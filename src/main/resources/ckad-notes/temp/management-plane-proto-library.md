# Management Plane — Proto Library Reference

> **Purpose:** Concrete proto source for files to contribute to the shared `caas-grpc-api` repository in support of the Management Plane. Organized in commit-dependency order so this can be staged across multiple PRs if needed.
>
> **Companion to:** `management-plane-context.md` (architecture/decision rationale) and `management-plane-prototype.md` (operational design).
>
> **Last meaningful update:** 2026-05-15

---

## 0. How to use this document

This doc is structured so you can work through it top-to-bottom and have a complete proto contribution ready to PR. Each section gives you:

- The **file path** within `proto-spec/src/main/proto/`
- The **full proto source** to commit
- **Notes on design decisions** so you can defend them in code review

The order matters: messages must exist before services that reference them. If splitting across PRs:

- **PR 1:** Section 1 (common enums) — small, low-risk, unlocks everything else
- **PR 2:** Sections 2 + 3 (request and response messages)
- **PR 3:** Section 4 (service definitions) — depends on 2 and 3

Substitute your actual package prefix wherever you see `com.<org>.<bu>.<product>`.

## 0.1 V1 prototype vs production target

These protos are designed once and used in both v1 prototype and production. The differences between v1 and production are **runtime behavior**, not contract changes:

- **Auth:** v1 uses no mTLS; production uses CKMS-issued client certs with caller-identity verification in an interceptor.
- **Partition routing key:** v1 uses a shared dev key everywhere; production uses per-BH `PRK_BH` stored in HeraKMS. The proto fields (`bh_id`, `partition_id`) are identical in both — only the server-side implementation differs.
- **Single BH:** v1 has one BH; `bh_id` is a single value. Production has 5. The proto admits both.

Bottom line: write the protos once, ship them, and the v1 → production transition doesn't break callers.

---

## 1. Common enums

### 1.1 `v1/common/van_inventory_status.proto`

Used by the runtime `VanProvisioningService` response.

```protobuf
syntax = "proto3";

package com.<org>.<bu>.<product>.caas.proto.v1.common;
option java_package = "com.<org>.<bu>.<product>.caas.proto.v1.common";

// Status returned by the Management Plane when fulfilling a VAN batch request.
// Distinguishes scarcity (a normal operating condition) from failure
// (which is signaled via gRPC status codes, not this enum).
enum VanInventoryStatus {
  VAN_INVENTORY_STATUS_UNSPECIFIED = 0;
  FULLY_FULFILLED = 1;       // issued_count == requested_count
  PARTIALLY_FULFILLED = 2;   // 0 < issued_count < requested_count
  INVENTORY_EMPTY = 3;       // no inventory available; not an error
}
```

**Notes:** Zero value is `_UNSPECIFIED` per proto3 convention. Once published, **values must never be renumbered**. `INVENTORY_EMPTY` is deliberately a status, not an error — scarcity is a normal operating condition.

### 1.2 `v1/common/seeding_status.proto`

Used by push/seeding flows.

```protobuf
syntax = "proto3";

package com.<org>.<bu>.<product>.caas.proto.v1.common;
option java_package = "com.<org>.<bu>.<product>.caas.proto.v1.common";

enum SeedingStatus {
  SEEDING_STATUS_UNSPECIFIED = 0;
  COMPLETED = 1;
  COMPLETED_WITH_SKIPS = 2;
  FAILED_RETRYABLE = 3;
  FAILED_TERMINAL = 4;
}
```

---

## 2. Request messages

### 2.1 `v1/request/van_batch_request.proto`

Pull path: shard requesting a top-up.

```protobuf
syntax = "proto3";

package com.<org>.<bu>.<product>.caas.proto.v1.request;
option java_package = "com.<org>.<bu>.<product>.caas.proto.v1.request";

// Request from a Mapping Engine shard to the Management Plane
// for a batch of partition-aligned VANs.
message VanBatchRequest {

  // The Bulkhead ID the requesting shard belongs to.
  // V1: single value (e.g. "BH1"). Production: one of BH1..BH5.
  string bh_id = 1;

  // The shard ID making the request. Used for audit logging;
  // dispatch is keyed on (bh_id, partition_id).
  string shard_id = 2;

  // The partition the requesting shard wants VANs for.
  // Valid range: [0, 2^26).
  uint32 partition_id = 3;

  // Number of VANs requested. MP may return fewer.
  uint32 requested_count = 4;

  // Idempotency key. Caller-generated, typically a UUID.
  // MP de-duplicates within a TTL window (recommended: 5 minutes).
  string request_id = 5;
}
```

### 2.2 `v1/request/van_lookup_request.proto`

VAN → (bh, partition) lookup, called by Client SDK on cache miss.

```protobuf
syntax = "proto3";

package com.<org>.<bu>.<product>.caas.proto.v1.request;
option java_package = "com.<org>.<bu>.<product>.caas.proto.v1.request";

// Request from Client SDK (or other authorized caller) to resolve
// a VAN to its (bh_id, partition_id) location. Used to route
// VAN-keyed operations (lookups, updates) to the correct shard.
message VanLookupRequest {

  // The VAN to resolve.
  string van = 1;

  // Caller identifier for audit/rate-limiting. Optional in v1.
  string caller_id = 2;
}
```

**Notes:** Deliberately minimal. The lookup is one of the hottest read paths in production; the proto stays small so serialization overhead is minimized. No idempotency key needed — lookups are inherently idempotent.

### 2.3 `v1/request/seed_partition_batch_request.proto`

Push path: MP sending VANs to a shard.

```protobuf
syntax = "proto3";

package com.<org>.<bu>.<product>.caas.proto.v1.request;
option java_package = "com.<org>.<bu>.<product>.caas.proto.v1.request";

// Sent from MP to a shard, instructing the shard to write a batch
// of VANs into its savepoint pools. The shard must verify the
// partition IDs fall within its assigned range and reject any
// that don't.
message SeedPartitionBatchRequest {

  string bh_id = 1;

  // Identifier of the calling Management Plane instance.
  string caller_id = 2;

  // Idempotency key for this specific RPC call.
  string batch_id = 3;

  // The VANs to seed, grouped by partition.
  repeated PartitionSeed partition_seeds = 4;
}

message PartitionSeed {
  uint32 partition_id = 1;
  repeated string vans = 2;
}
```

**Notes:** In v1, MP holds the partition routing key (shared dev key) and computes assignments itself. In production, the same proto shape applies but a different component (likely the BH Control Plane) does the hashing under the per-BH PRK. The shard's behavior is the same in both cases: write to the named partition.

---

## 3. Response messages

### 3.1 `v1/response/van_batch_response.proto`

```protobuf
syntax = "proto3";

package com.<org>.<bu>.<product>.caas.proto.v1.response;
option java_package = "com.<org>.<bu>.<product>.caas.proto.v1.response";

import "google/protobuf/timestamp.proto";
import "v1/common/van_inventory_status.proto";

message VanBatchResponse {
  repeated string vans = 1;
  uint32 issued_count = 2;
  v1.common.VanInventoryStatus status = 3;
  google.protobuf.Timestamp issued_at = 4;
  string request_id = 5;
  string detail = 6;
}
```

**Notes:** `issued_count` is redundant with `vans.size()` but explicit — saves the caller `.size()` calls in hot paths. `detail` is for human-readable info only, not programmatic use.

### 3.2 `v1/response/van_lookup_response.proto`

```protobuf
syntax = "proto3";

package com.<org>.<bu>.<product>.caas.proto.v1.response;
option java_package = "com.<org>.<bu>.<product>.caas.proto.v1.response";

message VanLookupResponse {

  // The VAN that was looked up (echoed for correlation).
  string van = 1;

  // The BH where the VAN's mapping lives.
  string bh_id = 2;

  // The partition where the VAN's mapping lives.
  uint32 partition_id = 3;

  // Whether the VAN was found. False means the VAN is not in
  // MP's dispatched audit log — either unknown, or never issued.
  bool found = 4;
}
```

**Notes:** No timestamp, no auth claims, nothing extra. The hot path stays lean. If `found = false`, callers should treat it as a hard error — an unknown VAN is not a recoverable scarcity condition.

### 3.3 `v1/response/seed_partition_batch_response.proto`

```protobuf
syntax = "proto3";

package com.<org>.<bu>.<product>.caas.proto.v1.response;
option java_package = "com.<org>.<bu>.<product>.caas.proto.v1.response";

import "v1/common/seeding_status.proto";

message SeedPartitionBatchResponse {
  v1.common.SeedingStatus status = 1;
  string batch_id = 2;
  repeated PartitionSeedResult partition_results = 3;
  uint32 written_count = 4;
  string detail = 5;
}

message PartitionSeedResult {
  uint32 partition_id = 1;
  PartitionSeedOutcome outcome = 2;
  uint32 written_count = 3;
  string detail = 4;
}

enum PartitionSeedOutcome {
  PARTITION_SEED_OUTCOME_UNSPECIFIED = 0;
  WRITTEN = 1;
  PARTITION_NOT_OWNED = 2;       // partition outside this shard's range
  ALREADY_SEEDED = 3;            // idempotency: batch_id was already processed
  WRITE_FAILED_RETRYABLE = 4;    // CRDB transient failure
}
```

---

## 4. Service definitions

### 4.1 `v1/service/van_provisioning_service.proto`

Pull path: runtime top-up.

```protobuf
syntax = "proto3";

package com.<org>.<bu>.<product>.caas.proto.v1.service;
option java_package = "com.<org>.<bu>.<product>.caas.proto.v1.service";

import "v1/request/van_batch_request.proto";
import "v1/response/van_batch_response.proto";
import "v1/request/van_lookup_request.proto";
import "v1/response/van_lookup_response.proto";

// Service exposed by the Management Plane to Mapping Engine shards
// and Client SDK callers.
service VanProvisioningService {

  // Pull path: shard requests a batch of partition-aligned VANs.
  //
  // Errors (as gRPC status codes):
  //   INVALID_ARGUMENT  - malformed bh_id, partition_id out of range,
  //                       requested_count is 0 or exceeds server max
  //   NOT_FOUND         - bh_id does not exist in topology
  //   UNAVAILABLE       - MP backing store is down (caller should retry)
  //   DEADLINE_EXCEEDED - caller's deadline expired
  //
  // Scarcity (no inventory available) is NOT an error; returns a
  // successful response with status = INVENTORY_EMPTY.
  rpc RequestVanBatch (v1.request.VanBatchRequest)
      returns (v1.response.VanBatchResponse);

  // Lookup path: Client SDK resolves a VAN to its (bh, partition).
  // Hot read; callers should cache aggressively.
  //
  // Errors:
  //   INVALID_ARGUMENT  - malformed VAN
  //   UNAVAILABLE       - MP backing store is down
  //
  // Unknown VAN is NOT an error; returns found=false.
  rpc LookupVan (v1.request.VanLookupRequest)
      returns (v1.response.VanLookupResponse);
}
```

**Notes:** Bundling `RequestVanBatch` and `LookupVan` on the same service is a deliberate call — they share an audience (callers that need MP) and the auth boundary is the same. Splitting later if needed is a localized refactor.

### 4.2 `v1/service/shard_seeding_service.proto`

Push path: MP-to-shard dispatch.

```protobuf
syntax = "proto3";

package com.<org>.<bu>.<product>.caas.proto.v1.service;
option java_package = "com.<org>.<bu>.<product>.caas.proto.v1.service";

import "v1/request/seed_partition_batch_request.proto";
import "v1/response/seed_partition_batch_response.proto";

// Service exposed by Mapping Engine shards to the Management Plane
// (in v1) or BH Control Plane (in production).
//
// Used for bulk seeding (initial pre-go-live load) and operator-
// initiated push dispatches when MP detects partition under-supply.
service ShardSeedingService {

  rpc SeedPartitionBatch (v1.request.SeedPartitionBatchRequest)
      returns (v1.response.SeedPartitionBatchResponse);
}
```

**Notes:** Single-RPC service. Justified because seeding is one-direction caller→shard; nothing else belongs here.

---

## 5. Final repository layout

After these contributions, the proto repository should look like:

```
proto-spec/src/main/proto/
└── v1/
    ├── common/
    │   ├── account_instructions.proto           (existing)
    │   ├── account_state.proto                  (existing)
    │   ├── seeding_status.proto                 (NEW — 1.2)
    │   └── van_inventory_status.proto           (NEW — 1.1)
    ├── request/
    │   ├── account_mapping_request.proto        (existing)
    │   ├── seed_partition_batch_request.proto   (NEW — 2.3)
    │   ├── van_batch_request.proto              (NEW — 2.1)
    │   └── van_lookup_request.proto             (NEW — 2.2)
    ├── response/
    │   ├── account_mapping_response.proto       (existing)
    │   ├── seed_partition_batch_response.proto  (NEW — 3.3)
    │   ├── van_batch_response.proto             (NEW — 3.1)
    │   └── van_lookup_response.proto            (NEW — 3.2)
    └── service/
        ├── account_mapping_service.proto        (existing)
        ├── shard_seeding_service.proto          (NEW — 4.2)
        └── van_provisioning_service.proto       (NEW — 4.1)
```

**Nine new files total.** Three PRs is reasonable; one big PR is fine if the team prefers.

---

## 6. What's deliberately not here yet

- **Topology service** (`GetBHMap`, `GetShardEndpoints`). Different audience (BH Control Planes), different concern. Design separately when needed.
- **Ops-facing inventory loading service.** V1 uses REST (matching legacy `VanUploadController`). If the team upgrades to gRPC for procurement, a new service is needed.
- **Production-grade seeding flow** with `StreamUnassignedNumbers`, `ConfirmIssued`, `BH Control Plane orchestration.` Deferred to production phase when PRK isolation matters.
- **Reconciliation service** for audit-log drift. Build only if observed drift justifies it.
- **Streaming variant of `RequestVanBatch`.** Add if runtime batches grow large enough to need it.

---

## 7. Verification before PR

- [ ] All package declarations match the actual repo convention. Replace `<org>.<bu>.<product>` placeholders.
- [ ] All `java_package` options match the protobuf package.
- [ ] Imports use repo-relative paths (`v1/common/...`), not absolute.
- [ ] Every enum has an `_UNSPECIFIED = 0` zero value.
- [ ] Run `mvn compile` locally on a consumer to confirm generated Java compiles cleanly.
- [ ] Run `protolint` or equivalent if the team uses one.

---

*Last updated: 2026-05-15*
