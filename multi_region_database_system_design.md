# Multi-Region Database Deployment — Principal-Level System Design

> **Topic:** Multi-Region Database Deployment  
> **Level Target:** Principal / Staff Engineer (E6/E7)  
> **Style:** Narrative — Start Simple → Discover Pain → Evolve → Arrive at Depth

---

## SECTION 1: UNDERSTANDING THE PROBLEM

### What Is Multi-Region Database Deployment?

A multi-region database deployment is the practice of replicating, partitioning, or distributing a database's data and serving infrastructure across two or more geographically distinct data centers or cloud regions. The problem it solves is fundamental: a single-region database is a single point of failure — a regional power outage, a cloud provider incident, or a network partition between continents means your entire product goes dark. Beyond resilience, geography matters for latency: a user in Mumbai querying a database hosted exclusively in us-east-1 experiences 150–250ms of round-trip overhead on every single request, before your application logic even runs. Multi-region deployment solves both: it keeps data physically close to users (read latency), and it keeps the system alive when one region dies (availability). The users are any engineering organization whose product has global reach, strict SLAs, or data residency requirements — from fintech platforms to SaaS companies to social networks.

---

### Clarifying Questions

Before drawing a single box, I would ask the interviewer these questions:

**Q1: Is this read-heavy or write-heavy, and what is the read/write ratio?**  
*Why it matters:* A 99:1 read/write ratio means I can deploy read replicas across all regions and only route writes to a single primary — this is dramatically simpler than multi-master. A 50:50 write ratio means I need active-active, which forces me to reason deeply about conflict resolution, which is one of the hardest problems in distributed systems.

**Q2: Does the product have data residency or data sovereignty requirements (GDPR, India's PDPB, China's PIPL)?**  
*Why it matters:* If European user data must never leave the EU, then a simple "replicate everything everywhere" strategy is illegal. I need region-pinned sharding — users are mapped to a home region, and their data never replicates outside it. This turns a technical problem into a compliance architecture problem.

**Q3: What is the acceptable RPO (Recovery Point Objective) and RTO (Recovery Time Objective)?**  
*Why it matters:* RPO = how much data can we lose? RTO = how fast must we recover? RPO=0 forces synchronous replication across regions, which adds 100–300ms to every write. RPO=30s allows async replication, which is much faster but means we accept up to 30s of data loss on a regional failure. This single question determines 70% of the architecture.

**Q4: Do we need global transactions — writes that atomically span multiple regions?**  
*Why it matters:* Global transactions (e.g., Google Spanner's TrueTime-based 2PC) are possible but extremely expensive in latency and operational complexity. Most systems avoid them by designing their data model to avoid cross-region writes. Knowing whether we need them tells me whether to reach for CockroachDB/Spanner or whether standard async replication is sufficient.

**Q5: What is the user distribution — what percentage of users are in each region?**  
*Why it matters:* If 90% of users are in APAC and 10% are in US, I should put the primary write region in APAC and have the US as a read replica, not the other way around. The "obvious" default of us-east-1 as primary is wrong if your users aren't there.

**Q6: Is the application already region-aware, or does the app tier need to be rearchitected alongside the DB?**  
*Why it matters:* Routing reads to the nearest region requires the application layer to know which region it's in and to connect to the right replica. If the app tier is monolithic and uses a single hardcoded connection string, I need to include app-tier changes in my design scope.

**Q7: What database engine are we starting from — relational (PostgreSQL/MySQL), document (MongoDB), or are we open to purpose-built globally distributed databases?**  
*Why it matters:* PostgreSQL has excellent streaming replication but no native multi-master. MongoDB has built-in global clusters. CockroachDB and Google Spanner are designed from scratch for global distribution. The starting point constrains the solution space significantly.

---

### Functional Requirements

**P0 — Core Requirements:**

1. **Read locality:** Users in any region must read from a local replica. Cross-region reads are not acceptable for normal traffic.
2. **Write durability:** A committed write must survive a full regional failure without data loss (RPO = 0 for critical data, RPO ≤ 30s for eventual-consistency paths).
3. **Automatic failover:** When a region fails, traffic must be rerouted to a healthy region within the RTO window (target: < 60 seconds) without manual intervention.
4. **Schema and operational consistency:** DDL changes (schema migrations) must be applied consistently across regions without causing split-brain or downtime.
5. **Data residency enforcement:** User data must be constrained to its designated geographic region where compliance requires it.

**Out of Scope (with rationale):**

- **Cross-region distributed transactions (XA/2PC spanning regions):** The latency cost (2–3× round-trip time across regions) and failure mode complexity exceed what most systems justify. We design the data model to avoid this need.
- **Real-time analytics queries against the production replica set:** Analytics should be served from a dedicated OLAP layer (Snowflake, BigQuery, Redshift) fed via CDC. Running analytics queries against production replicas risks replication lag and resource contention.
- **Application tier architecture:** We assume the app tier can be made region-aware. The full app redesign is a separate workstream.

---

### Non-Functional Requirements

**Availability: 99.99%**  
99.99% = ~52 minutes of allowed downtime per year. This means a single-region deployment is immediately ruled out — any regional cloud provider incident would consume our entire annual budget in one event. We need active replicas in at minimum two regions so that a regional failure causes a failover, not an outage. At 99.99%, we also cannot tolerate manual failover — the human reaction time alone (5–15 minutes) would breach the SLO.

**Consistency: We choose CP with graceful AP degradation**  
For write operations, we choose consistency over availability — a user losing a confirmed write is catastrophic (financial record disappears, order is lost). For read operations, we accept stale reads up to our replication lag SLO (target: p99 < 500ms replication lag). This is a deliberate CAP position: we never sacrifice write durability, but we tolerate eventual consistency on reads. Concretely: a user in Tokyo reading a record written 200ms ago in São Paulo might see a slightly stale value — that is acceptable. A write that is acknowledged but not durable — never acceptable.

**Latency:**  
- Read p50: < 10ms (local replica, in-region)  
- Read p99: < 50ms (local replica, in-region; includes occasional cross-region fallback)  
- Write p50: < 50ms (synchronous replication to quorum)  
- Write p99: < 200ms (includes cross-region replication acknowledgment)  
Why? Our SLA to customers is 200ms end-to-end for write operations. The DB write must complete well within that budget, leaving headroom for network and application processing.

**Throughput:**  
- Reads: 500K reads/sec at peak (global, distributed across regions)  
- Writes: 50K writes/sec at peak  
Math: 100M DAU × 50 reads/day ÷ 86,400s = ~58K reads/sec average. Peak factor of 3× = ~175K reads/sec per region for a 3-region deployment = 525K global. Writes are 10% of reads = 50K/sec at peak.

**Durability: RPO = 0 for financial/transactional data; RPO ≤ 30s for non-critical metadata**  
Data loss is never acceptable for anything that touches money, identity, or audit trails. For user preferences, UI state, or non-critical cache-backing data, we can accept up to 30 seconds of loss. This two-tier RPO allows us to use synchronous replication selectively and async replication where it suffices.

**Scalability:**  
At 10× current scale (1B DAU), the first thing that breaks is the write primary — a single primary handling 500K writes/sec is beyond what most RDBMS can sustain vertically. The design must anticipate horizontal write sharding by user region or entity ID. We'll design the schema with this in mind from Day 1.

**Security:**  
- Encryption in transit: TLS 1.3 mandatory on all inter-region replication channels.  
- Encryption at rest: AES-256 at the storage layer.  
- Replication user credentials: separate service account with minimal privileges (REPLICATION role only), rotated every 90 days.  
- Network: replication traffic travels over private VPC peering or a dedicated VPN/backbone — never the public internet.

**Observability SLOs (defined upfront — as important as latency SLOs):**  
- We must know within **30 seconds** if replication lag exceeds 1 second in any region.  
- We must know within **60 seconds** if any region's read error rate exceeds 1%.  
- We must know within **5 minutes** if write quorum drops below our minimum.  
- We must know within **24 hours** if storage cost per region is trending anomalously.

---

### Capacity Estimation

**Users and Request Volume:**

```
Daily Active Users (DAU):          100M
Average reads per user per day:    50
Average writes per user per day:   5
Read QPS (average):                100M × 50 / 86,400 = ~57,900 ≈ 58K reads/sec
Write QPS (average):               100M × 5 / 86,400  = ~5,800  ≈  6K writes/sec
Peak multiplier:                   3× (evening traffic spike)
Peak read QPS:                     174K reads/sec
Peak write QPS:                    18K writes/sec
```

Distributed across 3 regions (US, EU, APAC), assuming roughly equal split:
- Per-region reads: ~58K reads/sec at peak
- Per-region writes: ~6K writes/sec at peak (all eventually hitting the primary)

**Storage:**

```
Average record size:                2 KB
New records per day:                100M users × 5 writes = 500M records/day
Raw data ingress:                   500M × 2KB = 1TB/day
With 3× replication:                3TB/day replicated
5-year projection:                  3TB × 365 × 5 = ~5.5 PB
```

**Replication bandwidth:**

```
WAL (Write-Ahead Log) size ≈ 2-3× data size for PostgreSQL-style streaming:
Write throughput:     18K writes/sec × 2KB = ~36MB/sec raw
WAL bandwidth:        36MB/sec × 2.5 = ~90MB/sec
Cross-region replication (2 replica regions): 90MB/sec × 2 = 180MB/sec egress
Monthly egress cost at $0.09/GB: 180MB/sec × 2.6M seconds/month / 1024 ≈ 458 TB × $0.09 ≈ $41K/month
```

This egress cost is not trivial — it's a real budget line item that shapes decisions about replication topology and tiering.

**Dominant Constraint:** This system is **I/O-bound and network-bound**. The limiting factor is disk I/O on the write primary (WAL writes) and network bandwidth between regions (replication stream). CPU usage is modest compared to these two bottlenecks. Our scaling strategy is therefore: shard writes horizontally to distribute I/O, and minimize replication bandwidth by only replicating what's necessary (row filtering, logical replication slots).

---

## SECTION 2: THE SETUP

### Core Entities

**1. User**  
Represents an authenticated user of the platform. Primary access pattern: lookup by `user_id` (point read). Home region is an attribute — critical for data residency routing.

```
user_id        UUID (PK)
email          TEXT (unique)
home_region    ENUM('us-east', 'eu-west', 'ap-south')
created_at     TIMESTAMPTZ
version        INTEGER
```

**2. Record**  
The core data entity being stored and replicated. Access pattern: point read by `(record_id)`, range scan by `(user_id, created_at)`.

```
record_id      UUID (PK)
user_id        UUID (FK → users)
region_key     TEXT       -- sharding key, same as user's home_region
payload        JSONB
checksum       TEXT       -- for consistency verification post-replication
created_at     TIMESTAMPTZ
updated_at     TIMESTAMPTZ
version        INTEGER    -- optimistic locking
is_deleted     BOOLEAN DEFAULT FALSE
```

**3. ReplicationCheckpoint**  
Tracks the WAL position (or LSN in PostgreSQL terminology) each replica has consumed. Access pattern: point read by `(replica_id)`. Used for lag monitoring and failover decisions.

```
replica_id     TEXT (PK)   -- e.g., 'eu-west-1-replica-01'
region         TEXT
last_lsn       TEXT        -- Log Sequence Number
last_applied   TIMESTAMPTZ
lag_bytes      BIGINT
lag_seconds    INTEGER
```

**4. FailoverEvent**  
Audit log of every regional failover. Immutable append-only. Critical for post-incident analysis and RTO measurement.

```
event_id       UUID (PK)
region         TEXT
event_type     ENUM('primary_promoted', 'replica_degraded', 'failover_completed')
old_primary    TEXT
new_primary    TEXT
detected_at    TIMESTAMPTZ
resolved_at    TIMESTAMPTZ
rto_seconds    INTEGER     -- computed: resolved_at - detected_at
```

**5. SchemaVersion**  
Tracks the current schema migration version per region. Used to detect split-brain migrations and ensure all regions are in sync before a migration is declared complete.

```
version        INTEGER (PK)
region         TEXT
applied_at     TIMESTAMPTZ
migration_hash TEXT        -- SHA256 of migration SQL, for verification
status         ENUM('pending', 'applying', 'applied', 'failed')
```

---

### API Design

These are the APIs exposed by the **Database Access Layer** (DAL) — the service that abstracts multi-region routing from application services. Applications never connect directly to the DB; they go through the DAL.

**Read a Record (routed to nearest replica)**
```
GET /records/{record_id}
Headers: Authorization: Bearer <JWT>
         X-Region-Hint: ap-south   ← app tier tells DAL which region to prefer
Response: 200 { record_id, user_id, payload, updated_at, region_served_from }
          404 { error: "NOT_FOUND" }
          503 { error: "REPLICA_UNAVAILABLE", fallback_region: "us-east" }

Error case: If the local replica is behind by > 30s lag, the DAL can either:
  (a) Serve the stale read and annotate the response with X-Read-Staleness: <ms>
  (b) Fall back to the primary — but this adds 150–300ms cross-region latency
  Default: serve stale with annotation. Let the client decide if staleness is acceptable.
```

**Write a Record (always routed to primary)**
```
POST /records
Headers: Authorization: Bearer <JWT>
Body: { user_id, payload, idempotency_key }
Response: 201 { record_id, created_at, region_written_to }
          409 { error: "IDEMPOTENT_DUPLICATE", existing_record_id }
          503 { error: "PRIMARY_UNAVAILABLE", retry_after_ms: 5000 }

Note: idempotency_key is mandatory. At-least-once write delivery means clients
will retry. Without idempotency keys, retries create duplicate records.
```

**Promote Replica to Primary (internal, operations only)**
```
POST /admin/regions/{region}/promote
Headers: Authorization: Bearer <ops-token>
Body: { reason, force: boolean }
Response: 200 { new_primary, old_primary, promotion_started_at }
          409 { error: "REGION_NOT_CAUGHT_UP", lag_seconds: 45 }

Design note: This API returns 409 if the replica is >30s behind unless force=true.
Forcing promotion with lag is accepting data loss — it must be a conscious choice.
```

**Get Region Health**
```
GET /admin/regions/health
Response: 200 {
  regions: [
    { region: "us-east", role: "primary", lag_ms: 0, status: "healthy" },
    { region: "eu-west", role: "replica", lag_ms: 145, status: "healthy" },
    { region: "ap-south", role: "replica", lag_ms: 312, status: "degraded" }
  ]
}
```

**Apply Schema Migration (fan-out to all regions)**
```
POST /admin/migrations
Body: { version, migration_sql, checksum }
Response: 202 { migration_id, regions_targeted, estimated_completion_ms }

Note: Returns 202 (Accepted) not 200. Migrations are async fan-outs.
The caller polls /admin/migrations/{migration_id} to confirm all regions are applied.
```

---

## SECTION 3: HIGH-LEVEL DESIGN — EVOLVING CORE

### Requirement 1: Read Locality — Users read from a local replica

#### ❌ Naive Solution: Single Primary, App Connects Directly

**Approach:** One database in us-east-1. All regions connect to it. Simple connection string, no routing logic.

```
[User in Tokyo] ──── 150ms RTT ────► [DB Primary: us-east-1]
[User in London] ─── 80ms RTT ─────► [DB Primary: us-east-1]
[User in New York] ─ 5ms RTT ──────► [DB Primary: us-east-1]
```

**Why it breaks:** At 58K reads/sec globally, a single node can handle this, but not at p99 < 50ms. Tokyo users are paying 150ms of pure network overhead before query execution. At p99, query jitter adds another 50–100ms. Tokyo p99 = 300ms+, which violates our SLO. This breaks at any scale — it's not a scale problem, it's a physics problem.

---

#### ⚠️ Better Solution: Read Replicas Per Region, Manual Connection String

**Approach:** Deploy a read replica in each region. Configure application connection strings to point to the local replica for reads, and to the primary for writes.

```
[App: Tokyo]    ──► [Replica: ap-south]   ──WAL stream──► [Primary: us-east]
[App: London]   ──► [Replica: eu-west]    ──WAL stream──► [Primary: us-east]
[App: New York] ──► [Primary: us-east]    (reads + writes)
```

**New problem introduced:** The application code is now responsible for using the right connection. Developers forget. A single `db.query()` call that accidentally goes to the primary for a read causes cross-region latency. With 50 engineers on the team, you will have cross-region reads in production within a week. Also: replication lag is invisible to the application. If the Tokyo replica is 10 seconds behind, Tokyo users see stale data with no awareness.

---

#### ✅ Good Solution (Senior Bar): Database Access Layer (DAL) with Smart Routing

**Approach:** Introduce a **Database Access Layer** — a thin service that:
1. Accepts all DB requests from application services
2. Automatically routes reads to the nearest healthy replica
3. Routes all writes to the current primary
4. Exposes replication lag to callers via response headers
5. Falls back to the primary if local replica is unhealthy or excessively lagged

```
                    ┌─────────────────────────────────────────────┐
                    │          Global Load Balancer (GeoDNS)       │
                    └──────┬───────────────┬───────────────┬───────┘
                           │               │               │
                    ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
                    │  DAL: APAC  │ │  DAL: EU    │ │  DAL: US    │
                    └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
                           │               │               │
              ┌────────────▼──┐     ┌──────▼──────┐ ┌─────▼───────┐
              │Replica: APAC  │     │Replica: EU  │ │Primary: US  │
              │ (read-only)   │     │ (read-only) │ │(read+write) │
              └───────────────┘     └─────────────┘ └─────────────┘
                        ▲                  ▲                │
                        └──────────────────┴──── WAL stream ┘
```

DAL routing logic (pseudo-code):
```python
def route_query(query, region_hint, max_lag_ms=500):
    if query.is_write():
        return connections["primary"]
    
    local_replica = replicas[region_hint]
    if local_replica.is_healthy() and local_replica.lag_ms < max_lag_ms:
        return local_replica
    
    # Fallback: find the least-lagged healthy replica
    best = min(
        (r for r in replicas.values() if r.is_healthy()),
        key=lambda r: r.lag_ms
    )
    metrics.increment("dal.cross_region_fallback", region=region_hint)
    return best
```

**Schema for replica health tracking (in-memory, refreshed every 5s):**
```json
{
  "ap-south": { "lag_ms": 145, "lag_bytes": 2048, "is_healthy": true },
  "eu-west":  { "lag_ms":  89, "lag_bytes":  512, "is_healthy": true },
  "us-east":  { "lag_ms":   0, "lag_bytes":    0, "is_healthy": true, "is_primary": true }
}
```

**And here's how we'd know if this is broken in production:** The DAL emits `dal.cross_region_fallback` counter per region. If Tokyo reads are consistently falling back to the US primary, we have a replica health or lag problem. This counter is the canary. Target: < 0.1% of reads should fall back cross-region.

**Remaining gap a Principal would notice:** Read-your-own-writes (RYOW) semantics. If I write a record and immediately read it back from the replica, I might not see it — the replica hasn't caught up yet. Session consistency is not guaranteed. We'll solve this in a deep dive.

---

### Requirement 2: Write Durability — No data loss on regional failure

#### ❌ Naive Solution: Async Replication, Trust the Replica

**Approach:** Configure streaming replication with `synchronous_commit = off`. The primary acknowledges writes immediately after writing to its local WAL. The replica catches up asynchronously.

**Why it breaks:** If the primary region dies at exactly the moment between "write acknowledged to client" and "write replicated to at least one replica," that write is lost permanently. For a payment system: money debited, order confirmed to user, but the record never reached the replica. When we promote the replica to primary after the outage, that transaction is gone. RPO is not zero — it's however many seconds of replication lag existed at the moment of failure. At 50K writes/sec, even 1 second of lag = 50,000 lost transactions.

---

#### ⚠️ Better Solution: Synchronous Commit to One Replica

**Approach:** PostgreSQL `synchronous_standby_names = 'FIRST 1 (eu-west-replica)'`. The primary only acknowledges a write after at least one replica has confirmed it.

```
Client ──WRITE──► Primary ──sync──► Replica-EU (must ACK before client gets response)
                         ──async──► Replica-APAC (best-effort, no ACK required)
```

**New problem introduced:** Write latency is now bound by the round-trip time to the synchronous replica. US-East to EU-West = ~80ms RTT. Every write adds ~80ms. Our write p99 SLO was 200ms — we've spent 80ms of that budget just on replication acknowledgment. If the synchronous replica goes down, PostgreSQL by default degrades to async mode (or blocks writes entirely depending on `synchronous_commit` setting). The system isn't truly resilient — it has a single-replica dependency.

---

#### ✅ Good Solution (Senior Bar): Quorum-Based Synchronous Replication

**Approach:** Use PostgreSQL's `FIRST N` or `ANY N` quorum syntax. Require acknowledgment from **2 out of 3** replicas before committing. This tolerates one replica being down while still guaranteeing durability.

```
synchronous_standby_names = 'ANY 2 (eu-west-replica, ap-south-replica, us-west-replica)'
```

```
                    ┌──────────────┐
Client ──WRITE──►   │   Primary    │ ── sync ACK required from ANY 2 of 3 replicas
                    │  (us-east)   │
                    └──────┬───────┘
                           │
          ┌────────────────┼─────────────────┐
          │                │                 │
    ┌─────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
    │ Replica EU │  │ Replica APAC│  │ Replica US-W│
    │  (sync)    │  │  (sync)     │  │  (sync)     │
    └────────────┘  └─────────────┘  └─────────────┘
    ANY 2 must ACK before primary responds to client
```

**Write latency math:**
- The commit completes when the 2 fastest replicas respond
- US-W ≈ 20ms, EU-W ≈ 80ms, APAC ≈ 150ms
- Fastest 2: US-W (20ms) + EU-W (80ms) → p99 write ≈ 80–100ms ✓ (within 200ms SLO)

**What a Principal notices:** We've solved durability for standard relational data, but we haven't addressed the **split-brain scenario** during failover. When us-east goes down and we promote eu-west, how do we prevent old us-east (if it partially recovers) from accepting writes simultaneously? That requires fencing — a Principal-level topic we'll address in the deep dives.

---

### Requirement 3: Automatic Failover — Sub-60-second recovery

#### ❌ Naive Solution: Manual Failover (Pagerduty + Human)

**Approach:** Monitor primary health via a cron job. When it fails, someone gets paged. They SSH in, run `pg_promote` on the replica, update DNS, and notify the team.

**Why it breaks:** Human reaction time is 5–15 minutes minimum. At 99.99% availability, our entire annual downtime budget is 52 minutes. One manual failover per year could consume it. At 3am in Bangalore with a groggy on-call engineer, it's 20–30 minutes. This violates our SLO by design.

---

#### ⚠️ Better Solution: Patroni / Replication Manager with Auto-Promotion

**Approach:** Use Patroni (PostgreSQL HA framework using etcd or Consul for distributed consensus). Patroni elects a leader, detects primary failure via health checks, automatically promotes the best replica, and updates the cluster state. DNS failover happens automatically.

```
┌──────────────────────────────────────────────┐
│              etcd Cluster (consensus store)  │
│          /patroni/leader = "us-east-1"       │
└──────────────────────────────────────────────┘
         ▲ reads/writes cluster state
         │
┌────────┴───────┐   ┌────────────────┐   ┌────────────────┐
│ Patroni Agent  │   │ Patroni Agent  │   │ Patroni Agent  │
│  (us-east-1)   │   │  (eu-west-1)   │   │  (ap-south-1)  │
│  PRIMARY ✓     │   │  REPLICA       │   │  REPLICA       │
└────────────────┘   └────────────────┘   └────────────────┘
```

**Failover sequence (automated, < 30s):**
1. Patroni agent on us-east-1 stops responding to health checks
2. etcd TTL expires — leadership lock is released
3. eu-west-1 and ap-south-1 agents race to acquire leadership in etcd
4. Winner (eu-west-1) runs `pg_promote`
5. Patroni updates `/patroni/leader = "eu-west-1"`
6. DAL reads cluster state from etcd, starts routing writes to eu-west-1
7. DNS record updated (TTL was pre-configured to 30s)

**New problem:** Network partitions. If us-east-1 is not truly down but has a network partition (it can still serve local traffic, it just can't reach etcd), we have two nodes that both think they're the primary — classic split-brain. Patroni handles this via fencing (STONITH — Shoot The Other Node In The Head), but only if we've configured it correctly. A Principal thinks about the STONITH configuration before the failover mechanism.

---

#### ✅ Good Solution (Senior Bar): Patroni + STONITH + Read-Only Fallback Mode

**Approach:** Add two layers of protection:
1. **STONITH/Fencing:** Before any replica can promote itself, it must first fence the old primary (e.g., call the cloud API to stop the EC2 instance or remove it from the load balancer's target group). If fencing fails, the replica refuses to promote — it's better to stay down than to have two primaries.
2. **Read-only mode during failover:** While the new primary is being promoted, configure replicas to serve reads in read-only mode. Users experience degraded writes (returns 503 with `Retry-After: 30s`) but reads continue uninterrupted. The system is degraded, not down.

```python
# DAL behavior during failover
def handle_write_during_failover(query):
    if cluster_state == FAILOVER_IN_PROGRESS:
        if query.is_read():
            return route_to_best_healthy_replica(query)  # reads continue
        else:
            raise ServiceUnavailable(
                message="Primary region failover in progress",
                retry_after_ms=5000,
                idempotency_key=query.idempotency_key  # return key so client can retry safely
            )
```

**And here's how we'd know if this is broken in production:** Two alerts:
- `failover_in_progress = true` → immediately fires a Sev-1 page
- `write_error_rate > 5%` for 60 seconds → Sev-1 independent of failover flag
- `rto_seconds > 60` on the FailoverEvent table → post-incident SLO breach alert

---

### Requirement 4: Schema Migration Consistency Across Regions

#### ❌ Naive Solution: Run Migrations Sequentially Per Region, Manually

**Approach:** DBA connects to each region's primary, runs the migration SQL, verifies it worked. Move to the next region.

**Why it breaks:** During the time between "migration applied to us-east" and "migration applied to eu-west," the schema is different across regions. If our application reads from eu-west (old schema) and writes to us-east (new schema), and the migration adds a NOT NULL column, the app will start generating constraint violation errors on writes immediately. This window is typically 10–30 minutes for large tables with `ALTER TABLE` locks.

---

#### ⚠️ Better Solution: Expand/Contract Pattern (Backward-Compatible Migrations)

**Approach:** Never make breaking schema changes in a single step. Always use the Expand-Contract pattern:
- **Phase 1 (Expand):** Add new column as nullable. Apply to all regions. No app change yet.
- **Phase 2 (Migrate):** Backfill existing rows with the new column's value. Do this as a background job, not a blocking migration.
- **Phase 3 (Contract):** Add NOT NULL constraint and drop old column after all regions and apps are using the new column.

This means at any given moment, the schema is either at version N or N+1, and both versions of the application can read/write correctly.

---

#### ✅ Good Solution (Senior Bar): Versioned Migration Orchestration with Region Sync Gates

**Approach:** Build a migration orchestration service that:
1. Applies migrations to **one canary region** first
2. Monitors application error rate and latency for 5 minutes
3. Only fans out to remaining regions if the canary is healthy
4. Tracks migration status in the `SchemaVersion` table per region
5. Blocks new application deployments until all regions are on the same schema version

```
Migration Service:
  1. Apply migration to: us-west (canary region, 5% of traffic)
  2. Wait 5 minutes, observe: error_rate < 0.1%, latency p99 < baseline × 1.1
  3. If healthy: fan-out to eu-west, ap-south, us-east
  4. Update SchemaVersion status='applied' per region as each completes
  5. Emit 'MIGRATION_COMPLETE' event when ALL regions report status='applied'
  6. Deployment gate: CI/CD pipeline checks that schema_version matches target
     before deploying any new app version that depends on the new schema
```

**And here's how we'd know if this is broken:** The `SchemaVersion` table is CDC-streamed to our monitoring. An alert fires if any region is more than 2 schema versions behind the most-advanced region for more than 10 minutes.

---

### Requirement 5: Data Residency — Region-Pinned User Data

#### ❌ Naive Solution: Replicate Everything Everywhere

**Approach:** Full replication — every region gets a full copy of all data. Simple operationally.

**Why it breaks:** GDPR Article 44 prohibits transferring EU personal data to non-adequate countries without safeguards. If we replicate EU user data to our APAC region (which may be hosted in Singapore, not an EU-adequate country), we are in violation. The fine can be up to 4% of global annual revenue — existential for most companies.

---

#### ✅ Good Solution: Regional Sharding with Logical Replication Filtering

**Approach:** Use PostgreSQL's **logical replication** (publication/subscription model) with row-level filters. Each region only receives rows where `region_key` matches its own region.

```sql
-- On us-east primary: publish only US records to APAC replica
CREATE PUBLICATION us_to_apac_pub FOR TABLE records
WHERE (region_key = 'us');

-- On APAC replica: subscribe to the filtered publication
CREATE SUBSCRIPTION apac_sub
CONNECTION 'host=us-east-primary ...'
PUBLICATION us_to_apac_pub;
```

For EU data, only the EU region has a full copy. APAC and US replicas have zero EU user rows. This is verifiable: a compliance query `SELECT count(*) FROM records WHERE region_key='eu'` on the APAC replica must return 0 at all times.

**And here's how we'd know if this is broken:** Run a scheduled compliance check query every 15 minutes on each non-EU replica. If any EU records appear in APAC or US replicas, fire an immediate Sev-1 with compliance escalation. This is not optional — it's a regulatory requirement.

---

### Tying It All Together — Full Architecture

```
                           ┌──────────────────────────────────┐
                           │         GeoDNS / Global LB        │
                           │   (routes users to nearest DAL)   │
                           └───────┬───────────┬──────────────┘
                                   │           │
             ┌─────────────────────┼───────────┼──────────────────────┐
             │                     │           │                      │
    ┌────────▼────────┐   ┌────────▼────────┐  ┌────────────────────┐ │
    │   DAL: US-East  │   │   DAL: EU-West  │  │   DAL: AP-South    │ │
    │  (primary write │   │ (replica reads) │  │  (replica reads)   │ │
    │   + reads)      │   └────────┬────────┘  └────────┬───────────┘ │
    └────────┬────────┘            │                    │             │
             │                     │                    │             │
    ┌────────▼────────┐   ┌────────▼────────┐  ┌────────▼───────────┐ │
    │  PG Primary     │◄──│  PG Replica EU  │  │  PG Replica APAC  │ │
    │  (us-east-1)    │   │  (eu-west-1)    │  │  (ap-south-1)     │ │
    │  WAL producer   │   │  WAL consumer   │  │  WAL consumer      │ │
    └────────┬────────┘   └─────────────────┘  └────────────────────┘ │
             │ WAL stream (synchronous to ANY 2 of 3 replicas)         │
             │                                                          │
    ┌────────▼────────┐                                                │
    │  etcd Cluster   │ ── Patroni agents watch this for leader state  │
    │  (distributed   │                                                │
    │   consensus)    │                                                │
    └────────┬────────┘                                                │
             │                                                          │
    ┌────────▼────────┐                                                │
    │ Migration Svc   │ ── fan-out coordinator for schema changes      │
    └─────────────────┘                                                │
                                                                        │
    ┌────────────────────────────────────────────────────────────────┐ │
    │  Observability Stack                                            │ │
    │  Prometheus (metrics) + Loki (logs) + Tempo (traces)           │ │
    │  Grafana (dashboards) + PagerDuty (alerting)                   │ │
    └────────────────────────────────────────────────────────────────┘ │
    └──────────────────────────────────────────────────────────────────┘
```

**Critical path — a write from a Tokyo user:**

1. Tokyo user's app hits GeoDNS → resolves to DAL:AP-South (nearest)
2. DAL identifies the query as a WRITE → routes to Primary (us-east)
3. Cross-region write traverses the backbone network (~150ms)
4. Primary writes to WAL
5. Quorum reached: eu-west replica ACKs (fastest synchronous replica, ~80ms additional)
6. Primary responds 201 Created to DAL
7. DAL returns response to Tokyo user — total write latency: ~150ms (network) + ~80ms (quorum) + ~20ms (disk/processing) = ~250ms
   - **This exceeds our 200ms write p99 SLO** — which means our assumption that APAC users write through US-East is wrong at this scale.
   - **The honest observation:** We need a write primary in APAC for APAC users, which means active-active, which is a deep dive topic.

---

## SECTION 4: DEEP DIVES

### Deep Dive 1: Active-Active vs. Active-Passive — The Write Routing Problem

The critical path walkthrough exposed a real problem: Tokyo users paying 150ms network overhead on every write. Let's solve this properly.

#### ❌ Bad: Accept the Cross-Region Write Latency

Just document that APAC users have slower writes. This is a business decision, not a technical one — but it's often wrong. If your APAC market is 40% of revenue, making their writes 3× slower than US users is a product problem.

#### ⚠️ Better: Active-Active with Last-Write-Wins (LWW)

Deploy a write-capable primary in each region. Route writes to the local primary. Use timestamp-based conflict resolution: the write with the higher timestamp wins.

```
[Tokyo User] ──WRITE t=100──► [Primary: APAC]
[London User] ─WRITE t=101──► [Primary: EU]
-- Both update the same record within 150ms --
-- Conflict: APAC primary gets EU's write at t=150, EU primary gets APAC's write at t=152
-- LWW: t=101 (EU write) wins globally
-- But APAC primary already acknowledged t=100 to the Tokyo user!
-- Tokyo user sees their write confirmed, then silently overwritten.
```

The problem: clocks. Distributed systems clocks are not perfectly synchronized even with NTP. A 1ms clock skew can cause the "earlier" write to silently win. Users see confirmed writes disappear. This is not a theoretical concern — it is a well-documented production failure mode at Amazon, Google, and Facebook.

#### ✅ Best: Active-Active with Routing Affinity + CRDTs or Application-Level Conflict Resolution

**The correct answer for most systems:** Don't do true active-active unless you absolutely must. Instead, use **regional write affinity**: each user is assigned to a home region, and their writes always go to that region's primary. A Tokyo user with `home_region=ap-south` always writes to APAC primary. A London user with `home_region=eu-west` always writes to EU primary. The key insight: **two users in different regions editing the same shared resource** is the only true conflict scenario. If your data model minimizes shared-write surfaces (most SaaS products can achieve this by partitioning data by user or by tenant), you get the latency benefit of local writes without the conflict problem.

```
User Table: home_region column
DAL routing rule: writes go to home_region's primary

[Tokyo User, home_region=ap-south]  ──WRITE──► [Primary: APAC]  ← fast, local
[London User, home_region=eu-west]  ──WRITE──► [Primary: EU]   ← fast, local
[Shared resource write from both]   ──────────► Choose one primary as owner (e.g., record's creating region)
```

For shared resources that can be written from any region (e.g., a shared document), use CRDTs (Conflict-free Replicated Data Types) — data structures designed to merge concurrent updates deterministically without coordination. This is what Notion, Figma, and Google Docs use under the hood.

**What does a Principal see here that a Senior might miss?**

The Senior says "use active-active." The Principal asks: "What is the conflict rate? What is the consistency model your users expect?" Most products have a 99%+ user-partitioned write pattern — home region affinity solves the problem without CRDT complexity. CRDTs are the right answer only for collaborative editing tools. Reaching for CRDTs when user affinity would suffice is over-engineering that introduces operational complexity (CRDT library versioning, merge correctness bugs) for no real benefit. A Principal knows when *not* to use the sophisticated solution.

---

### Deep Dive 2: Split-Brain Prevention and Fencing

Split-brain is the most dangerous failure mode in any distributed system. Two nodes both believe they are the primary and accept writes. When connectivity is restored, we have two diverged histories and no automatic way to reconcile them. This is how you lose data at scale.

#### ❌ Bad: Hope the Network Never Partitions

No explicit fencing. Just trust that Patroni will always elect one leader. Real-world: AWS has multiple documented cases where VPC internal networking had asymmetric failures — node A could see node B, but B couldn't see A. Patroni would elect B as primary while A kept accepting writes from clients that could still reach it.

#### ⚠️ Better: etcd-Based Leadership Lock with TTL

Patroni's default: the primary holds a lock in etcd with a TTL (e.g., 30 seconds). If the primary can't renew the lock (because it can't reach etcd), it immediately fences itself — stops accepting writes. The replica then acquires the lock and promotes.

This works if: the primary can reach etcd but not replicas (partition between primary and replicas). The primary loses its lock when the TTL expires.

This fails if: etcd itself is partitioned. Or if the primary has degraded networking that lets it reach etcd but not replicas — it keeps renewing the lock while replicas can't receive WAL.

#### ✅ Best: STONITH + Lease-Based Fencing + Epoch Numbers

Three layers of protection:

**Layer 1 — STONITH (Shoot The Other Node In The Head):**
Before promoting a replica, use the cloud provider API to forcibly stop the old primary's database process or remove it from all load balancers. This is the "nuclear option" — it guarantees the old primary cannot accept writes before the new primary takes over.

```python
def promote_replica(replica_id, old_primary_id):
    # Step 1: Fence the old primary first
    cloud_api.stop_instance(old_primary_id)  # or: remove from target group
    # Step 2: Verify old primary is truly down (poll for 10 seconds)
    for _ in range(10):
        if not is_reachable(old_primary_id):
            break
        time.sleep(1)
    # Step 3: Only now promote the replica
    run_pg_promote(replica_id)
```

**Layer 2 — Epoch Numbers:**
Every promotion increments a global epoch stored in etcd. Write operations include the current epoch in the WAL. If an old primary (from epoch N) somehow comes back online and tries to write, replicas (already at epoch N+1) reject writes from old-epoch primary immediately.

```sql
-- WAL entry structure
{ epoch: 5, lsn: "0/ABC1234", record: {...} }

-- Replica rejection logic
if incoming_epoch < current_epoch:
    log.error("Stale primary detected", epoch=incoming_epoch)
    reject_write()
```

**Layer 3 — Client-Side Write Rejection via Epoch Headers:**
The DAL includes the current epoch in every write request. The database validates the epoch before committing. If the client is connected to a stale primary (old epoch), the write is rejected with `STALE_PRIMARY` error, and the client retries against the new primary.

**What does a Principal see here that a Senior might miss?**

The Senior implements Patroni and declares split-brain solved. The Principal asks: "What happens if STONITH fails? What happens if the cloud API is unreachable during the same network partition that caused the failover?" The answer: implement STONITH with a timeout, and if STONITH fails, do NOT promote — keep the cluster in a degraded state and page the on-call engineer. An automated system that promotes despite failed fencing is more dangerous than a system that refuses to promote and waits for human intervention. The safe default is: **when in doubt, don't promote**. Users experience writes being unavailable for longer, but they don't lose confirmed data.

---

### Deep Dive 3: Replication Lag — The Invisible Consistency Problem

Replication lag is the most common source of confusing production bugs in multi-region systems. Users see data disappear, or see stale data after a write — and the root cause is the replica serving reads that haven't yet received the latest WAL entries.

#### ❌ Bad: Ignore Lag, Serve All Reads from Replica

No lag awareness. Users write, read back from the replica 50ms later, see stale data. Support tickets flood in: "I just saved but my changes are gone."

#### ⚠️ Better: Expose Lag via Response Headers, Let Client Decide

The DAL includes `X-Replica-Lag-Ms: 312` in every read response. Smart clients can choose to retry if lag is too high. But this puts the burden on every client to implement this logic — fragile, inconsistent, and requires all clients to be updated.

#### ✅ Best: Read-Your-Writes (RYOW) via Session Tokens + LSN Tracking

**Approach:** Implement Read-Your-Writes consistency at the DAL level using LSN (Log Sequence Number) tracking.

After every write, the DAL records the LSN of the committed write and returns it to the client in a session token:

```
POST /records
Response headers: X-Write-LSN: 0/A1B2C3D4
```

On subsequent reads, the client sends this LSN back:

```
GET /records/{id}
Request headers: X-Min-LSN: 0/A1B2C3D4
```

The DAL checks whether the local replica's applied LSN is ≥ the requested minimum:

```python
def route_read_with_ryow(query, min_lsn):
    local_replica = replicas[region_hint]
    
    if local_replica.applied_lsn >= min_lsn:
        return local_replica  # Safe: replica has the write
    
    # Replica hasn't caught up yet
    if time_since_write < 500ms:
        # Option A: Wait for replica to catch up (up to 500ms)
        wait_for_lsn(local_replica, min_lsn, timeout_ms=500)
        return local_replica
    else:
        # Option B: Read from primary (guaranteed up-to-date)
        metrics.increment("dal.ryow_primary_fallback")
        return primary
```

**The clever part:** This uses the replica by default (fast, local) but falls back to the primary only for the specific reads that need to see a specific write. The session token is a compact opaque string (LSN + region + timestamp = ~32 bytes). Most of the time, the replica catches up within 100–200ms and serves the RYOW read locally. Primary fallback is rare — triggered only when the user reads back their write within milliseconds of writing.

**Schema for tracking LSN per connection (in DAL memory):**
```json
{
  "session_token": "eyJ...",
  "last_write_lsn": "0/A1B2C3D4",
  "write_region": "us-east",
  "write_timestamp": "2024-01-15T10:30:45Z"
}
```

**And here's how we'd know if this is broken:** Track `dal.ryow_primary_fallback` rate. If > 5% of reads are falling back to the primary for RYOW reasons, replication lag is too high. This triggers a lag investigation. Target: < 1% RYOW fallback rate.

**What does a Principal see here that a Senior might miss?**

The Senior implements RYOW and calls it done. The Principal asks: "What happens to the LSN session token in a failover?" If the primary fails and we promote eu-west, the LSN namespace changes (PostgreSQL LSNs are not portable across primaries). The session token with LSN `0/A1B2C3D4` from the old primary is meaningless to the new primary. The Principal's answer: in a failover, invalidate all outstanding LSN session tokens and force a brief "strong consistency mode" for 60 seconds post-failover — route all reads to the new primary until the system stabilizes. This is a temporary degradation in read performance (all reads hit the new primary), but it prevents users from seeing stale data in the post-failover window.

---

## SECTION 5: OBSERVABILITY — THE PRINCIPAL'S FIRST-CLASS CONCERN

### 5a. The Three Pillars

#### METRICS (The What)

**Business/Product Metrics:**

| Metric | Why It Matters |
|--------|---------------|
| `write_success_rate` (per region) | < 99.9% means users are losing data. Sev-1 even if infra metrics look green. |
| `read_success_rate` (per region) | < 99.9% means reads are failing, not just slow. |
| `replication_lag_p99` (per replica) | Our "invisible consistency" canary. > 1s means RYOW fallbacks spike. |
| `ryow_primary_fallback_rate` | Leading indicator of replication health. |
| `cross_region_fallback_rate` | Measures how often a region's replica is unhealthy. |
| `failover_count` (per 30d) | SRE-level: how stable is the system? Target: 0 unplanned failovers. |

**Service-Level Metrics (RED framework per service):**

*Database Access Layer (DAL):*
- Rate: reads/sec + writes/sec, per region
- Errors: `4xx` (bad requests), `5xx` (infrastructure failures), `REPLICA_UNAVAILABLE`, `PRIMARY_UNAVAILABLE` — separately
- Duration: p50/p95/p99 for read vs. write latency, per region

*PostgreSQL Primary:*
- Rate: TPS (transactions per second)
- Errors: constraint violations, deadlock rate, connection refused
- Duration: query latency p99, WAL write latency

*PostgreSQL Replicas:*
- Rate: replay TPS (WAL replay speed)
- Errors: replication slot invalidation, LSN mismatch errors
- Duration: lag_seconds, lag_bytes

**Infrastructure Metrics (USE framework):**

*For each PostgreSQL node:*
- Utilization: CPU %, memory %, disk I/O %, WAL disk usage %
- Saturation: connection pool exhaustion %, lock wait queue depth, WAL sender queue depth
- Errors: OOM kills, disk errors, network packet drops, replication slot overflow

*For etcd:*
- Utilization: leader election latency, follower log apply rate
- Saturation: Raft proposal queue depth
- Errors: election timeouts, leader changes per hour

**Derived/Computed Metrics:**
- `replication_bandwidth_MB_per_sec` = WAL sender bytes / time (monitors egress cost)
- `schema_version_drift` = max(schema_version) - min(schema_version) across regions (should be 0 in steady state)
- `RYOW_wait_time_p99` = time spent waiting for replica LSN catch-up (measures replica health)
- `failover_rto_seconds` = `resolved_at - detected_at` on FailoverEvent table

**Tool Choice:** Prometheus + Grafana.  
Why not Datadog? At our scale (500K reads/sec × 20 metrics per request = 10M metric data points/sec), Datadog's per-host pricing becomes prohibitive. Prometheus with remote write to Thanos for long-term storage gives us full control and predictable cost. Grafana provides dashboarding. For this specific system, query-by-label (region, replica_id, operation_type) is the primary access pattern — Prometheus + Grafana handles this natively.

---

#### LOGGING (The Why)

**Mandatory Log Structure:**
```json
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "INFO",
  "service": "dal",
  "region": "ap-south",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "request_id": "req-uuid-v4",
  "operation": "read.route",
  "target_replica": "ap-south-replica-01",
  "min_lsn_required": "0/A1B2C3D4",
  "replica_current_lsn": "0/A1B2C3E0",
  "ryow_fallback": false,
  "duration_ms": 12,
  "outcome": "success",
  "replication_lag_ms": 89
}
```

**Log Levels for This System:**
- **ERROR:** Primary unreachable; replica promoted without successful STONITH; write acknowledged but replication failed (data inconsistency!); LSN gap detected between regions; compliance check found EU data in non-EU replica
- **WARN:** RYOW fallback triggered (replica not caught up within 500ms); cross-region replica fallback triggered; replication lag > 500ms; connection pool > 80% utilized; STONITH API call returned non-200
- **INFO:** Failover started; failover completed; migration fan-out initiated; migration completed per region; replica health status changed
- **DEBUG:** Individual query routing decisions; LSN comparison; replica health poll results — NEVER in production

**What NOT to Log:**
- Database row content (PII, payload data)
- Connection strings or credentials
- AWS API keys used for STONITH calls (would appear in centralized logs)
- Health check polling results at INFO level (1 per 5s × 100 nodes = 12M log lines/hour of noise)

**Log Aggregation:** Fluentd sidecar per service → Kafka (buffers spikes) → Loki (label-indexed storage). Why Loki over Elasticsearch? We query by `{region="ap-south", level="ERROR"}` — label-based queries. Loki is 10× cheaper for label-query workloads. We never need full-text search within log bodies for this system.

---

#### DISTRIBUTED TRACING (The Where)

**Instrumentation:** OpenTelemetry SDK across all services. Exporter: OTLP → Grafana Tempo. Why OTel? Today we're a 50-person engineering team using Tempo. When we have 500 engineers, we may need Honeycomb or Lightstep for advanced analysis. OTel decouples instrumentation from backend — we re-point the exporter, not re-instrument every service.

**Critical Trace — Write Operation (annotated with expected latency budget):**
```
[POST /records: DAL ap-south] ──── 250ms total ────
  ├── [auth.validate_jwt] ─────────────────────── 3ms
  ├── [dal.determine_write_region] ──────────────  1ms  ← home_region lookup
  ├── [dal.cross_region_connect: us-east] ─────── 150ms ← network RTT (this is the bottleneck)
  │     └── [pg.write_with_wal_sync] ──────────── 85ms  ← includes quorum replication ACK
  │           ├── [wal.sync.eu-west] ──────────── 80ms  ← synchronous replica ACK
  │           └── [wal.sync.us-west] ──────────── 20ms  ← synchronous replica ACK (faster, not bottleneck)
  └── [dal.return_lsn_token] ───────────────────   1ms
```

This trace immediately reveals that for APAC users, 150ms is network overhead and 80ms is quorum ACK — total 230ms before the user gets a response. The trace makes visible what is otherwise invisible: the 150ms cross-region network cost. This is the data that motivates the regional write affinity discussion.

**Sampling Strategy:** Tail-based sampling at Grafana Tempo:
- 100% of traces with any span status=ERROR
- 100% of traces where total duration > 300ms (our write p99 SLO threshold)
- 5% random sample of everything else

Why not 100%? At 50K writes/sec + 500K reads/sec = 550K traces/sec. At ~1KB/span and 5 spans/trace = 5KB/trace × 550K/sec = 2.75GB/sec of trace data. At $0.10/GB: $8M/year just for traces. Tail-based at 5% + 100% errors = ~$800K/year, capturing 99% of the debugging value.

---

### 5b. SLI / SLO / SLA

| User Journey | SLI | SLO (Internal) | SLA (Customer) |
|---|---|---|---|
| Write a record | % writes completing < 200ms | 99.9% | 99.5% |
| Read a record (local replica) | % reads completing < 50ms | 99.5% | 99% |
| Read-your-own-write | % RYOW reads returning correct data | 99.99% | 99.9% |
| Failover RTO | Time from primary failure to write restoration | p95 < 60s | < 120s |
| Replication lag | p99 lag < 500ms | 99.5% of time | Not customer-visible |
| Data residency compliance | % time EU data stays in EU | 100% | 100% (regulatory) |

**Error Budget:**
- Write SLO: 99.9% = 0.1% error budget = ~43.8 minutes/month of degraded writes
- If we burn > 50% (21.9 min) in the first week → freeze feature deployments
- Error budget alert: `burn_rate_1h > 14.4` (consumes 2% budget in 1 hour) → Sev-1

---

### 5c. Alerting Strategy

**Tier 1 — Sev-1 (Page immediately):**
- Write success rate < 98% for 2 consecutive minutes
- Read success rate < 95% for 5 minutes
- Replication lag > 30 seconds on any replica
- Failover in progress (any epoch change detected)
- EU data detected in non-EU replica (compliance breach — also pages legal team)
- etcd cluster has lost quorum (< 2 of 3 members healthy)

**Tier 2 — Sev-2 (Investigate within 1 hour):**
- Replication lag > 1 second (not 30s, but trending)
- RYOW primary fallback rate > 2%
- Cross-region fallback rate > 0.5%
- Connection pool utilization > 80% on any node
- WAL disk usage > 70% (could cause replication slot invalidation)
- STONITH API latency > 5 seconds (fencing will be slow if needed)

**Tier 3 — Sev-3 (Fix this sprint):**
- P99 write latency trending up 10% week-over-week
- Replication bandwidth growing faster than write QPS (indicates WAL bloat)
- Error budget burn > 20% consumed in one week

**Alert Fatigue Rule:** Every alert has a runbook. Every Sev-2 is reviewed monthly — if it hasn't required action in 60 days, it's demoted to Sev-3 or removed. The on-call rotation reviews alert quality quarterly. We target < 3 Sev-1 alerts per week at steady state. More than 5 Sev-1s in a week triggers an alert quality review.

---

### 5d. Observability for Async Components

Our WAL replication is effectively an async stream. Per-partition (per-replica-slot) lag monitoring is essential:

```
# Prometheus query: replication lag per replica slot
pg_replication_slot_lag_bytes{slot_name="eu_west_sub"} 
pg_replication_slot_lag_bytes{slot_name="ap_south_sub"}
```

**Critical insight:** If `ap_south_sub` lag is 10MB but `eu_west_sub` lag is 0 bytes, we have a problem specific to the APAC replica, not a primary throughput problem. Without per-slot visibility, we'd mistake this for a write throughput issue and scale the primary unnecessarily.

**Dead replication slot risk:** If a replica goes offline and its replication slot on the primary is not dropped, the primary accumulates WAL indefinitely (it can't discard WAL until the slot consumer has caught up). This is a well-known PostgreSQL footgun — it can fill the disk of the primary and crash it. Alert: `pg_replication_slot_lag_bytes > 10GB` on any slot → Sev-2 immediately.

---

### 5e. Observability-Driven Database Design

**Audit columns on every table:**
```sql
created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
created_by   UUID REFERENCES users(id)
version      INTEGER NOT NULL DEFAULT 1  -- optimistic locking
is_deleted   BOOLEAN DEFAULT FALSE       -- soft delete, never hard delete
region_key   TEXT NOT NULL               -- always know which region owns this row
```

**Why soft deletes?** During the 3am incident where "user's records disappeared after the failover," we need to answer: was this a bug that deleted the rows, or a user action? Hard deletes make this question unanswerable. Soft deletes give us the audit trail. We archive `is_deleted=true` rows to cold storage (S3 Glacier) after 90 days to contain storage cost.

**CDC for free observability:** Tap the PostgreSQL WAL via Debezium → Kafka. This produces a real-time stream of every INSERT/UPDATE/DELETE across all tables. Use cases:
1. Power the compliance monitoring service (detect EU rows appearing in non-EU replicas)
2. Invalidate application-layer cache entries when records change
3. Feed the analytics warehouse without querying production
4. Build real-time audit logs

This is a free observability stream — the WAL is already produced for replication. Consuming it for these additional purposes adds zero write overhead to the primary.

**Query performance observability:**
```sql
-- Weekly review: top-10 queries by total_time
SELECT query, calls, total_time/calls AS avg_ms, 
       total_time, rows 
FROM pg_stat_statements 
ORDER BY total_time DESC 
LIMIT 10;
```
Log ALL queries > 100ms (not sampled — all of them). Track explain plan changes for the top-20 queries — an index being dropped silently can turn a 5ms query into a 30-second sequential scan overnight.

---

### 5f. Cost Observability

**Cost metrics:**
- Replication egress cost per region per day (target: < $1,500/day; alert if > $2,000/day)
- Storage cost per region (hot vs. archived): tag by `region`, `team`, `environment`
- STONITH API call cost (negligible, but tracks failover frequency)
- etcd storage cost (monitor: etcd should stay < 2GB; bloat indicates stale keys)

**Cost anomaly detection:**
- Alert if cross-region replication bandwidth increases > 50% day-over-day (could indicate replication loop, WAL bloat, or a runaway bulk insert job)
- Alert if WAL disk on primary grows > 5% per hour (could indicate a replica is down and its slot is accumulating WAL)
- Monthly cost-per-region review: if APAC region costs are 2× US despite lower user count, investigate WAL amplification

**Tagging strategy (every cloud resource):**
```
team: data-infra
service: postgres-primary|postgres-replica|etcd|dal
region: us-east|eu-west|ap-south
environment: production|staging
cost-center: infrastructure
```

Without tagging at creation time, cost attribution is impossible retroactively.

---

## SECTION 6: PRINCIPAL ENGINEER LENS

### 6a. The 3 Hidden Time Bombs

**Time Bomb 1: Replication Slot WAL Accumulation (Breaks at Scale)**

*What breaks:* As write volume grows, a single offline replica (e.g., APAC goes down for a maintenance window) accumulates WAL on the primary's disk. At 90MB/sec replication rate, a 2-hour outage = 648GB of WAL accumulated. At 500K writes/sec (10× scale), that's 6.5TB in 2 hours. Primary disk fills. Primary crashes. Now you've turned a replica outage into a primary outage.

*Cost of fixing later:* Extremely high — requires redesigning the replication topology to use streaming replication without persistent slots, or implementing WAL archiving to S3 as a buffer. Schema changes required to all monitoring.

*Early signals:* `pg_replication_slot_lag_bytes` growing linearly when a replica is down. Alert threshold: > 10GB triggers Sev-2. Mitigation now: set `max_slot_wal_keep_size` to cap WAL retention per slot.

**Time Bomb 2: The 10x Write Scale Cliff**

*What breaks:* Our single write primary is designed for 18K writes/sec peak. At 10× user growth = 180K writes/sec, a single PostgreSQL primary on the largest available instance tops out around 100K TPS. We hit the wall around 8× growth. The fix — horizontal sharding — is one of the most expensive migrations in database history (application changes, data migration, routing layer overhaul).

*Cost of fixing later:* 6–12 months of engineering time, high risk of data inconsistency during migration, likely requires a full application rewrite of DB access patterns.

*Early signals:* Primary CPU > 70% sustained, WAL write latency p99 creeping above 20ms, connection pool saturation above 60% — these appear at 5–6× scale, giving us 18 months of warning if we're watching.

**Time Bomb 3: Schema Drift in Logical Replication**

*What breaks:* Logical replication filters rows by `region_key` but replicates all columns. If a developer adds a column to the `records` table and only the EU region has the new column applied (due to a failed migration fan-out), logical replication breaks silently — PostgreSQL's publisher and subscriber have different schemas. The replication stream stalls. EU data stops replicating. No loud error — just silent drift.

*Cost of fixing later:* Requires pausing replication, fixing schema in all regions, resynchronizing data, and verifying consistency — typically 4–8 hours of downtime risk for large tables.

*Early signals:* `schema_version_drift > 0` for more than 10 minutes fires a Sev-2. Also: monitor `pg_replication_slot_lag_bytes` for sudden spikes that coincide with migration events.

---

### 6b. Technology Selection Justification

**Replication Technology:**

| Criteria | PostgreSQL Streaming + Logical | MySQL Group Replication | CockroachDB |
|---|---|---|---|
| RYOW consistency support | ✅ LSN-based, precise | ⚠️ GTID-based, workable | ✅ Built-in |
| Conflict resolution | N/A (home region routing) | N/A | ✅ MVCC-based |
| Data residency (row filtering) | ✅ Logical pub/sub with WHERE | ✅ Binlog filtering | ⚠️ Zone configs, less granular |
| Operational complexity | Medium (Patroni) | Medium (MGR) | High (distributed transactions) |
| Cost at our scale | Low (self-managed) | Low | High (license + compute) |
| Verdict | ✅ Chosen | ❌ Less ecosystem | ❌ Over-engineered for our pattern |

**The specific reason PostgreSQL beats CockroachDB for this system:** Our write pattern is region-affined (users write to their home region). True global distributed transactions (CRDB's main selling point) are not needed. We'd be paying for CRDB's complexity, licensing cost, and operational overhead for a capability we don't use.

**Consensus/Coordination:**

| Criteria | etcd | ZooKeeper | Consul |
|---|---|---|---|
| Patroni integration | ✅ Native | ✅ Legacy | ✅ Works |
| Operational simplicity | ✅ Simple binary | ❌ JVM, complex | ✅ Simple |
| Watch/lease semantics | ✅ Excellent | ⚠️ Adequate | ✅ Good |
| Verdict | ✅ Chosen | ❌ JVM overhead | ❌ Consul's strength is service mesh, not DB consensus |

---

### 6c. The Full Tradeoff Table

| Decision | Chosen | Alternative | What We Gain | What We Give Up | Revisit When |
|---|---|---|---|---|---|
| Write routing | Home-region affinity | Active-active LWW | Simplicity, no conflicts | APAC write latency (+150ms) | >40% of writes are cross-user conflicts |
| Replication | Async + quorum sync (ANY 2/3) | Full sync to all replicas | Low write latency | Risk of 1 replica being stale | RPO requirement tightens to 0ms everywhere |
| Failover | Patroni + STONITH | Manual | 30s RTO, no human needed | Complexity of fencing config | Never — manual failover can't meet 99.99% SLO |
| Consistency | RYOW via LSN tokens | Synchronous reads from primary | Local replica reads (fast) | Session token complexity | RYOW fallback rate > 5% sustained |
| Log storage | Loki | Elasticsearch | Cost (3-5× cheaper at label-query) | Full-text search within log bodies | Need full-text search on log content |
| Trace sampling | Tail-based 5% + 100% errors | Head-based 100% | Cost ($12M/yr vs $1M/yr) | Slightly delayed sampling decisions | Budget allows 100% sampling |
| Schema migrations | Expand-contract pattern | Big-bang migrations | Zero downtime | Slower feature delivery (2-3 phases) | Never — big-bang migrations are always wrong |

---

### 6d. Scale Evolution Plan

**Phase 1 — MVP (0 → 100K users):**

Architecture: Single PostgreSQL primary in one region, one warm standby, Patroni for failover, manual promotion only. No multi-region yet — build the data model with `region_key` from day one so the sharding migration is possible later.

Observability minimum viable: 3 dashboards (write latency, read latency, replication lag) and 5 alerts (primary down, replica lag > 30s, write error rate > 1%, disk > 80%, connection pool > 80%). No cost observability yet — it's not worth the complexity.

What we deliberately don't build: geographic routing, compliance filtering, DAL (just connect directly), RYOW — all of this is premature at 100K users.

**Phase 2 — Growth (100K → 10M users):**

What breaks first: At ~500K users (~2.9K writes/sec sustained), the single node starts showing > 50% CPU. At ~2M users, cross-region user count justifies replica deployment for read latency.

Components added: Read replicas per region, DAL with smart routing, RYOW via LSN tokens, compliance filtering (if expanding to EU). Patroni STONITH configured (previously manual).

Observability additions: Per-service SLO dashboards, error budget tracking, cost-per-region monthly review, automated compliance scans.

**Phase 3 — Scale (10M+ users):**

At 10M users (58K writes/sec sustained, 180K peak), the single primary is near its limit. This is the phase where we implement regional write affinity properly — each major region gets its own write primary. Cross-region replication becomes bidirectional (all regions replicate to all others, filtered by region_key). This is a 6-month project.

Observability at this scale: 50+ services, 200+ engineers. Manual alert review is impossible. Implement automated SLO burn-rate alerts (already designed), ML-based anomaly detection on replication lag and write latency, dedicated observability platform team (3–5 engineers) owning Prometheus/Grafana/Loki/Tempo infrastructure as an internal product. Alert quality review is quarterly and data-driven (false positive rate tracked per alert).

What becomes its own team: Database Infrastructure team (manages PostgreSQL, Patroni, migrations), Data Residency team (manages compliance filtering, audit), DAL team (owns the routing layer as a reliability product).

---

### 6e. Failure Mode Analysis

**Failure 1: Primary Region Network Partition**

*Scenario:* us-east-1 is alive but cannot communicate with eu-west or ap-south. etcd in us-east loses quorum. Primary cannot renew Patroni lock.

*Detection:* etcd `leader_changes_total` metric increments. Within 30 seconds (TTL expiry), Patroni on eu-west acquires lock. Alert: `failover_in_progress=true` → Sev-1 fires within 35 seconds.

*Impact:* Writes unavailable for ~45 seconds (30s TTL + 15s promotion). Reads from eu-west and ap-south replicas continue uninterrupted (they serve from their local replica). ~45s of write unavailability.

*Recovery:* Patroni promotes eu-west. DAL automatically routes writes to eu-west (reads cluster state from etcd). RTO: 45 seconds. RPO: 0 (quorum replication was synchronous — eu-west already had the data).

*Prevention:* STONITH ensures us-east primary is forcibly stopped before eu-west promotes. Epoch number prevents any stale us-east from accepting writes if networking partially recovers.

**Failure 2: Replication Lag Spike (Heavy Batch Write Job)**

*Scenario:* A batch import job on us-east writes 50M records in 2 hours. WAL replication stream overwhelms eu-west replica's network bandwidth. Lag grows from 100ms to 45 seconds.

*Detection:* `pg_replication_slot_lag_seconds > 30` → Sev-1 fires. Read success rates drop as DAL falls back to primary (RYOW triggers, primary under additional read load).

*Impact:* Read latency increases for eu-west users (primary fallback = +80ms). Primary CPU increases (serving reads it shouldn't be).

*Recovery:* Throttle the batch job (add `pg_sleep()` between batches to limit WAL production rate). Replica catches up. DAL RYOW fallback rate drops back to < 0.1%. Runbook: "Identify the batch job causing WAL storm using `pg_stat_activity`, kill or throttle, monitor `pg_replication_slot_lag_bytes` for recovery."

*Prevention:* Batch jobs should be rate-limited to 10% of peak write throughput. Implement a batch job governor that monitors WAL lag and auto-throttles when lag > 1 second.

**Failure 3: Logical Replication Schema Mismatch**

*Scenario:* Developer applies a migration (adds column `priority INTEGER DEFAULT 0`) to the us-east primary. Migration fan-out to eu-west fails silently (network hiccup during migration apply). eu-west still has the old schema. Logical replication publisher (us-east) starts sending rows with `priority` column. eu-west subscriber receives rows with a column it doesn't know about. Subscription stalls.

*Detection:* `schema_version_drift` alert fires: us-east is on version 47, eu-west is on version 46 for > 5 minutes. Also: eu-west replication lag starts growing as the subscription stalls.

*Impact:* eu-west is serving reads from an increasingly stale snapshot. Any writes from eu-west users (via home-region affinity) still work (eu-west has its own primary for EU users). But eu-west readers see data from before the migration.

*Recovery:* Runbook: (1) Identify failed migration via `SchemaVersion` table, (2) Reapply migration to eu-west primary, (3) Resume logical replication subscription, (4) Verify lag recovers to < 500ms within 10 minutes.

*Prevention:* Migration fan-out service verifies schema hash via `SchemaVersion.migration_hash` before declaring migration complete. Block application deployments that depend on the new schema until all regions report `status='applied'`.

---

### 6f. Observability Maturity Assessment

Our design achieves **Level 3 — Observability**:

We can answer arbitrary questions about system behavior without deploying new code:
- "Why did Tokyo users see stale reads at 10:32am?" → RYOW trace + replication lag metric timeline
- "Did the failover cause any data loss?" → FailoverEvent table + WAL LSN comparison at promotion time
- "Which region is responsible for the compliance risk?" → CDC stream from compliance monitoring service

To reach **Level 4 — Proactive:**
- Implement ML-based anomaly detection on replication lag (predict lag spikes from batch job patterns, not just threshold alerts)
- Auto-throttle batch jobs when WAL lag crosses 500ms (auto-remediation, not just alerting)
- Predictive disk-full alerts: extrapolate WAL accumulation rate and alert when "disk full in < 4 hours" (not when disk is at 90%)
- Chaos engineering: monthly regional failover drills in staging to validate RTO measurements (not just monitor them)

---

### 6g. Conway's Law Implications

This architecture implies at least 3 distinct teams with clear ownership boundaries:

**Database Infrastructure Team** — owns PostgreSQL primary/replica topology, Patroni configuration, failover runbooks, WAL archiving. This team's KPI is RTO + RPO. They are on-call for all Tier-1 database alerts.

**DAL / Data Platform Team** — owns the Database Access Layer, RYOW implementation, routing logic, the migration fan-out orchestrator. Their KPI is cross-region read latency and RYOW correctness. They own the service that all application teams depend on.

**Compliance / Data Residency Team** — owns the logical replication filter configuration, the compliance monitoring service, EU data scan jobs, and the legal team interface. Their KPI: 100% compliance, zero breaches.

**Coordination risk:** The write SLO spans all three teams. If APAC writes are slow (DAL routing), that's the DAL team. If the primary is overwhelmed (DB Infra team), that's a different on-call. If the latency spike is caused by replication lag (which affects RYOW primary fallbacks, which causes the DAL to route to primary, which overwhelms it) — the root cause traverses all three teams.

**Resolution:** The write success rate SLO is owned by a **virtual reliability team** that includes representatives from all three. The error budget is shared. When the SLO is burning, all three teams are paged together. This prevents finger-pointing and forces co-ownership of the end-to-end user experience.

As the Google SRE book documents: "Hope is not a strategy." Neither is "that's the other team's component."

---

### 6h. The Three Hard Questions

**Q1: "If you had to rebuild this system for 10× scale from Day 1, what are the top 3 things you would design differently?"**

First, I would implement regional write affinity with per-region primaries from day one — not as an afterthought when the single primary hits its ceiling. The migration from single-primary to multi-primary is one of the most expensive database migrations in existence, and it becomes harder as the dataset grows. Second, I would choose CockroachDB or Google Spanner for the primary data store — systems designed for global distribution from the ground up, rather than bolting multi-region onto a system (PostgreSQL) designed for single-machine operation. The operational complexity is comparable, but the CRDB approach eliminates entire categories of failure modes (split-brain, RYOW gymnastics). Third, I would design the application data model around region affinity from the first day — making `region_key` a mandatory first-class routing attribute enforced at the ORM layer, not a column that engineers add to queries inconsistently.

**Q2: "If your on-call engineer gets paged at 3am with 'users can't write records,' walk me through exactly which dashboard they open first."**

First dashboard: **Write Success Rate by Region** (Grafana). Which regions are failing? Is it all regions (primary is down) or one region (that region's DAL or write routing is broken)? Second: **DAL Error Rate by Error Type** — is it `PRIMARY_UNAVAILABLE` (Patroni failover needed), `TIMEOUT` (network issue), or `CONNECTION_POOL_EXHAUSTED` (too many connections)? Third: **etcd Cluster Health** — is Patroni consensus broken? If the etcd cluster has lost quorum, no failover can happen. The runbook step: check if etcd has < 2 of 3 members healthy. If yes, restore etcd first. The engineer should be able to identify the root cause within 5 minutes using these three dashboards, without touching production systems. If they can't, the runbook is insufficient and needs to be updated after the incident.

**Q3: "What assumption in your design are you least confident about, and what early signal in production would tell you that you got it wrong?"**

The assumption I'm least confident about is that **99% of writes are user-affined and cross-region write conflicts are rare enough to be handled by application-level logic.** My design avoids true active-active conflict resolution by assuming users mostly write their own data. If the product evolves to include real-time collaborative features (multiple users editing shared documents), this assumption breaks — conflict rates will spike and LWW or CRDT-based resolution becomes necessary. The early signal: monitor `cross_region_write_conflict_rate` (writes touching the same record ID from different regions within 1 second). If this exceeds 0.01% of writes, my assumption is wrong and we need to revisit the conflict resolution strategy. This is a business product metric as much as a technical one — the on-call engineer seeing this rate climb should escalate to the product team, not just the infra team.

---

## SECTION 7: LEVEL EXPECTATIONS

**Mid-level (E4):** Describes single-region replication with a standby. Mentions read replicas for geographic distribution. Can explain the difference between synchronous and asynchronous replication. Observability: knows that lag is a metric you should monitor.

**Senior (E5):** Designs DAL with read routing, explains quorum-based replication with specific PostgreSQL syntax, identifies the RYOW problem and proposes LSN-based tokens. Defines RED metrics per service, articulates SLIs/SLOs for write and read paths. Knows Patroni exists and how leader election works. May miss STONITH fencing details, cost observability, Conway's Law implications.

**Principal/Staff+ (E6/E7):** All of the above, plus: raises data residency compliance unprompted, designs logical replication filtering for region-pinned data, identifies the WAL slot accumulation time bomb and proposes `max_slot_wal_keep_size` mitigation, brings up tail-based trace sampling cost math, designs the 3am runbook experience before finishing each component, connects alert ownership to team structure, explicitly calls out the "10× scale cliff" and when to revisit the single-primary assumption, and discusses the tradeoffs of when NOT to use active-active (rather than defaulting to it as the "advanced" answer).

---

*Document generated for interview preparation purposes. References: PostgreSQL Streaming Replication documentation, Patroni HA framework, Google SRE Book (Chapters 4–6 on SLOs and Alerting), "Designing Data-Intensive Applications" (Kleppmann) — Chapter 5 on Replication, Discord's database migration case studies, Notion's CRDT implementation.*
