# Distributed Lock Manager — Principal-Level System Design

> **Style:** Bad → Better → Best · Full Observability · Principal Engineer Depth  
> **References:** Kleppmann (2016) "How to do distributed locking", Google Chubby (Burrows 2006), etcd docs, Martin Fowler on fencing tokens

---

## Table of Contents

1. [Understanding the Problem](#section-1-understanding-the-problem)
2. [The Setup](#section-2-the-setup)
3. [High-Level Design — The Evolving Core](#section-3-high-level-design--the-evolving-core)
4. [Deep Dives](#section-4-deep-dives)
5. [Observability — The Principal's First-Class Concern](#section-5-observability--the-principals-first-class-concern)
6. [Principal Engineer Lens](#section-6-principal-engineer-lens)
7. [Level Expectations](#section-7-level-expectations)

---

## Section 1: Understanding the Problem

### What Is a Distributed Lock Manager?

A distributed lock manager (DLM) is a service that provides mutual exclusion across multiple processes or machines. When a single-machine `synchronized` block or a local mutex isn't enough — because your application runs across dozens of pods in Kubernetes, or across multiple data centers — you need a coordination primitive that the entire cluster agrees on. Concretely: a payment service that must deduct a wallet balance exactly once, a cron scheduler that must not run the same job on two nodes simultaneously, or an inventory system that must not oversell the last item all need a distributed lock. The DLM is the service they call to serialize that critical section.

---

### Clarifying Questions

**Q: What's the primary use case — short critical-section locks (seconds) or long-lived resource leases (minutes to hours)?**  
*Why it matters:* This changes everything. Short locks favor in-memory stores with low TTLs and aggressive lease renewal. Long-lived leases require durable storage, because a Redis restart would silently unlock all held locks and cause double-processing. The failure model is completely different.

**Q: What consistency guarantee does the business actually need — can we tolerate a brief period where two holders think they hold the lock?**  
*Why it matters:* This is the hardest question. If the answer is "absolutely never" (financial transactions), we must sacrifice availability and use a CP system. If "extremely rarely is fine" (background job scheduling), an AP system with fencing tokens becomes viable. Getting this wrong costs money or correctness.

**Q: What is the target scale — how many distinct locks, how many concurrent acquisition attempts per second?**  
*Why it matters:* 10K lock ops/sec on a few hundred distinct locks is a very different design from 1M lock ops/sec across millions of keys. The former fits in a single replicated Raft group; the latter requires lock namespace sharding.

**Q: What happens when the lock service itself is unavailable — should callers fail-open or fail-closed?**  
*Why it matters:* Fail-open means the service degrades gracefully under lock-service outages but loses safety. Fail-closed means the entire system halts when the lock service is down. Neither is always right; the answer determines our availability SLO for the lock service itself.

**Q: Do lock holders need to perform work that outlasts the lock TTL (e.g., a database migration that takes 10 minutes)?**  
*Why it matters:* If yes, we need lease renewal (heartbeat) APIs, and we need to carefully design what happens when a holder goes silent — automatic expiry is a safety net but must not expire too aggressively.

**Q: Is cross-datacenter locking required, or is this within a single region?**  
*Why it matters:* Cross-DC locking at correctness levels requires a global consensus protocol like Paxos or Raft across regions, which means 100–200ms round-trip latencies. Single-region allows sub-5ms locks. The answer shifts the whole technology stack.

**Q: Do callers need to know the identity of the current lock holder?**  
*Why it matters:* Storing holder identity enables deadlock detection graphs and observability into "who is holding the lock and for how long" — invaluable at 3am. But it adds schema complexity.

---

### Functional Requirements

| Priority | Requirement | Description |
|---|---|---|
| **P0** | **Acquire** | A caller can try to acquire a named lock, optionally with a timeout. Returns success+token or failure. |
| **P0** | **Release** | A lock holder can explicitly release its lock using the token received at acquisition — no other caller can release it. |
| **P0** | **Auto-expiry (TTL)** | Every lock has a TTL. If not renewed or released before TTL, the lock is automatically freed — prevents dead-lock on holder crash. |
| **P0** | **Renew (Heartbeat)** | A holder can extend its TTL before expiry — allows long-running critical sections without holding a permanent lock. |
| **P0** | **Fencing Token** | Every lock acquisition returns a monotonically increasing token. Downstream systems use this to reject stale requests from zombie holders. |

**Out of Scope:**
- **Distributed deadlock detection graph** — complex to maintain at scale; operators can detect via TTL expiry and lock timeout metrics.
- **Read-write locks** — shared/exclusive semantics require significantly more protocol complexity; out of scope for V1.
- **Cross-datacenter global locks** — requires Paxos/Raft across WAN; we assume single-region for this design.

---

### Non-Functional Requirements

#### Availability
**Target: 99.99%** (~52 minutes downtime/year). A distributed lock service is infrastructure — if it's down, all services that depend on it are also degraded or down. This forces at minimum 3-node Raft quorum with no SPOF. Single-region, multi-AZ deployment is the minimum viable topology.

#### Consistency
**We choose CP.** This system is purpose-built to provide mutual exclusion. A brief window where two callers both believe they hold the same lock defeats the entire purpose. We accept that under a network partition, the minority partition will block rather than proceed. *Analogy: A traffic light that shows green on both sides to avoid "unavailability" is worse than one that goes dark.*

#### Latency

| Metric | Target | Reasoning |
|---|---|---|
| Acquire p50 | < 2ms | Single-region Raft commit |
| Acquire p99 | < 10ms | Must include full Raft round-trip |
| Release p99 | < 5ms | Also a Raft write |
| Renew p99 | < 5ms | Critical — must beat TTL reliably |

Why p99 < 10ms for acquire? Because most callers are on hot paths — a payment checkout that must acquire a wallet lock on every transaction. If 1% of those take 100ms, users perceive checkout as slow, even though the lock service isn't technically "broken."

#### Throughput
**Math:** Assume 5,000 application pods each making ~10 lock operations/second on average, with 5x burst headroom = **250K lock ops/sec peak**. Each lock op is a Raft-committed write of ~100 bytes. At 250K ops/sec × 100 bytes = 25MB/s write throughput — well within a modern NVMe node's capabilities. The bottleneck is Raft round-trips (consensus latency), not disk I/O.

#### Durability
Lock state does NOT need to survive a full cluster restart. Locks have TTLs; they will expire naturally. What must be durable: the fencing token counter (monotonically increasing — must never go backwards after a restart, or an old zombie could use a valid-looking token). **RPO = 0 for the fencing token sequence.**

#### Scalability
10x growth means 2.5M lock ops/sec. A single Raft group tops out around 500K–1M ops/sec depending on hardware. At 10x, we need lock namespace sharding: route locks by `hash(lock_name) % num_shards`, each shard backed by an independent Raft group. This is the architectural inflection point to design for.

#### Security
Callers authenticate via mTLS (service-to-service). Lock operations require a service identity (SPIFFE/SVID). Authorization: a service can only acquire locks in its designated namespace (e.g., `payments/*`). Release requires presenting the exact token received at acquisition.

#### Observability SLOs
We must know within **30 seconds** if lock acquisition failure rate exceeds 1%. Within **5 minutes** if p99 acquire latency trends above 10ms. Within **1 minute** if any lock is held for more than 5× its requested TTL (zombie holder).

---

### Capacity Estimation

| Metric | Estimate | Reasoning |
|---|---|---|
| Peak lock ops/sec | **250K ops/sec** | 5K pods × 10 ops/s × 5x burst |
| Avg lock state per entry | **~300 bytes** | lock_name + holder_id + token + TTL + metadata |
| Concurrent locks held | **~500K** | avg lock hold time ~200ms × 250K ops/s = 50K; with TTL headroom ×10 |
| Total in-memory state | **~150MB** | 500K × 300B — trivially fits in RAM |
| Raft log throughput | **~25MB/s** | 250K × 100B/entry |
| Raft log retention (1 hour) | **~90GB** | 25MB/s × 3600s; compact aggressively |
| Network (intra-cluster replication) | **~75MB/s** | 25MB/s × 3 replicas |

**Dominant constraint: Raft consensus latency.** This is not CPU-bound or I/O-bound — it's network-round-trip-bound. Our latency SLOs are directly determined by network topology.

---

## Section 2: The Setup

### Core Entities

| Entity | What It Represents | Primary Access Pattern |
|---|---|---|
| **Lock** | A named mutual exclusion resource. Identified by `namespace/resource_name`. | Point lookup by lock key. Extremely hot reads; writes must be linearizable. |
| **LockHolder** | An active grant. Contains holder identity, token, expiry timestamp, and acquired_at. | Read on every acquire attempt to check current holder. Written on acquire/release. |
| **FencingSequence** | A per-lock monotonic counter. Increments on every successful acquire. | Read+increment on acquire. Must never regress. |
| **WaitQueue** | Ordered list of callers waiting for a lock. Used for fairness (FIFO acquisition). | Push on blocked acquire, pop on release. Append-heavy. |
| **LockAuditLog** | Append-only log of all acquire/release/expire/renew events with timestamps. | Write-only hot path; read during incident investigation. |

---

### API Design

We use **gRPC** for the lock service API — not REST. Justification: lock operations are high-frequency, latency-sensitive, and benefit from HTTP/2 multiplexing and binary protobuf encoding. REST would add ~30–50% overhead.

```protobuf
// Proto definition (simplified)

service LockService {
  rpc Acquire(AcquireRequest) returns (AcquireResponse);
  rpc Release(ReleaseRequest) returns (ReleaseResponse);
  rpc Renew(RenewRequest)    returns (RenewResponse);
  rpc Inspect(InspectRequest) returns (InspectResponse); // debug/observability
}

message AcquireRequest {
  string lock_key    = 1; // "payments/wallet:user_123"
  string caller_id   = 2; // SPIFFE SVID of caller
  int32  ttl_seconds = 3; // requested TTL; server may cap
  int32  wait_ms     = 4; // 0=try-lock, -1=block forever, >0=timeout
}

message AcquireResponse {
  bool   acquired      = 1;
  string token         = 2; // UUID; required for release/renew
  int64  fencing_token = 3; // monotonically increasing; pass to downstream
  int64  expires_at_ms = 4;
  string holder_id     = 5; // who currently holds it (if not acquired)
}

message ReleaseRequest {
  string lock_key = 1;
  string token    = 2; // must match token from AcquireResponse
}
```

**Auth:** mTLS at the transport layer. The `caller_id` is extracted from the client certificate's SPIFFE SVID — not trusted from the request body.

> ⚠️ **API Evolution Note:** When we introduce sharding, the client library must handle routing transparently. When we add wait queues, the `Acquire` response must support a streaming/long-poll variant.

> 🔴 **3AM Test:** If all Acquire RPCs start returning `UNAVAILABLE`, our observability must catch it within 30 seconds via error rate alert on the `LockService/Acquire` gRPC method.

---

## Section 3: High-Level Design — The Evolving Core

### Requirement: Acquire a Distributed Lock

#### ❌ Naive — Single Redis SETNX

Use Redis `SETNX lock_key holder_id EX ttl`. If it returns 1, you hold the lock. If 0, someone else does. Release by `DEL lock_key`.

```
Client A            Redis (single node)      Client B
   |                        |                        |
   |--- SETNX lock_key ---->|                        |
   |<-- 1 (acquired) -------|                        |
   |                        |<--- SETNX lock_key ----|
   |                        |---- 0 (denied) ------->|
   |--- DEL lock_key ------>|
   |                        |<--- SETNX lock_key ----|
   |                        |---- 1 (acquired) ------>|
```

**Why it breaks:**
- **Race condition on release:** Client A acquires with TTL=10s. A GC pause causes A to take 15s in its critical section. Redis auto-expires the lock at 10s. Client B acquires. Client A wakes up and `DEL`s — deleting B's lock. Now B is doing unsafe work without a lock.
- **Single point of failure:** One Redis node. When it dies, all lock operations fail. No quorum, no replication guarantee.
- **No fencing tokens:** A slow A can submit a stale write to a downstream DB after its lock has expired. The downstream can't detect this.

---

#### ⚠️ Better — Redis with Lua Script + Token Validation

Fix the release race with an atomic Lua script that checks token before deleting. Fix single-node failure with Redis Sentinel.

```lua
-- Atomic release: only delete if token matches
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

**New problems introduced:**
- **Sentinel failover is not atomic:** During the ~30s sentinel failover window, a new primary may not have received the last few writes from the old primary — two holders simultaneously.
- **Still no fencing tokens:** Downstream services still can't detect zombie writes from expired-lock holders.
- **Clock skew on TTL:** If the Redis node's clock drifts, TTLs behave unpredictably across restarts.

---

#### ✅ Good (Senior Bar) — Raft-Based Lock Service

Build (or deploy) a dedicated lock service backed by a Raft consensus group: 3 or 5 nodes. Every lock operation is a Raft-committed entry — only visible after a quorum of nodes acknowledges it.

```
     Client                  Lock Service Cluster
        |                 ┌──────────────────────────┐
        |─── Acquire ────►│   Leader Node (Raft)     │
        |                 │   ┌──────────────────┐   │
        |                 │   │  In-Memory Lock  │   │
        |                 │   │  State (applied) │   │
        |                 │   └──────────────────┘   │
        |                 │      ▲  replicate  ▼      │
        |                 │  Follower A   Follower B  │
        |                 └──────────────────────────┘
        |◄── token + fencing_token=42 ───────────────|

     Client ────► Payment DB: UPDATE wallet SET balance=...
                  WHERE fencing_token > last_seen_token (= 41)
                  ✅ Accepted (42 > 41)

     Zombie A ──► Payment DB: UPDATE wallet...
                  WHERE fencing_token > last_seen_token (= 42)
                  ❌ Rejected (40 < 42) — stale write blocked!
```

```sql
-- Lock table schema (replicated via Raft)
CREATE TABLE locks (
    lock_key        TEXT PRIMARY KEY,       -- "ns/resource"
    token           UUID NOT NULL,          -- random; required for release
    fencing_token   BIGINT NOT NULL,        -- global monotonic counter per lock
    holder_id       TEXT NOT NULL,          -- SPIFFE SVID
    acquired_at     TIMESTAMPTZ NOT NULL,
    expires_at      TIMESTAMPTZ NOT NULL,
    ttl_seconds     INT NOT NULL,
    renew_count     INT DEFAULT 0
);

-- Fencing sequence (never resets, even across restarts)
CREATE TABLE fencing_sequences (
    lock_key        TEXT PRIMARY KEY,
    next_token      BIGINT NOT NULL DEFAULT 1
);
```

**Remaining gaps a Principal would notice:**
- The downstream database must actually implement the fencing token check — most don't do this out of the box. Requires a `last_seen_fencing_token` column with a check constraint on writes.
- TTL expiry computed from wall clock on every read (not stored as boolean), but wall clock drift between nodes can cause subtle bugs.
- Wait queue fairness is unimplemented — callers must busy-poll or implement exponential backoff themselves.

> 🔴 **3AM Test:** If a Raft leader election is in progress (~150–300ms), all acquire operations return `LEADER_NOT_AVAILABLE`. Alert: `lock_acquire_errors > 5%` for 10 seconds → Sev-1.

---

### Requirement: TTL Auto-Expiry

#### ❌ Naive — Background Reaper Thread
A single background thread scans the locks table every second, finds rows where `expires_at < now()`, and deletes them. At 500K concurrent locks, this is O(n) — a CPU bottleneck. A reaper crash leaves expired locks held indefinitely.

#### ✅ Good — Lazy Expiry + Periodic Compact

**Primary expiry: lazy.** On every `Acquire` or `Inspect`, the service checks `expires_at < now()` before returning state. If expired, treat the lock as unheld. O(1) per access.

**Secondary expiry: periodic compaction.** Every 30 seconds, the leader runs a compaction sweep on a bounded range (10K rows), deleting verified-expired entries. This keeps the table clean without blocking.

**Expiry as a Raft entry:** The expiry of a lock is recorded as an explicit Raft log entry with a timestamp — all followers agree on when the lock expired, no clock-skew ambiguity.

---

### Requirement: Fair Waiting on Busy Locks

#### ❌ Naive — Client-Side Busy Poll
Callers retry in a loop: `while !acquired: sleep(50ms); acquire()`. Creates a thundering herd: when a popular lock is released, all 1000 waiting callers hammer the service simultaneously.

#### ⚠️ Better — Exponential Backoff with Jitter
Add jitter: `sleep(50ms + random(0, 50ms))`. Spreads out retries. But at 10K concurrent waiters on a hot lock, this is still 10K RPCs hitting the service within a 100ms window.

#### ✅ Good — Server-Side Wait Queue with gRPC Streaming

The `Acquire` RPC with `wait_ms > 0` enqueues the caller server-side and holds the gRPC stream open. When the lock is released, the server wakes exactly the head of the FIFO queue. O(1) thundering herd eliminated.

```
Lock "pay:123" released
       │
       ▼
 Wait Queue: [B, C, D, E, ...]
       │
       ▼ notify head only
      [B] ─── AcquireResponse{acquired=true, token=...} ──► Client B
      [C, D, E] remain in queue, streams still open
```

> **Principal insight:** The wait queue itself must be durable via Raft — if the leader crashes mid-wait, waiters on the new leader must be re-notified. Waiters reconnect and re-enqueue on leader change, which the client library handles transparently.

---

### Requirement: Lease Renewal (Heartbeat)

#### ❌ Naive — Manual Renew with Long TTL
Give callers a 10-minute TTL so they don't have to worry about renewal. A caller that crashes while holding a lock ties up that resource for 10 minutes. In a payment system, that means a user can't process payments for 10 minutes.

#### ✅ Good — Auto-Renewal via Client Library

The client library spawns a background goroutine that calls `Renew` at `TTL / 3` intervals. If the application crashes, the goroutine dies with it, the heartbeat stops, and the lock expires at TTL.

```go
// Client library (Go pseudocode)
func WithLock(key string, ttl time.Duration, fn func()) error {
    resp, err := client.Acquire(key, ttl, 0)
    if err != nil { return err }

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Auto-renew at TTL/3 intervals
    go func() {
        ticker := time.NewTicker(ttl / 3)
        for {
            select {
            case <-ticker.C:
                client.Renew(key, resp.Token, ttl)
            case <-ctx.Done():
                return
            }
        }
    }()

    defer client.Release(key, resp.Token)
    fn() // execute critical section
    return nil
}
```

---

### Full Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Application Tier (Callers)                       │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│   │ Payment Pod  │  │ Scheduler Pod│  │ Inventory Pod│  ...        │
│   │ [DLM Client] │  │ [DLM Client] │  │ [DLM Client] │            │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘            │
│          │ gRPC/mTLS        │                  │                    │
└──────────┼──────────────────┼──────────────────┼────────────────────┘
           │                  │                  │
           ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Lock Service (DLM)                               │
│   ┌──────────────────────────────────────────────────────────────┐ │
│   │          gRPC Load Balancer (L7, client-side via Envoy)      │ │
│   └────────────────────────┬─────────────────────────────────────┘ │
│                            │                                        │
│   ┌────────────────────────▼─────────────────────────────────────┐ │
│   │            Raft Cluster (3 nodes, multi-AZ)                  │ │
│   │  ┌──────────────────┐  ┌────────────────┐  ┌─────────────┐  │ │
│   │  │  LEADER (AZ-1)   │  │ Follower (AZ-2)│  │ Follower    │  │ │
│   │  │  Lock State      │◄─┤  replicated    ├──►  (AZ-3)     │  │ │
│   │  │  (in-memory)     │  │                │  │             │  │ │
│   │  │  + WAL           │  └────────────────┘  └─────────────┘  │ │
│   │  │  + Wait Queues   │◄──── Leader handles writes             │ │
│   │  └──────────────────┘      Followers handle reads            │ │
│   └──────────────────────────────────────────────────────────────┘ │
│   ┌──────────────────────────────────────────────────────────────┐ │
│   │  Metrics Exporter (Prometheus) │ Audit Log → Kafka           │ │
│   └──────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
           │ fencing_token in response
           ▼
┌────────────────────────────────────┐
│      Downstream Resource           │
│  e.g. Payment DB                  │
│  CHECK: fencing_token >            │
│         last_accepted_token        │
└────────────────────────────────────┘
```

**Critical Path: Payment Lock Acquisition**

1. Payment Pod calls `Acquire("payments/wallet:user_123", ttl=5s, wait=200ms)` via gRPC/mTLS.
2. Lock Service leader receives request. Checks current lock state (in-memory O(1)). Lock is free.
3. Leader proposes a Raft log entry: `{op: ACQUIRE, key: "payments/wallet:user_123", token: uuid, ttl: 5s}`.
4. Raft entry is replicated to at least 2/3 followers (quorum). Leader commits.
5. Lock state updated in-memory. Fencing token counter incremented to 42.
6. Response: `{acquired: true, token: "abc-uuid", fencing_token: 42, expires_at: now+5s}`.
7. Payment Pod executes critical section. Passes `fencing_token=42` to DB write.
8. DB checks: `42 > last_seen_token(41)` → proceeds. Updates `last_seen_token = 42`.
9. Payment Pod calls `Release("payments/wallet:user_123", token="abc-uuid")` → committed via Raft.

> **Assumption to validate:** The downstream DB (steps 7–8) implements fencing token validation. This is NOT automatic. It requires schema changes and application-level enforcement on every write path. In practice, this is the hardest adoption challenge.

---

## Section 4: Deep Dives

### Deep Dive 1: The Fencing Token Problem — Making Correctness End-to-End

Client A holds a lock and pauses for 20 seconds (GC pause). The lock expires. Client B acquires the lock. Client A resumes — it has no idea its lock expired. Both A and B now write to the same resource. The lock service did its job perfectly; the problem is Client A doesn't know it lost the lock.

#### ❌ Naive — "Check-then-Act" by Client
Client A checks if it still holds the lock → yes → proceeds to write. Between the check and the write, the lock could expire. Classic TOCTOU race. No amount of client-side checking fixes this.

#### ⚠️ Better — Lease Renewal / Heartbeat
Client renews the lock every TTL/3. If the renewal fails, the client aborts its critical section. But: Client A's renewal succeeds at T=0, Client A pauses for TTL+1 seconds (very long GC pause), lock expires, Client B acquires, Client A resumes thinking it still holds the lock. Heartbeats reduce but don't eliminate the window.

#### ✅ Good — Monotonic Fencing Tokens + Downstream Enforcement

```
Timeline:
  T=0:  Client A acquires lock → fencing_token = 41
  T=5s: Client A pauses (GC pause)
  T=10s: Lock expires (TTL=10s)
  T=10s: Client B acquires lock → fencing_token = 42
  T=15s: Client B writes to DB with fencing_token=42 → ACCEPTED
         DB records: last_seen_token = 42
  T=20s: Client A resumes, tries to write with fencing_token=41
         DB checks: 41 ≤ 42 → REJECTED ✅

The downstream DB is the ultimate gatekeeper, not the lock service.
```

```sql
-- Downstream DB schema (payment example)
ALTER TABLE wallets ADD COLUMN
    lock_fencing_token BIGINT NOT NULL DEFAULT 0;

-- Write with fencing token check (atomic)
UPDATE wallets
SET
    balance = balance - :amount,
    lock_fencing_token = :fencing_token
WHERE
    user_id = :user_id
    AND lock_fencing_token < :fencing_token; -- reject stale writes

-- If 0 rows updated: either user not found,
-- OR fencing_token check failed (zombie write blocked)
```

> **🔍 What a Principal sees that a Senior might miss:** The fencing token counter must be durable across Raft leader elections and cluster restarts. If the fencing sequence resets to 0 after a restart, a zombie client with token=500 would see its token accepted by the downstream (500 > 0), defeating the guarantee. The solution: persist the high-water mark of the fencing counter to the WAL before any acquisition. On restart, load the last committed counter value before serving any traffic. This is a subtle initialization ordering bug that only surfaces after a crash.

---

### Deep Dive 2: Lock Hotspots — The "Celebrity User Lock" Problem

Most locks are rarely contended. But some locks are extremely hot — a celebrity user's wallet, a globally popular inventory item on sale, a cron job scheduler lock.

#### ❌ Naive — Single Raft Group for All Locks
At 250K ops/sec, a single Raft group becomes CPU-bound on the leader. Hot locks consume 80% of leader capacity, starving all other locks. Normal lock latency degrades even though the contention is on a single key.

#### ⚠️ Better — Static Sharding by Hash
`shard = hash(lock_key) % N`, each shard is an independent Raft group. Spreads load evenly on average. But hot lock contention within one shard still serializes all acquires for that one key on one shard leader.

#### ✅ Good — Hierarchical Locking + Lock Coalescing

**Lock coalescing:** The leader holds the lock "on behalf of" a micro-batch of consecutive callers, granting them each a token and processing their critical sections sequentially without a Raft commit per operation. One Raft commit grants 10 callers in sequence — amortizes consensus cost.

**Hierarchical locks:** For high-fan-out resources, introduce a parent lock. Rather than locking `payments/wallet:user_123` per-transaction, lock `payments/wallet:user_123:shard_3` (the user's balance is split into N shards). Trades off balance precision for throughput.

> **🔍 What a Principal sees that a Senior might miss:** Hot locks are often a symptom of a data model problem, not a locking problem. If every payment transaction requires exclusive access to a single wallet balance row, the real fix is sharding the balance or using an append-only ledger that doesn't require exclusive locks at all. The Principal asks: "Why does this resource need a distributed lock in the first place?" and challenges the upstream data model before reaching for a more complex locking protocol.

---

### Deep Dive 3: Network Partition During Lock Hold — The Split-Brain Scenario

A Raft cluster is split: Client A holds a lock, the Raft leader it acquired from is now in the minority partition. A new leader is elected in the majority partition. Does the new leader allow a new acquisition of the same lock?

#### ❌ Naive — Trust the New Leader Immediately
New leader checks its state: lock is expired (TTL has lapsed during the partition). Grants it to Client B. Client A resumes when the partition heals — it still thinks it holds the lock. Both A and B in critical section simultaneously.

#### ⚠️ Better — Fencing Tokens Save the Day (Partially)
If the downstream implements fencing token checks, Client A's stale operations are rejected. The lock service may transiently show two holders, but the downstream rejects A's writes. Safety preserved, but liveness suffers: Client A is executing a critical section that will be silently rolled back.

#### ✅ Good — Epoch Leases + Epoch-Aware Client Reconnection

Raft leaders include their term number (epoch) in every response. Client libraries track the epoch. On reconnection after a partition, if the client's cached epoch is lower than the current leader's epoch, the client knows a leader change occurred and invalidates its local lock state. It re-acquires or aborts, rather than proceeding with a potentially invalid lock.

Additionally: every lock response includes a "check-in deadline" — the client must re-confirm the lock by this time or the downstream should assume the lock is released. Similar to how Chubby (Google's DLM) handles leases with a safety period.

> **🔍 What a Principal sees that a Senior might miss:** There is no such thing as a "distributed lock that guarantees mutual exclusion without fencing tokens." Even with perfect Raft consensus, a slow client can hold a lock conceptually after it has expired from the cluster's perspective. The lock service provides a coordination mechanism; the downstream resource provides enforcement. Mutual exclusion is a property of the entire system, not just the lock service. A Principal designs both layers and documents their contract explicitly — including what happens if the downstream is upgraded to remove fencing token support (must be treated as a breaking change).

---

## Section 5: Observability — The Principal's First-Class Concern

A Senior builds a lock service that works. A Principal builds a lock service where, at 3am, any on-call engineer can answer: "Which lock is contended right now? Who holds it? How long have they held it? Are any locks in a zombie state? What's the p99 acquire latency for payments locks vs. scheduler locks?" — all without touching production.

---

### 5a. Metrics — The What

#### Business / Product Metrics

| Metric | Why It Matters | Alert On |
|---|---|---|
| `lock_acquire_success_rate` | The primary health signal. If this drops, services waiting on locks are stalling. | <99% for 2 min → Sev-1 |
| `lock_wait_duration_p99` by namespace | How long are callers actually waiting? Tells you if contention is user-visible. | >500ms → Sev-2 |
| `lock_zombie_count` | Locks held longer than 5× their requested TTL. Indicates a stuck holder. | Any non-zero → Sev-2 |
| `fencing_token_rejections_rate` | How often is the downstream rejecting stale writes? High rate = zombie problem. | >0.1% → Sev-2 |
| `lock_contention_ratio` by key | Requests that had to wait / total requests, per lock key. Identifies hotspots. | Any key >50% → Sev-3 |

#### RED Metrics (per Service)

| Service | Rate (RPS) | Errors (by type) | Duration (p50/p99) |
|---|---|---|---|
| Lock Service (Acquire) | `lock_acquire_rps` | TIMEOUT / LEADER_UNAVAIL / NOT_OWNER | 2ms / 10ms |
| Lock Service (Release) | `lock_release_rps` | TOKEN_MISMATCH / EXPIRED / NOT_FOUND | 1ms / 5ms |
| Lock Service (Renew) | `lock_renew_rps` | EXPIRED / TOKEN_MISMATCH | 1ms / 5ms |
| Raft Consensus Layer | `raft_commit_rps` | LEADER_ELECTION / QUORUM_LOST | 1ms / 8ms |

#### USE Metrics (per Infrastructure Component)
- **Raft Leader:** CPU utilization, commit throughput, election count (should be near-zero in steady state), log compaction lag.
- **Raft Followers:** Replication lag (entries behind leader), apply rate, snapshot transfer progress.
- **Network (intra-cluster):** Round-trip latency between AZs, packet loss rate. A 1ms increase in cross-AZ latency directly adds 1ms to all lock acquire times.
- **Memory (each node):** Lock state size, wait queue depth. Alert if lock state grows >80% of available RAM.

#### Derived Metrics (What Senior Engineers Check First)
- `lock_hold_time_p99 by namespace` — proxy for "is the critical section healthy?"
- `raft_commit_to_response_latency` — identifies serialization bottlenecks in the leader's apply loop.
- `lock_expiry_vs_release_ratio` — what % of locks expire via TTL vs. explicit release? High TTL expiry = callers crashing mid-lock.

---

### 5b. Structured Logging

```json
{
  "timestamp":    "2026-04-07T03:14:22.341Z",
  "level":        "INFO",
  "service":      "lock-service",
  "trace_id":     "a3f2c1d4-...",
  "span_id":      "b8e1...",
  "caller_id":    "spiffe://cluster/payments-service",
  "operation":    "lock.acquire",
  "lock_key":     "payments/wallet:user_123",
  "duration_ms":  3,
  "outcome":      "success",
  "fencing_token": 42,
  "wait_ms":      0,
  "raft_term":    7,
  "ttl_seconds":  5
}
```

**Log levels:**
- **ERROR:** TOKEN_MISMATCH releases, quorum loss, fencing sequence regression, Raft leader panics.
- **WARN:** Lock acquired after wait >200ms, renewal within 100ms of expiry, TTL expiry (caller may have crashed).
- **INFO:** All acquire/release/renew events — these are the audit trail. **Never sample these down.**
- **DEBUG:** Raft log replication details, wait queue operations, compaction sweeps — never in production by default.

---

### 5c. Distributed Tracing

Instrument with OpenTelemetry SDK. Every Acquire/Release/Renew RPC creates a root span. All downstream calls create child spans.

```
[POST Acquire "payments/wallet:user_123"] — 7ms total
├── [auth.validate_mtls_cert] — 0.5ms
├── [lock.check_current_state (in-memory)] — 0.1ms ← should dominate at 0
├── [raft.propose_entry] — 5ms ← this IS the bottleneck (network round-trip)
│   ├── [raft.replicate → follower_az2] — 2ms
│   └── [raft.replicate → follower_az3] — 2.5ms  ← cross-AZ latency
└── [lock.apply_and_respond] — 0.4ms
    └── [fencing_seq.increment_and_persist] — 0.2ms
```

**Sampling strategy:** Tail-based. Buffer all spans, emit 100% where any span has ERROR status or total duration >p99 threshold (10ms). 5% random sample otherwise. At 250K ops/sec, 100% sampling = 25GB/sec of span data — unacceptable. 5% + 100% errors gives full debugging coverage at ~1.25GB/sec.

---

### 5d. SLI / SLO / SLA

| User Journey | SLI | SLO (internal) | SLA (committed) |
|---|---|---|---|
| Lock Acquisition | % acquires completing in <10ms p99 | 99.9% | 99.5% |
| Lock Service Availability | % time serving requests without quorum loss | 99.99% | 99.9% |
| Mutual Exclusion Guarantee | % of releases where no concurrent holder existed | 100% (binary) | 99.999% |
| Fencing Token Monotonicity | % of tokens issued strictly greater than previous | 100% | 100% (non-negotiable) |

**Error budget:** 99.99% availability SLO = 4.38 minutes downtime/month. A Raft leader election takes ~300ms. We can afford ~880 leader elections per month before breaching SLO. Monitor: `raft_election_count_per_hour`. If it exceeds 2/hour, investigate immediately.

---

### 5e. Alerting Strategy

**🔴 Sev-1 — Page Immediately (User Impact Now)**
- Lock acquire success rate <95% for 1 minute
- Raft quorum loss on any cluster
- Fencing token sequence regression detected (critical correctness violation)
- >3 dependent services reporting elevated error rates simultaneously

**🟠 Sev-2 — Investigate Within 1 Hour**
- Lock acquire p99 >10ms sustained for 5 minutes
- Zombie lock count >0 for more than 5 minutes
- Raft leader election rate >2/hour
- Wait queue depth >10K
- Lock namespace reaching 90% of any node's memory budget

**🔵 Sev-3 — Fix This Sprint**
- Any lock key with contention ratio >50%
- Raft log compaction lag growing week-over-week
- Error budget burn >20% consumed in one week
- Lock hold time p99 increasing >10% week-over-week

> **Alert fatigue rule:** Every alert must be (1) Actionable — there is a runbook for it, (2) Urgent — it can't wait until morning, (3) Real — false positive rate <5%. Review and prune monthly.

---

### 5f. Cost Observability

- **Intra-cluster replication bandwidth cost:** At 75MB/s × $0.01/GB = ~$64/day. Alert if it grows >20% unexpectedly.
- **Raft log disk cost:** Alert if WAL disk usage grows >2× weekly average (indicates compaction failure).
- **Cost per namespace:** Tag all gRPC calls with namespace. Attribute Raft throughput cost per calling service. "If the payments team is responsible for 80% of lock ops, they should know that before requesting a 5× throughput increase."

---

## Section 6: Principal Engineer Lens

### 6a. The 3 Hidden Time Bombs

**⏱ Time Bomb 1 — Fencing Token Adoption Debt**  
At month 6, you discover 3 of your 12 downstream services never implemented fencing token validation. At 10M lock ops/month, one bad week with GC pauses causes a double-debit incident. Cost to fix later: schema migrations on production tables, application code changes across multiple services, 3–4 weeks of engineering time under pressure. Early signal: `fencing_token_enforcement_coverage` metric from weekly audits. Alert if coverage <100%.

**⏱ Time Bomb 2 — Lock Namespace Explosion**  
At 18 months, 50 new services have adopted the lock service with no namespace governance. You have 500 namespaces, 50 of which are abandoned. Lock state grows unboundedly; memory grows 40% over 6 months. Early signal: `namespace_count` metric growing >10% month-over-month without a corresponding new service launch. Alert at 100 namespaces — require governance review.

**⏱ Time Bomb 3 — Client Library Version Drift**  
At 24 months, 30% of services still run v1 of the client library. A Raft leader election causes v1 clients to enter a retry storm — generating 50× normal traffic against the new leader during a 300ms election window, causing cascading failure. Early signal: track `client_library_version` as a label on every gRPC call. Alert if any version <(current-1) represents >5% of traffic after 90 days of new version GA.

---

### 6b. Technology Selection Justification

| Criteria | Raft-based DLM (etcd/custom) | Redis + Redlock | ZooKeeper |
|---|---|---|---|
| Consistency under partition | ✅ CP — minority partitions block | ❌ Not CP — Redlock has known correctness holes (Kleppmann 2016) | ✅ CP — ZAB protocol |
| Acquire latency p99 | ✅ 5–10ms (single-region) | ✅ 2–5ms (no consensus overhead) | ⚠️ 15–30ms (ZAB is slower) |
| Fencing token support | ✅ Native monotonic sequence | ❌ Must bolt on externally | ✅ zxid serves as fencing token |
| Operational complexity | ⚠️ Must run and maintain Raft cluster | ✅ Redis ops teams exist everywhere | ❌ ZooKeeper is notoriously hard to operate |
| **Verdict** | **✅ Chosen — correctness is non-negotiable** | ❌ Correctness holes at edge cases | ❌ Operational burden, slower latency |

> **Note:** In practice, most teams use **etcd** rather than a custom Raft implementation. etcd provides exactly this: a CP key-value store with linearizable reads/writes and a monotonic revision number that serves as a perfect fencing token (`ModRevision` on each key).

---

### 6c. Full Tradeoff Table

| Decision | Chosen | Alternative | We Gain | We Give Up | Revisit When |
|---|---|---|---|---|---|
| Consistency model | CP (Raft) | AP (Redis) | Guaranteed mutual exclusion | Availability under partition | Never — correctness is the product |
| Transport protocol | gRPC/mTLS | REST/HTTPS | 30–50% lower latency, streaming | Browser/curl accessibility | If we need a web dashboard client |
| Lock expiry mechanism | Lazy + compaction | Active reaper thread | O(1) per access, no background load | Stale entries linger in storage | If storage costs become significant |
| Wait queue | Server-side FIFO | Client-side backoff | No thundering herd, fairness | Server memory for queue state | If queue state becomes a memory concern at 10M waiters |
| Trace sampling | Tail-based 5%+100% errors | Head-based 100% | 90% cost reduction on trace storage | Missed rare errors not caught by tail rules | If trace storage cost <$10K/month |

---

### 6d. Scale Evolution Plan

**Phase 1 — MVP (0 → 10K ops/sec)**  
Architecture: Single etcd cluster (3 nodes, same AZ). One Lock Service deployment (3 replicas for HA). Client library for Go and Java. No sharding. No wait queue — client-side backoff with jitter.  
Minimum viable observability: 3 dashboards (acquire success rate, p99 latency, Raft election count). 3 alerts (quorum loss, success rate <95%, p99 >50ms).  
Deliberately not built: namespace sharding, mandatory fencing token enforcement, wait queue, cross-AZ Raft.

**Phase 2 — Growth (10K → 250K ops/sec)**  
What breaks first: Single etcd cluster tops out around 50K ops/sec for writes. Raft commit latency climbs from 5ms to 20ms.  
What changes: 8 etcd clusters (namespace-based sharding). Server-side wait queues. Fencing token enforcement mandatory. Cross-AZ Raft for 99.99% SLO.  
Observability additions: Per-namespace latency dashboards, contention ratio per lock key, error budget burn rate alerting.

**Phase 3 — Scale (250K+ ops/sec, global)**  
Fundamental shifts: Cross-region Raft consensus requires 100–200ms round trips. At this scale, the answer is usually: "Redesign the data model — don't use distributed locks globally." If unavoidable: region-local locks with async conflict detection and saga-style compensation.  
What becomes its own team: Lock service becomes a platform product with its own SRE team, namespace governance process, and client library guild.

---

### 6e. Failure Mode Analysis

| Component | Failure Scenario | Detection | Impact | RTO/RPO | Prevention |
|---|---|---|---|---|---|
| Raft Leader | Leader node crashes | raft_election_count spikes; acquire errors spike → Sev-1 within 30s | All acquires fail for ~300ms | RTO: 300ms; RPO: 0 (WAL replicated) | 3-node quorum; multi-AZ |
| Network Partition | AZ-2 isolated from AZ-1,3 | Raft replication lag → quorum_unavailable → Sev-1 | AZ-2 clients can't acquire; AZ-1,3 continue | RTO: immediate on partition heal; RPO: 0 | Client library handles leader redirect |
| Client Library | Heartbeat goroutine panics | lock_expiry_vs_release_ratio spikes | Caller loses lock mid-operation | Depends on caller retry logic | Fencing token enforcement makes this safe |

---

### 6f. Observability Maturity Assessment

This design achieves **Level 3 — Observable**. We can answer arbitrary questions about lock state, contention, holder identity, wait times, and Raft health without deploying new code.

**To reach Level 4 (Proactive):** Build a lock contention predictor that analyzes trends in `lock_contention_ratio` and proactively shards namespaces before they become hot. Implement auto-remediation for Raft leader elections (auto-restart unhealthy nodes before they become unavailable). Set up anomaly detection on lock hold time p99 — a slow critical section often predicts an upstream timeout cascade within 10 minutes.

---

### 6g. Conway's Law Implications

The lock service architecture implies a dedicated Platform Engineering team owning the lock service and client libraries. Application teams (Payments, Inventory, Scheduling) are consumers. The risk: application teams implement their own ad-hoc locking (Redis SETNX scattered across codebases) because the DLM team's lock service is "too slow to integrate." The DLM team must provide fast onboarding — a 30-minute integration guide, a working example per language.

**SLO ownership:** The DLM team owns the lock acquire success rate SLO. But the "wallet payment success rate" SLO is jointly owned by Payments team and DLM team — because a DLM outage causes payment failures. This cross-team SLO dependency must be explicitly documented and have a named owner at the director level who can resolve priority conflicts.

---

### 6h. The Three Hard Questions

**Q1: Designing for 10× from Day 1**  
Design namespace sharding into the client library from day one — not the server topology. The client knows which namespace it's operating in, so routing to a shard is a client-side concern changeable without server API changes. Mandate fencing token enforcement from day one: make the client library refuse to compile without a `FencingTokenValidator` being supplied by the caller. Design the Raft log format to be forward-compatible — adding new log entry types without breaking followers on the old version.

**Q2: The 3am Runbook — "Users Can't Complete Payments"**  
**Dashboard 1:** Payment service error rate dashboard — confirms scope. **Dashboard 2:** Lock service acquire success rate — is it below 99%? If yes, pivot to lock service. **Step 1:** Check `raft_election_count` — is a leader election in progress? Wait 30 seconds and re-check. **Step 2:** Check `lock_zombie_count` — are locks stuck? If yes, identify the holder via `lock_key → holder_id` in the Inspect API, escalate to that service's on-call. **Step 3:** Check `lock_wait_duration_p99` — are callers waiting too long? If yes, check the contention dashboard for the specific lock key. Total time to root cause: <5 minutes with these dashboards.

**Q3: Least Confident Assumption**  
I'm least confident that all downstream services will correctly implement fencing token validation. In my experience, teams under delivery pressure skip the "hard but boring" correctness work. The early signal: run a weekly query across all services for `fencing_token_enforcement_coverage`. If any service reports zero fencing-token-rejected writes over a 30-day period, either they never have zombie writes (unlikely at scale) or they haven't implemented the check. Zero fencing rejections is a warning sign, not a green light.

---

## Section 7: Level Expectations

| Level | Design Bar | Observability Bar | Differentiating Gaps |
|---|---|---|---|
| **Mid (E4)** | Knows SETNX + TTL. Identifies single-node failure. Proposes Redis Sentinel. | Mentions metrics and logs. Knows Prometheus exists. | Misses fencing tokens entirely. Misses Raft correctness guarantees. Doesn't ask about consistency model. |
| **Senior (E5)** | Knows Redlock exists. Proposes Raft or ZooKeeper. Identifies TTL expiry races. Introduces fencing tokens. | Defines SLIs/SLOs. Knows RED/USE frameworks. Designs gRPC error rate alerts. Describes distributed tracing for lock acquisition latency breakdown. | Misses client library version drift risk. Doesn't raise fencing token adoption as org-level challenge. Misses cost observability. |
| **Principal/Staff (E6/E7)** | Challenges whether a distributed lock is the right primitive at all. Raises fencing token as a full-system concern. Designs namespace governance. Plans scale evolution. Identifies Conway's Law implications. References Chubby/etcd with specific operational tradeoffs. | Designs observability before finishing the architecture. Defines "what does a lock zombie look like in our metrics?" before building the lock. Treats alert fatigue as a first-class problem. Connects CDC from Raft WAL as free operational intelligence. | The Principal *raises* the hard questions before they're asked. |

---

> **The Single Most Distinguishing Principal Signal**
>
> A Principal engineer, after designing the lock service, says: *"Now let me challenge whether this is the right design at all. Are we locking because our data model requires serialized access to a shared mutable resource? If so, can we redesign that resource to not require mutual exclusion — an append-only ledger, a CRDT, an idempotent write with a unique constraint? The fastest distributed lock is the one you don't need."* This question — whether to build the thing at all — is the clearest signal of principal-level thinking.

---

*End of Document · Distributed Lock Manager · Principal-Level System Design*  
*References: Kleppmann (2016) "How to do distributed locking", Google Chubby paper (Burrows 2006), etcd documentation, Martin Fowler on fencing tokens*
