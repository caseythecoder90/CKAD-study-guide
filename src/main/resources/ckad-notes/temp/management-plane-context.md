# Management Plane — Project Context

> **Purpose of this document:** Persistent context for working on the Management Plane component of the new tokenization (TAN→VAN mapping) system. Drop this into any future Claude conversation so we don't have to re-establish the architecture from scratch.
>
> **Last meaningful update:** 2026-05-15 — switched database from Oracle to multi-region CockroachDB (multi-region support is a required production capability). Earlier sections still reflect single-BH v1 scope, three-table data model, feature-flagged hashing trigger, and prototype-vs-production split.

---

## 1. The system at a glance

We're building a new tokenization service that maps **Physical Account Numbers (TANs)** to **Virtual Account Numbers (VANs)**. There's an existing CRUD-style Spring Boot system in production today that does this — it works but doesn't scale or isolate the way leadership wants. The new design is heavily inspired by the **Position Gate Keeper (PGK)** team's architecture, which uses the LMAX **Disruptor** framework for high-throughput microbatching.

The core idea: a VAN is just an alias for a TAN. Every mapping creation, lookup, or migration flows through this system.

## 2. High-level architecture (production target)

The data plane is sharded into **Bulkheads (BHs)** — each BH is an isolated failure domain with its own CockroachDB cluster, its own keys, and its own routing config. The production target supports up to 5 physical BHs (sized for 5 BHs × 1T mappings = 5T total capacity).

Routing happens in two tiers:

1. **Tier-1 (Client SDK):** `JumpHash(acctNum, 1024)` → produces a **logical BH** in `0..1023`, then a config map collapses that to a **physical BH** in `1..5`. No secret needed at this tier.
2. **Tier-2 (BH-local Envoy + Rust WASM filter):** `JumpHash(HMAC(PRK_BH, acctNum), 2^26)` → produces a **partition** in `0..67,108,863`. The Envoy WASM filter computes this, injects an `x-partition-id` header, and routes the gRPC call to the right Mapping Engine shard.

Each BH contains:
- **Control Plane** (Spring Boot + xDS server) — distributes shard endpoints and the partition routing key to the local Envoy.
- **Envoy + Rust WASM router** — does tier-2 partition computation and shard routing.
- **Mapping Engine shards** — each shard is a BOL2T MBE JVM running T Disruptor threads. Each shard owns a partition range assigned by the Control Plane.
- **CockroachDB cluster** — dedicated per-BH for savepoint persistence.

## 3. Partitioning math

- **2^26 = 67,108,864 partitions per BH**
- Each partition has exactly **one savepoint** (one row in the BOL2T savepoint table)
- The savepoint stores partition state as an encrypted protobuf blob (<1MB)
- Why 2^26? Savepoint must fit in a CRDB byte column under 1MB; per-mapping size is ~37B; max mappings per partition ≈ 28,339; target 5T mappings across 5 BHs = 1T per BH; minimum partitions needed = 1T / 28,339 ≈ 35.3M; next power-of-2 is 2^26 = 67M, giving 49% headroom.

## 4. The Management Plane — what *I* own

The Management Plane is **its own SEAL deployment** — independent lifecycle, scaling, and failure domain from the BHs.

### 4.1 V1 prototype scope vs production target

**Critical:** the prototype phase deliberately defers most security infrastructure. This is by design — the team wants to prove the data flow before layering in the security model.

| Aspect | V1 prototype | Production target |
|---|---|---|
| Partition routing key | Shared dev key, no BH isolation | Per-BH `PRK_BH` in HeraKMS |
| MP holds keys | Yes (shared dev key) | No (production design says MP holds no PRK) |
| BH count | Single BH | 5 BHs |
| Transport security | Plaintext or basic TLS | mTLS with CKMS certs |
| Savepoint encryption | None or trivial | AppProtect AES-GCM |
| Key storage | Config file | HeraKMS (HSM-backed) |
| Lookup response signing | None | AppProtect signature |
| Active regions | Single region | Multiple regions (REGIONAL BY ROW + GLOBAL localities active) |
| Survival goal | ZONE survival | REGION survival |

The proto contracts are designed so that production migration is a localized change — no caller is affected by the security tightening. See the proto library doc for specifics.

### 4.2 V1 responsibilities

The Management Plane at v1 prototype scope:

1. **Inventory holder** — stores un-allocated VANs in its own database
2. **Hashing/indexing** — assigns each VAN to a partition under the shared dev key
3. **Partition-keyed dispatcher** — serves runtime VAN requests for `(bh_id, partition_id)`
4. **VAN lookup** — given a VAN, returns its `(bh_id, partition_id)` so Client SDK knows where to route
5. **Audit log** — every dispatched VAN is recorded
6. **Replenishment monitor** — watches per-partition inventory levels, alerts when low

For the prototype with single BH, `bh_id` is a single fixed value but the column exists in all tables so the schema scales to 5 BHs without modification.

### 4.3 Three operational endpoints

The prototype exposes three primary endpoints (REST or gRPC — to be confirmed):

- **`POST /procurement/uploadRange`** — Ops triggers VAN procurement. Service inserts raw VANs into `mp_van_main` with `indexed_state='PENDING'`. Mirrors the legacy `VanUploadController` flow.
- **`POST /indexing/run`** — Triggers a hashing pass. Worker reads pending rows from `mp_van_main`, hashes to assign partition, inserts into `mp_van_inventory`, marks main rows as `INDEXED`. Returns a run_id; actual work happens async.
- **Watermark monitor** — scheduled task, runs every few minutes, counts per-partition inventory, alerts when total available drops below threshold.

A feature flag controls whether the upload endpoint auto-triggers the indexing job on completion (chained mode) or whether ops must call `/indexing/run` explicitly (triggered mode). **Default for v1 is triggered mode** — explicit operator control.

### 4.4 The runtime VAN provisioning flow

Once VANs are indexed into `mp_van_inventory`, shards consume them via `RequestVanBatch`:

- **Off the critical path.** Failure of MP does not block batch processing — shards continue draining their existing savepoint VAN pool.
- **Partial fulfillment is allowed.** If MP has 73 VANs available for the requested partition, it returns 73, not an error.
- **Errors are reserved for actual failures** (DB unavailable, malformed request, unknown BH).

The proto for this is `VanProvisioningService.RequestVanBatch`. See proto library doc.

### 4.5 Push and pull both supported

The prototype implements both:

- **Pull** — shard's background top-up thread calls `RequestVanBatch` when its partition pool is below watermark.
- **Push** — MP can proactively call shards via `SeedPartitionBatch` when it detects partition under-supply or during bulk seeding operations.

Pull is the steady-state pattern. Push is for bursts, bulk seeding, and operator-initiated dispatching. Both call paths exist in the protos; MP's logic decides which to use based on state.

### 4.6 VAN → (bh, partition) lookup

Client SDK calls MP to resolve a VAN to its location. This is a point lookup against `mp_van_dispatched` (the audit table) returning the BH and partition. Client SDK should cache aggressively — VAN locations are immutable once assigned.

### 4.7 What MP explicitly does NOT do

- MP does not hold mapping data — (TAN, VAN) pairs live in BH savepoints, not MP.
- MP does not serve TAN-based lookups — only VAN-based.
- MP does not directly write to shard savepoints — it dispatches batches that shards write themselves.
- MP does not compute tier-1 BH routing for *client requests* — that's Client SDK's job.

## 5. Security model (production target)

| Layer | Mechanism | Scope |
|---|---|---|
| Transport | mTLS (CKMS CA certs) | All gRPC |
| Response authenticity | AppProtect digital signature | Lookup responses |
| Partition routing key | HMAC-SHA256 (PRK_BHn) | BH-local only |
| Data-at-rest | AppProtect AES-GCM | Savepoint blobs |
| Lookup cache | LRU + CRDB consistent read | Per-shard |
| Key storage | HeraKMS (HSM-backed) | Per-BH keys |
| Certificate mgmt | CKMS (PKI CA) | TLS identity |

**V1 prototype defers all of these.** See section 4.1 for the deferred items list.

## 6. Data model

Five tables in MP's database. See the prototype design doc for full schema and rationale. Summary:

- **`mp_van_main`** — every VAN ever ingested. Append-mostly. Holds `(van, ingest_id, ingested_at, indexed_state, indexed_at)`. The `indexed_state` flag drives the hashing pipeline.
- **`mp_van_inventory`** — operational hot working set. Indexed VANs not yet dispatched. Partitioned by `bh_id` (LIST partition). Deleted-on-dispatch.
- **`mp_van_dispatched`** — audit log of issued VANs. Holds `(van, bh_id, partition_id, shard_id, dispatched_at)`. The hot lookup path queries this table by PK on `van`.
- **`mp_ingest_jobs`** — operational visibility for the hashing pipeline (not load-bearing for correctness).
- **`mp_van_upload_history`** — audit of upload ranges, mirroring legacy `GlasDigitalVanHistoryEntity`.

The three-table lifecycle: a VAN lives in main → moves to inventory after hashing → moves to dispatched after issuance.

## 7. Database choice: CockroachDB (multi-region)

**Decision:** Multi-region CockroachDB. Single region active for v1; production target is multi-region with REGION survival.

**Driving requirement:** multi-region support is required in production. Oracle's multi-region story (Active Data Guard, GoldenGate) is asynchronous with failover gaps that don't meet the requirement. CRDB provides synchronous multi-region consensus via Raft with native support for region-aware table localities.

**Note on prior recommendation:** Earlier drafts of this design recommended Oracle, before the multi-region requirement was known. Once that requirement surfaced, CRDB became the right choice despite the operational learning curve. The change is documented here for the record.

Trade-offs accepted:
- **Operational learning curve.** The team is still establishing CRDB practices for the data plane; MP adds a second CRDB cluster. V1 doubles as a learning vehicle for CRDB operations.
- **Correlated failure with data plane.** Both MP and BHs run on CRDB. Mitigated by separate clusters with separate operational lifecycles.
- **Per-row overhead higher than Oracle.** CRDB stores more per row due to MVCC and consensus metadata. Sized into the storage estimates in the prototype design doc.

**Multi-region structure built in from day one** even though only one region is active in v1:
- Adding `crdb_region` columns later requires backfilling billions of rows
- Changing table locality (REGIONAL ↔ GLOBAL) triggers data movement on a live system
- Survival goal changes require cluster reconfiguration
- Region-aware query patterns are easier to write from the start than retrofit

The cost of this up-front structure is small; the cost of retrofitting later is large. See the prototype design doc for full schema and the GLOBAL-vs-REGIONAL flagged decision on `mp_van_dispatched`.

Legacy patterns (`FOR UPDATE SKIP LOCKED` for atomic claim, batch commits, randomized insert order) port to CRDB cleanly; the temp-table + `INSERT...SELECT` pattern from legacy translates to JDBC batched multi-row INSERTs in CRDB.

## 8. gRPC tooling

We are **not** using `net.devh:grpc-spring-boot-starter` or Spring's official `spring-grpc-spring-boot-starter`. We are using **plain `io.grpc` (google grpc-java)** libraries directly, wired into Spring Boot manually.

### 8.1 The shared proto repository

A teammate maintains a Maven module (`caas-grpc-api`) that holds all `.proto` files for the platform. Layout:

```
<repo-root>/
├── proto-spec/
│   └── src/main/proto/
│       └── v1/
│           ├── common/      # shared messages and enums
│           ├── request/     # *Request messages
│           ├── response/    # *Response messages
│           └── service/     # service definitions
├── .gitignore
├── Jenkinsfile
├── jules.yml
└── pom.xml
```

The pom runs `protoc` + the gRPC Java plugin at build time, generates Java sources, packages them into a JAR, and publishes to the internal Maven repo.

### 8.2 Workflow

1. Add or edit `.proto` files in `proto-spec/src/main/proto/v1/{request,response,service,common}/`
2. PR review + CI build publishes a new artifact version
3. In the consuming Spring Boot service, add the artifact as a Maven dependency
4. Generated classes (`*Grpc.java`, `*Request.java`, `*Response.java`) are on the classpath; extend `*ImplBase` for servers, use `*BlockingStub`/`*Stub` for clients

### 8.3 Proto file conventions

- `syntax = "proto3";`
- Package: `com.<org>.<bu>.<product>.caas.proto.v1.{request,response,service,common}`
- `option java_package` mirrors the protobuf package
- Imports use repo-relative paths: `import "v1/common/account_instructions.proto";`
- Services in their own files, importing the request/response files they need

### 8.4 Consuming service dependencies

```xml
<properties>
    <java.version>21</java.version>
    <protobuf-java.version>4.31.1</protobuf-java.version>
    <grpc.version>1.76.0</grpc.version>
    <javax.annotation.version>1.3.2</javax.annotation.version>
</properties>

<dependencies>
    <dependency><groupId>io.grpc</groupId><artifactId>grpc-services</artifactId><version>${grpc.version}</version></dependency>
    <dependency><groupId>io.grpc</groupId><artifactId>grpc-protobuf</artifactId><version>${grpc.version}</version></dependency>
    <dependency><groupId>io.grpc</groupId><artifactId>grpc-stub</artifactId><version>${grpc.version}</version></dependency>
    <dependency><groupId>io.grpc</groupId><artifactId>grpc-netty-shaded</artifactId><version>${grpc.version}</version></dependency>
    <dependency><groupId>com.google.protobuf</groupId><artifactId>protobuf-java</artifactId><version>${protobuf-java.version}</version></dependency>
    <dependency><groupId>com.google.protobuf</groupId><artifactId>protobuf-java-util</artifactId><version>${protobuf-java.version}</version></dependency>
    <dependency><groupId>javax.annotation</groupId><artifactId>javax.annotation-api</artifactId><version>${javax.annotation.version}</version></dependency>
    <dependency><groupId>com.<org>.<bu>.<product></groupId><artifactId>caas-grpc-api</artifactId><version>X.Y.Z</version></dependency>
</dependencies>
```

`io.grpc.Server` and `ManagedChannel` beans get wired manually in a `@Configuration` class with `@PostConstruct` start / `@PreDestroy` shutdown.

## 9. Lessons from the legacy app

The legacy `VanUploadController` + `VanUploadService` + `VanQueueService` system is informative for v2 design. Key patterns to carry forward:

- **Range-based procurement** — ops uploads `(startNumber, endNumber)`, validated for non-overlap with existing inventory. Mirrors the new `POST /procurement/uploadRange`.
- **Temp table + bulk `INSERT...SELECT`** for fast bulk inserts into a large primary table.
- **`Collections.shuffle()` on insert** — randomises insertion order so a client seeing VAN X cannot trivially infer X+1 is next. Security-relevant; preserved in v2.
- **`FOR UPDATE SKIP LOCKED`** for atomic claim-and-allocate without thread contention. The legacy `VanService.allocateVanFromBackup` and `allocateVanFromQueue` use this. Same pattern in v2's allocator.
- **Two-mode replenishment** (normal sequential vs critical parallel) — the legacy `VanQueueService` has this; v2 doesn't strictly need it because per-partition inventory is naturally distributed, but the pattern is available if hot partitions emerge.

Patterns to **change** in v2:

- **History insert outside transaction** is a known legacy gap. V2 fixes this by including audit history in the same transaction as the upload commit.
- **DDL inside application code** (CREATE/DROP TABLE for temp tables) — operationally sensitive. V2 should consider using a fixed staging table with truncation, or session-level global temporary tables (Oracle feature) instead of true CREATE/DROP.
- **Single global queue** — replaced by per-partition inventory rows. The legacy queue's 10K-row hot set becomes 67M-partition distributed inventory.

## 10. Open questions still to resolve

1. **Mapping creation rate** — drives burn rate, replenishment frequency, and inventory sizing. Need from product/SME.
2. **Procurement lead time** — how long from ops requesting a new range to delivery? Drives watermark thresholds.
3. **Maximum range size per procurement** — affects ingest chunking strategy.
4. **Real-time CCB integration** — currently push-loaded, production target may be on-demand pull. Out of v1 scope.
5. **Topology service protos** — `GetBHMap`, `GetShardEndpoints`. Different audience (BH Control Planes), separate from VAN flow.
6. **Production security migration** — when and how the deferred items (mTLS, PRK isolation, AppProtect, HSM) get layered in.

## 11. Companion documents

- **`management-plane-proto-library.md`** — concrete proto contributions for `caas-grpc-api`
- **`management-plane-prototype.md`** — detailed prototype design (data model, math, worker design, API endpoints)

---

*Last updated: 2026-05-15*
