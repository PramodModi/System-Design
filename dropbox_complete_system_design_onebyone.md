# Dropbox System Design — Principal Engineer Interview Guide
## Progressive Design: Simple → Versions → Deduplication → Sharing

---

> **Interview strategy:** Start simple. Every feature you add must be justified by a requirement. Every component you introduce must address a specific bottleneck. Name the trade-off before the interviewer asks.

---

## Table of Contents

1. [Clarifying Questions & Assumptions](#1-clarifying-questions--assumptions)
2. [Stage 1 — Core: Upload, Download, Delete + Sync](#2-stage-1--core-upload-download-delete--sync)
3. [Stage 2 — Add Versioning](#3-stage-2--add-versioning)
4. [Stage 3 — Add Deduplication](#4-stage-3--add-deduplication)
5. [Stage 4 — Add File and Folder Sharing](#5-stage-4--add-file-and-folder-sharing)
6. [Observability and Traceability](#6-observability-and-traceability)

---

## 1. Clarifying Questions & Assumptions

In a real interview, ask these before touching the whiteboard.

### Questions I would ask

| Category | Question |
|----------|----------|
| Scale | How many total users? DAU? |
| File size | Max file size? Average file size? |
| Sync | How many devices per user? Offline support? |
| Geo | Single region or global? |
| Consistency | Strong or eventual for sync? |
| Versioning | How many versions? Retention period? |
| Sharing | Public links? Collaboration? |
| Compliance | GDPR? HIPAA? Data residency? |

### Assumptions (state these explicitly)

```
Users:            500 million total, 100 million DAU
Devices per user: avg 3 (laptop, phone, tablet)
File size:        avg 1 MB, max 5 GB
Storage quota:    15 GB per user
Geo:              Global deployment
Consistency:      Read-your-writes for uploader; eventual for sync
Conflict model:   Conflict copy (Dropbox model)
Offline:          Yes — sync when reconnected
```

### Back-of-Envelope Calculations

These numbers drive every architectural decision that follows.

**Storage:**
```
500M users × 15 GB = 7.5 exabytes raw
× 3 replication factor = 22.5 EB total
→ Must use managed object storage (S3-class). No self-hosted disk.
```

**Upload throughput:**
```
100M DAU × 10% upload daily = 10M uploads/day
10M / 86,400 sec = ~116 uploads/sec average
× 5 peak factor = ~580 uploads/sec peak
× 1 MB avg = 580 MB/sec peak bandwidth
→ App servers cannot handle this bandwidth. Direct-to-S3 is mandatory.
```

**Metadata DB size:**
```
500M users × 1,000 files avg = 500 billion file records
Each file row ≈ 500 bytes = 250 TB metadata
→ Single PostgreSQL instance impossible. Sharding from day one.
```

**Sync QPS:**
```
100M DAU × 3 devices = 300M device connections
Poll / push every 30 sec = 10M sync requests/sec
→ Sync service is the highest-QPS component in the system
```

**Read:Write ratio:** ~10:1 (downloads >> uploads). System is read-heavy. CDN is essential.

---

## 2. Stage 1 — Core: Upload, Download, Delete + Sync

### 2.1 Functional Requirements

**Must have:**
- Upload a file from any device
- Download a file to any device
- Delete a file
- Sync changes across all of a user's devices in near-real-time
- Offline support — work offline, sync on reconnect

**Out of scope for Stage 1:**
- Versioning (Stage 2)
- Deduplication (Stage 3)
- Sharing (Stage 4)
- Real-time collaborative editing (out of scope entirely)

### 2.2 Non-Functional Requirements

| Property | Target | Why |
|----------|--------|-----|
| Availability | 99.99% (52 min/year) | Files are work-critical |
| Upload latency | p99 < 500ms for first ACK | User feels responsive |
| Download latency | p99 < 200ms TTFB | CDN-served |
| Sync propagation | < 30 seconds end-to-end | Acceptable for async |
| Durability | 99.999999999% (11 nines) | Never lose a file |
| Consistency | Read-your-writes for uploader | User sees own upload immediately |
| Throughput | 580 MB/sec peak upload | Back-of-envelope derived |
| Scalability | Handle 10× traffic spikes | Black Friday, viral moments |

---

### 2.3 Why Cloud Object Storage (S3) Over Self-Hosted Disk

This question comes up in every interview. The answer is not "because AWS is good." It is because the math makes self-hosted disk architecturally impossible at this scale.

**The self-hosted disk problem:**

```
22.5 EB storage needed
Enterprise SAS disk: 20 TB = ~1.1 million disks
Annual failure rate: 4% = ~44,000 disk failures/year = 120/day
Each failure requires: detect → repair → rebalance

Your team would spend more time on disk ops than on product.
```

**What S3 gives you that you cannot build yourself:**

| Property | S3 / Cloud Blob | Self-hosted |
|----------|----------------|-------------|
| Durability | 99.999999999% (erasure coding + multi-AZ) | Whatever you build |
| Scaling | Infinite, on-demand | Capacity planning, procurement cycles |
| Global presence | 30+ regions, edge endpoints | Multi-datacenter = years of work |
| Lifecycle policies | S3 Standard → IA → Glacier, automated | Custom ETL jobs |
| Presigned URLs | Native, cryptographically signed | You build the whole auth layer |
| Operational burden | Zero disk management | 120 disk failures/day to manage |
| Egress cost | Paid, but predictable | Hidden in hardware + bandwidth |

**Trade-off to acknowledge:** S3 has per-request and egress costs that add up at scale. Dropbox famously moved some storage in-house ("Magic Pocket") after reaching exabyte scale to save ~75% on storage costs. But this requires hundreds of engineers and years of work. At any reasonable scale, managed object storage wins on total cost of ownership.

---

### 2.4 API Design — Stage 1

#### File Upload (2-Phase)

```
POST /v1/files/init
Headers:
  Authorization: Bearer {jwt}
  Idempotency-Key: {client_uuid}
Body: {
  name:        "report.pdf",
  folder_id:   "folder_xyz",       // null = root
  size_bytes:  10485760,
  mime_type:   "application/pdf",
  chunk_hashes: ["sha256_A", "sha256_B", ...]  // client-computed, in sequence order
}
Response 200: {
  upload_id:      "upload_abc",
  missing_chunks: ["sha256_A", "sha256_B"],    // only missing ones
  presigned_urls: {
    "sha256_A": "https://s3.amazonaws.com/chunks/sha256_A?X-Amz-Signature=...",
    "sha256_B": "https://s3.amazonaws.com/chunks/sha256_B?X-Amz-Signature=..."
  }
}

POST /v1/files/{file_id}/complete
Body: {
  upload_id: "upload_abc",
  chunks:    [{ hash, sequence, etag }, ...]   // etag from S3 PUT response
}
Response 200: { file_id, name, size_bytes, created_at }
```

**Why two-phase?** The `/init` call lets the server check which chunks already exist, enabling deduplication (Stage 3). Even in Stage 1, this pattern sets up the architecture correctly without wasted refactoring.

#### File Download

```
GET /v1/files/{file_id}
Response 200: {
  file_id, name, mime_type, size_bytes,
  chunks: [
    { sequence: 0, hash: "sha256_A", url: "https://s3.../sha256_A?sig=..." },
    { sequence: 1, hash: "sha256_B", url: "https://s3.../sha256_B?sig=..." }
  ]
}
// Client fetches all chunk URLs in parallel, reassembles in sequence order
// Client verifies SHA-256 of reassembled file
```

#### File Delete

```
DELETE /v1/files/{file_id}
Response 204: (no body)
// Soft delete: sets is_deleted=true, schedules S3 GC async
```

#### Sync

```
GET /v1/sync/changes?cursor={change_id}&device_id={id}
// Long-poll: holds 30s if no changes
Response 200: {
  events: [
    { change_id: 4822, file_id, event_type: "created"|"modified"|"deleted", timestamp }
  ],
  next_cursor: 4823
}
```

---

### 2.5 Database Model — Stage 1

```sql
-- Sharded by owner_user_id (all user's files on one shard)

CREATE TABLE users (
  user_id       UUID PRIMARY KEY,
  email         VARCHAR UNIQUE NOT NULL,
  name          VARCHAR,
  quota_bytes   BIGINT DEFAULT 16106127360,  -- 15 GB
  used_bytes    BIGINT DEFAULT 0,
  created_at    TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE files (
  file_id           UUID PRIMARY KEY,
  owner_user_id     UUID NOT NULL,            -- shard key
  folder_id         UUID,                     -- NULL = root
  name              VARCHAR NOT NULL,
  mime_type         VARCHAR,
  size_bytes        BIGINT NOT NULL,
  s3_bucket         VARCHAR NOT NULL,
  s3_key            VARCHAR NOT NULL,         -- for single-chunk files (< 1 MB)
  is_deleted        BOOLEAN DEFAULT FALSE,
  created_at        TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMP NOT NULL DEFAULT NOW()
);

-- In Stage 1, multi-chunk files store chunk list here
CREATE TABLE file_chunks (
  file_id       UUID NOT NULL,
  sequence_num  INTEGER NOT NULL,
  chunk_hash    VARCHAR(64) NOT NULL,
  s3_key        VARCHAR NOT NULL,
  size_bytes    INTEGER NOT NULL,
  PRIMARY KEY   (file_id, sequence_num)
);

-- Append-only. Never updated. Used for sync.
-- Stored in Cassandra in production (linear append scale)
CREATE TABLE change_log (
  change_id     BIGINT NOT NULL,    -- monotonically increasing
  user_id       UUID NOT NULL,      -- partition key
  file_id       UUID NOT NULL,
  event_type    VARCHAR NOT NULL,   -- created | modified | deleted
  device_id     UUID,               -- which device originated this change
  created_at    TIMESTAMP NOT NULL,
  PRIMARY KEY   (user_id, change_id)
);
-- Cassandra equivalent:
-- PARTITION KEY: user_id
-- CLUSTERING KEY: change_id ASC

-- Tracks per-device sync cursor
CREATE TABLE device_cursors (
  device_id     UUID PRIMARY KEY,
  user_id       UUID NOT NULL,
  last_change_id BIGINT NOT NULL DEFAULT 0,
  last_seen_at  TIMESTAMP,
  device_name   VARCHAR
);
```

**Sharding key = `owner_user_id`.** All queries for a user's sync (give me all changes since cursor X) hit one shard. No cross-shard joins for the hot path.

---

### 2.6 Client-Side Design — Chunking, Reassembly, and Sync

The client is not just a UI. It contains significant logic.

#### Client Upload Flow

```
1. User selects file → client reads file into memory / stream

2. CHUNK DECISION:
   if file_size < 1 MB:
     → single direct upload (skip chunking entirely)
   elif file_size < 100 MB:
     → fixed 4 MB chunks
   else:
     → content-defined chunking via rolling hash (Rabin fingerprinting)
       Cut point: rolling_hash(4KB window) mod 4096 == 0
       Min chunk: 512 KB, Max chunk: 8 MB
       Why CDC for large files: insertion of bytes only affects 1 chunk,
       not all downstream chunks (boundary-shift problem)

3. HASH COMPUTATION (local, no network):
   SHA-256(each chunk) → chunk_hashes[]
   Client stores: { hash → bytes, sequence → hash }

4. POST /files/init with chunk_hashes[]
   Receive: missing_chunks[], presigned_urls{}

5. PARALLEL UPLOAD:
   For each missing chunk:
     PUT presigned_url with chunk bytes
     Store returned ETag
   Max concurrent: 6 parallel HTTP connections
   (saturates most network connections without overwhelming them)

6. POST /files/complete with { hash, sequence, etag } per chunk

7. Update local DB: file is now synced
```

#### Client Download Flow

```
1. GET /files/{file_id}
   Receive: chunk manifest [{sequence, hash, url}]

2. PARALLEL DOWNLOAD:
   Fetch all chunk URLs simultaneously
   Store chunks to temp directory as they arrive

3. REASSEMBLY:
   Sort by sequence_num ASC (never by arrival order)
   Concatenate in order → write to final destination

4. INTEGRITY VERIFICATION:
   SHA-256(reassembled_file) == declared file hash?
   If no → delete and retry (request fresh presigned URLs)
   If yes → move from temp to final location

5. Update local DB: file is synced at this version
```

#### Client Sync Engine

```
State machine per device:
  OFFLINE → CONNECTING → SYNCING → IDLE → OFFLINE

On CONNECTING:
  1. GET /sync/changes?cursor={last_known_change_id}&device_id={id}
  2. Apply each event:
     - created: download the new file
     - modified: download new version, replace local file
     - deleted: move local file to trash
  3. Advance cursor to max(change_id) in response
  4. Persist cursor to local SQLite DB (survives app restart)
  5. Go to IDLE

On IDLE:
  1. Long-poll: GET /sync/changes?cursor={cursor}
  2. Server holds connection 30s if no changes
  3. On event received → apply → loop
  4. On timeout → re-poll

Conflict detection:
  If local file modified AFTER last sync AND server file also changed:
    → Do NOT overwrite. Create conflict copy:
       "report (Pramod's conflicted copy 2024-01-15).pdf"
    → Both files exist. User resolves manually.
    → This is the Dropbox model: simple, no data loss.
```

#### Local Client Database (SQLite)

```sql
-- On each device, a local SQLite DB tracks sync state

CREATE TABLE local_files (
  file_id         TEXT PRIMARY KEY,
  name            TEXT NOT NULL,
  folder_path     TEXT NOT NULL,
  size_bytes      INTEGER,
  local_path      TEXT,           -- absolute path on this device
  server_hash     TEXT,           -- SHA-256 of last known server version
  local_hash      TEXT,           -- SHA-256 of current local file
  sync_state      TEXT,           -- synced | uploading | downloading | conflict
  last_synced_at  INTEGER         -- unix timestamp
);

CREATE TABLE sync_cursor (
  key             TEXT PRIMARY KEY DEFAULT 'cursor',
  change_id       INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE upload_sessions (
  upload_id       TEXT PRIMARY KEY,
  file_id         TEXT,
  chunk_hashes    TEXT,           -- JSON array, ordered
  missing_chunks  TEXT,           -- JSON array
  created_at      INTEGER
);
```

---

### 2.7 System Architecture — Stage 1

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                     │
│     Desktop (sync engine)  │  Mobile  │  Web Browser               │
└──────────────┬──────────────────────────────────┬───────────────────┘
               │ API calls (upload metadata,       │ file downloads
               │ sync polling)                     │
               ▼                                   ▼
┌──────────────────────────────┐    ┌──────────────────────────────────┐
│  API Gateway                 │    │  CDN (CloudFront)                │
│  - TLS termination           │    │  - Caches: thumbnails, shared    │
│  - JWT validation            │    │    public files                  │
│  - Rate limiting (token      │    │  - NOT for private user files    │
│    bucket per user)          │    │  - p99 download < 5ms on hit     │
│  - DDoS mitigation (WAF)     │    └──────────────────────────────────┘
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Load Balancer (L7, e.g. ALB)                                       │
│  - Round robin across Upload / Download / Sync service instances    │
│  - Health checks every 5s                                           │
│  - Sticky sessions NOT needed (all services are stateless)          │
└──────┬───────────────────────────┬──────────────────────────────────┘
       │                           │
       ▼                           ▼
┌──────────────┐         ┌──────────────────┐         ┌─────────────┐
│ Upload       │         │ Download         │         │ Sync        │
│ Service      │         │ Service          │         │ Service     │
│              │         │                  │         │             │
│ - Validates  │         │ - Auth check     │         │ - Long-poll │
│   JWT        │         │ - Queries DB     │         │   or SSE    │
│ - Checks     │         │ - Generates      │         │ - Reads     │
│   quota      │         │   presigned GET  │         │   change_log│
│ - Queries    │         │   URLs           │         │ - Pushes    │
│   chunk_store│         │ - Never touches  │         │   events to │
│ - Generates  │         │   file bytes     │         │   devices   │
│   presigned  │         └──────────────────┘         └─────────────┘
│   PUT URLs   │
│ - Never sees │
│   file bytes │
└──────────────┘
       │ all services read/write
       ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Redis Cluster (cache + sessions)                                   │
│  - File metadata cache (TTL 5 min, invalidated by CDC)              │
│  - Upload session state (TTL 30 min)                                │
│  - Rate limit counters                                              │
│  - Sync: active SSE connections per user                            │
└──────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Metadata DB — PostgreSQL (sharded by owner_user_id)                │
│  Shard 0: user_id 0..N/3  │  Shard 1  │  Shard 2                  │
│  Each shard: 1 primary + 2 read replicas in same AZ                │
│  Consistent hashing for future shard expansion                      │
└─────────────────────────┬────────────────────────────────────────────┘
                          │ CDC (Debezium tails WAL)
                          ▼
              Redis cache invalidation
              + Change log consumer

┌──────────────────────────────────────────────────────────────────────┐
│  Change Log — Cassandra (append-only)                               │
│  Partition key: user_id                                             │
│  Clustering key: change_id ASC                                      │
│  Retention: 30 days                                                 │
│  → Linear write scale. Per-user cursors never need cross-shard scan │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  S3 Blob Storage (multi-region)                                     │
│  Bucket: dropbox-chunks                                             │
│  Key pattern: chunks/{sha256_hash}                                  │
│  Lifecycle: Standard → S3-IA (30d) → Glacier (90d)                 │
│  Client uploads/downloads directly via presigned URLs               │
│  → App servers NEVER handle file bytes                              │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  Kafka (async processing)                                           │
│  Topics: file-upload-complete, file-deleted                         │
│  Consumers: thumbnail-worker, virus-scan-worker, quota-update-worker│
│  → Upload returns fast. Side effects happen async.                  │
└──────────────────────────────────────────────────────────────────────┘
```

---

### 2.8 Handling High Throughput, Scalability, Spikes

#### Throughput — The Presigned URL Pattern

The single most important decision for throughput: **app servers never touch file bytes.**

```
WITHOUT presigned URLs:
  Client → App Server → S3
  580 MB/sec flowing through your app servers
  Each server needs massive network I/O
  Can't horizontally scale beyond network card limits

WITH presigned URLs:
  Client → S3 directly
  App server only signs a URL (microseconds, no bytes)
  Throughput scales with S3 (effectively unlimited)
  App servers scale for API logic, not bandwidth
```

#### Scalability — Horizontal Stateless Services

Every service (Upload, Download, Sync) is stateless. No session affinity needed. Add instances behind the load balancer when CPU/memory crosses 70%. Auto-scaling group triggers on:
- Upload service: active presigned URL generation rate
- Sync service: active long-poll connections
- Download service: DB query latency p99

#### Handling Spikes

Three mechanisms, applied in layers:

**1. Token bucket rate limiting (API Gateway)**
```
Per user: 100 API calls/min (configurable)
Per IP: 1000 API calls/min (DDoS protection)
Burst: allow 2× for 60 seconds, then throttle
Returns HTTP 429 with Retry-After header
```

**2. Request queuing (Kafka)**
```
upload-complete events queued in Kafka
Consumers (thumbnail, virus scan) process at their own pace
Spike in uploads → Kafka queue grows → consumers catch up
Upload API response is not blocked by downstream processing
```

**3. S3 bucket policy rate limiting**
```
Max upload size per presigned URL: 4 MB (prevents storage abuse)
Content-type restriction: enforced at S3 bucket policy level
Cannot be bypassed even with a valid presigned URL
```

#### Reliability

**Multi-AZ deployment:** All services deployed across 3 AZs. Load balancer health-checks every 5 seconds. AZ failure → traffic rerouted in < 30 seconds.

**Database HA:** PostgreSQL with synchronous replication to one replica in same AZ. Patroni manages automatic failover. RTO < 60 seconds.

**S3 durability:** 99.999999999% — effectively never loses data. Cross-region replication for disaster recovery.

**Idempotency:** All mutating API endpoints require `Idempotency-Key` header. Safe to retry without duplicate side effects.

#### Low Latency

**For uploads (p99 < 500ms for first ACK):**
- `/init` only queries Redis (cache hit) or one DB shard (cache miss): ~20ms
- Presigned URL generation: ~5ms
- Total server-side work: ~30ms. Remaining budget is network.

**For downloads (p99 < 200ms TTFB):**
- Cache hit (Redis): return presigned URLs in ~10ms
- Cache miss: 2 DB queries + presigned URL gen: ~50ms
- Chunk bytes from S3/CDN: separate from API latency

**For sync propagation (< 30s):**
- CDC detects DB write: ~50ms
- Kafka event: ~5ms
- SSE push to device: ~5ms
- Total: ~60ms for online devices. Offline devices see it on reconnect.

#### Geo Upload and Download

**Problem:** User in Bangalore uploading to S3 in us-east-1 = 200ms latency per chunk.

**Solution: S3 Transfer Acceleration (upload) + CloudFront (download)**

```
Upload with Transfer Acceleration:
  Client → CloudFront Edge (nearest, e.g. Mumbai)
  → AWS backbone network (optimised, not public internet)
  → S3 us-east-1
  Latency reduction: 30-60% on long-distance uploads

Download with CloudFront:
  First request: CloudFront → S3 → cache at edge
  Subsequent requests: served from Mumbai edge node
  Latency: 5-20ms vs 200ms+ from origin

Multi-region S3 replication (for compliance or extreme latency):
  Replicate to ap-south-1 (Mumbai) for APAC users
  Cost: ~$0.02/GB replication fee
  Benefit: local-region download for 30% of users
```

#### Load Balancer Design

```
Tier 1: AWS Global Accelerator (or anycast routing)
  - Routes user to nearest regional entry point
  - Health-check: routes around regional outages

Tier 2: Application Load Balancer (L7) per region
  - Path-based routing:
    /v1/files/init      → Upload Service target group
    /v1/files/*/complete → Upload Service target group
    GET /v1/files/*     → Download Service target group
    /v1/sync/*          → Sync Service target group
  - Sticky sessions: OFF (all services stateless)
  - Health check: GET /health → 200 within 2s

Tier 3: Internal load balancing between service instances
  - Service mesh (e.g. AWS App Mesh / Istio)
  - mTLS between services
  - Circuit breaker (if 50% of requests fail in 10s → open circuit)
```

---

### 2.9 Security

**Authentication:** JWT (15-minute expiry) + refresh token rotation. JWT contains `user_id`, `device_id`, `exp`. Validated at API Gateway — services never re-validate.

**Presigned URL security:**
```
Presigned URL parameters are HMAC-signed:
  - Expiry locked in signature (changing it breaks the signature)
  - S3 key locked (cannot upload to a different path)
  - HTTP method locked (PUT for upload, GET for download)
  - Max content-length enforced by S3 bucket policy
  - HTTPS-only enforced by bucket policy

If a presigned URL is stolen and used within 15 minutes:
  - Attacker can only write to that one 4 MB S3 key
  - ETag validation at /complete detects substituted bytes
  - Short expiry bounds the risk window
```

**Encryption:**
```
In transit: TLS 1.3 everywhere
  Client → API Gateway: TLS
  Client → S3 (presigned): TLS (enforced by bucket policy)
  Service → Service: mTLS

At rest: AES-256-GCM
  S3: SSE-KMS (envelope encryption)
  DEK (Data Encryption Key): unique per file
  KEK (Key Encryption Key): in AWS KMS, never leaves KMS plaintext
  Key rotation: every 90 days
  Crypto-erasure: delete DEK → file unreadable instantly (GDPR compliance)
```

**Authorization:**
```
Files table has owner_user_id
Every API request: verify JWT user_id == file owner_user_id
Rate limiting prevents enumeration attacks
```

**Virus scanning:**
```
Async post-upload (via Kafka)
ClamAV + commercial AV engine
File quarantined if flagged → download blocked → user notified
Does NOT block upload response (would violate p99 < 500ms SLO)
```

---

### 2.10 Stage 1 Trade-offs

| Decision | What you gain | What you give up |
|----------|--------------|-----------------|
| Presigned URLs (direct to S3) | Infinite upload throughput | Lose centralised throttling on bytes |
| S3 over self-hosted | 11-nines durability, zero ops | Egress cost, vendor lock-in |
| Shard by owner_user_id | All user's files co-located | Hot shard risk for power users |
| Eventual sync (SSE/long-poll) | Works offline, highly available | 30-60s propagation delay |
| Conflict copy (vs CRDT) | Simple, no data loss | User must resolve manually |
| Cassandra for change_log | Linear append scale | No cross-user joins, eventual consistency |
| CDC for cache invalidation | Crash-safe, DB is truth | 10-500ms stale window |
| Async virus scan | Upload returns in <500ms | Brief window where unscanned file accessible |

---

## 3. Stage 2 — Add Versioning

### 3.1 What Changes — The One-Line Summary

> In Stage 1, uploading a new version overwrites the file. In Stage 2, we keep the last 30 versions and allow restore.

### 3.2 New Functional Requirements

- Every upload creates a new version (not an overwrite)
- List all 30 versions of a file
- Restore a file to any previous version
- Version retention: keep last 30; oldest auto-pruned when 31st is created

### 3.3 Data Model Changes

**Stage 1 had:** `files` table with `s3_key` directly.

**Stage 2 splits this into:**

```sql
-- files: now a pointer to its current version
CREATE TABLE files (
  file_id            UUID PRIMARY KEY,
  owner_user_id      UUID NOT NULL,         -- shard key
  folder_id          UUID,
  name               VARCHAR NOT NULL,
  mime_type          VARCHAR,
  current_version_id UUID,                  -- FK to file_versions
  is_deleted         BOOLEAN DEFAULT FALSE,
  created_at         TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at         TIMESTAMP NOT NULL DEFAULT NOW()
);

-- NEW: one row per version
CREATE TABLE file_versions (
  version_id           UUID PRIMARY KEY,
  file_id              UUID NOT NULL,       -- FK to files
  version_number       INTEGER NOT NULL,    -- 1, 2, 3 ...
  size_bytes           BIGINT NOT NULL,
  created_by_device_id UUID,
  created_at           TIMESTAMP NOT NULL DEFAULT NOW(),

  INDEX (file_id, version_number)           -- for listing versions
);

-- NEW: chunk sequence per version
CREATE TABLE version_chunks (
  version_id    UUID NOT NULL,              -- FK to file_versions
  sequence_num  INTEGER NOT NULL,
  chunk_hash    VARCHAR(64) NOT NULL,       -- FK to chunk_store (Stage 3)
  s3_key        VARCHAR NOT NULL,           -- redundant in Stage 3, needed in Stage 2
  size_bytes    INTEGER NOT NULL,
  PRIMARY KEY   (version_id, sequence_num),
  INDEX         (chunk_hash)                -- for ref_count lookups in Stage 3
);
```

**Why `version_chunks` instead of storing `s3_key` on `file_versions`?**

A version is a sequence of chunks in order. The `version_chunks` table is the join table that maps a version to its ordered list of chunk storage locations. This is the structure that enables deduplication in Stage 3 — two versions pointing to the same physical chunk without copying bytes.

### 3.4 New API Endpoints

```
-- List versions
GET /v1/files/{file_id}/versions
Response 200: {
  versions: [
    { version_id, version_number: 5, size_bytes, created_at, device_name },
    { version_id, version_number: 4, ... },
    ...
  ]
}

-- Download specific version (extends existing download API)
GET /v1/files/{file_id}?version=3
// Same response shape as before, just serves version 3's chunks

-- Restore to a previous version
POST /v1/files/{file_id}/restore
Body: { version_id: "version_abc" }
Response 200: { new_version_id, version_number: 6 }
// Creates version 6 pointing to version 3's chunks. Zero bytes copied in S3.
```

### 3.5 Upload Flow Changes

**/init is unchanged.** Client still sends chunk_hashes[].

**/complete now creates a version record:**

```
Transaction at /complete:
  1. INSERT file_versions (version_id=NEW, file_id, version_number=max+1, size_bytes)
  2. INSERT version_chunks rows (one per chunk, pointing to S3 keys)
  3. UPDATE files SET current_version_id=NEW, updated_at=NOW()
  4. INSERT change_log (event_type='modified', version_id=NEW)
  5. CHECK: if version_count > 30 → prune_oldest_version()

prune_oldest_version():
  1. Find oldest version_id for this file
  2. Delete its version_chunks rows
  3. Delete the S3 objects (or mark for GC — see Stage 3)
  4. Delete the file_versions row
```

**Note:** In Stage 2, deleting a version means deleting the S3 objects. In Stage 3 (deduplication), we will NOT delete S3 objects immediately — we decrement a ref_count and let GC handle it.

### 3.6 Download Flow Changes

**Download current version (most common):**
```sql
-- files.current_version_id gives version_id directly (no extra query)
SELECT vc.sequence_num, vc.s3_key
FROM   version_chunks vc
WHERE  vc.version_id = :current_version_id
ORDER  BY vc.sequence_num ASC
```

**Download specific version:**
```sql
-- Need one extra query to resolve version_number → version_id
SELECT version_id FROM file_versions
WHERE  file_id = :file_id AND version_number = :requested_version

-- Then same chunks query
```

### 3.7 Restore is Zero-Cost (Copy-on-Write)

```
User: "Restore file to version 3"

Server:
  1. Read version_chunks WHERE version_id = version_3_id
     → list of { sequence_num, s3_key }

  2. INSERT file_versions (version_id=NEW, version_number=6)

  3. INSERT version_chunks for NEW version
     pointing to the SAME s3_key values as version 3
     → No S3 copy. No bytes transferred.

  4. UPDATE files SET current_version_id = NEW

  5. INSERT change_log (event_type='restored')

Cost: a few DB inserts. S3 untouched.
```

### 3.8 Version Metadata Explosion — Trade-off

**The problem:**
```
500M users × 1000 files × 30 versions × 2.5 chunks avg
= 37.5 trillion version_chunks rows

This is 10-50× larger than the files table.
Metadata DB must shard aggressively.
```

**Mitigations:**

1. **Keep shard key = owner_user_id.** All version_chunks for a user's files co-locate with the files and file_versions rows on the same shard. No cross-shard joins for any user operation.

2. **Compress chunk list per version.** Instead of one row per chunk, store chunk hashes as an ordered array in a JSONB column on `file_versions`. Trades query flexibility for storage efficiency:

```sql
-- Alternative schema for version_chunks (lower row count)
ALTER TABLE file_versions ADD COLUMN chunk_list JSONB;
-- chunk_list = ["sha256_A", "sha256_B", "sha256_C"]  (ordered array)
-- Pro: 1 row per version instead of N rows
-- Con: cannot index individual chunk hashes (needed for dedup in Stage 3)
-- Decision: keep separate rows for Stage 3 compatibility
```

3. **Archive old version metadata.** Versions older than 90 days move to cold Cassandra partition. Restore still works but takes longer.

### 3.9 Stage 2 Trade-offs

| Decision | Gain | Cost |
|----------|------|------|
| version_chunks as join table | Enables dedup in Stage 3, clean separation | More rows than JSONB column |
| 30-version limit | Bounds storage growth | Users lose older history |
| Prune on write (not on schedule) | Simple, immediate | Adds latency to the 31st upload |
| Restore = new version (not rollback) | Preserves history, simple | Version number grows unboundedly |
| Storage class per chunk | Old chunks → Glacier at 90d | Restore of old version takes 3-12h from Glacier |

**Recommendation on version limit:** Expose as a configurable setting per plan tier. Free: 10 versions. Pro: 30 versions. Business: unlimited (with cost per version above 100).

---

## 4. Stage 3 — Add Deduplication

### 4.1 What Changes — The One-Line Summary

> In Stage 2, every chunk is stored once per version. In Stage 3, identical chunks across any user, any file, any version are stored exactly once.

### 4.2 Why Deduplication

**Storage math at scale:**
```
Scenario: 1 million users each upload the same 10 MB PDF

Stage 2: 1M × 10 MB = 10 TB stored
Stage 3: 10 MB stored (one copy, 1M references)
Saving: 99.999%

In practice, dedup saves 30-60% of storage at Dropbox scale.
At 22.5 exabytes total storage, that is 6-12 exabytes saved.
At $0.023/GB/month: $138M-$276M/year saved.
```

### 4.3 Data Model Changes

**Add one table, modify the join table:**

```sql
-- NEW: global registry of all unique chunks
CREATE TABLE chunk_store (
  chunk_hash     VARCHAR(64) PRIMARY KEY,   -- SHA-256, hex-encoded
  s3_bucket      VARCHAR NOT NULL,
  s3_key         VARCHAR NOT NULL,          -- chunks/{chunk_hash}
  size_bytes     INTEGER NOT NULL,
  ref_count      INTEGER NOT NULL DEFAULT 0, -- # of version_chunks rows pointing here
  storage_class  VARCHAR NOT NULL DEFAULT 'STANDARD',  -- STANDARD | IA | GLACIER
  etag           VARCHAR(34),               -- MD5 from S3, for integrity verification
  created_at     TIMESTAMP NOT NULL DEFAULT NOW()
);

-- MODIFIED: version_chunks now references chunk_store
-- Remove s3_key (now in chunk_store), keep chunk_hash as FK
ALTER TABLE version_chunks DROP COLUMN s3_key;
-- version_chunks.chunk_hash is now a FK to chunk_store.chunk_hash
```

**The two-table dedup architecture:**

```
chunk_store                    version_chunks
────────────────────           ──────────────────────────────────────
chunk_hash = sha256_A    ←──   (version_5_pramod, seq=0, sha256_A)
  s3_key = chunks/sha256_A     (version_1_anita,  seq=2, sha256_A)
  ref_count = 3           ←──   (version_2_raj,    seq=0, sha256_A)
  storage_class = STANDARD

One physical blob in S3. Three version_chunks rows. ref_count = 3.
```

### 4.4 Upload Flow Changes — /init Query

**The dedup check at /init:**

```sql
-- Client sends all chunk_hashes[]
-- Server queries ONLY chunk_store (not files, file_versions, version_chunks)

SELECT chunk_hash
FROM   chunk_store
WHERE  chunk_hash = ANY(ARRAY['sha256_A', 'sha256_B', 'sha256_C'])
```

`chunk_store.chunk_hash` is the PRIMARY KEY → batch primary key lookup, O(1) per hash. Returns only existing hashes. Client uploads only the difference.

**The /complete transaction with deduplication:**

```sql
BEGIN;

-- A: Insert newly uploaded chunks (ON CONFLICT = safe for race conditions)
INSERT INTO chunk_store (chunk_hash, s3_bucket, s3_key, size_bytes, etag)
VALUES ('sha256_B', 'bucket', 'chunks/sha256_B', 4194304, '"md5_B"')
ON CONFLICT (chunk_hash) DO NOTHING;

-- B: Increment ref_count for ALL chunks this version references
--    (both existing sha256_A and newly uploaded sha256_B)
UPDATE chunk_store
SET    ref_count = ref_count + 1        -- atomic row-level increment
WHERE  chunk_hash IN ('sha256_A', 'sha256_B', 'sha256_C');

-- C: Insert file_versions row
INSERT INTO file_versions (...);

-- D: Insert version_chunks rows (pointing to chunk_store, no s3_key needed)
INSERT INTO version_chunks (version_id, sequence_num, chunk_hash)
VALUES (...);

-- E: Update files.current_version_id
UPDATE files SET current_version_id = ... ;

-- F: Append change_log
INSERT INTO change_log (...);

COMMIT;
```

### 4.5 The ref_count — The Most Dangerous State

`ref_count` must be perfectly accurate at all times. A bug that decrements it to zero prematurely causes **silent, unrecoverable data loss for other users** (their version_chunks point to a deleted S3 object).

**Protection mechanisms:**

```
1. Soft delete: never delete S3 object immediately.
   Mark chunk_store row as pending_gc.
   Hard-delete S3 object 48 hours later.

2. GC double-check before hard delete:
   SELECT COUNT(*) FROM version_chunks WHERE chunk_hash = ?
   If count > 0: ref_count was wrong. Do NOT delete. Alert on-call.

3. Weekly reconciliation job:
   Full scan of version_chunks → recount references per chunk_hash
   Compare to chunk_store.ref_count
   Alert if mismatch > 0

4. ref_count update is inside DB transaction:
   Cannot have version_chunks written without ref_count incremented.
```

### 4.6 Version Pruning Changes (with deduplication)

**Stage 2 pruning:** Delete S3 objects directly.

**Stage 3 pruning:**
```
1. Find oldest version for this file
2. For each chunk in version_chunks:
   UPDATE chunk_store SET ref_count = ref_count - 1 WHERE chunk_hash = ?
3. Find all chunk_hashes WHERE ref_count = 0 (now orphaned)
4. Mark as pending_gc (do NOT delete immediately)
5. Delete version_chunks rows
6. Delete file_versions row

GC job runs hourly:
  SELECT chunk_hash FROM chunk_store
  WHERE  ref_count = 0 AND pending_gc = TRUE
  AND    updated_at < NOW() - INTERVAL '48 hours'
  → DELETE from S3
  → DELETE from chunk_store
```

### 4.7 Race Condition: Two Users Upload Same Chunk Simultaneously

```
T=0: Pramod's /init → sha256_X not found in chunk_store
T=0: Anita's /init  → sha256_X not found in chunk_store
T=1: Both get presigned URL for sha256_X
T=2: Both PUT to S3 (idempotent — same bytes, same key, no problem)
T=3: Pramod's /complete arrives first
     INSERT INTO chunk_store (sha256_X ...) → succeeds
T=4: Anita's /complete arrives
     INSERT INTO chunk_store (sha256_X ...) → ON CONFLICT DO NOTHING
     (no error, no duplicate row)
T=5: Both UPDATE ref_count = ref_count + 1
     Atomic row-level locks make these serial
     Final ref_count = 2. Correct.
```

### 4.8 Stage 3 Trade-offs

| Decision | Gain | Cost |
|----------|------|------|
| chunk_store as global table | Cross-user dedup, massive storage savings | Global table = potential hot shard |
| ref_count | Precise GC, no orphan blobs | Bug risk: decrement error = data loss |
| Soft delete (48h grace) | Safety net for ref_count bugs | Storage cost of deleted-but-not-GC'd chunks |
| ON CONFLICT DO NOTHING | Safe concurrent uploads | Second uploader wasted their bandwidth |
| Shard chunk_store by hash prefix | Even distribution (hashes are random) | Different shard than user's files |

**The chunk_store sharding question:**

`chunk_store` is a global table. It cannot shard by `user_id` (it has no user). Options:

1. **Shard by `chunk_hash` prefix** (first 2 hex chars = 256 shards). Even distribution since SHA-256 is uniform. Cross-shard is unavoidable when a version references chunks on multiple shards — mitigate with batching.

2. **Separate dedicated cluster.** `chunk_store` is write-once, read-often. Optimise for read throughput. Run it on a separate PostgreSQL cluster from the user metadata.

3. **Redis for hot chunk lookups.** Cache the most-referenced chunk_hashes in Redis. 90% of lookups hit a small set of popular files (Linux ISOs, common PDFs). Cache hit rate very high.

---

## 5. Stage 4 — Add File and Folder Sharing

### 5.1 What Changes — The One-Line Summary

> In Stage 3, only the owner can access a file. In Stage 4, owners can grant access to specific users or generate public links.

### 5.2 New Functional Requirements

- Share a file or folder with specific users (viewer or editor role)
- Generate a public link with optional expiry and optional password
- Revoke access at any time
- Collaborators see shared files in their own file tree

### 5.3 Data Model Changes

```sql
-- NEW: tracks explicit shares between users
CREATE TABLE file_shares (
  share_id      UUID PRIMARY KEY,
  resource_id   UUID NOT NULL,          -- file_id or folder_id
  resource_type VARCHAR NOT NULL,       -- 'file' | 'folder'
  owner_user_id UUID NOT NULL,          -- who granted access
  grantee_user_id UUID,                 -- NULL for public links
  permission    VARCHAR NOT NULL,       -- 'view' | 'edit'
  created_at    TIMESTAMP NOT NULL DEFAULT NOW(),
  expires_at    TIMESTAMP,              -- NULL = no expiry
  INDEX (grantee_user_id, resource_id)  -- for listing "files shared with me"
);

-- NEW: public link tokens
CREATE TABLE share_links (
  token         VARCHAR(32) PRIMARY KEY,  -- random URL-safe token
  resource_id   UUID NOT NULL,
  resource_type VARCHAR NOT NULL,
  owner_user_id UUID NOT NULL,
  permission    VARCHAR NOT NULL DEFAULT 'view',
  password_hash VARCHAR,                  -- NULL = no password
  expires_at    TIMESTAMP,
  created_at    TIMESTAMP NOT NULL DEFAULT NOW(),
  access_count  INTEGER DEFAULT 0         -- analytics
);

-- NEW: tracks which shared items appear in which user's view
CREATE TABLE user_shared_items (
  user_id       UUID NOT NULL,           -- grantee
  resource_id   UUID NOT NULL,
  resource_type VARCHAR NOT NULL,
  owner_user_id UUID NOT NULL,           -- who shared it
  share_id      UUID NOT NULL,
  added_at      TIMESTAMP NOT NULL DEFAULT NOW(),
  PRIMARY KEY   (user_id, resource_id)
);
```

### 5.4 Authorization Model — The Critical Change

**Before Stage 4:** Auth check = `JWT user_id == file.owner_user_id`. Simple, fast.

**After Stage 4:** Auth check = `user is owner OR user has a valid share`.

```python
def check_file_access(file_id, user_id, required_permission='view'):
    # Check Redis cache first (permission is checked on every request)
    cache_key = f"perm:{file_id}:{user_id}"
    cached = redis.get(cache_key)
    if cached:
        return cached['permission'] >= required_permission

    # Owner always has full access
    file = db.query("SELECT owner_user_id FROM files WHERE file_id = ?", [file_id])
    if file.owner_user_id == user_id:
        redis.setex(cache_key, 300, {'permission': 'edit'})
        return True

    # Check file_shares
    share = db.query("""
        SELECT permission FROM file_shares
        WHERE  resource_id = ? AND grantee_user_id = ?
        AND    (expires_at IS NULL OR expires_at > NOW())
    """, [file_id, user_id])

    if share:
        redis.setex(cache_key, 300, {'permission': share.permission})
        return share.permission >= required_permission

    raise HTTP403("Access denied")

# IMPORTANT: when a share is revoked, invalidate the cache:
redis.delete(f"perm:{file_id}:{user_id}")
```

### 5.5 The Shared Folder Cross-Shard Problem

**The problem:**
```
User A (owner, Shard 0) shares a folder with User B (Shard 2).
User B uploads a file to that shared folder.
Where does the file row live?
```

**Option 1: File lives on the owner's shard**
- All folder contents always on one shard
- Cross-shard write when B uploads (B's request → hits B's shard → cross-shard write to A's shard)
- Clean model, higher write latency for collaborators

**Option 2: File lives on the uploader's shard**
- Fast writes for collaborators
- Folder listing requires scatter-gather across multiple shards
- Inconsistent data locality

**Recommendation: Option 1 (owner's shard)**
```
Folder contents always live on owner's shard.
When User B uploads to a shared folder:
  1. Check share permission (file_shares table)
  2. Write file row to Shard 0 (owner's shard), not Shard 2
  3. This is an async cross-shard write — acceptable for collaborative access patterns
  
Trade-off: B's uploads to shared folders are ~5ms slower (cross-shard).
Benefit: folder listing is always a single-shard query.
```

### 5.6 New API Endpoints

```
-- Share a file or folder
POST /v1/shares
Body: {
  resource_id:   "file_abc",
  resource_type: "file",
  permission:    "view",      // or "edit"
  expires_at:    "2025-12-31T00:00:00Z",  // optional
  password:      "secret123"  // optional
}
Response 200: {
  share_id:  "share_xyz",
  share_url: "https://dropbox.com/s/tok_abc"  // for public links
}

-- List people a resource is shared with
GET /v1/shares?resource_id={id}
Response 200: { shares: [{ grantee_email, permission, expires_at }] }

-- Revoke a share
DELETE /v1/shares/{share_id}
Response 204

-- List files shared with me
GET /v1/shared-with-me
Response 200: { items: [{ resource_id, resource_type, owner_name, permission }] }

-- Resolve a public link (no auth required)
GET /v1/public/{token}
// If password required: returns 401 with password_required: true
// If valid: returns same structure as GET /v1/files/{file_id}
```

### 5.7 Sync Changes for Shared Files

**The problem:** When User A modifies a shared file, User B needs to see the update in their sync.

**Solution: Shared change log events**

```
On /complete for a shared file:
  1. Find all share_id records for this file
  2. For each grantee_user_id:
     INSERT INTO change_log (user_id=grantee_user_id, file_id, event_type='modified')

Now B's sync cursor will pick up A's changes.
B's sync service sees: "file_abc was modified" → downloads new version
```

**Trade-off:** This multiplies change_log writes by the number of collaborators. For a folder shared with 100 users, one file modification = 101 change_log entries (owner + 100 grantees). At scale, Cassandra's linear write throughput handles this.

### 5.8 Stage 4 Trade-offs

| Decision | Gain | Cost |
|----------|------|------|
| Permission cache in Redis | Sub-ms auth checks on every request | Stale cache window after revocation |
| Owner-shard file placement | Clean folder listing | Cross-shard write for collaborators |
| Fanout change_log per share | Simple sync for collaborators | Write amplification proportional to share count |
| Soft expiry (check at read time) | No background job needed | Expired shares use DB resources until eviction |
| Public link as separate table | Clean separation of concerns | Two tables to check for access |

---

## 6. Observability and Traceability

### 6.1 The Three Pillars

**Metrics (Prometheus + Grafana):**

| Metric | Alert threshold | What it means |
|--------|----------------|---------------|
| `upload_success_rate` | < 99.5% for 5 min | Upload service degraded |
| `upload_latency_p99` | > 500ms for 5 min | SLO breach |
| `download_latency_p99` | > 200ms for 5 min | CDN or DB issue |
| `sync_propagation_lag_p99` | > 30s for 5 min | Change log consumer falling behind |
| `change_log_consumer_lag` | > 10,000 events | Kafka consumer falling behind |
| `chunk_store_ref_count_errors` | > 0 ever | DATA INTEGRITY ALERT — page immediately |
| `glacier_restore_queue_depth` | > 100 | Old version downloads backing up |

**Logs (structured JSON, Elasticsearch):**

Every log line must include:
```json
{
  "trace_id": "abc123",
  "span_id": "def456",
  "user_id": "user_pramod",
  "file_id": "file_xyz",
  "device_id": "device_abc",
  "service": "upload-service",
  "event": "chunk_upload_complete",
  "chunk_hash": "sha256_A",
  "duration_ms": 45,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

`trace_id` is the single most important field. A single upload operation spans: Upload Service → S3 → Kafka → Thumbnail Worker → Sync Service → SSE push. Without trace_id, debugging a failed upload is archaeology.

**Traces (OpenTelemetry → Jaeger):**

```
Trace: user uploads file.pdf (total: 312ms)
├── Upload Service: validate JWT + check quota (8ms)
├── Redis: cache lookup for file metadata (2ms)
├── DB: query chunk_store for dedup check (15ms)
├── AWS SDK: generate presigned URLs × 3 (6ms)
├── [client uploads to S3 — separate trace]
├── Upload Service: /complete handler (25ms)
│   ├── DB transaction: insert version_chunks etc. (18ms)
│   └── Kafka: publish upload-complete event (4ms)
└── Sync Service: SSE push to 2 devices (3ms)
```

### 6.2 SLI / SLO / SLA

```
SLI (what we measure):      p99 upload latency over 5-minute windows
SLO (internal target):      p99 < 500ms
SLA (external commitment):  99.9% of uploads complete within 1s
                            (weaker than SLO — gives engineering buffer)

Error budget: 99.9% availability = 43 minutes/month of allowed downtime
When error budget < 20% remaining → freeze feature deploys → focus on reliability
```

### 6.3 Key Failure Modes and Mitigations

| Failure | Symptom | Mitigation |
|---------|---------|------------|
| Change log consumer lag | Sync propagation > 30s | Alert at 10k events; scale consumer group; fallback to direct DB poll |
| Metadata DB shard down | Upload/download failures for ~33% of users | Patroni auto-failover; circuit breaker; graceful degradation |
| S3 regional outage | All chunk uploads fail | Multi-region S3 replication; Transfer Acceleration re-routes |
| ref_count corruption | Silent data loss (other users' files broken) | Soft delete; GC double-check; weekly reconciliation job |
| Kafka consumer crash | Thumbnails/virus scans stop | Consumer group re-assignment; dead-letter queue for failed events |
| Redis cache poisoned | Stale file metadata served | CDC invalidation; TTL as safety net (5 min max stale) |

---

## System Evolution Summary

```
Stage 1 (Core):
  files table → s3_key directly
  Simple upload → S3, sync via change_log
  Challenge: No history, no sharing, no dedup

Stage 2 (Versioning):
  files → file_versions → version_chunks
  Upload creates a new version, not an overwrite
  Restore = new version pointing to old chunks (zero-copy)
  Challenge: Metadata row explosion (shard hard)

Stage 3 (Deduplication):
  + chunk_store (global, sharded by hash)
  /init queries chunk_store only (all 4 other tables untouched)
  /complete: ON CONFLICT DO NOTHING + ref_count increment
  Challenge: ref_count correctness = most dangerous state in system

Stage 4 (Sharing):
  + file_shares + share_links + user_shared_items
  Auth check: owner OR valid share (cached in Redis)
  Sync: fanout change_log events to all grantees
  Challenge: cross-shard file placement for shared folders

Observability (throughout):
  trace_id on every log line from day one
  Metrics + alerts defined before launch
  ref_count integrity: the one alert that must never fire
```

---

## Quick Reference — All Tables

| Table | Shard key | Stage added | Primary use |
|-------|-----------|-------------|-------------|
| `users` | user_id | 1 | Identity, quota |
| `files` | owner_user_id | 1 | File identity, current version pointer |
| `file_chunks` (Stage 1 only) | file_id | 1 | Replaced by version_chunks in Stage 2 |
| `change_log` | user_id | 1 | Sync cursor, Cassandra |
| `device_cursors` | device_id | 1 | Per-device sync state |
| `file_versions` | owner_user_id | 2 | Version history |
| `version_chunks` | owner_user_id | 2 | Chunk sequence per version |
| `chunk_store` | chunk_hash | 3 | Global dedup registry, ref_count |
| `file_shares` | resource_id | 4 | User-to-user access grants |
| `share_links` | token | 4 | Public link tokens |
| `user_shared_items` | user_id | 4 | "Shared with me" view |

---

## Quick Reference — All API Endpoints

| Endpoint | Stage | Description |
|----------|-------|-------------|
| `POST /v1/files/init` | 1 | Begin upload, get presigned URLs |
| `POST /v1/files/{id}/complete` | 1 | Confirm upload, write metadata |
| `GET /v1/files/{id}` | 1 | Download current version |
| `DELETE /v1/files/{id}` | 1 | Soft delete |
| `GET /v1/sync/changes?cursor=N` | 1 | Long-poll sync |
| `GET /v1/files/{id}/versions` | 2 | List versions |
| `GET /v1/files/{id}?version=N` | 2 | Download specific version |
| `POST /v1/files/{id}/restore` | 2 | Restore to previous version |
| `POST /v1/shares` | 4 | Create share |
| `DELETE /v1/shares/{id}` | 4 | Revoke share |
| `GET /v1/shared-with-me` | 4 | List files shared with me |
| `GET /v1/public/{token}` | 4 | Resolve public link |
