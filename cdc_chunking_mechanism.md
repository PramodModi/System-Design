# Content-Defined Chunking (CDC)
### How Dropbox Achieves 60-70% Storage Deduplication

---

## Table of Contents

1. [The Problem — Why Chunking Exists](#1-the-problem--why-chunking-exists)
2. [Naive Approach — Fixed-Size Chunking](#2-naive-approach--fixed-size-chunking)
3. [The Boundary Shift Problem](#3-the-boundary-shift-problem)
4. [Content-Defined Chunking — The Idea](#4-content-defined-chunking--the-idea)
5. [Rabin Fingerprinting — The Algorithm](#5-rabin-fingerprinting--the-algorithm)
6. [Rolling Hash — Step by Step](#6-rolling-hash--step-by-step)
7. [How Boundaries Are Content-Stable](#7-how-boundaries-are-content-stable)
8. [The Anchor Zone Concept](#8-the-anchor-zone-concept)
9. [Client SDK Implementation](#9-client-sdk-implementation)
10. [Cross-File Deduplication](#10-cross-file-deduplication)
11. [Server-Side View — Chunk-Agnostic by Design](#11-server-side-view--chunk-agnostic-by-design)
12. [File Reconstruction](#12-file-reconstruction)
13. [CDC vs Fixed Chunking — Full Comparison](#13-cdc-vs-fixed-chunking--full-comparison)
14. [Key Numbers to Remember](#14-key-numbers-to-remember)

---

## 1. The Problem — Why Chunking Exists

Before understanding CDC, understand why we chunk files at all.

Large files uploaded as a single unit have three critical problems:

**Problem 1: Resume is impossible.**
If a 10GB file upload fails at 99% completion, the client must restart from zero. No partial progress is saved.

**Problem 2: API servers become bottlenecks.**
The server holds the entire file in memory during transit. At 1,200 concurrent 100MB uploads, that is 120GB of RAM needed on API servers — not feasible.

**Problem 3: Deduplication is coarse.**
If two users upload files that are 99% identical (different header, same body), you store both copies entirely. No sharing of identical content.

Chunking solves all three:
- Each chunk uploads independently → resume at the failed chunk
- Chunks upload directly to S3 via presigned URLs → API servers are never in the data path
- Identical chunks across different files share one S3 object → deduplication

The question is not *whether* to chunk. The question is *where* to cut.

---

## 2. Naive Approach — Fixed-Size Chunking

The simplest answer: split the file into equal-sized blocks.

```
File: "The complete guide to system design..."  (20MB)

Fixed chunks (4MB each):
┌─────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│   Chunk 1   │   Chunk 2   │   Chunk 3   │   Chunk 4   │   Chunk 5   │
│ bytes 0→4MB │bytes 4→8MB  │bytes 8→12MB │bytes12→16MB │bytes16→20MB │
│  hash = abc │  hash = def │  hash = ghi │  hash = jkl │  hash = mno │
└─────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
```

Upload initiation:
```
Client sends chunk_hashes: [abc, def, ghi, jkl, mno]
Server checks: all 5 are new → upload all 5 chunks
Total upload: 20MB
```

Second upload (first time, cold state):
```
Server stores all 5 chunks.
chunks table:
  abc → s3_key='chunks/ab/abc', ref_count=1
  def → s3_key='chunks/de/def', ref_count=1
  ghi → s3_key='chunks/gh/ghi', ref_count=1
  jkl → s3_key='chunks/jk/jkl', ref_count=1
  mno → s3_key='chunks/mn/mno', ref_count=1
```

So far so good. The problem emerges on the first edit.

---

## 3. The Boundary Shift Problem

User inserts one sentence at the beginning of the document. Just **50 bytes** added.

```
Before: "The complete guide to system design..."
After:  "PREFACE: Read this first. The complete guide to system design..."
         ← 50 new bytes here →
```

What happens to fixed chunks?

```
BEFORE EDIT:
┌─────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│   Chunk 1   │   Chunk 2   │   Chunk 3   │   Chunk 4   │   Chunk 5   │
│ bytes 0→4MB │bytes 4→8MB  │bytes 8→12MB │bytes12→16MB │bytes16→20MB │
│  hash = abc │  hash = def │  hash = ghi │  hash = jkl │  hash = mno │
└─────────────┴─────────────┴─────────────┴─────────────┴─────────────┘

AFTER EDIT (50 bytes inserted at position 0):
┌─────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│   Chunk 1   │   Chunk 2   │   Chunk 3   │   Chunk 4   │   Chunk 5   │
│ bytes 0→4MB │bytes 4→8MB  │bytes 8→12MB │bytes12→16MB │bytes16→20MB │
│  hash = XYZ │  hash = PQR │  hash = STU │  hash = VWX │  hash = YZA │
└─────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
      ↑               ↑               ↑               ↑               ↑
   changed         changed         changed         changed         changed
   (has new        (shifted        (shifted        (shifted        (shifted
   50 bytes)       by 50)          by 50)          by 50)          by 50)
```

All 5 chunks have different hashes. The server sees 5 entirely new chunks and uploads all 20MB again.

**A 50-byte edit caused a 20MB upload.**

This is the **boundary shift problem**: because chunks are defined by byte position, any insertion or deletion shifts every subsequent byte, which shifts every subsequent chunk boundary, which changes every subsequent chunk hash. The deduplication system is completely defeated.

At scale:
```
10M DAU × 1 edit/day × avg 2MB wasted re-upload
= 20 TB/day of unnecessary egress
= $1.8M/day in unnecessary S3 egress cost
```

Fixed-size chunking is not an optimization that can be tuned — it is fundamentally broken for mutable files.

---

## 4. Content-Defined Chunking — The Idea

Instead of cutting at fixed byte positions, cut at positions determined by the **content itself**.

The core insight: find natural split points in the data by looking at patterns of bytes. If the content hasn't changed, the pattern is still there, and the cut happens at the same relative position in the unchanged content — regardless of what happened earlier in the file.

```
                 content pattern detected → CUT HERE
                        ↓
[... byte 3981, byte 3982, byte 3983 ✂ byte 3984, byte 3985 ...]
                                     ↑
                             boundary fires because
                             bytes 3936-3983 produce
                             a specific hash value
```

If you insert 50 bytes at the start of the file, those 50 bytes shift everything by 50 positions. But as soon as the algorithm reaches the unchanged content (starting at position 50), the byte patterns are identical to before. The same rolling hash values appear at the same relative positions in the unchanged content, and the same boundaries fire.

---

## 5. Rabin Fingerprinting — The Algorithm

The most widely used CDC algorithm is **Rabin fingerprinting**, invented by Michael Rabin in 1981 and used by Dropbox, rsync, and most modern sync systems.

It has two components:
1. A **rolling hash** — efficiently computes a hash over a sliding window of bytes
2. A **boundary condition** — fires when the hash satisfies a specific pattern

### The Polynomial Foundation

Rabin fingerprinting treats a sequence of bytes as a polynomial over a Galois Field (GF(2^k)). For our purposes, the important property is:

```
hash(bytes[i+1 ... i+W]) can be computed from:
  hash(bytes[i ... i+W-1])
  + the new byte entering the window (bytes[i+W])
  - the old byte leaving the window (bytes[i])

Without recomputing the hash of all W bytes from scratch.
```

This is the "rolling" property — updating the hash as the window slides costs O(1) per byte, not O(W) per byte. For a 48-byte window, this is a 48× speedup vs recomputing from scratch at every position.

### The Irreducible Polynomial

Rabin uses a specific irreducible polynomial p(x) over GF(2). A common choice for CDC:

```
p(x) = x^53 + x^6 + x^2 + x + 1
```

In practice this is implemented as a lookup table over 256 entries (one per byte value) for maximum speed. The algorithm precomputes `T[b]` = contribution of byte value `b` to the hash, and `U[b]` = contribution of removing byte `b` from the window.

```python
# Simplified Rabin hash update (conceptual — real impl uses lookup tables)
def rabin_update(hash_val, byte_entering, byte_leaving, window_size):
    # Remove contribution of byte_leaving (now exiting the window)
    hash_val ^= U[byte_leaving]
    # Left-shift hash (slide the polynomial)
    hash_val = ((hash_val << 1) | (hash_val >> 63)) & 0xFFFFFFFFFFFFFFFF
    # Add contribution of byte_entering
    hash_val ^= T[byte_entering]
    return hash_val
```

### The Boundary Condition

After computing the rolling hash at each position, check:

```python
if (rolling_hash & MASK) == TARGET:
    # CUT HERE — this is a chunk boundary
```

Where:
- `MASK` is a bitmask that determines average chunk size
- `TARGET` is any fixed value (e.g., all zeros, or a chosen constant)

```
MASK = 0x00001FFF  (13 bits set)
→ boundary fires when bottom 13 bits of hash == TARGET
→ probability of match at any position = 1/2^13 = 1/8192
→ average chunk size = 8192 bytes = 8KB

MASK = 0x000FFFFF  (20 bits set)
→ probability = 1/2^20 = 1/1,048,576
→ average chunk size = 1MB

Dropbox uses ~4MB average:
MASK = 0x003FFFFF  (22 bits set)
→ probability ≈ 1/4,194,304
→ average chunk size ≈ 4MB
```

The mask is the single tuning knob. Larger mask = larger average chunks = fewer chunks per file = less metadata overhead but coarser dedup granularity.

---

## 6. Rolling Hash — Step by Step

Let's trace through a concrete example with a tiny window (4 bytes instead of 48) to see exactly what happens.

```
File bytes: [72, 65, 6C, 6C, 6F, 20, 77, 6F, 72, 6C, 64, 21]
             H   e   l   l   o  ' '  w   o   r   l   d   !

Window size: 4 bytes
Boundary mask: 0x3  (2 bits → average chunk every 4 bytes, for illustration)
Target: 0x0
```

**Slide the window across:**

```
Position 0: window = [72, 65, 6C, 6C]  →  hash = 0xA3  →  0xA3 & 0x3 = 0x3  ≠ 0x0
Position 1: window = [65, 6C, 6C, 6F]  →  hash = 0x7C  →  0x7C & 0x3 = 0x0  == 0x0  ✂ CUT
Position 2: window = [6C, 6F, 20, 77]  →  hash = 0x51  →  0x51 & 0x3 = 0x1  ≠ 0x0
Position 3: window = [6F, 20, 77, 6F]  →  hash = 0xB8  →  0xB8 & 0x3 = 0x0  == 0x0  ✂ CUT
Position 4: window = [20, 77, 6F, 72]  →  hash = 0x2D  →  0x2D & 0x3 = 0x1  ≠ 0x0
Position 5: window = [77, 6F, 72, 6C]  →  hash = 0x64  →  0x64 & 0x3 = 0x0  == 0x0  ✂ CUT
...
```

**Result: cuts fire at positions 1, 3, 5 — driven by content, not byte count.**

The key property: these same cuts will fire at the same positions in the **content** even if bytes are inserted before position 0. Once the algorithm passes the inserted bytes and reaches the original content, the rolling hash produces the same values and the same cuts fire.

---

## 7. How Boundaries Are Content-Stable

This is the fundamental property that makes CDC work. Trace through what happens when 2 bytes are inserted at position 0:

```
ORIGINAL FILE:
Position:  0    1    2    3    4    5    6    7    8    9   10   11
Bytes:    [72,  65,  6C,  6C,  6F,  20,  77,  6F,  72,  6C,  64,  21]

AFTER INSERT of [AA, BB] at position 0:
Position:  0    1    2    3    4    5    6    7    8    9   10   11   12   13
Bytes:    [AA,  BB,  72,  65,  6C,  6C,  6F,  20,  77,  6F,  72,  6C,  64,  21]
               ↑         ↑         original content starts here, shifted by 2
```

Now trace the rolling hash (window=4) on the new file:

```
Position 0: window = [AA, BB, 72, 65]  →  hash = ???  →  may or may not fire
Position 1: window = [BB, 72, 65, 6C]  →  hash = ???  →  may or may not fire
Position 2: window = [72, 65, 6C, 6C]  →  hash = 0xA3  ← SAME AS ORIGINAL POSITION 0
Position 3: window = [65, 6C, 6C, 6F]  →  hash = 0x7C  ← SAME AS ORIGINAL POSITION 1  ✂ CUT
Position 4: window = [6C, 6C, 6F, 20]  →  hash = 0x51  ← SAME AS ORIGINAL POSITION 2
Position 5: window = [6C, 6F, 20, 77]  →  hash = 0xB8  ← SAME AS ORIGINAL POSITION 3  ✂ CUT
...
```

From position 2 onwards, the rolling hash values are **identical** to the original file's rolling hash values starting from position 0. The same boundary decisions fire — just shifted by 2 positions to account for the inserted bytes.

**The boundary that was at original position 1 is now at position 3. The content of the chunk is identical. The hash is identical. The server already has this chunk.**

This is why CDC produces stable chunk boundaries in the face of insertions and deletions. The boundaries are anchored to content, not to byte addresses.

---

## 8. The Anchor Zone Concept

After any insertion or deletion, there is a brief **ripple zone** where boundaries may shift or new boundaries may appear due to the interaction of the new bytes with the rolling window. Once the window has completely "flushed" the new bytes, it enters the **anchor zone** where boundaries are identical to the pre-edit file.

```
                   insertion point
                         ↓
[unchanged][new bytes][RIPPLE ZONE][─────── ANCHOR ZONE ──────────────────]
                            ↑                      ↑
                     1-3 chunks affected    all chunks identical
                     new hashes             same hashes as before
                     must upload            dedup hits 100%

Ripple zone size ≈ window_size / average_chunk_size
For 48-byte window, 4MB average chunks:
  ripple zone ≈ 48 / 4,194,304 ≈ negligible
  In practice: 1-2 chunks affected regardless of file size
```

**This is the key insight:** A 50-byte insertion in a 10GB file affects 1-2 chunks (~4-8MB), not 2,500 chunks (10GB). The damage is bounded by chunk size, not by file size.

### Full Walkthrough: 50-Byte Edit in a 20MB File

```
FIXED CHUNKING:
Before:  [abc][def][ghi][jkl][mno]   (5 × 4MB chunks)
After:   [XYZ][PQR][STU][VWX][YZA]   (all 5 changed)
Upload:  20MB

CDC CHUNKING:
Before:  [abc][def][ghi][jkl][mno]   (5 × ~4MB chunks, variable)
After:   [XYZ][def][ghi][jkl][mno]   (only chunk 1 changed — ripple zone)
          ↑     ↑     ↑     ↑     ↑
       changed  ← all unchanged, same hashes, dedup hits →
Upload:  ~4MB  (one chunk re-uploaded)
Savings: 80%
```

---

## 9. Client SDK Implementation

CDC runs entirely on the client. The server never participates in chunking decisions — it only stores and retrieves chunks by hash.

### Full Pseudocode

```python
import hashlib
from collections import deque

# ─── Configuration ────────────────────────────────────────────────────────────
WINDOW_SIZE    = 48           # bytes in rolling window (Rabin standard)
AVG_CHUNK_SIZE = 4 * 1024 * 1024   # 4MB target average
MIN_CHUNK_SIZE = 1 * 1024 * 1024   # 1MB minimum — prevent tiny chunks
MAX_CHUNK_SIZE = 16 * 1024 * 1024  # 16MB maximum — prevent huge chunks

# Mask derived from target average:
# boundary fires with probability 1/AVG_CHUNK_SIZE
# mask has log2(AVG_CHUNK_SIZE) bits set
BOUNDARY_MASK  = AVG_CHUNK_SIZE - 1   # 0x3FFFFF for 4MB average
BOUNDARY_VALUE = 0                    # fire when (hash & MASK) == 0

# Precomputed Rabin lookup tables (computed once at startup)
T = precompute_rabin_table()          # T[byte] = hash contribution entering
U = precompute_rabin_table_out()      # U[byte] = hash contribution leaving

# ─── Rolling Hash State ───────────────────────────────────────────────────────
def rabin_init():
    return {
        'hash':   0,
        'window': deque(maxlen=WINDOW_SIZE),
        'filled': False   # window not full yet — don't check boundary
    }

def rabin_slide(state, byte_in):
    """
    Slide the window one byte forward.
    Returns True if a chunk boundary should be cut here.
    """
    window = state['window']

    # If window is full, the oldest byte is about to leave
    if len(window) == WINDOW_SIZE:
        byte_out = window[0]
        state['hash'] ^= U[byte_out]    # remove leaving byte's contribution
        state['filled'] = True

    # Rotate hash left by 1 bit (slide the polynomial)
    h = state['hash']
    state['hash'] = ((h << 1) | (h >> 63)) & 0xFFFFFFFFFFFFFFFF

    # Add entering byte's contribution
    state['hash'] ^= T[byte_in]

    # Append new byte to window
    window.append(byte_in)

    # Only check boundary once window is full
    return state['filled'] and (state['hash'] & BOUNDARY_MASK) == BOUNDARY_VALUE


# ─── CDC Splitter ─────────────────────────────────────────────────────────────
def cdc_split(file_path):
    """
    Split a file into variable-size content-defined chunks.
    Returns list of (chunk_bytes, sha256_hash) tuples.
    Runs in a background thread — do not call from UI thread.
    """
    chunks = []
    rabin = rabin_init()

    with open(file_path, 'rb') as f:
        file_bytes = f.read()   # stream in 64KB blocks in production

    chunk_start = 0

    for i, byte in enumerate(file_bytes):
        is_boundary = rabin_slide(rabin, byte)
        chunk_size  = i - chunk_start + 1

        # Enforce minimum chunk size — ignore boundary signals
        if chunk_size < MIN_CHUNK_SIZE:
            continue

        # Enforce maximum chunk size — force a cut regardless of hash
        if chunk_size >= MAX_CHUNK_SIZE:
            chunk_data = file_bytes[chunk_start : i + 1]
            chunks.append((chunk_data, sha256(chunk_data)))
            chunk_start = i + 1
            rabin = rabin_init()   # reset rolling hash state
            continue

        # Natural boundary detected by content pattern
        if is_boundary:
            chunk_data = file_bytes[chunk_start : i + 1]
            chunks.append((chunk_data, sha256(chunk_data)))
            chunk_start = i + 1
            rabin = rabin_init()   # reset rolling hash state for next chunk

    # Final partial chunk (always present unless file ends exactly on a boundary)
    if chunk_start < len(file_bytes):
        chunk_data = file_bytes[chunk_start:]
        chunks.append((chunk_data, sha256(chunk_data)))

    return chunks

def sha256(data):
    return hashlib.sha256(data).hexdigest()
```

### Why Reset the Hash After Each Chunk?

After cutting a chunk, the rolling hash state is reset to zero. This is a deliberate design choice:

```
Without reset: boundary decisions in chunk N depend on bytes from chunk N-1
               → inserting a byte in chunk N-1 could change boundary in chunk N
               → defeats the purpose of CDC

With reset: each chunk's boundaries depend only on its own content
            → independent, stable chunks
```

### Performance Characteristics

```
Algorithm         Speed per core    Notes
──────────────────────────────────────────────────────────
Rabin (naive)     ~200 MB/sec       Byte-by-byte with lookup tables
FastCDC           ~3,000 MB/sec     Skip ahead when hash can't match
gear-hash CDC     ~1,500 MB/sec     Simplified polynomial, fast on modern CPUs

For a 100MB file:
  Rabin:   ~500ms   → background thread, imperceptible
  FastCDC: ~33ms    → near-instant

Threading requirement: ALWAYS run in a background thread.
If run on the UI thread:
  100MB file × 500ms = UI frozen for half a second per upload
  10 files queued = 5 seconds of UI freeze
  Users notice anything > 100ms
```

### FastCDC — The Production Variant

Dropbox and most modern systems use FastCDC (2016) rather than classic Rabin. The key optimization: use a **gear hash** (simpler polynomial, fits in a register) and **skip ahead** when the hash cannot possibly match the boundary condition within the minimum chunk size.

```python
# FastCDC key optimization:
# If we're at position i and chunk_size < MIN_CHUNK_SIZE,
# don't compute hash at all — skip ahead to MIN_CHUNK_SIZE
# This eliminates 50-90% of hash computations

def fastcdc_split(file_bytes):
    chunks = []
    i = 0
    n = len(file_bytes)

    while i < n:
        chunk_start = i
        hash_val = 0

        # Skip to minimum chunk size without hashing
        i += min(MIN_CHUNK_SIZE, n - chunk_start)

        # Now look for natural boundary up to max chunk size
        while i < chunk_start + MAX_CHUNK_SIZE and i < n:
            hash_val = (hash_val >> 1) + GEAR[file_bytes[i]]   # gear hash
            if (hash_val & BOUNDARY_MASK) == 0:
                break   # natural boundary found
            i += 1

        chunk_data = file_bytes[chunk_start:i]
        chunks.append((chunk_data, sha256(chunk_data)))

    return chunks

# GEAR table: 256 random 64-bit values, one per byte value
# Generated once, hardcoded into the SDK
GEAR = [random 64-bit integer for each byte value 0-255]
```

---

## 10. Cross-File Deduplication

CDC's benefit extends beyond a single file's edit history. Because chunks are content-addressed globally, identical content across completely different files shares storage.

### Scenario: Two Users Upload Similar Documents

```
User A uploads: "Q4 Strategy v1.docx"  (12MB)
CDC produces:
  Chunk 1: [executive summary section]  hash = abc  4.1MB
  Chunk 2: [charts and tables section]  hash = def  3.8MB
  Chunk 3: [appendix section]           hash = ghi  4.1MB

Server stores: abc, def, ghi  →  12MB in S3

────────────────────────────────────────────────────

User B uploads: "Q4 Strategy FINAL.docx"  (12MB)
(Same document, only appendix was updated)
CDC produces:
  Chunk 1: [executive summary section]  hash = abc  4.1MB  ← identical content
  Chunk 2: [charts and tables section]  hash = def  3.8MB  ← identical content
  Chunk 3: [new appendix section]       hash = xyz  4.1MB  ← changed

Server dedup check:
  abc → EXISTS  →  skip
  def → EXISTS  →  skip
  xyz → NEW     →  upload 4.1MB

User B's upload: 4.1MB instead of 12MB  (66% bandwidth savings)
S3 storage:      16.1MB instead of 24MB  (33% storage savings)
```

### Scenario: Same File on Multiple Devices

```
User syncs "project.zip" (500MB) to 3 devices:
  Device 1 (laptop):   downloads 500MB  — first device, full download
  Device 2 (desktop):  downloads 0MB    — already synced from Device 1's upload
  Device 3 (phone):    downloads 0MB    — already synced from Device 1's upload

Total egress without dedup: 1,500MB
Total egress with dedup:      500MB  (67% savings)
```

### Why 60-70% Dedup Rate is Realistic

Dropbox's publicly disclosed dedup rates make sense when you think about real-world file behavior:

```
File Type           Typical Dedup Rate    Why
──────────────────────────────────────────────────────────────────────
Office documents    70-85%                Documents evolve incrementally.
                                          Each edit touches 1-3 chunks.
                                          Body content rarely changes entirely.

Source code         60-80%                Commits touch few lines.
                                          Most files in a repo unchanged.

Photos              30-50%                Photos are unique per capture.
                                          But re-edits (crop, filter) share
                                          original content chunks.

Videos              10-20%                Highly compressed already.
                                          CDC finds few boundaries.
                                          Each frame is unique.

Compressed archives 5-10%                 ZIP/tar content is pre-compressed.
                                          Rolling hash finds no patterns.
                                          CDC largely ineffective.
```

**The overall 60-70% figure holds because documents and source code dominate user storage at Dropbox's consumer base.** An enterprise product serving video production companies would see 15-20% and need a different economic model.

This is why monitoring `dedup_hit_rate` in production is critical — it directly validates whether your storage cost projections are correct.

---

## 11. Server-Side View — Chunk-Agnostic by Design

The server never knows or cares about CDC. Its interface is completely simple:

```
Client says: "Here are the SHA-256 hashes of my chunks"
Server says: "Here are the ones I already have. Upload the rest."
```

```sql
-- The entire server-side dedup logic is this one query:
SELECT hash FROM chunks
WHERE hash = ANY(:client_chunk_hashes);
-- Returns hashes already stored → client skips those
-- Missing from result → client uploads those
```

This architectural separation is powerful:
- You can improve the chunking algorithm (switch from Rabin to FastCDC, change chunk size) by shipping a new client SDK version
- The server doesn't change at all
- Old clients using old CDC parameters and new clients using new parameters both work simultaneously
- The server is just a content-addressed blob store: `hash → bytes`

### What the Server Stores

```sql
-- chunks table: the content-addressed blob index
CREATE TABLE chunks (
  hash        VARCHAR(64) PRIMARY KEY,  -- SHA-256 hex string (64 chars)
  s3_key      TEXT NOT NULL,            -- 'chunks/ab/abc123...'
  size_bytes  INTEGER NOT NULL,
  ref_count   INTEGER NOT NULL DEFAULT 1,  -- how many file_versions reference this
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- S3 key derivation: use first 2 chars of hash as prefix
-- avoids S3 hot partition on a single prefix
s3_key = 'chunks/' + hash[0:2] + '/' + hash
-- e.g., hash = 'abc123...' → s3_key = 'chunks/ab/abc123...'
```

### The ref_count — Chunk Garbage Collection

Every `file_version` that references a chunk increments `ref_count`. Every deletion decrements it. When `ref_count` reaches 0, the chunk is eligible for GC.

```sql
-- On file version creation: increment ref_count for each new chunk
UPDATE chunks SET ref_count = ref_count + 1
WHERE hash = ANY(:new_chunk_hashes);

-- On file version deletion / expiry: decrement
UPDATE chunks SET ref_count = ref_count - 1
WHERE hash = ANY(:deleted_version_chunk_hashes);
-- SINGLE SQL STATEMENT = atomic. No race condition.

-- GC job (runs daily, S3 deletes are cheap):
DELETE FROM chunks WHERE ref_count = 0 AND created_at < now() - interval '1 day';
-- The 1-day grace period handles the edge case where:
-- a new upload is in progress (UPLOADING status) and references a chunk
-- that just hit ref_count = 0 due to a concurrent deletion elsewhere.
```

> **Critical:** Never delete an S3 object without first setting `ref_count = 0` and waiting for the GC grace period. The ref_count decrement and the S3 deletion must not happen in the same transaction — they are in different systems.

---

## 12. File Reconstruction

When a client downloads a file, it reconstructs it from ordered chunks:

```sql
-- Get chunks in correct order for a file version
SELECT
    h.ord,
    h.hash,
    c.s3_key,
    c.size_bytes
FROM file_versions fv
CROSS JOIN LATERAL
    unnest(fv.chunk_hashes) WITH ORDINALITY AS h(hash, ord)
JOIN chunks c ON c.hash = h.hash
WHERE fv.version_id = :version_id
ORDER BY h.ord ASC;
```

Client reconstruction:
```python
def reconstruct_file(version_id, output_path):
    # 1. Get ordered chunk list from server
    chunks = api.get_version_chunks(version_id)
    # Returns: [{ord: 1, hash: 'abc', s3_key: '...'}, ...]

    with open(output_path, 'wb') as out:
        for chunk in sorted(chunks, key=lambda c: c['ord']):
            # 2. Check local cache first
            local_path = local_chunks.get(chunk['hash'])
            if local_path:
                # Already have this chunk on disk — read from local cache
                with open(local_path, 'rb') as f:
                    out.write(f.read())
            else:
                # Download from S3 via presigned URL
                chunk_bytes = download_from_s3(chunk['hash'])
                out.write(chunk_bytes)
                # Save to local chunk cache
                local_chunks.save(chunk['hash'], chunk_bytes)
```

This is why the client's `local_chunks` table is file-independent and hash-keyed. Any chunk from any version of any file can satisfy a download request for any other file that shares that chunk. The client's local disk is a content-addressed cache with the same addressing scheme as S3.

---

## 13. CDC vs Fixed Chunking — Full Comparison

| Property | Fixed-Size Chunking | Content-Defined Chunking |
|---|---|---|
| **Boundary type** | Byte position | Content pattern (rolling hash) |
| **Small edit near file start** | ALL subsequent chunks change → full re-upload | 1-2 chunks change (ripple zone) → minimal upload |
| **Small edit near file end** | Only last chunk changes | Only 1-2 chunks change — same result |
| **Identical files by different users** | No dedup (different file_ids, no hash check) | Full dedup — shared chunks, shared S3 storage |
| **Partially identical files** | No dedup | Dedup on identical segments |
| **Chunk size** | Exactly fixed (e.g., exactly 4MB) | Variable (e.g., 1MB to 16MB, avg 4MB) |
| **Boundary stability** | Changes with any byte shift | Anchored to content — stable after ripple zone |
| **Client computation** | O(1) per chunk (just count bytes) | O(n) per file (hash every byte) — background thread |
| **Server changes needed** | None | None (server is chunk-agnostic) |
| **Algorithm complexity** | Trivial | Moderate (rolling hash, lookup tables) |
| **Dedup rate at Dropbox scale** | ~5-10% (accidental hash collisions only) | ~60-70% |
| **Storage cost at 22 PB/year** | ~22 PB/year | ~7-8 PB/year |
| **Economic viability** | Requires 3× more storage | Makes unit economics work |

---

## 14. Key Numbers to Remember

```
Window size (Rabin):        48 bytes
Average chunk size:         ~4MB (Dropbox)
Min chunk size:             1MB   (prevents tiny fragments)
Max chunk size:             16MB  (prevents huge monolithic chunks)
Boundary probability:       1 / avg_chunk_size
Ripple zone after edit:     1-2 chunks regardless of file size
Rabin fingerprint speed:    ~200 MB/sec per core
FastCDC speed:              ~3,000 MB/sec per core
100MB file processing:      ~500ms (Rabin) / ~33ms (FastCDC)
Healthy dedup rate:         60-70% (document/code heavy workload)
Dedup rate for video:       10-20% (pre-compressed, few patterns)
Alert threshold:            dedup_hit_rate < 30% over 1-hour window
SHA-256 hash size:          64 hex chars = 32 bytes per chunk reference
S3 key prefix:              first 2 chars of hash (avoids hot partitions)
GC grace period:            1 day after ref_count reaches 0
```

---

## Summary: Why CDC Is Not Optional

Fixed-size chunking is not a starting point you can optimize later. It is fundamentally broken for mutable files because the boundary shift problem defeats deduplication entirely — every small edit uploads the entire file.

CDC solves the boundary shift problem by anchoring cuts to content patterns rather than byte positions. The result:

- A 50-byte insertion in a 10GB file uploads ~4MB, not 10GB
- Two users storing similar documents share chunks transparently
- Storage costs scale sub-linearly with the number of edits
- The server stays simple — it's just a content-addressed blob store

At Dropbox's scale (22 PB/year raw growth), the difference between 10% and 70% dedup rate is **~13 PB/year** of avoided storage, or roughly **$130M/year** in S3 costs at $0.01/GB-month. CDC is not an optimization — it is what makes the economics of cloud file sync viable.
