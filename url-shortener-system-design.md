# System Design: URL Shortener
> With custom aliases, expiry, and bad-URL safety filtering

---

## Section 1: Understanding the Problem

### What is a URL Shortener?

A URL shortener takes a long, unwieldy URL — say, a 400-character e-commerce product link — and maps it to a short alias like `sho.rt/aB3k9Z`. Anyone who visits the short URL gets transparently redirected to the original destination. The value isn't just aesthetics: short URLs are trackable, shareable in constrained formats (SMS, tweets), and can be made to expire. At scale, a service like bit.ly handles billions of redirects per day, making this deceptively simple system a classic distributed systems design problem.

---

### Clarifying Questions

| Question | Why It Matters |
|---|---|
| Consumer product or internal tooling? | Consumer = optimize hard for redirect latency at global scale. Internal = simpler infra, no CDN edge nodes required. |
| Expected read-to-write ratio? | URL shorteners are read-heavy. 1000:1 → entire caching strategy changes. 100:1 → write path needs more attention. |
| Do premium users get vanity/custom aliases? | Custom aliases need a different ID generation path; explicit collision detection; reserved namespace protection. |
| Do short URLs expire? Can they be updated/deleted? | Expiry means checking TTL on every redirect. Deletion/update changes cache invalidation story entirely. |
| What SLA does the gov't bad-URL API offer? | If API P99 = 500ms, we cannot call it inline during redirects. Need local blocklist cache. |
| Do we need click analytics? | Analytics requires async write on every redirect — adds Kafka + analytics DB. Out of scope initially. |
| Max URL length? Do we normalize URLs? | Normalization affects deduplication — same URL with different UTM params could be stored twice. |

---

### Functional Requirements

| # | Requirement | Priority |
|---|---|---|
| F1 | Given a long URL, generate and return a short URL | P0 |
| F2 | Redirect short URL to original long URL | P0 |
| F3 | Block creation/redirect if URL is on gov't bad-URL list | P0 |
| F4 | Premium users can specify custom aliases | P0 |
| F5 | Premium users can set an expiry time on short URLs | P0 |

**Out of Scope:**
- Click analytics/geographic dashboards — this is a full sub-system requiring Kafka + Clickhouse + a separate service
- URL normalization/deduplication — a product decision, not a technical one
- QR code generation — purely presentational, no systems challenge

---

### Non-Functional Requirements

**Availability: 99.99%** (~52 min downtime/year). Redirect is a core product function — if redirects are down, all shared links are broken. This means no single-region deployment; we need active-active multi-region or at minimum a hot standby.

**Consistency: AP with eventual consistency on reads.** For redirect lookups, we favor availability. A user seeing a stale redirect for 1-2 seconds after update is acceptable. A user hitting a 500 because the DB is unavailable is not. However — writes (create short URL) must be durable. We cannot lose a write: the user has already shared that link.

**Latency:**
- Redirect (read) P50: <10ms, P99: <50ms — needs to be edge-cache fast
- Create short URL (write) P50: <100ms, P99: <300ms — humans waiting for a UI response
- Bad URL check must **not** be on the redirect critical path

**Throughput:**
- 100M DAU × 0.1 creates/day = 10M creates/day = **~116 writes/sec**
- 100M DAU × 10 redirects/day = 1B redirects/day = **~11,600 reads/sec baseline; peak 3× = ~35,000 reads/sec**

**Durability:** Zero tolerance for losing a URL mapping. If `sho.rt/aB3k9Z → long-url.com` is lost, every existing share of that link breaks. RPO = 0. RTO < 30 seconds.

**Security:**
- Creates require auth (JWT Bearer token)
- Bad URL checks on both create and redirect paths
- Validate scheme allowlist — no `javascript:`, `data:`, etc.
- Short codes must be non-enumerable (no sequential IDs)

**Observability SLOs (set upfront):**
- Know within **30 seconds** if redirect success rate drops below 99.9%
- Know within **2 minutes** if the bad URL check is failing open
- Know within **5 minutes** if any URL mapping is returning wrong destinations
- Know within **1 hour** if the blocklist is stale beyond its refresh interval

---

### Capacity Estimation

```
DAU: 100M
URL creates/day:  100M × 0.1 = 10M    →  116 writes/sec
Redirects/day:    100M × 10  = 1B     →  11,600 reads/sec; peak ~3× = 35K reads/sec

Storage per URL:
  short_code:   7 bytes
  long_url:     200 bytes avg
  metadata:     ~100 bytes (user_id, created_at, expires_at, is_custom)
  = ~310 bytes/URL

Storage growth:
  10M URLs/day × 310 bytes = 3.1 GB/day
  Per year: ~1.1 TB/year
  5-year:   ~5.5 TB — manageable in a single cluster with archiving

Cache sizing:
  20% of URLs get 80% of traffic (power law)
  20% of 10M daily = 2M hot URLs × 310 bytes ≈ 620 MB
  Fits comfortably in a 2–4 GB Redis instance

Bandwidth:
  Writes: 116/sec × 310 bytes ≈ 36 KB/sec (negligible)
  Redirects: 35K/sec × 310 bytes ≈ 10.8 MB/sec (also modest)
```

**Dominant constraint: read latency, not throughput.** At 35K reads/sec the database can handle raw volume, but P99 < 50ms requires cache-first architecture. Every cache miss hits the DB; we want cache hit rate > 95%.

---

## Section 2: The Setup

### Core Entities

| Entity | Represents | Primary Access Pattern |
|---|---|---|
| **URL Mapping** | Maps short code → long URL | Point lookup by `short_code` (hot read path) |
| **User** | Free or premium account | Lookup by `user_id` for permission checks at create time |
| **Blocklist Entry** | Bad URL from government API | Membership check: "is this domain/URL in the blocklist?" |

---

### API Design

#### POST /api/v1/urls — Create short URL

```
Headers: Authorization: Bearer <jwt>

Request:
{
  "long_url":      "https://example.com/very/long/path?with=params",
  "custom_alias":  "my-brand",            // optional, premium only
  "expires_at":    "2024-12-31T23:59:59Z" // optional, premium only
}

Response 201:
{
  "short_url":  "https://sho.rt/aB3k9Z",
  "short_code": "aB3k9Z",
  "long_url":   "https://example.com/...",
  "expires_at": null
}

Error cases:
  400 - long_url is malformed or uses non-http(s) scheme
  403 - long_url matches bad URL blocklist  → { "error": "URL_BLOCKED" }
  409 - custom_alias already taken          → { "error": "ALIAS_TAKEN" }
  403 - custom_alias/expiry by non-premium  → { "error": "PREMIUM_REQUIRED" }
```

#### GET /{short_code} — Redirect

```
No auth required.

Response 302: Location: <long_url>
  (302 not 301 — preserves ability to update/deactivate URLs)

Error cases:
  404 - short_code not found
  410 - short_code expired  → { "error": "URL_EXPIRED" }
  451 - destination on blocklist → { "error": "URL_BLOCKED" }
```

#### DELETE /api/v1/urls/{short_code} — Deactivate

```
Headers: Authorization: Bearer <jwt>
Auth: must be owner of the short_code
Response 204: No Content
```

---

## Section 3: High-Level Design

### Requirement F1: Create a Short URL

#### ❌ Naive Solution: MD5 hash of the long URL

Take the long URL, MD5 hash it, take the first 7 characters.

**Why it breaks:** MD5 produces hex characters only (`[0-9a-f]`), giving 16^7 = 268M combinations. With 10M URLs/day this collides catastrophically within weeks. Two users shortening the *same* long URL get the *same* short code — User A's expiry or deactivation kills User B's link. Hash-based generation also cannot support custom aliases in the same namespace.

#### ⚠️ Better Solution: Base62-encoded auto-increment ID

Use a database auto-increment ID, then Base62 encode it (`[0-9A-Za-z]`, 62 chars). ID 1 = "1", ID 62 = "Z", ID 3.5B = "aB3k9Z" (7 chars covers 62^7 = 3.5 trillion).

**New problem:** Sequential IDs are trivially enumerable. A malicious actor can walk `sho.rt/0000001` through `sho.rt/ZZZZZZZ` and harvest every URL in the system, including private ones. Also, a single DB as the ID source is a global bottleneck and SPOF.

#### ✅ Good Solution: Distributed ID generation with pre-generated pool

A background service continuously generates random Base62 strings, checks uniqueness in Postgres, and inserts them into a Redis `LPUSH url_code_pool` queue. App servers do `RPOP url_code_pool` — a single atomic Redis operation, sub-millisecond. No coordination between app servers needed.

```sql
CREATE TABLE url_mappings (
  short_code   VARCHAR(10) PRIMARY KEY,  -- "aB3k9Z" for generated, "my-brand" for custom
  long_url     TEXT NOT NULL,
  user_id      UUID NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at   TIMESTAMPTZ,              -- NULL = never expires
  is_custom    BOOLEAN NOT NULL DEFAULT FALSE,
  is_active    BOOLEAN NOT NULL DEFAULT TRUE,
  version      INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX ON url_mappings (user_id);
CREATE INDEX ON url_mappings (expires_at) WHERE expires_at IS NOT NULL;
```

**Observability hook:** Track `id_pool_depth` metric. Alert if < 1M (Sev-2), alert if < 100K (Sev-1 — writes will fail within minutes).

**Remaining gap a Principal notices:** Need a `reserved_codes` table to block `/api`, `/admin`, `/health` from being claimed as custom aliases.

---

### Requirement F2: Redirect Short URL to Long URL

#### ❌ Naive Solution: Direct DB lookup on every request

```
User → App Server → DB lookup WHERE short_code = 'aB3k9Z' → Return 302
```

At 35K reads/sec peak with ~5ms average DB round-trip, P99 latency is already blown. At 10x scale (350K reads/sec) this simply doesn't work.

#### ⚠️ Better Solution: Redis cache in front of DB

```
User → App Server → Redis GET short_code
  HIT  → Return 302 immediately (~1ms)
  MISS → DB lookup → Cache with TTL → Return 302
```

**New problem:** With TTL=1hr, a deactivated URL stays live for 3600 seconds. For expired premium URLs, this is a correctness violation — we promised the URL would stop working at a specific time.

#### ✅ Good Solution: Two-layer cache (CDN + Redis) with smart invalidation

**Layer 1 — CDN edge (CloudFront/Akamai):** Caches 302 responses for non-expiring URLs at the edge. Handles 95%+ of traffic without touching origin.

**Layer 2 — Redis (origin cache):** 10-minute TTL. Cached payload includes `expires_at` for client-side expiry check:

```json
{
  "long_url":   "https://example.com/...",
  "expires_at": "2024-12-31T23:59:59Z",
  "is_active":  true
}
```

On deactivation: Write Service issues `DEL short_code` from Redis immediately + CDN surrogate key purge (near-instant with Cloudflare). Expiry is checked on read against `now()` — not inferred from TTL.

**Observability hook:** `cache_hit_ratio` metric (target >95%). Alert if drops below 90%.

---

### Requirement F3: Bad URL Safety Check

#### ❌ Naive Solution: Synchronous gov't API call on every redirect

P99 500ms external API on a P99 50ms path = non-starter. If the gov't API goes down, do we fail open (serve all URLs) or fail closed (block all redirects)? Both are unacceptable.

#### ⚠️ Better Solution: Full blocklist cached in Redis

Sync every 15 minutes. 35K redirects/sec × 1 Redis SISMEMBER each = manageable but adds Redis load. 15-minute stale window is a safety risk for newly flagged content.

#### ✅ Good Solution: Two-tier — Bloom filter (in-process) + Redis (confirmation)

**Tier 1 (in-process Bloom filter):** Each app server maintains a Bloom filter loaded from the current blocklist. False positives (incorrectly blocking a safe URL) acceptable at <0.01%. False negatives are not acceptable.

A Bloom filter for 10M bad URLs with 0.01% FP rate requires ~240 MB per node. Lookup is O(k) hash functions — effectively instant.

**Tier 2 (Redis exact lookup):** Bloom filter positives trigger a Redis exact lookup to eliminate false positives before returning 451.

**Blocklist refresh flow:**

```
Gov't API (every 5 min) → Sync Service →
  1. Pull delta/full blocklist
  2. Build new Bloom filter bitarray
  3. Publish to Kafka topic "blocklist.v{n}"
  4. App servers subscribe → hot-swap Bloom filter atomically
  5. Update Redis exact-match set
```

**Observability hook:** `blocklist_staleness_seconds` metric. Alert if > 600s. Track `bloom_filter_false_positive_rate` — if growing, filter is under-sized.

---

### Requirement F4 & F5: Custom Aliases + Expiry (Premium)

#### ✅ Good Solution: DB-level uniqueness + reserved namespace + idempotent expiry worker

For custom aliases — DB PRIMARY KEY on `short_code` provides the write-level collision guarantee. Application pre-check:

```sql
-- Pre-check: alias available?
SELECT 1 FROM url_mappings WHERE short_code = $alias;
-- Check reserved namespace
SELECT 1 FROM reserved_codes WHERE code = $alias;
-- Then INSERT — if race condition causes PK violation, return 409
```

For expiry — background worker runs every 60 seconds:

```sql
SELECT short_code FROM url_mappings
WHERE is_active = TRUE
  AND expires_at IS NOT NULL
  AND expires_at <= now()
LIMIT 1000;
-- Then UPDATE is_active = FALSE + Redis DEL in batch
-- Uses SELECT ... FOR UPDATE SKIP LOCKED for multi-worker safety
```

**Observability hook:** `expiry_job_last_run_timestamp` — alert if age > 120s. `expiry_violations` counter (redirects served after expiry) — should always be 0; any non-zero value is Sev-1.

---

### Full Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   Client ──→ CDN Edge ──(miss)──→ Load Balancer                    │
│                                        │                           │
│                          ┌─────────────┼──────────────┐           │
│                          ▼             │              ▼            │
│                   Write Service        │       Read Service        │
│                    (POST /urls)        │       (GET /{code})       │
│                          │             │              │            │
│              ┌───────────┤             │    ┌─────────┼────────┐  │
│              ▼           ▼             │    ▼         ▼        ▼  │
│         Auth Service  Redis            │   Redis  Bloom     Read  │
│         (JWT check)   (ID pool)        │   (cache) Filter  Replica│
│              │                         │              │            │
│              ▼                         │              ▼            │
│         Primary DB ←── replication ───┘         Gov't API        │
│              │                                        │            │
│         Expiry Worker                          Sync Service        │
│         (every 60s)                            (every 5 min)       │
└─────────────────────────────────────────────────────────────────────┘
```

**Critical redirect path (most important flow):**

1. Client hits `sho.rt/aB3k9Z` → DNS resolves to CDN edge
2. CDN cache **HIT** (95% of traffic): return 302, done in ~5ms
3. CDN **MISS**: forward to Load Balancer → Read Service
4. Read Service checks local Bloom filter for destination domain
5. Redis `GET aB3k9Z`
   - **HIT**: check `expires_at`, return 302
   - **MISS**: query Read Replica, populate Redis, return 302
6. If Bloom filter positive → Redis exact blocklist lookup → 451 if confirmed

**Assumptions to validate:**
- 302 (not 301) redirects are acceptable
- Gov't API provides a full exportable list, not just a per-URL query endpoint
- Free-tier users are rate-limited but not blocked from creating URLs

---

## Section 4: Deep Dives

### Deep Dive 1: Short Code Generation at Scale

**The problem:** Generating unique, non-enumerable, URL-safe short codes at 116 writes/sec with no collisions and no coordination bottleneck.

#### ❌ Naive: Single sequence + Base62 encode

One Postgres sequence. Sequential = enumerable. Single DB sequence = global mutex. At 10x write rate, sequence becomes a bottleneck.

#### ⚠️ Better: Distributed counter with per-server ranges

Each app server requests a range of IDs (e.g., 5000000–5001000). Reduces DB calls. But IDs are still partially sequential. Range coordinator is a new SPOF.

#### ✅ Best: Pre-generated randomized pool

A background `id-generator` service:
1. Generates random Base62 strings
2. INSERTs into Postgres (`ON CONFLICT IGNORE`) — DB is source of truth for uniqueness
3. Only pushes successfully-inserted codes to Redis `url_code_pool` list
4. App servers `RPOP url_code_pool` — atomic, sub-millisecond, no server coordination

```
Pool target: 10M ready codes (covers ~24 hours of peak writes)
Generator: continuously refills when pool drops below 5M
Alert Sev-2: pool_depth < 1M
Alert Sev-1: pool_depth < 100K (writes will fail)
```

**Fallback if Redis pool is unavailable:** Write Service generates codes inline with collision-retry loop (max 3 retries). Tracked via `pool_fallback_mode` metric.

**What a Principal sees that a Senior might miss:**
- The hot-swap of the Bloom filter needs atomic reference swap (`AtomicReference<BloomFilter>`) — a mid-flight request reading the old filter while another thread NULLs it causes a NPE
- The generator race condition: two instances producing the same random code both passing the uniqueness check. Solution: INSERT is the authoritative check, not the pre-check

---

### Deep Dive 2: Bad URL Check — Safety Without Killing Latency

The core tension: a safety requirement that conflicts with a latency requirement.

#### ✅ Best: Two-tier Bloom filter + Redis confirmation (detailed)

**Data structure choice — why Bloom filter over Redis SISMEMBER:**

At 35K redirects/sec, 35K Redis calls/sec just for safety checks adds significant Redis load and network overhead. An in-process Bloom filter costs ~240MB of memory per node but zero network hops.

**False positive handling:**
- Bloom filter says "NOT in set" → guaranteed safe → serve immediately
- Bloom filter says "possibly in set" → Redis exact lookup → confirm or clear
- At 0.01% false positive rate with 10M entries, ~3.5 extra Redis calls/sec at 35K QPS — negligible

**Hot-swap during blocklist update:**

```python
class BlocklistManager:
    def __init__(self):
        self._filter = AtomicReference(BloomFilter())

    def check(self, domain: str) -> bool:
        return self._filter.get().might_contain(domain)

    def refresh(self, new_entries: List[str]):
        new_filter = BloomFilter(capacity=len(new_entries) * 1.2, error_rate=0.0001)
        for entry in new_entries:
            new_filter.add(entry)
        self._filter.set(new_filter)  # atomic swap
```

**Emergency mode:** If gov't API is down for > 30 minutes, alert Sev-2 and serve using last-known-good filter. Log every redirect during stale period for retroactive audit.

**What a Principal sees that a Senior might miss:**
- The gov't API might provide *domain*-level blocks or *full-URL* blocks — different matching strategies, different Bloom filter key formats
- A format change to the gov't API's export schema silently breaks the sync service. Need a versioned schema contract and a parse-error metric (`blocklist_parse_error_rate > 0` → Sev-1)

---

### Deep Dive 3: Cache Consistency on Deactivation and Expiry

**The problem:** How quickly does a deactivated/expired URL stop working, and what does "correct" mean here?

#### ❌ Naive: Let cache expire naturally

For an expired premium URL (user set expiry = "Dec 31 midnight"), TTL-based expiry gives up to 1 hour of continued serving after the promised cutoff. Correctness violation.

#### ⚠️ Better: Write-through cache invalidation on deactivate

Write Service issues Redis `DEL` on deactivation. Expiry worker issues Redis `DEL` on expiry. Clears the origin cache immediately.

**Problem:** CDN edge nodes have their own cache. CDN cache invalidation APIs are asynchronous (CloudFront: 60-180 seconds), have cost ($0.005/path), and don't guarantee instant global propagation.

#### ✅ Best: Short TTLs for premium URLs + CDN surrogate keys for deactivation

| URL type | CDN Cache-Control | Redis TTL | Invalidation on deactivation |
|---|---|---|---|
| Free, no expiry | `max-age=3600` | 10 minutes | Surrogate key purge |
| Premium with expiry | `max-age=60` | 2 minutes | Surrogate key purge + auto-expire |
| Deactivated | `no-store` | DEL immediately | Surrogate key purge |

**Surrogate key flow:**
- On create: tag CDN response with `X-Cache-Tag: short_code_{aB3k9Z}`
- On deactivate: call CDN purge API with that tag → near-instant propagation (Cloudflare: <5s)

**Expiry worker — multi-worker-safe design:**

```sql
-- SELECT ... FOR UPDATE SKIP LOCKED ensures two workers grab non-overlapping batches
SELECT short_code FROM url_mappings
WHERE is_active = TRUE
  AND expires_at <= now()
FOR UPDATE SKIP LOCKED
LIMIT 1000;
```

**What a Principal sees that a Senior might miss:**
- Two expiry worker instances running simultaneously must be safe. `FOR UPDATE SKIP LOCKED` is the Postgres primitive that makes this correct without a distributed lock
- The partial index `ON url_mappings (expires_at) WHERE expires_at IS NOT NULL` is load-bearing — without it, the expiry worker does a full table scan at 10B rows

---

## Section 5: Observability — The Principal's First-Class Concern

A Senior builds a system that works. A Principal builds a system you can **understand, debug, and improve at 3am**.

---

### 5a. The Three Pillars

#### Metrics (The What)

**Business/Product Metrics:**

| Metric | What it tells you | Alert threshold |
|---|---|---|
| `redirect_success_rate` | Single most important metric | < 99.9% → Sev-1 |
| `url_creates_total` | Write service health | Drop > 20% → investigate |
| `urls_blocked_at_create` | Safety filter at creation | Spike → possible phishing campaign |
| `redirects_blocked_safety` | Safety filter at redirect | Should be near-zero |
| `expiry_violations` | Correctness of expiry | Any non-zero → Sev-1 |

**Service-Level Metrics (RED framework):**

*Read Service:*
- Rate: `redirect_requests_per_sec` (by result: hit_cache / miss_to_db / blocked / expired / not_found)
- Errors: `redirect_5xx_rate` by type (DB_ERROR, CACHE_ERROR, TIMEOUT)
- Duration: P50, P95, P99 of `redirect_latency_ms`

*Write Service:*
- Rate: `url_create_requests_per_sec`
- Errors: `url_create_error_rate` (ALIAS_TAKEN / URL_BLOCKED / VALIDATION_FAILED / DB_ERROR)
- Duration: P50, P99 of `url_create_latency_ms`

**Infrastructure Metrics (USE framework):**

| Component | Utilization | Saturation | Errors |
|---|---|---|---|
| Redis | Memory %, connection count | `cache_hit_ratio` (target >95%), eviction rate | `redis_command_errors_total` |
| Postgres Primary | Active connections vs max | Replication lag, WAL write rate, lock waits | Query errors, OOM kills |
| Expiry Worker | — | `expiry_job_lag_seconds` | `expiry_job_failures` |

**Derived Metrics:**
- `bloom_filter_false_positive_rate` = `redis_blocklist_confirmations_negative / redis_blocklist_lookups` — high rate means filter is under-sized
- `id_pool_depth` — Redis `LLEN url_code_pool`; must stay above 1M
- `blocklist_staleness_seconds` — `now() - last_successful_sync_timestamp`; alert if > 600s

**Tool choice: Prometheus + Grafana** over Datadog (cost at our read volume) or CloudWatch (vendor lock-in). Prometheus pull model suits predictable, low-cardinality metrics. `short_code` is never a label — that would be infinite cardinality.

---

#### Logging (The Why)

Every log line:

```json
{
  "timestamp":        "2024-01-15T10:23:45.123Z",
  "level":            "INFO",
  "service":          "read-service",
  "trace_id":         "4bf92f3577b34da6",
  "span_id":          "00f067aa0ba902b7",
  "user_id":          "hashed:sha256:a3f...",
  "short_code":       "aB3k9Z",
  "operation":        "redirect.lookup",
  "result":           "success|blocked|expired|not_found",
  "cache_result":     "hit|miss",
  "duration_ms":      4,
  "destination_domain": "example.com"
}
```

> Note: `destination_domain` not full `long_url` — long URLs may contain PII (email addresses in query params, session tokens). Never log full URLs at INFO level.

**Log levels:**

| Level | Use for |
|---|---|
| ERROR | Wrong destination served, blocklist check threw exception, DB write succeeded but cache invalidation failed |
| WARN | Bloom filter hot-swap > 5s, Redis pool depth < 500K, gov't API returning 5xx |
| INFO | URL created, URL deactivated, blocklist refreshed, expiry batch summary |
| DEBUG | Individual cache GET/SET, Bloom filter query results — OFF in production |

**Log sampling:**
- 100% of ERROR and WARN — always
- 100% of redirects resulting in 4xx or 5xx — always
- 100% of blocklist-triggered events (audit trail) — always
- 5% of successful redirects — sampled by `trace_id hash mod 20 = 0`, ensuring consistent cross-service sampling

**Pipeline:** Fluentd → Kafka → Loki (label-indexed). Loki over Elasticsearch: primary query patterns are label-based (`service=read-service`, `trace_id=X`), not full-text search. Loki is 10-20x cheaper at this volume.

---

#### Distributed Tracing (The Where)

Instrumented with OpenTelemetry SDK (vendor-neutral — export to Jaeger today, Tempo tomorrow without re-instrumentation).

**Trace tree — normal cache hit redirect:**
```
[GET /aB3k9Z] — 2ms total
  ├── [cache.redis_get:aB3k9Z] — HIT — 1.8ms
  └── return 302 → example.com
```

**Trace tree — Bloom filter positive (the interesting case):**
```
[GET /aB3k9Z] — 12ms total
  ├── [auth.rate_limit_check] — 0.5ms
  ├── [cache.redis_get:aB3k9Z] — MISS — 1.2ms
  ├── [bloom.check:example-bad.com] — 0.02ms → positive
  ├── [redis.blocklist_confirm:example-bad.com] — 1.1ms → confirmed
  └── return 451 Unavailable For Legal Reasons
```

**Sampling strategy: tail-based**
- 100% of traces with any span status=ERROR
- 100% of traces with total_duration > 100ms (our P99 budget)
- 2% random sample of everything else

At 35K redirects/sec × 2% = 700 traces/sec × 1KB/trace ≈ 60GB/day — manageable.

> Why not 100%? At 35K QPS, 100% sampling = ~35K spans/sec. At ~1KB/span = 35MB/sec = ~3TB/day. At $0.10/GB storage = $109,000/year just for traces. Tail-based at 2% + 100% errors gives 95% of debugging value at <5% of the cost.

---

### 5b. SLI / SLO / SLA

| User Journey | SLI (what we measure) | SLO (internal target) | SLA (customer commitment) |
|---|---|---|---|
| Redirect | % redirects completing < 50ms | 99.9% | 99% |
| URL create | % creates completing < 300ms | 99.5% | 99% |
| Safety blocking | % bad URLs blocked within 10 min of gov't update | 99% | 95% |
| URL availability | % redirects returning correct destination | 99.99% | 99.9% |
| Expiry correctness | % URLs inactive within 120s of expiry time | 99.9% | 99% |

**Error budget:** Redirect SLO = 99.9% → 0.1% error budget = ~43 minutes/month.
- > 50% budget burned in one week → freeze feature deployments, focus on reliability
- Fast burn alert: 50% of monthly budget burned in 1 hour
- Slow burn alert: 10% of monthly budget burned in 6 hours

---

### 5c. Alerting Strategy

**Tier 1 — Sev-1 (page immediately):**
- `redirect_success_rate < 99%` for 2 consecutive minutes
- `expiry_violations > 0` for any 1-minute window
- `bloom_filter_age_seconds > 1800` (30 min stale — safety bypass risk)
- Any service returning HTTP 5xx > 2% rate for 3 minutes
- `id_pool_depth < 100K`

**Tier 2 — Sev-2 (investigate within 1 hour):**
- `cache_hit_ratio < 90%` for 10 minutes
- `redirect_p99_latency > 200ms` for 5 minutes
- `blocklist_staleness_seconds > 600`
- `id_pool_depth < 1M`
- Replication lag > 10s

**Tier 3 — Sev-3 (fix this sprint):**
- P99 redirect latency trending up > 15% week-over-week
- Cache hit rate declining trend
- Bloom filter false positive rate > 0.1%

**Every Sev-1 alert must link to a runbook.** Example runbook for `redirect_success_rate < 99%`:
1. Check `redirect_error_by_type` breakdown — is it DB errors, Redis errors, or timeouts?
2. If DB errors: check `pg_replication_lag_seconds` — failover may be needed
3. If Redis errors: check Redis memory utilization — may need to evict or scale
4. Escalate to DB/infra on-call if unresolved in 15 minutes

---

### 5d. Async System Observability

**Expiry worker:**
- `expiry_job_last_run_timestamp` — alert if age > 120s
- `expiry_job_duration_ms` — growth indicates batch too large or DB query degrading
- `expiry_job_urls_processed` — sudden spike means a cohort of premium subscriptions expired simultaneously

**Blocklist sync:**
- Instrument a `born_on_timestamp` in each Kafka sync event payload
- `blocklist_end_to_end_latency_ms` = gov't API response → all app servers hot-swapped
- DLQ on the Kafka consumer: if a hot-swap message fails to process, alert within 5 minutes

---

### 5e. Observability-Driven Database Design

Every critical table includes audit columns:

```sql
CREATE TABLE url_mappings (
  short_code          VARCHAR(10) PRIMARY KEY,
  long_url            TEXT NOT NULL,
  user_id             UUID NOT NULL,
  is_custom           BOOLEAN NOT NULL DEFAULT FALSE,
  is_active           BOOLEAN NOT NULL DEFAULT TRUE,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at          TIMESTAMPTZ,
  version             INTEGER NOT NULL DEFAULT 1,
  deactivated_by      UUID,
  deactivation_reason VARCHAR(50)  -- 'USER_REQUEST', 'EXPIRY', 'SAFETY_BLOCK'
);
```

**Soft deletes over hard deletes:** If a user reports a redirect going to the wrong destination, we need to answer "was this ever changed?" Hard deletes make incident investigation impossible. Soft delete (`is_active = FALSE`) + archive to cold storage after 90 days.

**CDC (Change Data Capture):** Debezium taps the Postgres WAL → Kafka. Every `url_mappings` change is streamed for:
- Real-time audit log at zero application code cost
- Cache invalidation driven by DB changes (more reliable than app-layer invalidation)
- Future analytics pipeline without polling primary DB

> This is a Principal-level insight: **the WAL is free observability data.**

**Query performance:** Log all queries > 50ms (ALL, not sampled). Review `pg_stat_statements` top-10 by `total_time` weekly. The main query `WHERE short_code = $1` should always be a PK index scan. If it ever becomes a seq scan, something dropped the primary key index.

---

### 5f. Cost Observability

| Cost driver | Metric | Alert threshold |
|---|---|---|
| Storage | `storage_growth_gb_per_day` | > 5% day-over-day for 3 consecutive days |
| CDN egress | `cdn_origin_egress_mb_per_sec` | > 2x 7-day average |
| CDN offload ratio | `cdn_offload_ratio` | < 90% for 1 hour (10x origin egress cost) |
| Gov't API calls | `blocklist_api_calls_per_day` | If API has per-call pricing: > 1000/day |

> Senior engineers observe correctness. Principal engineers observe correctness **and cost**.

Track cost-per-user monthly. If CAC is $10 but infrastructure cost per user is $8, the business model is broken. Tag every cloud resource with `team`, `service`, `environment`, `feature` at creation time.

---

## Section 6: Principal Engineer Lens

### 6a. The 3 Hidden Time Bombs

**Time Bomb 1: CDN Cache Incoherence at Scale**
At 100M DAU, a major content creator shares a link to 50M followers simultaneously — fine for the CDN. But when a safety team starts issuing bulk takedowns of 10K URLs/hour at month 20, CDN invalidation queues back up. Users see blocked content for 5-10 minutes. Early signal: `redirects_blocked_by_safety` rising while `cdn_invalidation_queue_depth` also rises. Fix: batched surrogate-key invalidation, or short TTLs for safety-flagged URLs.

**Time Bomb 2: The Bloom Filter Memory Problem**
At 10M URLs/day × 10 years = 36.5B URLs, the Bloom filter for the ID generator's uniqueness check grows to ~44GB per node. Early signal: `id_generator_memory_mb` trending upward. Fix: move to 8-char short codes (62^8 = 218 trillion combinations) before namespace pressure grows.

**Time Bomb 3: Gov't API Format Lock-In**
In 24 months, the gov't API changes its export format. The sync service AND Bloom filter builder both break simultaneously — in production, under pressure. Early signal: `blocklist_parse_error_rate > 0`. Fix: a versioned adapter layer with a schema contract between the sync service and the Bloom filter builder so format changes can be rolled out independently.

---

### 6b. Technology Selection Justification

| Criteria | Postgres | DynamoDB | Cassandra |
|---|---|---|---|
| Primary access pattern | Point lookup by PK + range scan for expiry | Point lookup by PK only | High write throughput |
| Secondary index support | Excellent, expressive | Limited, GSI reads eventually consistent | Good with careful modeling |
| Expiry range scan | Simple partial index | Requires GSI, eventually consistent | Possible but complex |
| Operational complexity | Moderate | Low (managed) | High (compaction, GC tuning) |
| Cost at our scale | Low — single cluster comfortable at 116 writes/sec | Medium | High — overprovisioned for write rate |
| **Verdict** | ✅ **Chosen** | ❌ GSI range scans expensive | ❌ Overcomplicated for this write rate |

**The specific reason Postgres beats DynamoDB here:** the expiry worker needs `WHERE expires_at <= now()`. DynamoDB doesn't support range scans on non-key attributes without a GSI, and those reads are eventually consistent. Postgres's partial index makes this query trivially fast.

---

### 6c. The Full Tradeoff Table

| Decision | Chosen | Alternative | What We Gain | What We Give Up | Revisit When |
|---|---|---|---|---|---|
| 302 vs 301 redirects | 302 | 301 | Ability to update/deactivate URLs | Browser-level client caching | If URLs are truly immutable |
| Bloom filter placement | In-process per node | Redis SISMEMBER only | Zero network hop, sub-ms safety check | 240MB memory per node, stale window | If nodes become memory-constrained |
| Consistency model | AP (eventual) | CP (strong) | 99.99% availability, low latency | Stale reads for ~1-10s after deactivation | If compliance requires instant deactivation |
| Log storage | Loki | Elasticsearch | 10-20x cheaper, label-based queries | No full-text search on log bodies | If complex log body search is needed |
| Deletes | Soft delete | Hard delete | Full audit trail, safe incident investigation | Storage growth (needs archive job) | After 90-day archive to cold storage |

---

### 6d. Scale Evolution Plan

#### Phase 1 — MVP (0 → 100K users)

Single-region. Postgres primary + one read replica. Redis single instance. One app server cluster. No CDN — direct origin responses. No Bloom filter — just Redis SISMEMBER for blocklist checks (simpler, and P99 latency budget isn't stressed yet).

**Minimum viable observability:** 3 dashboards (redirect success rate, P99 latency, blocklist staleness). 5 alerts: the Sev-1 list above. Don't build cost observability yet — premature.

**Deliberately not built:** multi-region, CDN, Bloom filter. Premature complexity at this scale.

#### Phase 2 — Growth (100K → 10M users)

**What breaks first:** Redis single instance becomes a memory/throughput bottleneck as hot URL working set grows. Solution: Redis Cluster with consistent hashing.

**What's added:** CDN (absorbs 90%+ of redirect load), Bloom filter (eliminates safety check overhead), Read replicas (multiple, for geographic distribution).

**Observability additions:** Per-service SLO dashboards, error budget tracking, cost-per-user metric, CDN offload ratio tracking.

#### Phase 3 — Scale (10M+ users)

Multi-region deployment. DNS-based geographic routing. Read replicas per region. The blocklist sync becomes a multi-region problem — each region needs its own Bloom filter, synchronized from a regional Kafka cluster.

**Observability shifts:** Manual threshold alerts don't scale across 20+ services. Need automated SLO burn-rate alerts (Google SRE book §5), ML-based anomaly detection on redirect latency and cache hit rate, dedicated observability platform team.

---

### 6e. Failure Mode Analysis

**Component 1: Postgres Primary**
- Failure: Primary goes down (hardware failure, OOM kill)
- Detection: `db_connection_errors_total` spikes → Sev-1 in <30 seconds
- Impact: Writes fail. Reads degrade to cache-only (still serving 95% from cache). Cache misses return 503.
- Recovery: Failover to read replica promoted to primary. RTO = 30-60 seconds with automated failover (Patroni). RPO = 0 (synchronous replication to standby).
- Prevention: Multi-AZ deployment. Synchronous standby replica.

**Component 2: Redis Cache**
- Failure: Redis evicts hot entries under memory pressure (misconfigured `maxmemory`)
- Detection: `cache_hit_ratio` drops from 95% to 60% — Sev-2 fires within 5 minutes. DB read replica load spikes concurrently.
- Impact: Degraded — redirects still work, P99 latency jumps. Not a total failure unless DB also gets overwhelmed.
- Recovery: Increase Redis memory, fix `maxmemory-policy = allkeys-lru`. RTO = minutes.
- Prevention: Alert at 70% Redis memory utilization. Size instance for 2x expected hot working set.

**Component 3: Blocklist Sync Service**
- Failure: Sync service crashes — Bloom filter becomes stale
- Detection: `blocklist_staleness_seconds > 600` → Sev-2 alert
- Impact: Degraded safety, not availability. Continue serving redirects on stale filter. Bad URLs added after last sync continue to be served.
- Recovery: Restart sync service. Hot-swap Bloom filter. RTO < 5 minutes.
- Prevention: Two sync service replicas. Atomic Bloom filter swap supports concurrent syncs safely.

---

### 6f. Observability Maturity Assessment

This design reaches **Level 3 — Observability** ("we can answer arbitrary questions about system behavior without deploying new code"):

| Level | Description | Achieved? |
|---|---|---|
| Level 1 — Reactive | Find out from users that something is broken | Surpassed |
| Level 2 — Monitoring | Alerts when services are down | Surpassed |
| Level 3 — Observability | Answer arbitrary questions without new code | ✅ Yes |
| Level 4 — Proactive | Predict degradation before user-visible | Path defined |

**Path to Level 4:**
- Automated SLO burn-rate alerts (not just threshold alerts) — detects degradation before it's user-visible
- ML-based anomaly detection on cache hit rate and redirect latency — predicts Redis memory pressure 30 minutes before evictions start
- Auto-remediation: if `id_pool_depth < 500K`, automatically trigger emergency pool generation

---

### 6g. Conway's Law Implications

This architecture naturally maps to two teams:

| Team | Owns | SLO Responsibility |
|---|---|---|
| URL Platform Team | Write Service, Read Service, Postgres, Redis, ID Pool | Redirect success rate |
| Safety Team | Sync Service, Bloom filter pipeline, Gov't API integration | Blocklist staleness SLO |

**Critical cross-team interface:** The Kafka topic `blocklist.v{n}` and its message schema. If the Safety Team changes the blocklist schema without coordination, the Platform Team's Read Service breaks. This interface needs a **versioned schema registry contract** — not just Kafka topic naming conventions.

**The SLO ownership question:** If blocking is failing because Redis is down → Platform Team. If blocking is failing because gov't API is down → Safety Team. Define this in writing before going to production. Without it, the 3am page becomes an argument between two teams rather than a resolution.

---

### 6h. The Three Hard Questions

**Q1: "If you had to rebuild for 10x scale from Day 1, what are the top 3 things you'd design differently?"**

First: CDN integration from day one — the 302 → CDN cache invalidation strategy shapes every downstream decision, and retrofitting it means re-testing all edge cases. Second: design the Bloom filter update pipeline with version vectors from the start — at 10x we'd have dozens of app servers and the hot-swap coordination gets complex. Third: separate the create and redirect services into independent deployment units immediately — they have completely different scaling profiles (writes: CPU-bound validation; reads: I/O-bound cache lookup) and co-deploying them couples their scaling decisions.

**Q2: "On-call engineer paged at 3am with 'users can't redirect.' Walk through the first 10 minutes."**

Open the **redirect success rate dashboard** first — single graph, should be 99.9%+, tells you immediately if it's total failure or partial. If partial: filter by `cache_result=miss` — if miss rate is 100%, Redis is down. If success rate looks fine but users are reporting issues: check `redirect_p99_latency` — maybe it's slowness perceived as broken. If all infra looks green: check `bloom_filter_age_seconds` — if stale > 30min, safety checks might be incorrectly blocking safe URLs (false positives from stale filter). The decision tree in the runbook covers all branches in order: check infra → cache → DB → safety pipeline.

**Q3: "What assumption are you least confident about, and what early signal would tell you it's wrong?"**

The assumption I'm least confident about is the **10:1 redirect-to-create ratio**. If this service gets adopted for high-frequency link tracking (marketing teams reusing the same short URL across millions of ad impressions), the actual ratio could be 1000:1 or higher — which changes the CDN sizing, Redis hot-set estimation, and the entire capacity model. The early signal: `redirects_per_unique_short_code_p99` growing orders of magnitude above baseline. If that metric shows a handful of short codes receiving millions of hits each, we have a hot-key problem requiring per-URL rate limiting and dedicated hot-key caching tiers.

---

## Section 7: Level Expectations

| Level | Minimum Bar | Observability Bar | Typical Gaps |
|---|---|---|---|
| **Mid-level (E4)** | Functional redirect flow, basic API + schema, Redis caching when prompted | Knows metrics/logs/traces exist; can name Prometheus and ELK | Custom alias collisions, expiry correctness, Bloom filter, enumeration vulnerability |
| **Senior (E5)** | Proactively adds caching, identifies 302 vs 301, handles custom aliases with collision detection, implements expiry worker | Defines SLIs/SLOs, designs RED metrics per service, describes distributed tracing and why it matters | CDN cache invalidation at deactivation, Bloom filter concurrency, cost observability, distributed expiry worker coordination |
| **Principal/Staff+ (E6/E7)** | All Senior capabilities + treats observability as a design constraint from Section 1 | SLOs before dashboards, error budgets, 3am runbook as first-class deliverable, cost observability unprompted | *(Distinguishing signals below)* |

**Principal distinguishing signals:**
- Raises CDN cache invalidation complexity unprompted
- Designs Bloom filter hot-swap with explicit atomic reference analysis
- Connects Conway's Law to SLO ownership before being asked
- Brings up CDC/WAL as a free observability stream
- Treats alert fatigue as a first-class engineering problem
- Asks "how will we know this is working correctly in production?" before finishing any component
- Mentions tail-based sampling tradeoffs without prompting
- Raises the gov't API format lock-in time bomb

---

*System Design Document — URL Shortener with Custom Aliases, Expiry, and Bad-URL Safety*
*Principal Engineer Interview Reference*
