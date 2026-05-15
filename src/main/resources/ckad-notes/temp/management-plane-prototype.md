# Management Plane — V1 Prototype Design

> **Purpose:** Operational design for the Management Plane v1 prototype. Covers data model, capacity math, worker behavior, API surface, and explicit deferred-to-production items.
>
> **Companion to:** `management-plane-context.md` (high-level architecture and decisions) and `management-plane-proto-library.md` (proto contracts).
>
> **Last meaningful update:** 2026-05-15 — switched database to multi-region CockroachDB.

---

## 1. Scope statement

### 1.1 What v1 prototype does

- Procures large ranges of VANs (operator-driven, mirrors legacy)
- Hashes VANs to partition assignments under a shared dev key
- Holds per-partition inventory for fast partition-keyed allocation
- Serves runtime VAN top-up requests from shards
- Serves VAN → (bh, partition) lookups from Client SDK
- Records dispatched VANs in an audit log
- Monitors inventory levels and alerts on low supply
- Supports both pull (shard initiates) and push (MP initiates) dispatch
- **Deploys on multi-region CockroachDB from day one** (single region active in v1, but schema and topology designed for multi-region)

### 1.2 What v1 prototype does NOT do (deferred to production)

| Capability | V1 | Production |
|---|---|---|
| Multi-BH support | Single BH only | 5 BHs with tier-1 routing |
| Partition routing key isolation | Shared dev key, MP holds it | Per-BH `PRK_BH` in HeraKMS, MP holds none |
| Transport security | Plaintext or basic TLS | mTLS with CKMS certs |
| Savepoint encryption | None or trivial | AppProtect AES-GCM |
| Lookup response signing | None | AppProtect digital signature |
| Real-time CCB integration | Manual range upload by ops | On-demand pull from CCB |
| BH Control Plane orchestration | N/A (single BH) | CP-orchestrated bulk seeding under PRK_BH |
| Audit reconciliation job | Not built | Offline scan to back-fill missing audit entries |
| **Active regions** | Single region | Multiple regions, with REGIONAL BY ROW + GLOBAL table localities active |
| **Survival goal** | ZONE survival | REGION survival |

These are not gaps — they're explicit deferrals. The proto contracts and schema are designed so the production migration is a localized change without breaking callers or requiring data backfills.

### 1.3 Acceptance criteria for prototype completion

The prototype is "done" when:

1. Ops can upload a range of N VANs via `POST /procurement/uploadRange`
2. An indexing run can be triggered (manual or auto via feature flag) and completes
3. A shard can request VANs via gRPC `RequestVanBatch(bh_id, partition_id, count)` and receive partition-aligned VANs
4. Client SDK can call `LookupVan(van)` and receive the correct `(bh_id, partition_id)`
5. MP can push VANs to shards via `SeedPartitionBatch` (used for bulk seeding scenarios)
6. The watermark monitor correctly detects low inventory and emits an alert
7. All operations are restartable — crashes mid-job recover cleanly
8. CRDB cluster runs with `SURVIVE ZONE FAILURE` and is configured as multi-region capable (regions list defined, even if only one is active in v1)

---

## 2. Database choice: CockroachDB

**Decision:** Multi-region CockroachDB from day one, single active region in v1.

### 2.1 Why CRDB

The driving requirement is **multi-region support in production**. CRDB provides synchronous multi-region consensus via Raft, with built-in support for region-aware table localities (REGIONAL BY TABLE, REGIONAL BY ROW, GLOBAL). Oracle's multi-region story (Active Data Guard, GoldenGate) is asynchronous and has failover gaps that don't meet the requirement.

Other database choices were considered (see `management-plane-context.md` Section 8 for the Oracle vs CRDB trade-off, originally drafted before the multi-region requirement was known). Multi-region requirement makes CRDB the right call despite the operational learning curve.

### 2.2 Why multi-region structure from day one

We build with multi-region table localities and column structure from v1, even though only one region is active. Reasons:

- **Adding `crdb_region` columns later requires backfilling every row.** With billions of rows in `mp_van_main`, this is a multi-hour operation with production impact.
- **Changing table locality (REGIONAL ↔ GLOBAL) triggers data movement.** Doing this on a live, populated system is risky and slow.
- **Survival goal changes require cluster reconfiguration.** Easier to set correctly from the start than to migrate later.
- **Query patterns differ.** REGIONAL BY ROW tables benefit from region-aware queries; retrofitting these queries means touching every DAO.

The cost of building multi-region structure into v1 is small (some extra columns, longer DDL). The cost of retrofitting later is large. We pay the cost up front.

### 2.3 Trade-offs honestly stated

- **Operational learning curve.** The team is still establishing CRDB practices for the data plane; MP adds a second cluster. We mitigate this by treating v1 as a learning vehicle for CRDB operations.
- **Correlated failure with the data plane.** Both MP and BHs run on CRDB. A CRDB-version-specific bug could affect both. We mitigate by running separate clusters with separate operational lifecycles.
- **Per-row overhead higher than Oracle.** CRDB stores more per row due to MVCC and consensus metadata. Sized into the storage estimates in Section 4.

---

## 3. Data model

Five tables. The lifecycle of a single VAN moves it through three of them: `mp_van_main` (ingested) → `mp_van_inventory` (hashed, available) → `mp_van_dispatched` (issued). The other two tables (`mp_ingest_jobs`, `mp_van_upload_history`) provide operational visibility and audit.

### 3.1 Database-level configuration

Set up the database with multi-region capabilities even though only one region is active in v1.

```sql
-- Create the database with primary region
CREATE DATABASE mp_db PRIMARY REGION "us-east-1";

-- Add additional regions (production target; v1 may declare these but not activate)
ALTER DATABASE mp_db ADD REGION "us-west-2";
ALTER DATABASE mp_db ADD REGION "us-central-1";

-- Survival goal: production wants region survival; v1 starts with zone survival
ALTER DATABASE mp_db SURVIVE ZONE FAILURE;
-- Production migration: ALTER DATABASE mp_db SURVIVE REGION FAILURE;
```

**Notes:**

- `PRIMARY REGION` is where REGIONAL BY TABLE data defaults to.
- Additional regions can be declared in v1 even before they're physically deployed; CRDB just tracks them as known regions.
- `SURVIVE ZONE FAILURE` means the cluster survives a single node/zone going down. `SURVIVE REGION FAILURE` means it survives an entire region going down (requires at least 3 regions).

### 3.2 `mp_van_main` — the universal pool

Every VAN ever procured. Append-mostly: one INSERT at ingest, one UPDATE during indexing, frozen thereafter.

```sql
CREATE TABLE mp_van_main (
    van               STRING        NOT NULL,
    ingest_id         STRING        NOT NULL,
    ingested_at       TIMESTAMP     NOT NULL DEFAULT current_timestamp(),
    indexed_state     STRING        NOT NULL,    -- 'PENDING' or 'INDEXED'
    indexed_at        TIMESTAMP,
    PRIMARY KEY (van) USING HASH WITH (bucket_count = 16)
) LOCALITY REGIONAL BY TABLE IN PRIMARY REGION;

-- Supports the indexing worker's "find pending work" query
CREATE INDEX ix_main_pending
    ON mp_van_main (indexed_state, ingested_at)
    WHERE indexed_state = 'PENDING';
```

**Notes:**

- **`USING HASH WITH (bucket_count = 16)`** is critical. CRDB shards data across nodes by primary key range. Sequential VAN values (which they will be when procured from CCB) would create a write hotspot on whichever node owns the current end of the key range. The hash bucket distributes inserts across 16 ranges, eliminating the hotspot. Lookups by `van` still work — CRDB uses the bucket column internally.
- **`LOCALITY REGIONAL BY TABLE IN PRIMARY REGION`** — this table lives in one region. Procurement and indexing happen there. Other regions read it across the WAN if needed, but that's rare (the table is only read by the indexer, which can run in the primary region).
- **Partial index** (`WHERE indexed_state = 'PENDING'`) — CRDB supports this natively. The index only contains un-indexed rows, so it stays small as the worker burns through pending rows.

### 3.3 `mp_van_inventory` — the operational hot working set

Indexed VANs not yet dispatched. **VANs leave this table when dispatched** (delete on dispatch).

```sql
CREATE TABLE mp_van_inventory (
    van               STRING        NOT NULL,
    bh_id             STRING        NOT NULL,
    partition_id      INT4          NOT NULL,
    status            STRING        NOT NULL,    -- 'AVAILABLE' or 'RESERVED'
    reserved_by       STRING,
    reserved_session  STRING,
    reserved_at       TIMESTAMP,
    crdb_region       crdb_internal_region NOT NULL,  -- explicit region column for REGIONAL BY ROW
    PRIMARY KEY (van) USING HASH WITH (bucket_count = 16)
) LOCALITY REGIONAL BY ROW AS crdb_region;

-- Hot path: allocation queries
CREATE INDEX ix_inv_alloc
    ON mp_van_inventory (bh_id, partition_id, status);
```

**Notes:**

- **`LOCALITY REGIONAL BY ROW AS crdb_region`** is the multi-region setup. Each row carries a `crdb_region` column indicating which region "owns" it. Reads and writes from that region are local; cross-region access pays WAN latency. In v1 (single region), every row has the same `crdb_region` value, but the column and table locality are in place for production.
- **In production, `crdb_region` is set to the region serving the BH that owns this VAN.** When a shard in `us-east` requests VANs for BH1 (which lives in `us-east`), the inventory rows are in `us-east` — fast local read. When `us-west` BHs eventually exist, their VANs live in `us-west` rows.
- **Hash-bucketed PK** for the same hotspot-prevention reason as `mp_van_main`.

### 3.4 `mp_van_dispatched` — the audit log and lookup index

Every dispatched VAN. **This is the table that serves VAN lookups from Client SDK from any region**, which is why the locality choice matters.

```sql
CREATE TABLE mp_van_dispatched (
    van               STRING        NOT NULL,
    bh_id             STRING        NOT NULL,
    partition_id      INT4          NOT NULL,
    shard_id          STRING,
    dispatched_at     TIMESTAMP     NOT NULL DEFAULT current_timestamp(),
    PRIMARY KEY (van) USING HASH WITH (bucket_count = 16)
) LOCALITY GLOBAL;  -- [DECISION FLAG — see Section 3.4.1]
```

#### 3.4.1 GLOBAL vs REGIONAL — flagged design decision

**Status: open. Decide before production go-live; v1 can ship either way and migrate.**

The `LOCALITY GLOBAL` choice means every region has a strongly-consistent local read replica. Writes are slower (have to commit to all regions), but reads are fast from anywhere.

**Why GLOBAL might be right:**
- Client SDK calls `LookupVan` on cache miss. If SDKs run in multiple regions, GLOBAL gives them local read latency everywhere — likely 1-5ms instead of 50-100ms cross-region.
- Writes are infrequent (only happen on dispatch). The cost of slower writes is paid rarely.
- Read volume is potentially very high; latency multiplies through Client SDK fleet.

**Why GLOBAL might be wrong:**
- If Client SDK only ever runs in one region, GLOBAL is wasted overhead.
- Write latency increases proportionally with the number of regions. With 3+ regions, every dispatch becomes notably slower.
- If aggressive Client SDK caching makes lookups rare, the latency advantage doesn't matter.

**Inputs needed to decide:**
- Where will Client SDK fleet run? Single region or distributed?
- What's the expected lookup volume (cache miss rate × mapping creation rate)?
- What's the tolerance for slower dispatch writes?

**For v1:** start with `REGIONAL BY TABLE IN PRIMARY REGION`. This makes writes fast and is sufficient for single-region. Production-time decision: migrate to GLOBAL if SDK fleet distribution and lookup volume justify it. The migration itself requires careful planning but is well-supported by CRDB.

**Concrete v1 starting point:**

```sql
CREATE TABLE mp_van_dispatched (
    van               STRING        NOT NULL,
    bh_id             STRING        NOT NULL,
    partition_id      INT4          NOT NULL,
    shard_id          STRING,
    dispatched_at     TIMESTAMP     NOT NULL DEFAULT current_timestamp(),
    PRIMARY KEY (van) USING HASH WITH (bucket_count = 16)
) LOCALITY REGIONAL BY TABLE IN PRIMARY REGION;
```

### 3.5 `mp_ingest_jobs` — operational visibility

Tracks indexing runs. Not load-bearing for correctness — restart-ability comes from `mp_van_main.indexed_state`, not from this table.

```sql
CREATE TABLE mp_ingest_jobs (
    job_id            STRING        NOT NULL,
    job_type          STRING        NOT NULL,    -- 'INDEXING_RUN'
    status            STRING        NOT NULL,    -- 'PENDING', 'RUNNING', 'COMPLETED', 'FAILED'
    processed_count   INT8          DEFAULT 0,
    started_at        TIMESTAMP,
    updated_at        TIMESTAMP,
    completed_at      TIMESTAMP,
    error_detail      STRING,
    PRIMARY KEY (job_id)
) LOCALITY REGIONAL BY TABLE IN PRIMARY REGION;
```

### 3.6 `mp_van_upload_history` — procurement audit

Mirrors the legacy `GLAS_DIGITAL_VAN_HISTORY` table. **In the same transaction as the upload commit** (fixing the legacy gap where the history insert ran outside the upload transaction).

```sql
CREATE TABLE mp_van_upload_history (
    upload_id         STRING        NOT NULL,
    range_start       STRING        NOT NULL,
    range_end         STRING        NOT NULL,
    van_count         INT8          NOT NULL,
    uploaded_at       TIMESTAMP     NOT NULL DEFAULT current_timestamp(),
    uploaded_by       STRING,
    PRIMARY KEY (upload_id)
) LOCALITY REGIONAL BY TABLE IN PRIMARY REGION;

CREATE INDEX ix_history_uploaded_at ON mp_van_upload_history (uploaded_at);
```

### 3.7 ERD summary

```
mp_van_main (universal pool, REGIONAL BY TABLE)
   |
   | (indexing worker consumes PENDING rows)
   v
mp_van_inventory (hot working set, REGIONAL BY ROW)
   |
   | (dispatch deletes row, inserts into mp_van_dispatched)
   v
mp_van_dispatched (audit + lookup index, REGIONAL BY TABLE in v1, GLOBAL flagged)


mp_ingest_jobs            (operational visibility, not critical)
mp_van_upload_history     (procurement audit, mirrors legacy)
```

A VAN moves through three states; one row per VAN exists in main (forever), at most one in inventory (only while waiting to be dispatched), and one in dispatched (after issuance).

---

## 4. Capacity math (single-BH, single-region v1)

### 4.1 The two-buffer model

The system has **two distinct VAN buffers** in two different databases at two different layers, each serving a different purpose. They must be sized separately.

```
   ┌────────────────────────┐  top-up  ┌────────────────────────┐
   │  Buffer B: MP          │ ──RPC──→ │  Buffer A: Shard       │
   │  mp_van_inventory      │          │  savepoint pool        │
   │  (CRDB, in MP)         │          │  (CRDB, in shard BH)   │
   │                        │          │                        │
   │  Reserve for future    │          │  Active working pool   │
   │  top-up requests       │          │  for create operations │
   └────────────────────────┘          └────────────────────────┘
              ▲                                    ▲
              │ ops procures + indexing            │ shard top-up thread
              │ (rare, ~weeks/months apart)        │ (continuous, per-partition)
```

- **Buffer A (shard savepoint pool):** sized by *per-partition watermark + burn rate*. Drains continuously, refills via top-up RPC.
- **Buffer B (MP inventory):** sized by *replenishment cycle length*. Drains slowly, refills via ops procurement.

### 4.2 Variables and assumptions

| Variable | Symbol | V1 working value | Source / notes |
|---|---|---|---|
| Partitions per BH | P | 67,108,864 (2^26) | Architecture-fixed |
| Per-partition savepoint target (Buffer A) | T | **10 VANs** | **[ASSUMPTION — confirm with shard team]** |
| Per-partition savepoint watermark | W | **3 VANs** (30% of T) | **[ASSUMPTION]** |
| Mapping creation rate (system-wide) | R | **TBD** mappings/sec | **Need from product/SME** |
| Replenishment cycle (Buffer B reserve duration) | D | **TBD** days | **Need from product/ops** |
| Procurement lead time | L | **TBD** days | **Need from product/ops** |

### 4.3 Buffer A: savepoint pool sizing

Buffer A lives **in each shard's savepoint blob in CRDB**, not in MP. MP feeds Buffer A via top-up RPCs. Total VANs across all Buffer A pools at steady state:

```
Buffer_A_total = P × T
               = 67,108,864 × 10
               = 671,088,640 VANs
               ≈ 671 million VANs
```

These VANs flow *through* MP into shard savepoints during initial seeding. After seeding, they live in shards — not in MP.

### 4.4 Buffer B: MP reserve sizing

Buffer B holds the reserve in `mp_van_inventory` that MP draws from when serving top-up RPCs. It must last from one procurement cycle to the next.

```
Buffer_B_target = R × 86,400 × D
```

For various scenarios:

| Scenario | R (mappings/sec) | D (days reserve) | Buffer B target |
|---|---|---|---|
| Low burn, fast procurement | 100 | 30 | ~260M VANs |
| Low burn, slow procurement | 100 | 90 | ~778M VANs |
| Medium burn, normal procurement | 1,000 | 14 | ~1.2B VANs |
| Medium burn, slow procurement | 1,000 | 30 | ~2.6B VANs |
| High burn, fast procurement | 10,000 | 7 | ~6B VANs |

**Rule of thumb:** `D ≥ 2 × L`. Procure well before reserves run out.

### 4.5 Variance check

JumpConsistentHash distributes uniformly *on average* across the 67M partitions, with per-partition variance. Variance only matters when Buffer B is small enough that some partitions have zero.

Floor: when Buffer B has at least `P × 3 = 200M` VANs, every partition has ≥3 VANs on average. Even with variance, top-ups mostly succeed.

**Watermark/alert threshold:**

```
Buffer_B_alert = max(Buffer_B_target × 0.25, P × 3)
              = max(Buffer_B_target × 0.25, ~200M)
```

### 4.6 Total initial procurement

VANs ops must upload pre-go-live:

```
Initial_procurement = Buffer_A_total + Buffer_B_target
                    = 671M + Buffer_B_target
```

For the scenarios above:

| Scenario | Buffer A | Buffer B | **Total initial procurement** |
|---|---|---|---|
| Low burn, fast procurement | 671M | 260M | **~931M VANs** |
| Low burn, slow procurement | 671M | 778M | **~1.45B VANs** |
| Medium burn, normal procurement | 671M | 1.2B | **~1.87B VANs** |
| Medium burn, slow procurement | 671M | 2.6B | **~3.27B VANs** |
| High burn, fast procurement | 671M | 6B | **~6.67B VANs** |

### 4.7 Database storage estimate

Per-row sizes (CRDB, with PK indexes, MVCC overhead, and consensus metadata — higher per-row footprint than Oracle):

| Table | Row size estimate | Peak rows | Storage |
|---|---|---|---|
| `mp_van_main` | ~80 bytes | Initial_procurement | varies |
| `mp_van_inventory` | ~120 bytes | Buffer B target | varies |
| `mp_van_dispatched` | ~90 bytes | grows over time | varies |
| `mp_ingest_jobs` | small | hundreds | negligible |
| `mp_van_upload_history` | ~100 bytes | thousands | negligible |

For the **medium burn, normal procurement** scenario (~1.87B initial procurement, ~1.2B Buffer B steady-state):
- `mp_van_main`: 1.87B × 80 bytes ≈ **150 GB**
- `mp_van_inventory`: 1.2B × 120 bytes ≈ **144 GB**
- `mp_van_dispatched`: at R=1,000 mappings/sec, 31.5B rows/year × 90 bytes ≈ **2.8 TB/year**

**Plus CRDB replication overhead.** With 3 replicas (default), multiply storage by 3. For region-survival production (typically 5 replicas across 3 regions), multiply by 5.

**Year-1 CRDB cluster storage estimate:** ~9-15 TB total raw storage for the medium burn scenario. Smaller scenarios are proportionally smaller. **Get R confirmed before sizing the cluster** — it's the dominant variable.

`mp_van_dispatched` growth is unbounded. Plan for periodic archival to cold storage (S3 WORM is mentioned in the architecture doc), keeping only the most recent N months hot.

### 4.8 Hashing throughput estimate

CRDB write throughput is lower per-row than Oracle due to consensus overhead, but parallelizes well across nodes. Realistic single-worker throughput: **10-20K rows/sec**. With 8 parallel workers and proper batching: **50-100K rows/sec sustained**.

For the medium scenario (~1.87B VANs) at 80K rows/sec:
```
Indexing time = 1.87B / 80K = 23,400 seconds ≈ 6.5 hours
```

**Plan for overnight indexing for the initial load.** Subsequent refills (replenishing Buffer B between procurement cycles) finish in 1-2 hours depending on size.

### 4.9 Refill amount

When Buffer B drops to alert threshold:
```
Refill_amount ≈ 0.75 × Buffer_B_target
```

For the medium scenario, ~900M VANs per refill cycle. Procured via `POST /procurement/uploadRange`, potentially in multiple calls if product can't deliver in one range.

### 4.10 Summary: the numbers that matter most

If you can only get three numbers from product/ops, get these:

1. **R** (mapping creation rate) — drives everything operational
2. **L** (procurement lead time) — drives D, which drives Buffer B size
3. **Maximum procurement range size** — drives chunking strategy

T should be confirmed with the shard team; 10 is a reasonable starting value.

---

## 5. API surface

### 5.1 Procurement endpoint

```
POST /procurement/uploadRange
Content-Type: application/json

{
  "startNumber": "00000000001000000000",
  "endNumber":   "00000000001999999999",
  "uploadedBy":  "ops-team-id"
}

Response 200:
{
  "uploadId": "upload-uuid",
  "vanCount": 1000000000,
  "ingestedAt": "2026-05-15T10:30:00Z"
}
```

**Behavior:**
- Validates range: non-blank, parseable as non-negative long, start ≤ end
- Validates no overlap with existing inventory (query `mp_van_main` for any VAN in range)
- Batch-inserts into `mp_van_main` with `indexed_state='PENDING'`, using JDBC batched multi-row INSERTs
- Inserts audit row into `mp_van_upload_history` **in the same CRDB transaction** (fixing legacy gap)
- Randomises insertion order via `Collections.shuffle()` (security pattern preserved from legacy)
- Returns immediately on commit; does not wait for indexing

If feature flag `mp.indexing.chained=true`, auto-triggers `/indexing/run` after upload commits. Default `false` for v1.

### 5.2 Indexing trigger endpoint

```
POST /indexing/run
Response 200:
{
  "jobId": "job-uuid",
  "status": "RUNNING",
  "startedAt": "2026-05-15T10:35:00Z"
}

GET /indexing/run/{jobId}
Response 200:
{
  "jobId": "job-uuid",
  "status": "RUNNING|COMPLETED|FAILED",
  "processedCount": 500000,
  "startedAt": "...",
  "completedAt": "..."
}
```

### 5.3 gRPC services

See `management-plane-proto-library.md` for full schemas:

- `VanProvisioningService.RequestVanBatch` — pull path
- `VanProvisioningService.LookupVan` — Client SDK VAN resolution
- `ShardSeedingService.SeedPartitionBatch` — push path

---

## 6. Worker design

### 6.1 The hashing worker

Triggered by `POST /indexing/run` (or auto-triggered via feature flag).

```
worker_loop():
  while true:
    txn_with_retry():
      batch = SELECT van FROM mp_van_main
              WHERE indexed_state = 'PENDING'
              ORDER BY ingested_at
              LIMIT 5000
              FOR UPDATE SKIP LOCKED
      
      if batch is empty:
        mark job COMPLETED
        return DONE
      
      inventory_rows = []
      for each van in batch:
        partition_id = JumpHash(HMAC(SHARED_DEV_KEY, van), 2^26)
        bh_id = "BH1"  // v1: single BH
        crdb_region = "us-east-1"  // v1: single region
        inventory_rows.append((van, bh_id, partition_id, 'AVAILABLE', crdb_region))
      
      INSERT INTO mp_van_inventory (...) VALUES (... multi-row ...)
      ON CONFLICT (van) DO NOTHING  -- idempotency on retry
      
      UPDATE mp_van_main 
      SET indexed_state = 'INDEXED', indexed_at = current_timestamp()
      WHERE van = ANY(batch)
      
      UPDATE mp_ingest_jobs 
      SET processed_count = processed_count + batch.size, updated_at = current_timestamp()
      WHERE job_id = current_job_id
```

### 6.2 CRDB transaction retry — important

CRDB transactions can return a **retry error** under contention (SQLSTATE `40001`, "transaction restart"). Unlike Oracle, CRDB does not silently serialize conflicting transactions; it asks the application to retry.

```java
// In the DAO layer
public void runIndexingBatch() {
    int attempts = 0;
    while (attempts < MAX_RETRIES) {
        try {
            jdbcTemplate.execute(connection -> {
                connection.setAutoCommit(false);
                // ... do the work ...
                connection.commit();
                return null;
            });
            return;  // success
        } catch (TransactionRestartException e) {  // catch SQLSTATE 40001
            attempts++;
            sleep(exponentialBackoff(attempts));
        }
    }
    throw new IndexingFailedException("Max retries exhausted");
}
```

The Spring `JdbcTemplate` and Hibernate retry support handle this if configured. Make sure the team is aware — this is a real difference from Oracle.

### 6.3 Threading

Use Java 21 virtual threads. The bottleneck is database I/O. Initial pool size: 8 virtual threads. Tune based on CRDB cluster connection capacity (each worker needs one connection while running).

### 6.4 Restart-ability

- State lives in `mp_van_main.indexed_state`. Crashed batches release locks on rollback; next worker run claims the same rows.
- Idempotency on retry: `INSERT ... ON CONFLICT DO NOTHING` handles re-inserts safely.
- Multiple workers run in parallel without blocking each other thanks to `FOR UPDATE SKIP LOCKED`.

### 6.5 The watermark monitor

Scheduled task, runs every 5-10 minutes.

```
scheduled monitor():
  total_available = SELECT COUNT(*) FROM mp_van_inventory
                    WHERE bh_id = ? AND status = 'AVAILABLE'
  
  if total_available < Buffer_B_alert:
    emit alert: "Inventory low for BH X: {total_available} VANs"
  
  // Optional: per-partition depth check (expensive — scans many partitions)
  // Run less frequently (every 30-60 min)
  per_partition_low = SELECT partition_id, COUNT(*) as depth
                       FROM mp_van_inventory
                       WHERE bh_id = ? AND status = 'AVAILABLE'
                       GROUP BY partition_id
                       HAVING COUNT(*) < W
  
  if per_partition_low not empty:
    emit alert: "Partitions below watermark: {list}"
```

The monitor alerts but **does not procure VANs itself** — procurement is an ops decision.

### 6.6 The dispatch flow

**Pull (handler for `RequestVanBatch` RPC):**

```
txn_with_retry():
  if cached_response_for(request_id):
    return cached_response
  
  rows = SELECT van FROM mp_van_inventory
         WHERE bh_id = ? AND partition_id = ? AND status = 'AVAILABLE'
         LIMIT ?
         FOR UPDATE SKIP LOCKED
  
  if rows is empty:
    return VanBatchResponse(status=INVENTORY_EMPTY)
  
  DELETE FROM mp_van_inventory WHERE van = ANY(rows.vans)
  INSERT INTO mp_van_dispatched (van, bh_id, partition_id, shard_id, dispatched_at)
     VALUES (... rows ...)
  
cache_response(request_id, response)
return VanBatchResponse(vans=rows, issued_count=rows.size, status=...)
```

**Push (MP initiates `SeedPartitionBatch` on a shard):**
- Same DELETE-from-inventory and INSERT-into-dispatched pattern
- Then call target shard's `SeedPartitionBatch` via gRPC
- If shard ack fails: known v1 gap, VANs are in `mp_van_dispatched` but may not be in savepoint. Production adds reconciliation.

### 6.7 Failure modes summary

| Failure | Recovery |
|---|---|
| Procurement endpoint crashes mid-insert | Transaction rollback; ops retries same range |
| Indexing worker crashes mid-batch | `FOR UPDATE SKIP LOCKED` releases on rollback; next worker picks up |
| Multiple indexing workers contend | `FOR UPDATE SKIP LOCKED` lets them claim disjoint batches |
| **CRDB transaction restart (40001)** | DAO retry loop with exponential backoff |
| Dispatch crashes mid-transaction | Transaction rollback; VANs return to `AVAILABLE` |
| Push to shard succeeds but ack lost | V1: known drift, audit shows dispatched. Production: reconciliation |
| CRDB region unavailable | With ZONE survival (v1): cluster survives single node loss. With REGION survival (production): cluster survives region loss. gRPC returns UNAVAILABLE; caller retries |

---

## 7. Metrics for production-readiness

### 7.1 Inventory state metrics (most important)

Scraped every 30-60 seconds:

| Metric | Type | Purpose |
|---|---|---|
| `mp.inventory.buffer_b.total` | gauge | Total VANs in `mp_van_inventory` with `status='AVAILABLE'` |
| `mp.inventory.buffer_b.by_partition_min` | gauge | Smallest per-partition count (variance check) |
| `mp.inventory.buffer_b.by_partition_p50` | gauge | Median per-partition count |
| `mp.inventory.buffer_b.by_partition_p99` | gauge | 99th percentile per-partition count |
| `mp.inventory.pending_indexing` | gauge | Rows with `indexed_state='PENDING'` |
| `mp.inventory.dispatched_total` | counter | Cumulative VANs dispatched |

### 7.2 Request rate metrics

| Metric | Type | Purpose |
|---|---|---|
| `mp.rpc.request_van_batch.rate` | counter | Pull requests/sec from shards |
| `mp.rpc.request_van_batch.duration` | histogram | Latency distribution |
| `mp.rpc.request_van_batch.vans_issued` | counter | Total VANs handed out |
| `mp.rpc.lookup_van.rate` | counter | Lookup requests/sec from Client SDK |
| `mp.rpc.lookup_van.duration` | histogram | Hot path latency |
| `mp.rpc.lookup_van.cache_misses` | counter | Lookups hitting MP (vs SDK cache) |
| `mp.rpc.seed_partition_batch.rate` | counter | Push dispatches initiated by MP |

`mp.rpc.request_van_batch.vans_issued` validates the R assumption from product.

### 7.3 Worker and procurement metrics

| Metric | Type | Purpose |
|---|---|---|
| `mp.procurement.upload.rate` | counter | Upload calls (should be rare) |
| `mp.procurement.upload.vans` | counter | VANs uploaded |
| `mp.indexing.runs.rate` | counter | Indexing jobs started |
| `mp.indexing.throughput` | gauge | Rows/sec being indexed |
| `mp.indexing.duration` | histogram | End-to-end time per indexing run |

### 7.4 Failure metrics

| Metric | Type | Purpose |
|---|---|---|
| `mp.rpc.errors.by_status` | counter | gRPC errors by status code |
| `mp.inventory.empty_responses` | counter | INVENTORY_EMPTY responses (scarcity, not error) |
| `mp.indexing.failures` | counter | Indexing batches that failed |
| `mp.db.transaction.rollbacks` | counter | DB transaction rollbacks |
| **`mp.db.transaction.restarts`** | **counter** | **CRDB-specific: transactions that hit 40001 and retried** |

### 7.5 CRDB cluster metrics

CRDB exposes these natively via Prometheus; MP should integrate them into the same dashboards:

| Metric | Purpose |
|---|---|
| `crdb.ranges.unavailable` | Range-level health; alert if > 0 |
| `crdb.ranges.under_replicated` | Replication health |
| `crdb.replication.lag` (per region) | Cross-region replication health (matters more in production) |
| `crdb.sql.txn.restarts.rate` | Cluster-wide transaction restart rate |
| `crdb.sql.conn.count` | Active connection count |
| `crdb.sql.bytes` | Read/write throughput |

### 7.6 The four SLIs to anchor alerts on

If alerting on only four things:

1. **Buffer B available drops below alert threshold** (operational: need to procure)
2. **Buffer B per-partition min drops below 1** (correctness: starved partitions)
3. **LookupVan p99 latency exceeds threshold** (user-facing: Client SDK is slow)
4. **CRDB `ranges.unavailable` > 0** (capacity/health: cluster is unhealthy)

### 7.7 What to gather before go-live

Load tests replace assumptions with measurements:

- **Synthetic load test** hitting `RequestVanBatch` at expected R for sustained periods. Measure: latency, CRDB connection usage, Buffer B drain rate, transaction restart rate.
- **Synthetic load test** hitting `LookupVan` at expected lookup volume. Measure: latency at p50/p99 from different regions (relevant for the GLOBAL vs REGIONAL decision).
- **Full procurement → indexing → dispatch cycle** with a realistic-size range. Measure: wall-clock time, throughput stability, transaction restart frequency.
- **Failure injection:** kill an indexing worker mid-batch, drop a CRDB node, partition the cluster network. Confirm recovery.

---

## 8. Migration path to production

When v1 is validated and ready to layer in production capabilities:

1. **Activate additional regions.** Add `us-west-2` and `us-central-1` as fully active regions. `ALTER DATABASE mp_db SURVIVE REGION FAILURE` once at least 3 regions are populated.
2. **Multi-BH support.** Add BH2-BH5 routing logic; populate `crdb_region` column on inventory rows based on which region serves each BH. The schema is already in place for this — only data routing changes.
3. **Decide on `mp_van_dispatched` locality.** Promote to `GLOBAL` if cross-region lookup latency justifies it. Run load tests with Client SDK in multiple regions to inform the decision.
4. **PRK isolation.** Per-BH HMAC keys move into HeraKMS. MP loses the shared dev key. Hashing logic moves to BH Control Planes per original architecture (`StreamUnassignedNumbers`, `ConfirmIssued` flows).
5. **mTLS.** Add CKMS-issued certs; add interceptors for caller-identity verification.
6. **AppProtect encryption.** Savepoint blobs get AES-GCM encryption; lookup responses get signed.
7. **Audit reconciliation.** Build the offline scan-shard-savepoints job to catch drift.
8. **Real-time CCB integration.** If/when CCB exposes an API, MP gets a CCB client; procurement becomes pull-from-CCB rather than push-from-ops.

**Critically:** none of these changes break the proto contracts or the schema shape. Callers don't need to change. The migration is internal to MP.

---

*Last updated: 2026-05-15*
