# Dropbox System Design — Principal Engineer Interview Guide

> A complete, production-grade system design for a cloud file storage and synchronization service, with versioning, covering all technical decisions, trade-offs, and broader perspectives.

---

## Table of Contents

1. [Phase 0 — Clarifying Questions & Assumptions](#phase-0--clarifying-questions--assumptions)
2. [Phase 1 — Requirements](#phase-1--requirements)
3. [Phase 2 — Back of Envelope Calculations](#phase-2--back-of-envelope-calculations)
4. [Phase 3 — API Design](#phase-3--api-design)
5. [Phase 4 — Data Model](#phase-4--data-model)
6. [Phase 5 — Architecture (Simple → Production)](#phase-5--architecture-simple--production)
7. [Phase 6 — Upload Flow Deep Dive](#phase-6--upload-flow-deep-dive)
8. [Phase 7 — Sync & Versioning Deep Dive](#phase-7--sync--versioning-deep-dive)
9. [Phase 8 — Security Design](#phase-8--security-design)
10. [Phase 9 — Key Technical Concepts](#phase-9--key-technical-concepts)
11. [Phase 10 — Trade-offs Matrix](#phase-10--trade-offs-matrix)
12. [Phase 11 — Observability & Failure Modes](#phase-11--observability--failure-modes)
13. [Complete System Summary](#complete-system-summary)

---

## Phase 0 — Clarifying Questions & Assumptions

### Questions I would ask before designing anything

**Scale**
- How many total users and DAU?
- Maximum file size? Average file size?
- Total storage budget?

**Versioning**
- How many versions to keep per file?
- Version retention period?
- Can users restore to any version?

**Collaboration**
- Shared folders between users?
- Real-time co-editing (Google Docs style)?
- Public link sharing with expiry/password?

**Geo / Compliance**
- Global deployment or single region?
- Data residency requirements (GDPR, HIPAA)?

**Consistency**
- Strong or eventual consistency?
- Read-your-writes required?
- Conflict resolution strategy?

**Platform**
- Web + desktop + mobile?
- Offline support needed?
- Native sync client?

### Assumptions (stated explicitly before proceeding)

| Parameter | Assumption | Reasoning |
|-----------|-----------|-----------|
| Total users | 500 million | Dropbox-class scale |
| DAU | 100 million | 20% DAU/total ratio |
| Max file size | 5 GB | Covers most professional use cases |
| Average file size | 1 MB | Mix of docs, images, small media |
| Versions retained | 30 per file | Balance cost vs utility |
| Deployment | Global, multi-region | Latency requirements |
| Shared folders | Yes | Core collaboration feature |
| Real-time co-edit | **Out of scope** | Separate product (Google Docs) |
| Conflict model | Conflict copy (Dropbox approach) | Simple, no data loss |
| Consistency | Strong for metadata, eventual for blobs | Practical balance |
| Offline support | Yes | Core differentiator |
| Read-your-writes | Required | User sees their own upload immediately |

---

## Phase 1 — Requirements

### Functional Requirements

**Core (must have):**
- Upload, download, delete files and folders
- Cross-device synchronization — changes on one device appear on all others
- File versioning — keep last 30 versions, restore any version
- Shared folders — multiple users collaborate on the same folder
- Public link sharing with optional expiry and password
- Offline mode — work offline, sync when reconnected

**Out of scope (explicitly):**
- Real-time co-editing of document content
- Video streaming or media playback

### Non-Functional Requirements

| Property | Target | Reasoning |
|----------|--------|-----------|
| Availability | 99.99% (~52 min/year downtime) | Files are work-critical |
| Upload latency | p99 < 500ms for first chunk ACK | User feels responsive |
| Download latency | p99 < 200ms TTFB | CDN-served |
| Sync propagation | < 30 seconds end-to-end | Acceptable for async sync |
| Durability | 99.999999999% (11 nines) | Never lose a file |
| Consistency | Read-your-writes for uploader | User sees their upload immediately |
| Scale | 500M users, 100M DAU | Dropbox-class |

---

## Phase 2 — Back of Envelope Calculations

> Always do this before architecture. The numbers tell you which components will be under stress.

### Storage

```
500M users × 15 GB avg quota = 7.5 exabytes raw
With 3× replication              = 22.5 EB
Dedup saves ~30% in practice
Effective storage to manage      ≈ 16 EB
```

### Upload Throughput

```
100M DAU × 10% upload daily = 10M uploads/day
10M / 86,400 seconds         = 116 uploads/sec (avg)
Peak 5× factor               = ~580 uploads/sec
Avg 1 MB per upload          = 580 MB/s peak bandwidth
```

### Metadata DB Size

```
500M users × 1,000 files avg = 500 billion file records
Each file row ≈ 500 bytes    = 250 TB metadata
→ Sharding is mandatory from day one
```

### Versioning Storage (with dedup)

```
With chunk-level deduplication:
  - Versions share unchanged chunks
  - Average version delta ≈ 5% of file
  - 30 versions ≈ 2.5× storage overhead (not 30×)
  
Key insight: versioning is nearly free when chunks are deduped
```

### Sync Service QPS

```
100M DAU × 3 devices avg   = 300M active device connections
Long-poll every 30 seconds = 10M sync requests/sec
→ Sync service is the highest-QPS component in the system
```

### Read:Write Ratio

```
Downloads >> Uploads (estimated 10:1)
→ System is read-heavy
→ Optimize read path with CDN + aggressive caching
```

---

## Phase 3 — API Design

### File Upload (2-Phase Protocol)

```
POST /v1/files/init
Headers: Idempotency-Key: <uuid>
Body: {
  name: "report.pdf",
  size: 52428800,
  chunk_hashes: ["sha256_1", "sha256_2", ...],
  folder_id: "folder_xyz",
  mime_type: "application/pdf"
}

Response 200:
{
  file_id: "file_abc",
  upload_id: "upload_xyz",
  missing_chunks: ["sha256_3", "sha256_7"],   ← only chunks server doesn't have
  presigned_urls: {
    "sha256_3": "https://s3.amazonaws.com/...",
    "sha256_7": "https://s3.amazonaws.com/..."
  }
}
```

**Why 2-phase?** Server tells client which chunks are already stored (deduplication). Client uploads only the missing ones directly to S3 via presigned URL. Saves massive bandwidth when uploading common file types.

```
POST /v1/files/{file_id}/complete
Body: { upload_id: "upload_xyz", chunk_ids: [...] }
Response: { file_id, version_id, version_number: 5 }
```

### File Download

```
GET /v1/files/{file_id}?version=3
Response: {
  chunk_list: [{ sequence: 0, chunk_hash, download_url }, ...],
  metadata: { name, size, created_at, mime_type }
}
```

### Versioning

```
GET  /v1/files/{file_id}/versions
Response: [{ version_id, version_number, size, created_at, device_id }, ...]

POST /v1/files/{file_id}/restore
Body: { version_id: "v5_abc" }
Response: { new_version_id, version_number: 31 }
→ Creates version 31 as a copy of version 5. Zero bytes copied in S3.
```

### Sync (Most Critical API)

```
GET /v1/sync/changes?cursor=4821&device_id=device_abc

Behavior:
  - If changes exist: return immediately with events
  - If no changes: hold connection open for 30s (long-poll), then return empty

Response: {
  events: [
    { change_id: 4822, file_id, event_type: "modified", version_id, timestamp },
    { change_id: 4823, file_id, event_type: "deleted", timestamp }
  ],
  next_cursor: 4823
}
```

**Cursor semantics:** Each device stores its `last_seen_change_id`. On reconnect after days offline, it simply requests `?cursor=<last_seen>` and receives all missed events in order. No events are ever lost.

### Sharing

```
POST /v1/shares
Body: {
  resource_id: "folder_xyz",
  resource_type: "folder",
  permission: "edit",          ← view | edit | owner
  expiry: "2025-12-31T00:00:00Z",
  password: "optional_password"
}
Response: { share_token: "tok_abc", share_url: "https://drop.box/s/tok_abc" }

GET /v1/shares/{token}
→ Validates token, checks expiry+password → redirects to CDN presigned download URL
```

### Cross-Cutting Concerns

- All mutating endpoints require `Idempotency-Key` header
- Safe to retry on network failure without duplicate side effects
- JWT auth on all endpoints, short-lived (15 min) + refresh token rotation
- Rate limiting enforced at API Gateway (token bucket, per user)

---

## Phase 4 — Data Model

### Core Tables

#### `users`
```sql
user_id       UUID PRIMARY KEY        ← shard key for metadata DB
email         VARCHAR UNIQUE NOT NULL
name          VARCHAR
quota_bytes   BIGINT DEFAULT 15GB
used_bytes    BIGINT DEFAULT 0
created_at    TIMESTAMP
```

#### `files`
```sql
file_id           UUID PRIMARY KEY
owner_user_id     UUID NOT NULL        ← shard key, FK to users
folder_id         UUID
name              VARCHAR NOT NULL
mime_type         VARCHAR
current_version_id UUID               ← FK to file_versions
is_deleted        BOOLEAN DEFAULT FALSE
created_at        TIMESTAMP
updated_at        TIMESTAMP
```

#### `file_versions`
```sql
version_id         UUID PRIMARY KEY
file_id            UUID NOT NULL        ← FK to files
version_number     INTEGER NOT NULL
size_bytes         BIGINT NOT NULL
created_by_device_id UUID
created_at         TIMESTAMP

INDEX: (file_id, version_number) for listing versions
```

#### `version_chunks` (join table — the heart of deduplication)
```sql
version_id     UUID NOT NULL    ← FK to file_versions
sequence_num   INTEGER NOT NULL  ← ordering of chunks within version
chunk_hash     VARCHAR NOT NULL  ← FK to chunk_store

PRIMARY KEY (version_id, sequence_num)
INDEX: (chunk_hash) for ref_count lookups
```

#### `chunk_store` (dedup table)
```sql
chunk_hash     VARCHAR PRIMARY KEY  ← SHA-256 of chunk bytes
s3_bucket      VARCHAR NOT NULL
s3_key         VARCHAR NOT NULL
size_bytes     INTEGER NOT NULL
ref_count      INTEGER DEFAULT 0    ← # of version_chunks rows pointing here
storage_class  ENUM(STANDARD, IA, GLACIER)
created_at     TIMESTAMP
```

#### `change_log` (append-only — Cassandra/DynamoDB)
```
change_id    BIGINT (monotonic sequence)    ← partition by (user_id, month)
user_id      UUID
file_id      UUID
event_type   ENUM(created, modified, deleted, restored, moved)
version_id   UUID
device_id    UUID
created_at   TIMESTAMP

Partition key: (user_id, month_bucket)
Clustering key: change_id ASC
```

### Key Design Insights

**Deduplication via `chunk_store`:**
The `chunk_hash` is the primary key. If user A and user B both upload the same PDF, `chunk_store` has one row. `version_chunks` for both files point to the same row. One physical blob in S3. `ref_count` tracks when it is safe to delete.

**Versioning is zero-copy:**
Creating version 5 of a 1 GB file where only 4 KB changed means: `file_versions` gets one new row + `version_chunks` gets 250 rows (most pointing to existing chunks) + 1 new `chunk_store` row for the changed chunk. S3 stores only 4 KB of new data. Storage cost of a version ≈ cost of its delta only.

**Change log is separate from Metadata DB:**
Append-only access pattern → Cassandra or DynamoDB, not PostgreSQL. Partitioned by `(user_id, month_bucket)` so all of a user's recent events live on one partition. Never updated, only appended. Retained for 30 days then purged (old versions still queryable via `file_versions`).

---

## Phase 5 — Architecture (Simple → Production)

### Step 1: Naive Baseline

```
Client → API Server → PostgreSQL
              ↓
           S3 Blob Storage
```

**Problems this cannot handle:**
- Single API server = SPOF + throughput ceiling
- Single DB = 250 TB won't fit on one machine
- File bytes flowing through API server = 580 MB/s through one box
- No async processing = upload blocks thumbnail generation, virus scan, notification
- No CDN = 300ms+ global latency
- No sync mechanism

### Step 2: Production Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                 │
│              Desktop │ Mobile │ Web Browser                     │
└──────────────┬────────────────────────────────┬─────────────────┘
               │ uploads/metadata               │ downloads
               ▼                                ▼
┌──────────────────────────┐    ┌───────────────────────────────┐
│ API Gateway + Load       │    │ CDN (CloudFront / Akamai)     │
│ Balancer                 │    │ Caches thumbnails + shared     │
│ (rate limit, auth, SSL)  │    │ files at edge globally        │
└──────┬───────────────────┘    └───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│                    MICROSERVICES                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Upload       │  │ Metadata     │  │ Sync                 │  │
│  │ Service      │  │ Service      │  │ Service              │  │
│  │              │  │              │  │ (SSE long-poll)      │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────────────────┘  │
└─────────┼─────────────────┼───────────────────────────────────┘
          │                 │ reads/writes
          │                 ▼
          │  ┌──────────────────────────────────────────────────┐
          │  │  Redis Cluster                                   │
          │  │  - File metadata cache (TTL 5 min)              │
          │  │  - ACL / permission cache                       │
          │  │  - Session store                                │
          │  │  - Rate limit counters                          │
          │  └──────────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────────────┐
│  Kafka (Message Bus)                                             │
│  Topics: upload-events │ sync-events │ version-events           │
└──────────────────────────────────────────────────────────────────┘
          │
          ├──► Thumbnail generation worker
          ├──► Virus scan worker (ClamAV / commercial)
          └──► Notification worker (push/email)

┌──────────────────────────────────────────────────────────────────┐
│  Metadata DB — PostgreSQL, sharded by user_id                   │
│  Shard 0: user_id 0..N/3  │  Shard 1  │  Shard 2               │
│  Each shard: 1 primary + 2 read replicas                        │
│  Consistent hashing for future shard expansion                  │
└─────────────────────────┬────────────────────────────────────────┘
                          │ CDC (Debezium tails WAL)
                          ▼
                    Redis invalidation
                    + Search index update (Elasticsearch)

┌──────────────────────────────────────────────────────────────────┐
│  S3 Blob Storage (multi-region)                                  │
│  Standard → S3-IA (30d inactive) → Glacier (90d inactive)       │
│  ← Clients upload directly via presigned URLs (bypass servers)  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  Change Log DB — Cassandra / DynamoDB                           │
│  Append-only │ Partition by (user_id, month) │ Sync service     │
│  reads this to answer: "what changed since cursor X?"          │
└──────────────────────────────────────────────────────────────────┘
```

### Why Each Layer Was Added

| Layer | NFR it addresses |
|-------|-----------------|
| API Gateway | Rate limiting, auth, SSL termination, single entry point |
| CDN | Download latency (p99 < 200ms globally), caching shared files |
| Microservices | Independent scaling per bottleneck metric |
| Redis | Metadata read performance (10M sync req/sec can't all hit DB) |
| Kafka | Upload latency (don't block user for virus scan / thumbnail) |
| DB Sharding | 250 TB metadata won't fit single node |
| Presigned URLs | 580 MB/s upload bandwidth can't flow through app servers |
| Cassandra change log | Linear append scale, cursor-based sync for 300M devices |
| CDC (Debezium) | Crash-safe cache invalidation (DB = single source of truth) |
| Storage tiering | Cost optimization (old chunks shouldn't cost S3 Standard rates) |

---

## Phase 6 — Upload Flow Deep Dive

### Step-by-Step with Justification

**Step 1 — Client chunks file + computes hashes (local, no network)**
- Rolling hash (Rabin fingerprinting) → content-defined 4 MB chunks
- SHA-256 per chunk computed locally
- Content-defined chunking means insertion of bytes only affects one chunk, not all downstream chunks
- *Why this matters:* Edit 4 KB in a 1 GB file → re-upload 4 KB, not 1 GB

**Step 2 — POST /files/init**
- Server checks `chunk_store`: which hashes already exist?
- Returns only `missing_chunks[]` and presigned URLs for those chunks
- *Deduplication:* If another user already uploaded the same PDF, 0 bytes transferred
- *Idempotency:* Client retries on timeout → server returns same `upload_id`. No duplicate.

**Step 3 — Client uploads only missing chunks directly to S3**
- PUT requests to presigned URLs (bypass app servers entirely)
- Chunks uploaded in parallel (saturate available bandwidth)
- *Resumable:* Network drops → retry only the failed chunk, not the whole file
- *Rate limiting:* S3 bucket policy enforces max upload size per presigned URL

**Step 4 — POST /files/complete**
- Atomically: write `file_versions` row, write `version_chunks` rows, update `files.current_version_id`, append `change_log` row
- *Saga pattern:* If metadata write fails after S3 write, chunks remain in S3. GC job runs hourly to find orphaned chunks (in S3 but no `chunk_store` row → safe to delete)

**Step 5 — Kafka event published**
- `upload-complete` event published to Kafka
- Thumbnail generation worker consumes → generates preview, stores in S3, updates metadata
- Virus scan worker consumes → scans asynchronously, quarantines if flagged
- Notification worker consumes → pushes to other user's devices
- *Why Kafka:* Upload returns 200 in ~200ms. Virus scan takes 5s. These are decoupled.

**Step 6 — Sync service notifies other devices**
- CDC detects `change_log` insert → invalidates Redis metadata cache
- Sync service has SSE connections open to all user's devices
- Pushes change event to all active connections immediately
- Offline devices will receive it when they next poll with their cursor

### Version Creation is Free (Copy-on-Write)

When step 4 creates version N, it inserts:
- 1 row in `file_versions`
- K rows in `version_chunks` (most pointing to existing `chunk_store` rows)
- Only NEW chunks (the delta) get new `chunk_store` rows

**Cost of creating version N = cost of storing only the changed chunks.** Unchanged chunks are referenced, not copied. For a 1 GB file with 4 KB of changes: version creation costs 4 KB in S3, not 1 GB.

---

## Phase 7 — Sync & Versioning Deep Dive

### Sync Protocol: Cursor-Based Change Log

```
Each device stores: last_seen_change_id (persisted locally)

On start / reconnect:
  GET /sync/changes?cursor=4821&device_id=device_abc

Server behavior:
  1. Query change_log WHERE user_id = X AND change_id > 4821
  2. If results exist → return immediately
  3. If no results → hold connection open 30s (long-poll) then return empty

Client behavior:
  1. Apply each event in order (create/modify/delete local file)
  2. Advance cursor to max(change_id) in response
  3. Immediately re-poll
```

**Key properties:**
- No events are ever lost. Cursor advances only on successful ACK.
- Offline device reconnects after 3 days → gets all missed events since cursor. Works perfectly.
- Change log retained 30 days → covers all practical offline durations
- Cassandra partition key `(user_id, month)` means all of a user's recent events are on one node

### Conflict Resolution: Conflict Copy Model

When two devices edit the same file offline and both sync:

```
Device A (syncs first):
  Uploads v2 → accepted → file is now at v2

Device B (syncs second):
  Server sees: base_version=v1, but server is at v2 → conflict
  Device B's version saved as: "report (Pramod's conflicted copy 2024-01-15).pdf"
  Both v2 and conflict copy exist. User resolves manually.
```

**Why this model:** No data loss ever. Simple to implement. The alternative (CRDT merge) is complex and works for text but is meaningless for binary files (images, PDFs, spreadsheets).

### Version Limit Enforcement

When version 31 is created (limit is 30):
1. Find the oldest version row for this file
2. For each chunk in that version: decrement `ref_count` in `chunk_store`
3. Any `chunk_store` row where `ref_count` reaches 0 → marked `pending_gc`
4. GC job (async, hourly) deletes the S3 object and removes the `chunk_store` row
5. Delete the `file_versions` row

**Critical:** `ref_count` must be decremented correctly. A chunk shared by versions 1 and 5 should only be deleted when BOTH versions are pruned. `ref_count` ensures this.

### Version Restore: O(1) Metadata Operation

```
User: "Restore to version 5"

1. Read version_chunks WHERE version_id = version_5_id
   → Get list of {sequence_num, chunk_hash}
   
2. INSERT INTO file_versions (version_id=NEW, file_id, version_number=31)

3. INSERT INTO version_chunks (version_id=NEW, ...) 
   pointing to the same chunk_hashes as version 5
   → No S3 copy. ref_counts incremented.

4. UPDATE files SET current_version_id = NEW

5. Append change_log event

Result: Instant. File at version 5 content. All chunks already in S3.
```

### Storage Tiering for Old Version Chunks

```
Chunk last accessed < 30 days ago  → S3 Standard      (~$0.023/GB/month)
Chunk last accessed 30–90 days    → S3-IA             (~$0.0125/GB/month)
Chunk last accessed > 90 days     → Glacier           (~$0.004/GB/month)

S3 Lifecycle Policy automates transitions.
For Glacier: proactive warming needed if version restore requested
→ Check chunk storage class before generating download URL
→ If Glacier: initiate restore (3–12 hr), notify user when ready
```

---

## Phase 8 — Security Design

### Defense in Depth

**1. Encryption in Transit**
- TLS 1.3 on all connections: Client → API Gateway, Client → S3 presigned, Service ↔ Service (mTLS)
- HSTS enforced on all web endpoints
- Certificate pinning on mobile clients

**2. Encryption at Rest (Envelope Encryption)**
```
File DEK (Data Encryption Key)
  ← unique per file
  ← encrypts the chunk bytes in S3 (AES-256-GCM)
  ← stored encrypted by the KEK

File KEK (Key Encryption Key)
  ← stored in AWS KMS
  ← never leaves KMS in plaintext
  ← rotated every 90 days automatically

Crypto-erasure on delete:
  When user deletes a file → delete the DEK from the metadata DB
  Chunks remain in S3 but are now unreadable (key is gone)
  GDPR right-to-erasure: instant, no need to wait for GC
```

**3. Authorization**
- JWT tokens (15-minute expiry) + refresh token rotation
- Per-file ACL stored in metadata DB, cached in Redis
- Shared folder roles: `viewer` / `editor` / `owner`
- Presigned URLs scoped to specific S3 key, 15-minute validity, one PUT operation only

**4. Rate Limiting**
- Token bucket algorithm per user at API Gateway
- S3 bucket policy: max upload size enforced per presigned URL
- DDoS mitigation via CloudFront WAF

**5. Virus Scanning**
- Async post-upload (does not block upload response)
- ClamAV + commercial AV engine
- File quarantined if flagged → user notified → download blocked until cleared
- Scan result stored in metadata DB

---

## Phase 9 — Key Technical Concepts

### Content-Defined Chunking (Rolling Hash)

**Problem with fixed chunking:** Insert 10 bytes in the middle of a 1 GB file → all downstream chunk boundaries shift → system sees the entire file as changed → re-uploads 1 GB.

**Rolling hash solution:**
```
hash = polynomial of bytes in sliding window
Cut point: rolling_hash(window) mod 4096 == 0

This fires ~once per 4096 bytes → average chunk ≈ 4 KB
The boundary is determined by content, not position
```

When bytes are inserted, the window keeps sliding until the same content pattern reappears downstream, re-establishing the same cut point. Chunks after the insertion region are content-identical to before → same hash → reused, not re-uploaded.

**Tuning the divisor:**
- `mod 256` → avg 256-byte chunks (max granularity, high metadata overhead)
- `mod 4096` → avg 4 KB chunks (general purpose, sweet spot)
- `mod 65536` → avg 64 KB chunks (coarse, lower overhead, less dedup)

**Implementation detail:** Since 4096 = 2¹², checking `hash mod 4096 == 0` is a single bitwise AND: `hash & 0xFFF == 0`. One CPU instruction per byte.

### Deduplication via SHA-256

After content-defined chunking, each chunk is SHA-256 hashed. The hash is the primary key of `chunk_store`. Before uploading, the client sends all chunk hashes to the server. Server responds with the subset it doesn't have. Client uploads only missing chunks.

**Cross-user deduplication:** Two users uploading the same ISO image → second user uploads 0 bytes. Both `version_chunks` rows point to the same `chunk_store` entry.

### Presigned URLs

The upload service generates a time-limited, cryptographically signed URL that authorizes the client to write exactly one object to exactly one S3 key, without any AWS credentials.

**Benefits:** App servers never handle file bytes → no bandwidth bottleneck → infinite upload scalability. S3 handles the storage directly.

**Risk:** Centralized rate control is lost. Mitigate with S3 bucket policies (max upload size per presigned URL) and per-user request limits at the API Gateway.

### Sharding Strategy

**Shard by `user_id`:** All of a user's files live on one shard. Sync queries (`give me all files changed for user X`) are single-shard with no cross-shard joins.

**Consistent hashing:** Adding a new shard remaps only 1/N of keys (where N = new shard count). Minimizes data migration.

**Hot shard problem:** A power user with 10M files saturates one shard. Mitigation: secondary shard key `(user_id, date_bucket)` for extreme cases, or manual rebalancing.

**Shared folders:** User A shares a folder with User B. A's metadata is on Shard 0, B's on Shard 2. Solution: ownership model — the folder owner's shard holds the authoritative metadata. B's shard holds an indirection record pointing to A's shard. Cross-shard read on access; write always goes to owner's shard.

### CDC-Based Cache Invalidation

**Problem with write-through invalidation:** App server writes to DB, then invalidates Redis. If app server crashes between these two steps → stale cache permanently. Race condition under high concurrency.

**CDC solution:** Debezium tails the PostgreSQL WAL (write-ahead log). Every committed DB change generates an event on Kafka. A consumer reads these events and invalidates Redis. The DB commit is the only operation the app server performs. Cache invalidation is guaranteed to happen eventually, even through app server crashes.

**Trade-off:** 10–500ms window where Redis is stale (CDC lag). Acceptable for file metadata.

### Replication and Quorum

Metadata DB: synchronous replication to one replica within the same AZ (W=2). Reads served from primary by default (read-your-writes), from replica for non-critical reads.

Change log (Cassandra): W=2, R=2, N=3 (quorum). Guarantees last write is always visible on read. `W + R > N` is the fundamental quorum condition.

---

## Phase 10 — Trade-offs Matrix

| Decision | You gain | You give up / risk |
|----------|----------|-------------------|
| Content-defined chunking | Insertion-resilient dedup; bandwidth savings on edit | Variable chunk sizes complicate parallelism and progress tracking |
| Presigned URLs (direct to S3) | App servers off critical path; infinite upload scalability | Lose centralized throttling; must enforce limits in S3 policy |
| Eventual consistency for sync | High availability; works offline-first | Window where devices see stale state (~30s lag) |
| Cassandra for change log | Unlimited append scale; linear write throughput | No cross-user joins; eventual consistency within cluster |
| Conflict copy model | Simple to implement; no data loss ever | User must manually merge; pollutes folder on bad connectivity |
| CDC for cache invalidation | Crash-safe invalidation; DB is single source of truth | 10–500ms stale window; Debezium infrastructure to maintain |
| Shard by user_id | All user's files co-located; no cross-shard sync queries | Hot shard risk for power users; shared folders need indirection layer |
| SSE instead of WebSocket | Simpler infrastructure; stateless; native reconnect | Server-to-client only; client actions must use separate REST calls |
| Conflict copy (vs CRDT) | Zero implementation complexity; works for binary files | User-visible conflict files; requires manual resolution |
| Async virus scan (vs sync) | Upload returns in 200ms | Brief window where unscanned file is accessible |
| Async thumbnail (vs sync) | Upload returns in 200ms | Thumbnail not immediately available after upload |
| Envelope encryption (DEK+KEK) | Instant crypto-erasure; GDPR compliant; key rotation | Key management complexity; KMS latency on each file access |

### The Trade-off Meta-Principle

Every decision is a dial between: **consistency ↔ availability**, **simplicity ↔ capability**, **performance ↔ cost**. Name which dial you are turning and why before making the choice. Never pick a technology blindly.

---

## Phase 11 — Observability & Failure Modes

### Three Pillars of Observability

**Metrics (Prometheus + Grafana):**
- Upload success rate (SLI: target 99.9%)
- p99 chunk upload latency
- Sync propagation lag (change_log write → SSE delivery to device)
- Change log consumer lag on Kafka
- `ref_count` GC queue depth
- Cache hit rate (Redis)

**Logs (Structured JSON, shipped to Elasticsearch):**
Every log line must include: `trace_id`, `user_id`, `file_id`, `chunk_id`, `device_id`. This allows correlating across services for any single operation.

**Traces (OpenTelemetry → Jaeger):**
Distributed trace across: API Gateway → Upload Service → S3 presign → Kafka publish → Worker. The S3 upload span (client-direct) is the longest — typically 70–80% of total upload time.

### SLI / SLO / SLA

| Term | Definition | Example |
|------|-----------|---------|
| SLI | Actual measurement | p99 upload latency = 320ms |
| SLO | Internal target | p99 upload < 500ms |
| SLA | External contract | 99.9% availability or refund |

### Top Failure Modes

**1. Change log consumer falls behind**
- *Symptom:* Sync propagation lag spikes from <5s to >60s
- *Alert at:* 10,000 unprocessed Kafka events
- *Mitigation:* Horizontal scale the consumer group; fallback: devices poll metadata DB directly if SSE times out after 60s

**2. Metadata DB shard goes down**
- *Symptom:* Upload and download failures for users whose `user_id` routes to that shard
- *Mitigation:* Synchronous replication to replica within same AZ; automatic failover via Patroni (PostgreSQL HA); circuit breaker on all DB call sites to prevent cascade

**3. S3 regional outage**
- *Symptom:* All uploads fail
- *Mitigation:* Multi-region S3 replication; route presigned URL generation to secondary region on primary degradation

**4. CDC falls behind (Debezium lag)**
- *Symptom:* Stale cache → users see outdated file lists
- *Mitigation:* Alert on WAL replication lag > 1000 events; TTL-based cache expiry as safety net (metadata expires after 15 min regardless)

### System Evolution Path

```
Phase 1 (0–1M users):
  Single PostgreSQL + single Redis + basic S3 upload
  No sharding, no CDN, manual operations

Phase 2 (1M–10M users):
  Add CDN for downloads
  Add Redis cluster for metadata caching
  Split upload service from metadata service

Phase 3 (10M–100M users):
  Shard PostgreSQL by user_id (start with 3 shards)
  Add Kafka for async processing
  Add Cassandra for change log (replace polling with SSE)
  Add consistent hashing for future shard expansion

Phase 4 (100M–500M users):
  Multi-region active-passive (then active-active for reads)
  Storage tiering (S3-IA + Glacier)
  ML-based storage pre-warming for frequent version restores
  Per-tenant encryption key management
```

**What NOT to build on day one:** Real-time co-editing, ML-based storage pre-warming, per-chunk encryption (file-level is sufficient), multi-region active-active writes. Each is a multiplier on complexity. Add only when a specific user need or regulatory requirement demands it.

---

## Complete System Summary

| Layer | Choice | Key Reason |
|-------|--------|-----------|
| File blob storage | S3 multi-region | 11-nines durability, infinite scale |
| Metadata DB | PostgreSQL, sharded by `user_id` | Strong consistency; co-locate user's files |
| Change log | Cassandra, partitioned by `(user_id, month)` | Append-only, linear write scale |
| Cache | Redis cluster | Sub-ms metadata reads; session storage |
| Sync protocol | SSE long-poll with cursor | Server-push only; stateless; resumable |
| Upload protocol | Content-defined chunks + presigned URL + delta | Resumable; dedup; no server bandwidth |
| Versioning | Copy-on-write chunks with `ref_count` | Version = metadata row, not S3 copy |
| Conflict resolution | Conflict copy | Simplest safe strategy for binary files |
| Async processing | Kafka + worker pool | Decouple upload latency from side effects |
| Cache invalidation | CDC via Debezium | Crash-safe; DB is the only authority |
| Security | Envelope encryption + crypto-erasure | GDPR compliant; instant key revocation |
| Observability | OpenTelemetry + Jaeger + Prometheus | `trace_id` on every log line |
| Message format | Idempotent operations + idempotency keys | Safe retries across all distributed writes |
| Rate limiting | Token bucket at API Gateway + S3 policy | Two layers; neither alone is sufficient |

---

*Document covers: Dropbox-class cloud storage with versioning, designed for 500M users at principal engineer depth. All major decisions include explicit trade-offs and alternatives considered.*
