# System Design: Distributed Messaging Queue (Kafka-Style)

> **Interview-Ready Principal Engineer Document**
> *Narrative style: Simple → Problems Discovered → Solutions Evolved → Principal Depth*

---

## SECTION 1: UNDERSTANDING THE PROBLEM

### What Is a Distributed Messaging Queue?

A distributed messaging queue is a durable, high-throughput communication backbone that decouples producers (services that generate events) from consumers (services that react to them). Instead of Service A calling Service B directly — creating tight coupling, backpressure risk, and availability dependencies — both talk through a persistent, ordered log. Think of it as a fault-tolerant, replayable event highway: producers write messages to named *topics*, consumers read from those topics at their own pace, and the system retains the full history so that a consumer can replay past events, recover from failures, or onboard a new downstream service without re-triggering every upstream producer. Kafka pioneered this model; it underpins LinkedIn's activity feeds, Netflix's real-time pipeline, Uber's dispatch system, and tens of thousands of other mission-critical workflows.

---

### Clarifying Questions

Before drawing a single box, I'd ask the interviewer these questions:

**Q1: Is this a general-purpose system, or is it optimized for a specific workload (e.g., log aggregation vs. transactional events)?**
*Why it matters:* Log aggregation tolerates some message loss (fire-and-forget); transactional events require at-least-once or exactly-once delivery semantics. This fundamentally changes our storage guarantees and consumer acknowledgment model.

**Q2: What is the expected peak throughput — messages per second and average message size?**
*Why it matters:* At 1M msg/sec with 1 KB messages, we're pushing ~1 GB/sec of raw throughput. That determines whether we batch writes, how many partitions we need, and whether a single broker can handle it or we need a full cluster from day one.

**Q3: What delivery guarantee is required — at-most-once, at-least-once, or exactly-once?**
*Why it matters:* At-most-once is trivially cheap. Exactly-once requires idempotent producers, transactional APIs, and two-phase coordination — a 3–5× complexity multiplier. Knowing the requirement upfront saves us from over-engineering.

**Q4: What is the message retention requirement — hours, days, indefinitely?**
*Why it matters:* If we retain 7 days of data at 1 GB/sec throughput, that's 600 TB. Storage tiering (hot SSDs → warm HDDs → cold object store) becomes mandatory, not optional. This shapes the cost model entirely.

**Q5: Do consumers need to replay messages from an arbitrary offset, or only read new messages?**
*Why it matters:* Replay is Kafka's killer feature but it requires treating the log as an immutable, indexed store — not a queue you drain. It changes how we think about offset management, compaction, and consumer group semantics.

**Q6: What are the consumer patterns — single consumer per topic, or multiple independent consumer groups each reading the full log?**
*Why it matters:* Multiple consumer groups (analytics AND alerting AND archiving all reading the same topic) is where Kafka's log abstraction shines over traditional queues. It also means we can't delete-on-consume — retention policy must be time/size-based.

**Q7: Is geographic distribution required — single region, multi-region active-passive, or multi-region active-active?**
*Why it matters:* Multi-region active-active with ordering guarantees is one of the hardest problems in distributed systems. It requires global sequence numbers or causal consistency guarantees and will dominate our complexity budget. Let's confirm scope before designing for it.

*For this document, I'll assume:* general-purpose, at-least-once delivery, 1M msg/sec peak throughput, 7-day retention, multi-consumer-group replay support, single region with cross-region replication (not active-active).

---

### Functional Requirements

**P0 — Core Requirements:**

1. **Produce messages to named topics** — Producers can publish messages to a topic. The system acknowledges durability.
2. **Consume messages from topics with offset tracking** — Consumers in a named consumer group read messages from a partition, and their progress (offset) is tracked persistently so they can resume after failure.
3. **Support multiple independent consumer groups** — Multiple groups can each read the full topic independently, at their own pace, without interfering with each other.
4. **Message retention with configurable policy** — Messages are retained for a configurable period (e.g., 7 days) or until a size limit is hit, regardless of whether they've been consumed.
5. **Topic partition management** — Topics are divided into ordered partitions; producers can target partitions by key for ordering guarantees within a key.

**Out of Scope:**
- **Exactly-once semantics (EOS) end-to-end** — Requires distributed transactions across producer, broker, and consumer. Too complex for the initial design; we'll note where it can be layered in.
- **Schema registry / message validation** — Important in production (Confluent Schema Registry), but orthogonal to the core messaging architecture.
- **Multi-region active-active** — Global total ordering across regions is a PhD-level problem. We'll cover single-region with async replication.

---

### Non-Functional Requirements

**Availability: 99.99% (≈52 minutes downtime/year)**
This means we cannot tolerate single-broker failures bringing down a topic. Every partition must have at least 2 replicas, and leader election must complete in <30 seconds. A 3-broker cluster with replication factor 3 means we can lose any 1 broker with zero downtime. *Design constraint: no single point of failure in the write path.*

**Consistency: AP with strong durability guarantees**
We choose availability over strict consistency. A producer that gets an acknowledgment (`acks=all`) can be confident the message is durable on N brokers. Consumers may see slightly stale data during leader re-election (seconds), but *committed* messages are never lost. We explicitly reject CP-style synchronous cross-node confirmation for every write — that would kill throughput. *This means: a consumer may briefly re-read a message after rebalance. Idempotent consumers are expected.*

**Latency:**
- Write p50: <5ms, p99: <20ms (end-to-end broker acknowledgment)
- Read p50: <10ms, p99: <50ms (consumer fetch latency)
- *Why these targets?* At 1M msg/sec, a p99 write latency of 20ms means even our slowest 1-in-100 writes completes in one "heartbeat interval." If we allow p99 > 100ms, producer retry storms become likely during GC pauses.

**Throughput:**
- Target: 1M messages/sec ingest, 3M messages/sec egress (3 consumer groups)
- At avg 1 KB/msg: ingress ~1 GB/sec, egress ~3 GB/sec
- *Design implication:* Sequential disk I/O is mandatory. Random-access writes at 1 GB/sec would require SSDs; sequential append to a log achieves this on commodity HDDs.

**Durability:**
- RPO = 0 for committed messages. A message that received `acks=all` must survive any single broker failure.
- *Design implication:* In-Sync Replicas (ISR) must acknowledge before the producer is told the write succeeded.

**Scalability:**
- 10× growth means 10M msg/sec. What breaks first? The per-broker network card (typically 10 Gbps = ~10M msg/sec at 1 KB). So horizontal partition distribution is our primary scaling lever — add brokers, rebalance partitions.

**Security:**
- mTLS for broker-to-broker and client-to-broker connections
- ACLs per topic per consumer group (SASL/SCRAM or OAuth2-based)
- Encryption at rest for all segment files on disk

**Observability:**
- Know within **30 seconds** if any broker's write error rate exceeds 0.1%
- Know within **60 seconds** if any consumer group's lag exceeds 10,000 messages
- Know within **5 minutes** if replication lag on any partition exceeds 5 seconds
- Know within **24 hours** if storage growth deviates >20% from forecast

---

### Capacity Estimation

**Baseline assumptions:**
- 10,000 producer clients, 30,000 consumer clients
- 1M messages/sec peak ingest, 3M msg/sec peak egress
- Average message size: 1 KB
- Retention: 7 days
- Replication factor: 3

**Storage:**
```
Ingest rate:     1M msg/sec × 1 KB = 1 GB/sec raw
Daily ingest:    1 GB/sec × 86,400 sec = 86.4 TB/day
7-day retention: 86.4 TB × 7 = ~605 TB raw
With RF=3:       605 TB × 3 = ~1.8 PB total disk capacity needed
```

**Broker count (I/O-bound sizing):**
```
Sequential write throughput per broker (12-disk HDD RAID): ~2 GB/sec
Brokers needed for ingest (3× for RF):  1 GB/sec × 3 replicas / 2 GB/sec = 2 brokers minimum
Brokers needed for egress (3 CGs):      3 GB/sec / 2 GB/sec = 2 brokers minimum
With headroom (50% utilization target): 6 brokers in production cluster
Partition count:                        6 brokers × 10 partitions/broker = 60 partitions (starting point)
```

**Memory (page cache is everything):**
```
"Hot" data (last 30 min of writes): 1 GB/sec × 1800 sec = 1.8 TB
Per broker (10 partitions): 1.8 TB / 6 = 300 GB hot data
Broker RAM target: 256–512 GB (maximize OS page cache for zero-copy reads)
```

**Network:**
```
Ingress: 1 GB/sec across 6 brokers = ~167 MB/sec per broker
Egress (3 CGs + replication): ~(3+2) × 1 GB/sec = 5 GB/sec total = ~833 MB/sec per broker
10 GbE NIC = 1.25 GB/sec → need to ensure we don't saturate
```

**Dominant constraint: I/O-bound** (disk write throughput and network egress). The scaling strategy is horizontal partitioning — add brokers and rebalance partitions.

---

## SECTION 2: THE SETUP

### Core Entities

**Topic**
A named, logical stream of messages. Topics are divided into partitions. Primary access pattern: created infrequently (administrative), but every produce and consume operation references a topic by name.

**Partition**
An ordered, immutable append-only log. Each partition is the unit of parallelism, assignment, and ordering. Primary access pattern: sequential append (producer) and sequential scan from an offset (consumer). The critical insight: ordering is only guaranteed *within* a partition, not across partitions.

**Message (Record)**
The atomic unit stored in a partition. Contains: key (optional, used for partition routing), value (payload bytes), headers (metadata k/v pairs), timestamp, and offset (assigned by broker on write). Primary access pattern: written once, read many times by multiple consumer groups.

**Offset**
A monotonically increasing integer that uniquely identifies a message's position within a partition. Consumers track their committed offset per partition. Primary access pattern: read and written frequently by the consumer group coordinator (high-frequency small writes).

**Consumer Group**
A named group of consumer instances collectively reading a topic. Each partition is assigned to exactly one consumer in the group at a time. Offsets are committed per (group, partition) tuple. Primary access pattern: updated on every consumer poll cycle.

---

### API Design

These are the core APIs. Note: Kafka uses a custom binary protocol over TCP, not REST — for a new system we'd likely provide both a native binary protocol (for throughput) and a REST gateway (for accessibility). I'll design the logical API; the transport layer is a deep dive.

```
POST /topics
  Request:  { "name": "user-events", "partitions": 12, "replication_factor": 3,
              "retention_ms": 604800000 }
  Response: { "topic_id": "uuid", "name": "user-events", "partitions": [...] }
  Auth:     JWT in Authorization header (admin scope)
  Error:    409 if topic exists, 400 if partition count exceeds broker capacity

POST /topics/{topic}/produce
  Request:  { "records": [{ "key": "user-123", "value": "<bytes>",
              "headers": {"source": "checkout-svc"}, "partition": null }] }
  Response: { "results": [{ "partition": 3, "offset": 10042, "timestamp": 1710000000 }] }
  Auth:     JWT (producer scope for topic)
  Error:    503 if ISR below min.insync.replicas (message NOT written — return error, not silent drop)
  Note:     null partition = broker routes by key hash. Explicit partition overrides routing.

GET /topics/{topic}/consume?group={group}&max_records=500&timeout_ms=1000
  Request:  (consumer must have previously called /join-group or held partition assignment)
  Response: { "records": [{ "partition": 3, "offset": 10042, "key": "...", "value": "...",
              "timestamp": 1710000000 }], "next_offsets": { "3": 10043 } }
  Auth:     JWT (consumer scope for topic)
  Error:    404 if group/topic not found, 409 if partition assignment changed (rebalance in progress)

POST /consumer-groups/{group}/offsets
  Request:  { "offsets": [{ "topic": "user-events", "partition": 3, "offset": 10043 }] }
  Response: { "committed": true }
  Auth:     JWT (consumer scope for topic+group)
  Error:    409 if this consumer no longer owns the partition (rebalance happened)

GET /topics/{topic}/metadata
  Response: { "partitions": [{ "id": 0, "leader": "broker-2", "replicas": ["broker-1","broker-2","broker-3"],
              "isr": ["broker-2","broker-3"] }] }
  Auth:     JWT (any authenticated client)
```

*These APIs will evolve. The produce API will need a batching envelope for high-throughput paths. The consume API will need a streaming/long-poll variant. I'll flag these when we get there.*

---

## SECTION 3: HIGH-LEVEL DESIGN

### Requirement 1: Produce Messages to Named Topics

#### ❌ Naive Solution: Single Broker In-Memory Queue

The simplest thing: one server accepts producer connections and stores messages in memory per topic. Producers POST to `/produce`, the server appends to an in-memory list, consumers GET to drain items.

```
Producer → [Single Broker (in-memory list)] → Consumer
```

**Why it breaks:** At 1M msg/sec × 1 KB = 1 GB/sec, we exhaust a 256 GB server's memory in 256 seconds. More critically, a single broker restart loses *everything*. No durability. No replay. No partitioning. At ~1K concurrent producers, connection handling alone saturates a single-threaded server. This breaks at day one.

And here's how we'd know it's broken in production: `process_memory_bytes` hits `node_total_memory_bytes`, then the process OOM-kills. No graceful degradation — just instant data loss. We'd find out from users, not alerts.

---

#### ⚠️ Better Solution: Single Broker with Disk-Based Log

Fix the memory problem by writing to disk. Each topic partition becomes an **append-only log file** on disk. Producers append to the log; consumers read from a byte offset. The OS page cache handles the "hot data" read path via `sendfile()` (zero-copy).

```
Producer → [Broker: append to segment file] → Consumer (reads at offset)
                     ↓
              [Disk: topic-partition-0.log]
```

This is actually Kafka's core insight — a write-ahead log as the primary data structure. Sequential disk writes on commodity hardware achieve 200–500 MB/sec, which is vastly better than random I/O.

**New problem introduced:** Still a single broker. Any broker failure = complete unavailability. Also, a single partition means all 1M msg/sec must go through one write path — sequential writes are fast but we still hit single-threaded bottlenecks in the network layer. No fault tolerance whatsoever.

---

#### ✅ Good Solution (Senior Bar): Multi-Broker Replicated Partition Log

Now we introduce the real architecture:

**Partitioning:** Split each topic into N partitions. Each partition is an independent append-only log. Producers route messages by `hash(key) % num_partitions`. This parallelizes both writes and reads.

**Replication:** Each partition has one *leader* and (RF-1) *followers*. Producers always write to the leader. The leader appends to its local log and fans out to followers in parallel. The leader tracks an **In-Sync Replica (ISR)** set — followers that are caught up within `replica.lag.time.max.ms` (default 10s).

**Durability guarantee:** With `acks=all`, the producer waits until *all ISR members* have written the message before the broker returns an acknowledgment. This gives us RPO=0 for committed messages even under single-broker failure.

**Consumer offset tracking:** Consumer groups commit offsets to a special internal topic (`__consumer_offsets`), replicated and partitioned just like any other topic.

```
                    ┌────────────────────────────────────────┐
                    │           ZooKeeper / KRaft             │
                    │  (cluster metadata, leader election)    │
                    └──────────────┬─────────────────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
   [Broker 1]                 [Broker 2]                 [Broker 3]
   Leader: P0, P3             Leader: P1, P4             Leader: P2, P5
   Follower: P1, P4           Follower: P2, P5           Follower: P0, P3
        │                          │                          │
   [disk: P0.log]            [disk: P1.log]            [disk: P2.log]
   [disk: P3.log]            [disk: P4.log]            [disk: P5.log]
        │                          │                          │
        └──────────────────────────┼──────────────────────────┘
                                   │
              ┌────────────────────┴────────────────────┐
              │                                         │
        [Producer Pool]                       [Consumer Group A]
                                              [Consumer Group B]
```

**Schema for a partition segment file:**

```
# Each segment: [base_offset].log + [base_offset].index + [base_offset].timeindex
# Log format (binary, big-endian):
Record Batch {
  base_offset:        int64
  batch_length:       int32
  partition_leader_epoch: int32
  magic:              int8  (2 for current format)
  crc:                int32 (CRC32C of the rest)
  attributes:         int16 (compression, timestamp type, transactional, etc.)
  last_offset_delta:  int32
  base_timestamp:     int64
  max_timestamp:      int64
  producer_id:        int64   ← for idempotent/transactional producers
  producer_epoch:     int16
  base_sequence:      int32
  records:            [Record...]
}

Record {
  length:             varint
  attributes:         int8
  timestamp_delta:    varint
  offset_delta:       varint
  key:                bytes
  value:              bytes
  headers:            [Header...]
}
```

*Observability hook:* For this good solution, we'd instrument: `messages_in_per_sec` per partition, `bytes_in_per_sec` per broker, `under_replicated_partitions` (Sev-1 if > 0 for >30s), and `isr_shrink_rate`. ISR shrinking silently is one of the most dangerous things that can happen — it means we're one more failure away from data loss.

**Remaining gaps (what a Principal notices):** The partition count is set at topic creation and *cannot be reduced without data loss* — this is an irreversible decision. A Senior picks a number; a Principal asks "what's our partition count governance process and what happens when we need to change it?" Also: ZooKeeper is a scaling bottleneck. KRaft (Kafka's Raft-based metadata mode) is the correct modern answer — more on this in Deep Dives.

---

### Requirement 2: Consume Messages with Offset Tracking

#### ❌ Naive Solution: Stateless Pull with No Offset Persistence

Consumers call a simple GET endpoint. The broker returns the next N messages. No state is tracked — the consumer just passes "give me messages after offset X" and manages X itself, in memory.

```
Consumer A: GET /consume?topic=events&offset=1000
Consumer B: GET /consume?topic=events&offset=1000
```

**Why it breaks:** When Consumer A crashes and restarts, it has no idea what offset it was at. It either re-reads from 0 (reprocesses everything) or misses messages. At scale with thousands of consumer instances, this coordination is impossible client-side. What happens when a consumer group has 20 instances and needs to hand off a partition during scaling? There's no handoff protocol.

*3am consequence:* Consumer restarts after a deploy, re-reads 7 days of events, triggers duplicate side effects (emails sent twice, payments doubled). We'd find out from angry users at 3am. No alerting would catch it.

---

#### ⚠️ Better Solution: Server-Side Offset Commit to a DB

Store committed offsets in a relational DB. Consumer commits `(group, topic, partition, offset)` after processing. On restart, consumer reads its last committed offset and resumes.

```
Consumer → [Broker: fetch from offset] → process → commit offset to [Postgres]
Consumer restart → read offset from [Postgres] → resume
```

**New problem:** At 1M msg/sec with 100 consumer groups, offset commits hit ~100K writes/sec to Postgres — trivially saturates a single relational DB. Also, partition rebalancing across consumer instances requires a coordination protocol (who owns which partition?) that Postgres can't provide natively.

---

#### ✅ Good Solution (Senior Bar): `__consumer_offsets` Topic + Group Coordinator

The key insight: store consumer offsets *in the same system* — a special compacted topic called `__consumer_offsets`. This gives us the same durability and throughput as regular topic writes, without any external DB dependency.

**Group Coordinator Pattern:**
- Each consumer group is assigned to one broker as its *Group Coordinator* (determined by `hash(group_id) % num(__consumer_offsets partitions)`).
- Consumers join the group via a `JoinGroup` request to the coordinator.
- The coordinator elects a *Group Leader* (the first consumer to join) which runs a partition assignment strategy (Range, RoundRobin, or Sticky).
- The leader sends the assignment back via `SyncGroup`. All consumers now know exactly which partitions they own.
- Consumers send periodic `Heartbeat` requests (every `heartbeat.interval.ms`, default 3s). If the coordinator doesn't see a heartbeat within `session.timeout.ms` (default 45s), it triggers a *rebalance*.

```
Consumer A joins → GroupCoordinator on Broker 2
Consumer B joins → GroupCoordinator on Broker 2
Coordinator triggers rebalance:
  Leader (Consumer A) assigns:
    Consumer A → Partition 0, 1, 2
    Consumer B → Partition 3, 4, 5
Consumers fetch from assigned partitions.
Consumer A commits: { group: "analytics", partition: 0, offset: 50042 }
  → written to __consumer_offsets topic → replicated across ISR
```

**Offset commit schema in `__consumer_offsets`:**
```json
{
  "key":   { "group": "analytics-pipeline", "topic": "user-events", "partition": 0 },
  "value": { "offset": 50042, "metadata": "worker-hostname", "commit_timestamp": 1710000000000,
             "expire_timestamp": 1710604800000 }
}
```

The topic uses **log compaction** — only the latest offset per key is retained. This means we never grow unboundedly even though offsets are committed frequently.

*Observability:* `consumer_lag` = `(log_end_offset - committed_offset)` per partition. This is the single most important consumer metric. We expose it via a JMX exporter → Prometheus. Alert: consumer group lag > 100K messages AND not decreasing for 5 minutes = Sev-2.

**What a Principal notices:** The rebalance problem is severe. During a rebalance, *all* consumers in a group pause and stop processing. At 1M msg/sec, a 10-second rebalance causes 10M unprocessed messages of lag. The answer is **Cooperative Incremental Rebalancing** (introduced in Kafka 2.4) — only the partitions being moved are paused, not the entire group. A Senior uses the default eager rebalance; a Principal specifically asks for `CooperativeStickyAssignor`.

---

### Requirement 3: Multiple Independent Consumer Groups

#### ❌ Naive Solution: Copy-on-Read — Duplicate Stored Messages Per Group

When Consumer Group B wants to read the same topic as Group A, duplicate the messages into a separate storage area for Group B.

**Why it breaks:** 3 consumer groups = 3× storage. 10 groups = 10× storage. At 600 TB base, that's 6 PB for 10 groups. Operationally insane.

---

#### ✅ Good Solution (Senior Bar): Shared Immutable Log, Per-Group Offsets

The log is written exactly once. Each consumer group maintains its own independent pointer (offset) into the *same* immutable log. There's no concept of "consuming" a message — reading doesn't remove it. Retention is time/size-based, not consumption-based.

```
Topic: user-events / Partition 0

[offset 0] [offset 1] ... [offset 10042] [offset 10043] ← log end
     ↑                          ↑
Group "analytics"          Group "alerting"           Group "archiver"
committed_offset=0         committed_offset=10042     committed_offset=9000
(catching up)              (near real-time)           (slightly behind)
```

This is the central abstraction that makes Kafka different from traditional queues. The log is a shared, ordered, immutable fact store. Consumer groups are just sliding windows over it.

*Observability:* Per-group, per-partition lag. Not just "is any consumer behind" but "which group, on which partition, by how many messages?" We'll come back to this in the deep dive.

---

### Tying It All Together

```
                           ┌──────────────┐
                           │  KRaft       │
                           │  (3-node     │
                           │  metadata    │
                           │  quorum)     │
                           └──────┬───────┘
                                  │ metadata (topic/partition/leader info)
          ┌───────────────────────┼───────────────────────────┐
          │                       │                           │
     [Broker 1]              [Broker 2]                 [Broker 3]
     Leader: P0,P3           Leader: P1,P4              Leader: P2,P5
     ISR replication ←──────────────────────────────────────→ ISR replication
     __consumer_offsets       Group Coordinator           __consumer_offsets
          │                       │                           │
          └──────────┬────────────┘                           │
                     │                                        │
            ┌────────┴────────┐                     ┌─────────┴──────────┐
            │  Producer Pool  │                     │  Consumer Groups   │
            │                 │                     │                    │
            │ - Key-based     │                     │ Group A: analytics │
            │   partitioning  │                     │ Group B: alerting  │
            │ - Batching      │                     │ Group C: archiving │
            │ - Idempotent    │                     │                    │
            │   producer ID   │                     │ Sticky partition   │
            └─────────────────┘                     │ assignment         │
                                                    └────────────────────┘

Critical write path for "user-123 clicks buy":
1. Producer hashes key "user-123" → Partition 3 (on Broker 1)
2. Producer sends RecordBatch to Broker 1 (TCP, compressed)
3. Broker 1 appends to P3.log on disk (sequential write, OS fsync batched)
4. Broker 1 fans out to Broker 2 and Broker 3 (ISR followers for P3)
5. Followers write to their replica logs and ACK Broker 1
6. Broker 1 advances High Watermark for P3 (now visible to consumers)
7. Broker 1 returns {partition: 3, offset: 10043} to producer
8. Consumer Group "analytics" fetches P3 from offset 10040
9. Consumer processes 4 records, commits offset 10044 to __consumer_offsets
```

**Assumptions to validate:**
- Replication factor 3 with `min.insync.replicas=2` is sufficient for our durability SLO
- `acks=all` is acceptable latency-wise for our producers (adds ~5–10ms for ISR ack round-trip)
- Consumer groups will implement idempotent processing (at-least-once delivery means occasional duplicates)
- Partition count (starting at 60) is sufficient for 18 months; we have a partition migration plan

---

## SECTION 4: DEEP DIVES

### Deep Dive 1: Partition Count as an Irreversible Decision

Let's start simple. A partition is the unit of parallelism: more partitions → more consumers in a group → more throughput. So why not just create 10,000 partitions?

#### ❌ Too Few Partitions

At 60 partitions for 1M msg/sec: each partition handles ~16,666 msg/sec. With consumer groups of 60 consumers, fine. But what if we need 100 consumers? Kafka assigns exactly one consumer per partition per group. 40 consumers sit idle. We can't scale out consumption without adding partitions, and **partitions cannot be added to an existing topic without re-partitioning**, which breaks key-based ordering guarantees for in-flight messages.

#### ⚠️ Too Many Partitions

Let's say we panic and create 10,000 partitions.

- **Broker memory:** Each partition requires a file handle for its leader log and potentially followers. With RF=3, 10,000 partitions = 30,000 open file handles across the cluster. Linux's default `ulimit -n` is 1024. At scale, this causes `Too many open files` errors at the worst possible time.
- **Leader election cost:** During a broker failure, ZooKeeper/KRaft must elect new leaders for all affected partitions. At 1,000 partitions/broker, leader election takes ~30 seconds per broker failure. At 10,000 partitions/broker: 5 minutes of unavailability.
- **End-to-end latency:** Each partition batch requires a separate network round-trip for replication ACK. More partitions = more parallel replication = more broker thread contention.
- **Real data point:** LinkedIn found that beyond ~4,000 partitions per broker, end-to-end replication latency degraded nonlinearly. Netflix limits to ~2,000 partitions per broker in their production clusters.

#### ✅ Principled Partition Sizing

The right formula:

```
target_partitions = max(
    ceil(peak_produce_throughput / throughput_per_partition),
    ceil(peak_consume_throughput / throughput_per_partition),
    target_consumer_group_max_parallelism
)

# For our system:
# throughput_per_partition ≈ 10 MB/sec (sequential disk + replication)
# 1 GB/sec / 10 MB/sec = 100 partitions minimum
# With headroom and multi-CG: start at 200 partitions

# Per-broker: 200 / 6 brokers = 33 partitions/broker → well within safe limits
```

**Governance protocol (Principal-level):**
1. Start with 200 partitions. Add brokers before adding partitions.
2. New partitions: only added when current partitions are >80% saturated for >30 days.
3. When partition count changes are unavoidable: use a *shadow topic* — create the new topic with more partitions, dual-write for a transition window, then cut over consumers.
4. Key-based ordering services get dedicated topics (never shared) to isolate partition count changes.

**What does a Principal see here that a Senior might miss?**
The Senior adds partitions when throughput degrades. The Principal asks: *"What is our partition count governance process, and who approves partition count changes?"* Because a partition count change mid-stream is a data migration, not a config change. It should require a change control process, not a one-liner CLI command. Also: the Principal models partition count as a capacity planning input with 18-month horizon, not a reactive dial.

---

### Deep Dive 2: ISR Shrinkage and the Silent Durability Cliff

This is one of Kafka's most dangerous failure modes. Understanding it separates a Principal from a Senior.

#### ❌ Ignoring ISR Dynamics

Most engineers set `replication.factor=3` and `acks=all` and feel safe. They are — until a GC pause or network partition causes a follower to fall behind. When a follower falls behind by more than `replica.lag.time.max.ms` (default 10s), the *leader removes it from the ISR*.

```
Normal:   ISR = [Broker1(leader), Broker2, Broker3]   ← acks=all requires all 3
After GC: ISR = [Broker1(leader), Broker3]            ← Broker2 fell behind
```

Now `acks=all` only requires 2 brokers (Broker1 + Broker3). We're one failure away from data loss. But **no alert fired, no incident was declared.** The system is silently less durable than our SLA implies.

#### ⚠️ Better: `min.insync.replicas`

Set `min.insync.replicas=2`. Now if ISR shrinks below 2, producers receive `NotEnoughReplicasException` rather than silently getting weaker durability guarantees. This is correct — fail loudly rather than succeed silently with weakened guarantees.

**But new problem:** When ISR shrinks to 1 (leader only), every producer write fails with a 503. At 1M msg/sec, this becomes a Sev-1 immediately. The operator gets paged, but has no clear runbook: *why did the ISR shrink? Is Broker2 down? In a GC pause? Network partition?*

#### ✅ Principled ISR Observability and Recovery

**The key metrics to instrument:**

```
# Per-partition ISR tracking (expose via JMX → Prometheus):
kafka_server_replicamanager_isrshrinks_total        # counter: ISR shrink events
kafka_server_replicamanager_isrexpands_total        # counter: ISR expand events (recovery)
kafka_server_replicamanager_underreplicatedpartitions  # gauge: partitions with ISR < RF

# Per-broker lag from follower perspective:
kafka_server_replicafetchermanager_maxlag            # max messages behind across all fetched partitions
```

**ISR shrink runbook (the 3am playbook):**

```
Alert: under_replicated_partitions > 0 for > 60 seconds

Step 1: Is the follower broker UP?
  → check broker health: if down, initiate broker recovery (ETA 5 min)
  → if up, proceed to step 2

Step 2: Is it a GC pause?
  → check jvm_gc_pause_seconds_sum on the follower broker
  → if high GC: JVM tuning needed, temporarily increase replica.lag.time.max.ms

Step 3: Is it a network partition?
  → check broker-to-broker TCP metrics
  → if network: escalate to infra team

Step 4: Is it a hot partition overloading the follower?
  → check bytes_in_per_sec by partition on follower
  → if hot partition: add replica brokers, trigger reassignment
```

**Architectural safeguard (Principal insight):** Track `isr_shrink_rate` as a *weekly trend metric*, not just an instant alert. A system where ISR shrinks 10 times/day but recovers quickly is silently bleeding toward a durability cliff. The Principal asks: "What is our ISR shrink event frequency, and is it trending up?" Three ISR shrinks in a week is a Sev-3 ticket. Fifteen in a day with a pattern (same partition, same time of day) is a capacity problem that needs rebalancing.

**What does a Principal see here that a Senior might miss?**
ISR shrinkage is not a binary event. A Senior fixes it when it fires an alert. A Principal tracks the *rate* of ISR churn as a leading indicator of cluster health degradation, and treats repeated shrink-expand cycles on the same partition as a signal that the partition is overloaded — not that the network is flaky.

---

### Deep Dive 3: Consumer Rebalance — The Coordination Tax

Rebalancing is necessary. It's also the #1 cause of consumer group latency spikes in production Kafka deployments.

#### ❌ Eager (Stop-the-World) Rebalance

Kafka's original rebalance protocol: when *any* consumer joins, leaves, or crashes — all consumers in the group **revoke all their partitions** simultaneously, then wait for the coordinator to issue a new assignment. During this window: zero consumption, unbounded lag accumulation.

```
Timeline:
t=0:  Consumer C joins group (scale-out event)
t=0:  All consumers revoke their partitions
t=0+: Coordinator runs assignment algorithm
t=5s: Coordinator issues new assignments
t=5s: All consumers begin re-fetching from committed offsets

Result: 5 seconds × 1M msg/sec = 5M messages of additional consumer lag
```

At a large company doing multiple deploys per day, this means dozens of stop-the-world pauses daily. Users of downstream systems (analytics, alerting) see delayed data.

#### ⚠️ Static Group Membership

Assign each consumer instance a persistent `group.instance.id`. The coordinator knows that `instance-0` temporarily disconnected (e.g., rolling restart) vs. a new consumer instance joining. It suppresses rebalance during the `session.timeout.ms` window for known instances.

**Better, but:** Still can't avoid rebalance when adding/removing consumer count. A rolling deploy of 20 consumers triggers 20 mini-rebalances.

#### ✅ Cooperative Incremental Rebalancing

Introduced in Kafka 2.4 via `CooperativeStickyAssignor`. The key insight: **only revoke partitions that need to move**. Partitions staying with the same consumer are never revoked.

```
Before rebalance:
  Consumer A: [P0, P1, P2, P3, P4, P5]

Consumer B joins. New assignment:
  Consumer A: [P0, P1, P2] (keeps these)
  Consumer B: [P3, P4, P5] (takes these)

Phase 1 (immediate): Consumer A revokes P3, P4, P5 — continues processing P0, P1, P2
Phase 2 (next poll): Consumer B assigned P3, P4, P5 — begins fetching

Result: Consumer A never stops. Lag accumulation = only the time for Consumer B to start.
```

**Combined with static membership:**

```
// Consumer config (Java):
props.put("partition.assignment.strategy", "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
props.put("group.instance.id", "worker-" + System.getenv("POD_NAME")); // Kubernetes pod name
props.put("session.timeout.ms", "60000");  // longer window for rolling restarts
props.put("heartbeat.interval.ms", "10000");
```

**What does a Principal see here that a Senior might miss?**

The Senior deploys `CooperativeStickyAssignor` and calls it done. The Principal asks: "What happens to our consumer lag SLO during a cluster-wide rolling restart of 50 consumer instances?" And the answer is: even with cooperative rebalancing, if you restart 50 consumers simultaneously (e.g., a Kubernetes node pool rotation), you can trigger 50 concurrent rebalance phases. The Principal implements a *canary consumer restart* strategy: restart consumers in waves of 10%, with lag stabilization checks between waves. The deployment pipeline should read consumer lag metrics and pause rollout if lag exceeds a threshold.

---

## SECTION 5: OBSERVABILITY — THE PRINCIPAL'S FIRST-CLASS CONCERN

A Senior builds a Kafka cluster that runs. A Principal builds a Kafka cluster that you can understand, debug, and improve at 3am without SSH-ing into a broker.

### 5a. The Three Pillars

#### METRICS (The What)

**Business/Product Metrics:**
- `message_delivery_success_rate` per topic (% of produced messages that were consumed by at least one consumer group within SLO window)
- `consumer_group_lag_p95` per group — "If our alerting consumer group is 30 minutes behind, someone is not getting alerted about production outages"
- `end_to_end_latency_ms` (born-on timestamp in message → consumer processing timestamp)
- `topics_count` and `partitions_count` trends — governance health metric

**Service-Level Metrics (RED per broker):**
- **Rate:** `messages_in_per_sec`, `bytes_in_per_sec` (per broker, per topic)
- **Errors:** `request_handler_io_pool_request_rate_by_type{error}` — broken down by RequestType (Produce, Fetch, Metadata, OffsetCommit)
- **Duration:** `request_total_time_ms` p50/p95/p99 by RequestType; produce p99 target < 20ms

**Infrastructure Metrics (USE per broker):**
- **Utilization:** `cpu_usage_percent`, `disk_used_percent` per broker, `network_bytes_per_sec` vs. NIC capacity
- **Saturation:** `request_handler_io_pool_idle_percent` (if < 30%, you're saturated), `network_processor_avg_idle_percent`, `log_flush_rate_and_time_ms` (if p99 > 100ms, disk is a bottleneck)
- **Errors:** `log_segment_corruption_count` (should always be 0; > 0 is Sev-1), `disk_io_errors`, OOM kills in broker JVM logs

**Derived/Computed Metrics:**
- `consumer_lag_per_partition` = `log_end_offset - committed_offset`. Computed by Kafka's consumer group coordinator and exported via JMX. Not available natively as a Prometheus metric — use `kafka-consumer-groups.sh` or the Kafka Exporter sidecar.
- `isr_shrink_rate` = `rate(isr_shrinks_total[5m])`. Trend this weekly.
- `batch_size_avg` per producer — if this is consistently low (< 1KB), producers aren't batching efficiently; add `linger.ms=10` to producer config.
- `replication_lag_max` across all follower replicas — leading indicator of ISR shrink.

**Tool choice: Prometheus + Grafana, not Datadog or CloudWatch.**
At our scale (6 brokers, 60 partitions, 30 consumer groups), we're emitting ~10,000 time series. Datadog at $0.05/metric/month = $6K/month just for metrics. Prometheus is free, runs on-cluster, and integrates natively with Kafka's JMX exporter. Grafana gives us the dashboards. We use Thanos or Cortex for long-term metrics storage (Prometheus' default 15-day retention is insufficient for trend analysis).

---

#### LOGGING (The Why)

**Structured log schema for every broker log line:**

```json
{
  "timestamp": "2024-03-15T10:30:00.123Z",
  "level": "ERROR",
  "service": "kafka-broker",
  "broker_id": "2",
  "trace_id": "a1b2c3d4-...",
  "request_id": "uuid-v4",
  "operation": "produce.ack",
  "topic": "user-events",
  "partition": 3,
  "offset": 10042,
  "producer_id": 1234,
  "duration_ms": 18,
  "outcome": "failure",
  "error_code": "NOT_ENOUGH_REPLICAS",
  "isr_size": 1,
  "min_isr": 2,
  "metadata": { "batch_size": 512, "compression": "snappy" }
}
```

**Log levels for Kafka:**
- **ERROR:** ISR below `min.insync.replicas` causing producer failures; log segment corruption; controller failover failure; `OutOfMemoryError` in broker JVM.
- **WARN:** ISR shrink event (follower dropped); consumer group rebalance triggered; request queue depth > 500; replica fetch latency > 5s.
- **INFO:** Topic creation/deletion; leader election completion; broker startup/shutdown; consumer group assignment change.
- **DEBUG:** Individual record batch acknowledgment; offset commit per partition. NEVER enabled in production — at 1M msg/sec, DEBUG generates 1M+ log lines/sec.

**What NOT to log:**
- Message payload/value content (customer PII, security risk)
- Producer client IP beyond first hop (GDPR)
- Presigned URLs or auth tokens from producer requests

**Log sampling strategy:**
- 100% of ERROR and WARN — always
- 100% of requests where `duration_ms > p99_threshold` (currently 20ms for produce)
- 5% of successful INFO-level produce/consume cycles
- Sampling decision propagated via trace header — if a request is sampled at the producer SDK, all broker and consumer logs for that trace are also sampled

**Log pipeline:** Kafka broker → Filebeat (log shipper, low-overhead) → Loki (label-indexed log storage) → Grafana (query and alerting). Why Loki over Elasticsearch? We query primarily by label: `{service="kafka-broker", level="ERROR", topic="user-events"}`. Loki's label-based index is 10× cheaper than Elasticsearch's full-text inverted index for this access pattern. At 1 GB/sec throughput, the difference is meaningful.

---

#### DISTRIBUTED TRACING (The Where)

**Instrumentation with OpenTelemetry:**
We use OTel because it decouples instrumentation from the backend. Today we export to Jaeger; if we need Honeycomb or Grafana Tempo for better tail-based sampling, we swap the exporter without re-instrumenting every producer and consumer.

**Trace context propagation through the message:**
This is non-obvious for async systems. The trace doesn't travel over TCP like in synchronous RPC — it travels *inside the message payload* as a header:

```
Producer creates span: "produce.user-events.P3"
  → injects trace context into message header: "traceparent: 00-a1b2c3d4...-01"
Broker receives message: creates child span "broker.append.P3"
Consumer receives message: extracts traceparent header → creates child span "consume.analytics.P3"
  → continues the same trace!
```

This gives us true end-to-end visibility: producer SDK → broker append → consumer processing, all in one trace.

**Critical trace tree for a produce call:**

```
[POST produce user-events P3] — 18ms total
├── [auth.validate_producer_credentials] — 2ms
├── [quota.check_producer_rate] — 1ms
├── [log.append_record_batch] — 8ms   ← sequential disk write
├── [replication.fan_out_to_ISR] — 5ms
│   ├── [replication.to_broker2] — 4ms
│   └── [replication.to_broker3] — 5ms  ← slowest ISR follower = bottleneck
└── [response.ack_producer] — 0ms
```

Expected bottleneck: ISR replication (`replication.to_broker_N`). Budget: < 10ms for replication ACK. If this span consistently > 10ms, ISR follower is under load — check `replica_fetch_lag` on that broker.

**Sampling strategy:** Tail-based with these rules:
- 100% of traces where any span has `status=ERROR` (ISR failures, auth rejections)
- 100% of traces where `total_duration > 20ms` (produce p99 threshold)
- 2% random sample of everything else

At 1M msg/sec: 2% sample = 20,000 traces/sec. At ~2KB/trace: 40 MB/sec = ~3.5 TB/day. With 100% error capture + 2% baseline, we get excellent debuggability at a fraction of full-capture cost.

---

### 5b. SLI / SLO / SLA

| User Journey | SLI | SLO (Internal) | SLA (External) |
|---|---|---|---|
| Message produce | % produces completing with durable ACK in <20ms | 99.9% | 99.5% |
| Message consume | Consumer fetch p99 latency | <50ms | <100ms |
| Consumer lag | % of consumer groups with lag < 10K messages | 99.5% | N/A (internal) |
| Broker availability | % of time all partitions have ISR ≥ min.insync.replicas | 99.99% | 99.9% |
| End-to-end latency | p95 time from produce to consumer receipt | <500ms | <2s |

**Error Budget:**
- Produce SLO: 99.9% → 0.1% error budget = ~43 minutes/month of degraded produce
- If we burn >50% of monthly budget in any single week → freeze feature releases, focus on reliability
- Fast-burn alert: >5% of monthly budget consumed in 1 hour → Sev-1 page

---

### 5c. Alerting Strategy

**Tier 1 — Sev-1 (page immediately):**
- Producer error rate > 1% for 2 consecutive minutes (users losing messages)
- `under_replicated_partitions > 0` for > 5 minutes (durability SLA at risk)
- Any broker offline (no heartbeat to KRaft metadata quorum for > 30s)
- Consumer group lag > 1M messages AND not decreasing for 10 minutes
- `log_segment_corruption_count > 0` — immediate data integrity incident

**Tier 2 — Sev-2 (investigate within 1 hour):**
- Producer error rate > 0.1% (healthy is < 0.01%)
- Consumer group lag > 100K messages for > 10 minutes
- ISR shrink event rate > 10/hour (cluster instability leading indicator)
- Broker disk > 75% utilized (7-day retention math breaking down)
- Dead letter queue (DLQ) depth > 0 for > 5 minutes

**Tier 3 — Sev-3 (fix this sprint):**
- P99 produce latency trending up > 15% week-over-week
- Storage growth > 20% above 30-day forecast
- `replication_lag_max > 5s` sustained (below ISR threshold but close)
- Error budget burn rate > 20% of monthly budget consumed in one week

**Alert fatigue prevention:**
Every Sev-1 and Sev-2 alert has a linked runbook. Monthly alert review: any Sev-2 that didn't result in human action in the past 30 days gets demoted to Sev-3. The Tier-2 `isr_shrink_rate` alert: last month we got 40 alerts, 38 self-resolved in 30s. Suppress if duration < 60 seconds and recovery_rate = 100% in trailing 7 days.

---

### 5d. Observability for Distributed Async Systems

**Queue health — per-partition visibility:**
```
# kafka_consumer_group_lag{group="analytics",topic="user-events",partition="3"} = 50023
# kafka_consumer_group_lag{group="analytics",topic="user-events",partition="4"} = 2
```

If partition 3 has 50K lag but partition 4 has 2 — this is a *hot partition* problem, not a throughput problem. The fix is partition key redistribution, not adding consumer instances. You'd never see this with aggregate lag metrics alone.

**Message age (oldest unprocessed message):**
```python
# Born-on timestamp embedded in every message:
message_born_on_ms = record.headers.get("born_on_timestamp")
consumer_lag_time_ms = time.now_ms() - message_born_on_ms
# Expose as histogram metric: message_age_seconds_bucket
```

This is the *true* SLO metric. A lag of 100K messages is meaningless without knowing how fast they're arriving. If produce rate dropped to 100 msg/sec, 100K lag = 1000 seconds of backlog = potential SLA breach.

**DLQ as a Sev-1 signal:**
Dead letter queue depth > 0 = silent data loss. We alert within 5 minutes of any DLQ message. The DLQ is not a garbage bin — it's an incident waiting to be discovered. Every DLQ message gets: original topic, partition, offset, consumer group, failure reason, and stack trace (structured). Monthly DLQ audit: each message needs a ticket.

**End-to-end latency tracking:**
```json
// Every message header includes:
{ "born_on": 1710000000000, "producer_id": "checkout-svc-pod-7d8f9", "trace_id": "a1b2c3d4" }

// Consumer emits after processing:
histogram: message_e2e_latency_ms{topic="user-events",group="alerting"} = 87
```

This is the *only* reliable way to measure true async latency. Wall-clock differences between producer and consumer Prometheus metrics are misleading if clocks drift (NTP skew of 10ms is common in cloud VMs).

---

### 5e. Observability-Driven Database Design

For Kafka's internal metadata (topic configs, ACLs, consumer group state in `__consumer_offsets`):

**Audit columns on every metadata record:**
```sql
CREATE TABLE topic_configs (
  topic_id       UUID PRIMARY KEY,
  topic_name     VARCHAR(255) NOT NULL,
  partition_count INT NOT NULL,
  replication_factor INT NOT NULL,
  retention_ms   BIGINT NOT NULL,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_by     VARCHAR(255),           -- service account or operator
  version        INTEGER NOT NULL DEFAULT 1,  -- optimistic locking
  is_deleted     BOOLEAN DEFAULT FALSE   -- soft delete
);
```

**Why soft deletes?** If a topic is "deleted" and messages disappear, we need to answer: was it a user action, a bug, or a bad migration? Hard deletes make incident forensics impossible.

**CDC for operational intelligence:**
The KRaft metadata log is itself a Kafka topic (`__cluster_metadata`). We tap it with a Debezium-style connector to stream all metadata changes to our observability pipeline. This gives us:
- Real-time audit trail of all topic and ACL changes
- Who created this topic? When was the partition count last changed?
- Replay of all configuration changes for incident forensics

This is a Principal-level insight: the metadata log is free data. We get a complete audit log without adding any write overhead to the control plane.

---

### 5f. Cost Observability

**Cost metrics:**
- Storage cost: `$0.023/GB-month` (EBS gp3) × broker disk = tracked monthly per topic
- Network egress: typically `$0.09/GB` — at 3 GB/sec egress = $23K/day if all goes to the internet. Tag replication traffic (internal) vs. consumer traffic (may be cross-AZ = $0.02/GB).
- Compute: broker EC2 instances tagged with `team`, `service=kafka`, `env=prod`

**Cost anomaly detection:**
- Alert if daily storage cost grows > 10% day-over-day for 3 consecutive days (topic explosion or retention misconfiguration)
- Alert if cross-AZ egress spikes > 2× 7-day average (possible misconfigured consumer connecting from wrong AZ)
- Per-topic storage cost tracked monthly: topics in the top 10% by cost reviewed with owning teams

**The unit economics insight (Principal-level):**
At 1M msg/sec × 86,400s × 1 KB × 7 days × RF=3 = ~1.8 PB storage at $0.023/GB = ~$41K/month in storage alone. Add egress and compute: total infrastructure cost per million messages ≈ $0.05. If we charge downstream teams for usage, we need this granularity. Without per-topic cost attribution, the infrastructure cost looks like undifferentiated overhead.

---

## SECTION 6: PRINCIPAL ENGINEER LENS

### 6a. The 3 Hidden Time Bombs

**Time Bomb 1: Partition Count Lock-In (18-month horizon)**
At launch we configure 200 partitions. In 18 months, if throughput 5×es to 5M msg/sec, we need 500+ partitions. Adding partitions to an existing topic breaks key-based ordering for in-flight messages and requires a complex shadow-topic migration. The cost of fixing it later: 2–3 engineering weeks of migration work, a production freeze window, and consumer reprocessing of potentially days of data. *Early signal:* `bytes_in_per_sec / max_partition_throughput > 0.7` sustained for 7 days on more than 10% of partitions.

**Time Bomb 2: ZooKeeper Controller Scalability (24-month horizon)**
Legacy Kafka uses ZooKeeper for controller election and metadata storage. At 200 partitions this is fine. At 10,000 partitions (post-growth), ZooKeeper's 3-node quorum becomes a write bottleneck: metadata updates are serialized through the ZooKeeper leader. Controller failover time scales linearly with partition count: at 10K partitions, a controller failover takes 3–5 minutes of unavailability. KRaft (Raft-based metadata, no ZooKeeper) is the answer — it handles millions of partitions with controller failover in <30 seconds. *We should start with KRaft today.* Migration from ZooKeeper to KRaft requires a broker restart in "migration mode" — doable but risky in production. Cost of waiting: a multi-hour maintenance window vs. building on KRaft from day one. *Early signal:* `controller_epoch_change_rate` trending up, `zookeeper_request_latency_ms_p99 > 100ms`.

**Time Bomb 3: Consumer Group Coordinator Hot Partition (12-month horizon)**
All consumer groups hash to `__consumer_offsets` partitions (default: 50 partitions). At 1,000 consumer groups committing offsets every 5 seconds: 200 offset commits/sec per `__consumer_offsets` partition. Fine. At 100,000 consumer groups (multi-tenant scenario or microservice explosion): 2,000 commits/sec per partition. The Group Coordinator broker starts struggling, offset commits lag, consumers see `COORDINATOR_LOAD_IN_PROGRESS` errors, rebalances cascade. Fix: increase `offsets.topic.num.partitions` (requires recreating the topic) or shard consumer groups across multiple Kafka clusters. *Early signal:* `group_coordinator_request_latency_ms` for `OffsetCommit` requests trending above 10ms p99.

---

### 6b. Technology Selection Justification

**Storage (Log Segments):**

| Criteria | Local Disk (XFS/ext4) | Apache BookKeeper | Cloud Object Store (S3) |
|---|---|---|---|
| Sequential write latency | <1ms (SSD) | 5–10ms (network) | 100–500ms |
| Throughput ceiling | 2 GB/sec per disk | Scales horizontally | Unlimited but expensive |
| Cost/TB/month | $0.023 (EBS gp3) | $0.05–0.10 | $0.023 (but egress expensive) |
| Operational complexity | Low | High (separate cluster) | Medium |
| **Verdict** | ✅ Chosen | ❌ Extra operational layer | ❌ Too slow for hot path |

*S3 is excellent for long-term tiered storage (after 7-day hot retention) — we use it for the cold tier, not the hot path.*

**Metadata Coordination:**

| Criteria | KRaft (Raft-native) | ZooKeeper | etcd |
|---|---|---|---|
| Kafka-native integration | Native, no external dep | Requires separate cluster | Not supported |
| Failover time at 10K partitions | <30s | 3–5min | N/A |
| Operational overhead | Zero (embedded in broker) | Full ZooKeeper cluster | N/A |
| **Verdict** | ✅ Chosen | ❌ Legacy, scaling cliff | ❌ Not supported |

---

### 6c. The Full Tradeoff Table

| Decision | Chosen | Alternative | What We Gain | What We Give Up | Revisit When |
|---|---|---|---|---|---|
| Delivery guarantee | At-least-once | Exactly-once | Simplicity, 3–5× higher throughput | Consumer must be idempotent | Business requires financial transactions |
| Partition count governance | Start at 200, grow with process | Start at 2000 | Lower initial overhead, safe ISR | Migration pain at 5× growth | Throughput per-partition > 70% for 30 days |
| Metadata: KRaft vs ZooKeeper | KRaft | ZooKeeper | No external dep, faster failover | KRaft still maturing (GA Kafka 3.3+) | Stable for 2 years; keep |
| Log compaction | Time-based retention (default) | Log compaction for state topics | Simpler, predictable storage | Can't use Kafka as key-value store | Need changelog/state topics |
| Rebalance protocol | CooperativeStickyAssignor | EagerAssignor | No stop-the-world pauses | Slightly more complex assignment | Consumer SDK drops support |
| Sampling strategy | Tail-based (5% + 100% errors) | Head-based (5%) | Catches all errors/slow traces | More infra (collector buffer) | Collector infra too expensive |
| Log storage | Loki (label-based) | Elasticsearch | 10× cheaper for label queries | No full-text search on message body | Need forensic text search across payloads |

---

### 6d. Scale Evolution Plan

**Phase 1 — MVP (0 → 10K producers, ~50K msg/sec):**
- Architecture: 3-broker KRaft cluster, RF=3, 60 partitions, 3 `__consumer_offsets` partitions per group
- Single AZ (accept AZ-level unavailability — acceptable at MVP)
- Observability minimum: 3 dashboards (produce error rate, consumer lag, broker disk usage); 5 alerts (broker down, under-replicated partitions > 0, disk > 80%, produce error > 1%, lag > 1M)
- What we deliberately skip: MirrorMaker2 (cross-region replication), tiered storage, per-topic quotas, Schema Registry
- Why: adding these at MVP adds 3 weeks of engineering for problems we don't have yet

**Phase 2 — Growth (10K → 100K producers, ~500K msg/sec):**
- What breaks first: broker disk (at 500K msg/sec × 86400s × 7 days × RF=3 ≈ 300 TB/broker for 3 brokers → add brokers and rebalance)
- Add: tiered storage (local disk → S3 for data > 2 days old), per-producer/consumer quotas (to prevent single tenant starving others), MirrorMaker2 for cross-region async replication
- Observability additions: per-service SLO dashboards for each major consumer group, cost-per-topic tracking, automated rebalance impact analysis (measure lag delta during each rebalance event)
- What becomes difficult: partition rebalancing as we add brokers needs tooling (Cruise Control for automated partition reassignment)

**Phase 3 — Scale (100K+ producers, 1M+ msg/sec):**
- Fundamental shifts: multi-cluster federation (separate clusters per team/domain), dedicated metadata cluster (KRaft quorum isolated from data brokers), automated tier management
- What becomes its own team: Kafka Platform Team (8–12 engineers) owning cluster operations, consumer group SLOs, schema governance, and cost attribution
- Observability at this scale: automated SLO burn-rate alerts replace manual threshold alerts; ML-based anomaly detection on cost and latency trends (similar to Netflix Mantis); dedicated observability platform team (3–4 engineers)

---

### 6e. Failure Mode Analysis

**Component 1: Broker Leader for Hot Partition**

| Dimension | Detail |
|---|---|
| Failure scenario | Broker holding leader for P0 (highest-traffic partition) goes OOM due to JVM GC pressure |
| Detection | `jvm_memory_heap_used_bytes` approaching `jvm_memory_heap_max_bytes`; followed by `under_replicated_partitions > 0`; then `produce_error_rate` spike |
| Time to detect | 30 seconds (memory metric → ISR shrink alert) |
| Impact | Degraded: producers for P0 see 5–30s timeout while leader re-election runs. Other partitions unaffected. |
| Recovery | KRaft elects new leader from ISR in <30s; producers retry with new leader metadata. RPO=0 (committed messages safe). RTO < 60s. |
| Prevention | JVM heap tuning (off-heap page cache is preferred over JVM heap for Kafka); set broker heap to 6–8 GB max, let OS own the rest for page cache; monitor `heap_used/heap_max` and alert at 80% |

**Component 2: Group Coordinator Failure**

| Dimension | Detail |
|---|---|
| Failure scenario | The broker acting as Group Coordinator for the "analytics" consumer group crashes |
| Detection | `group_coordinator_unavailable` error in consumer client logs; consumer group stops committing offsets; consumer group lag grows |
| Time to detect | 60 seconds (consumers detect coordinator gone after session timeout) |
| Impact | All consumers in "analytics" group trigger rebalance. Processing pauses ~30s during coordinator failover + rebalance |
| Recovery | KRaft elects new controller, reassigns group to new coordinator broker, triggers rebalance. Consumers resume from last committed offset. |
| Prevention | Spread consumer groups across multiple coordinator brokers (natural if `__consumer_offsets` has ≥ 50 partitions and groups are many); use static group membership to reduce rebalance depth |

**Component 3: KRaft Metadata Quorum (Controller) Failure**

| Dimension | Detail |
|---|---|
| Failure scenario | 2 of 3 KRaft controller nodes lose connectivity simultaneously (split-brain scenario) |
| Detection | `active_controller_count` drops to 0 in Prometheus; broker logs show `no active controller` errors |
| Time to detect | 15 seconds (controller election timeout) |
| Impact | **Total unavailability** — no new topics can be created, no partition reassignments, no new consumer group creations. Existing produce/consume continues for established partitions (data plane is separate from control plane in Kafka). |
| Recovery | Restore connectivity; KRaft quorum re-elects leader (< 10s once connectivity restored). RTO = time to fix network + 10s election |
| Prevention | 5-node KRaft quorum (tolerates 2 simultaneous failures, not just 1); place KRaft nodes in 3 separate AZs; use dedicated controller-mode nodes (not combined broker+controller) at scale |

---

### 6f. Observability Maturity Assessment

**Current design: Level 3 (Observability)**

We can answer arbitrary questions about broker behavior without deploying new code:
- "Why did consumer group X fall behind at 2:30am?" → Trace ID from born-on message header, correlated with ISR shrink event at 2:28am, correlated with GC pause on broker 2
- "Which topic is consuming the most disk?" → Per-topic storage metrics in Grafana
- "Is the hot partition issue on P3 a key distribution problem or a throughput problem?" → Per-partition lag + message age metrics

**What Level 4 requires (Proactive):**
- Automated SLO burn-rate prediction: "At current burn rate, you'll exhaust your monthly error budget in 48 hours" — trigger before the SLA is breached
- ML-based anomaly detection on consumer lag patterns (seasonal baseline models — Monday morning consumer lag spikes are normal; sudden off-hour spikes are not)
- Automated partition rebalancing via Cruise Control when partition skew exceeds 20%
- Self-healing: if ISR shrinks to min-ISR threshold, automatically throttle producer quotas to reduce replication pressure before producers start failing

---

### 6g. Conway's Law Implications

The architecture implies these ownership boundaries:

- **Kafka Platform Team** — owns broker fleet, KRaft, partition governance, storage costs
- **Consumer Platform Team** — owns consumer group coordinator, offset management, rebalance tooling
- **Observability Team** — owns the metrics pipeline, Prometheus/Grafana, alert definitions
- **Producer teams** (Checkout, User-Events, etc.) — own their topics and produce SLOs

**Cross-team risk:** Who owns the end-to-end produce success rate SLO? The Kafka Platform Team owns broker availability. The Checkout Team owns their producer config. If produce errors spike, both teams get paged — and both point fingers at each other. **Resolution:** Define a shared SLO owned jointly, with an agreed incident commander (rotating, platform team leads). The on-call runbook must specify: "If produce error rate > 1%, page BOTH platform team and the owning producer team simultaneously."

**Observability ownership = team ownership:**
Every consumer group lag SLO is owned by the *consuming team*, not the platform team. The platform team provides the metrics and tooling; each consuming team builds their own lag dashboards and lag alerts. If the analytics team's consumer group is behind, that's the analytics team's Sev-2, not a platform incident — unless it's caused by a broker failure.

---

### 6h. The Three Hard Questions

**Q1: "If you had to rebuild this system for 10× scale from Day 1, what are the top 3 things you'd design differently?"**

First: start with 500+ partitions and a formal partition governance process — never let partition count become a political conversation mid-incident. Second: build the tiered storage S3 cold tier from day one, because at 10× volume the 7-day retention disk math is unworkable on local disk alone; hot-cold tiering needs to be in the consumer-facing API from the start (consumers need to know if they're fetching from hot or cold tier because latency is 10–50× different). Third: federate into domain clusters from day one rather than a single monolithic cluster — a multi-tenant monolith at 10× scale means one rogue producer can starve all consumers; hard cluster boundaries enforce quotas and blast radius isolation by construction.

**Q2: "On-call at 3am: users can't consume messages. Walk me through your first 10 minutes."**

Open the Kafka Platform Overview dashboard — first look: `produce_error_rate` (green means producers are fine; the issue is consumer-side) and `consumer_lag_by_group` (which group, which partitions?). If lag is only growing for one group: check that group's consumer instances (are they running?), then check their committed offsets (are they advancing?). If offsets not advancing: consumer processing is stuck — check application logs with trace IDs from the message born-on header to find the exception. If lag is growing for ALL groups simultaneously: broker-side. Check `under_replicated_partitions` and `isr_shrink_rate`. If ISR is healthy: check broker network and disk (`bytes_in_per_sec` vs. NIC capacity, `log_flush_latency_ms`). Escalate to broker runbook within 5 minutes if root cause not identified.

**Q3: "What assumption are you LEAST confident about, and what early signal would tell you you got it wrong?"**

The assumption I'm least confident about is that `acks=all` with `min.insync.replicas=2` at RF=3 will deliver our p99 produce latency target of 20ms under sustained 1M msg/sec load. This requires the slowest ISR follower to acknowledge every batch in <15ms — achievable on local disk, but vulnerable to GC pauses, disk write spikes, and network jitter on the follower. The early signal: `replication.lag.max` on any follower broker trending above 2–3 seconds during peak load. If we see this consistently, we need to move to faster NVMe SSDs on broker nodes, tune the JVM GC (ZGC with 200ms max pause), or relax to `min.insync.replicas=1` during controlled windows — accepting a narrower durability guarantee to meet latency SLO. The worst outcome is silently settling for `acks=1` in production because engineers got tired of p99 alerts, without a formal decision having been made.

---

## SECTION 7: LEVEL EXPECTATIONS

**Mid-level (E4):**
Minimum: Can design topic/partition structure, produce/consume APIs, basic offset tracking. Knows Kafka is "append-only log with partitions." Observability: names Prometheus and ELK. Misses: ISR mechanics, rebalance protocol, partition count governance, exactly-once semantics.

**Senior (E5):**
Minimum: Proactively covers ISR, `acks=all` vs `acks=1` tradeoffs, consumer group coordinator, `__consumer_offsets` topic. Designs RED metrics per broker, defines consumer lag SLO. Describes cooperative rebalancing. Knows KRaft vs ZooKeeper. Misses: ISR shrink as a *trend* metric (not just instant alert), partition count as irreversible, cost observability, Conway's Law team ownership implications.

**Principal/Staff+ (E6/E7):**
Distinguishing signals in this document:
- Designed partition count governance as a *process* (not just a number)
- Raised ISR shrink *rate* as a weekly trend, not just an instant alert
- Designed born-on timestamp in message headers for true async latency measurement
- Flagged KRaft from day one (not ZooKeeper) based on 18-month scale horizon
- Defined per-topic cost attribution for unit economics
- Mapped Conway's Law to Sev-1 escalation policy for shared SLOs
- Designed the 3am runbook experience as part of the system architecture
- Treated DLQ depth as a Sev-1 signal of silent data loss, not a monitoring afterthought
- Discussed tail-based sampling tradeoffs with real numbers ($13M/year vs. 5% sampling)
- Raised cooperative incremental rebalancing AND the canary deployment rollout strategy as a combined solution, not just the protocol change

---

*Document length reflects Principal-level interview depth. In an actual 45-minute interview, sections 1–3 cover the first 20 minutes; the interviewer directs which deep dives to pursue. The observability and principal sections should be woven throughout the conversation, not delivered as a monologue at the end.*
