# Algorithmic System Design I – Learnings Summary

A collection of core system design concepts derived from **Algorithmic System Design I** session, focusing on three production-grade problems: **Impressions Counting**, **Remote File Sync**, and **Rate Limiting**.

---

## 1. Impressions Counting System (AdTech / Social Media Analytics)

### Problem

Count **unique visitors/impressions** in near real-time for posts or ads over the last *N* days, where *N* can change dynamically.

**Challenges:**

* Must be **real-time or near real-time**
* Must support **billions of events**
* Must allow **arbitrary time window queries**
* Each user counted **once per window**
* Approximation is acceptable

---

### Naïve Approach

Use a `Set<UserID>` per post/hour/day to compute unique visitors.

**Problems:**

* Memory explosion: 1M users ≈ 8 MB per post/hour
* CPU- and memory-intensive set unions
* Poor scalability for large ad volumes

---

### Solution: HyperLogLog (HLL)

A **probabilistic cardinality estimation algorithm** that approximates the number of unique elements using constant memory.

**Properties:**

* Uses hash representations instead of raw user IDs
* Supports: `PFADD`, `PFCOUNT`, and `PFMERGE`
* Approximation error: ±2%

**Memory Comparison:**

| Metric             | Set    | HyperLogLog |
| ------------------ | ------ | ----------- |
| Space for 1M users | 4–8 MB | 12 KB       |
| Relative size      | 100%   | 0.15%       |

**Trade-offs:**

* Cannot remove elements
* Approximate count (but sufficient for analytics)

---

### System Architecture

```text
Client → Event Collector Service → Kafka → Rule Engine → Kafka → Counter Service → Redis (HLL) → DynamoDB (backup)
```

**Redis Commands:**

* `PFADD key userId`
* `PFCOUNT key`
* `PFMERGE key resultKeys...`

**Persistence:**

* Dump Redis every 10s to DynamoDB
* On cache miss, restore from DynamoDB

**Real-World Usage:** Google Ads, YouTube Views, LinkedIn Posts, Instagram Insights

---

## 2. Remote File Sync (Dropbox Design)

### Problem

Synchronize local files with cloud efficiently — uploading, updating, and downloading with minimal bandwidth.

### Solution: Chunking + Hashing

Each file split into **4 MB chunks**, each hashed via SHA-256.

Example:

| Block | Size | Hash |
| ----- | ---- | ---- |
| B1    | 4 MB | H1   |
| B2    | 4 MB | H2   |
| B3    | 4 MB | H3   |
| B4    | 2 MB | H4   |

---

### Database Design

#### File Metadata DB

| Column          | Description                      |
| --------------- | -------------------------------- |
| `namespace_id`  | User/account ID                  |
| `relative_path` | e.g. `/video.avi`                |
| `block_list`    | [H1, H2, H3, H4]                 |
| `version_id`    | Monotonically increasing version |

* **Append-only DB** – never updates, only new entries on change.
* Enables **efficient version tracking**.

#### Blocks DB

| Column       | Description             |
| ------------ | ----------------------- |
| `account_id` | User ID                 |
| `hash`       | Block hash              |
| `block`      | Actual 4 MB binary data |

---

### Upload Flow

1. Client splits file → computes hashes
2. Sends hashes to **Meta Server**
3. Meta Server checks which hashes are missing on **Block Server**
4. Client uploads only missing blocks
5. Meta Server commits metadata

**Result:** Only changed or missing chunks are uploaded.

---

### Download / Sync Flow

1. Client polls: *I have version X, is there an update?*
2. Meta Server replies with latest version Y
3. Client compares hash lists, downloads only new blocks
4. Client reconstructs file sequentially

**Benefits:**

* No re-download of unchanged blocks
* Append-only schema simplifies versioning
* Seamless sync across devices

---

### Versioning Logic

* Each upload adds a new row with new `version_id`
* Client tracks its last synced version
* Server sends only delta block info

**Example:**
Old: [H1, H2, H3, H4] → New: [H1, H2, H5, H4] → Only H3 changed → Download H5

---

### Key Takeaways

* Chunked sync reduces bandwidth
* Append-only DB = easy change detection
* Version IDs enable efficient polling
* Used in Dropbox, Google Drive, Git, S3 multipart uploads

---

## 3. Rate Limiter Design

### Purpose

Protect system from overload or abuse by limiting API calls per user/IP/token.

**Use cases:**

* API abuse prevention
* Fair usage enforcement
* Tier-based billing

---

### Algorithms Overview

| Algorithm          | Description                        | Pros              | Cons                   |
| ------------------ | ---------------------------------- | ----------------- | ---------------------- |
| **Leaky Bucket**   | Queue requests, leak at fixed rate | Smooth traffic    | Latency due to queuing |
| **Fixed Window**   | Allow N requests per minute        | Simple            | Boundary spikes        |
| **Sliding Window** | Move window every second           | Smooth + accurate | Slightly complex       |

---

### Sliding Window Implementation

#### Data Structures

* `store[epochSecond] = count`
* `count = cumulative requests (last 60s)`
* `lastSecond = last updated timestamp`

#### Algorithm

```text
onRequest():
  currentSecond = now()
  if count >= LIMIT:
      return 429
  count += 1
  store[currentSecond] += 1

  if currentSecond > lastSecond:
      expiredSecond = currentSecond - WINDOW_SIZE
      count -= store[expiredSecond]
      delete store[expiredSecond]
      lastSecond = currentSecond
```

**Handles:**

* Smooth traffic control
* Prevents burst spikes
* Works efficiently with atomic Redis commands

---

### Redis-Based Rate Limiter

Use atomic operations:

* `INCR key`
* `EXPIRE key 60`

Or via Lua script for full atomicity.

**Response headers:**

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 20
X-RateLimit-Reset: 1730200000
```

**HTTP 429** → `Too Many Requests`

---

### Summary Table

| Topic                    | Technique                       | Core Store           | Benefit                            |
| ------------------------ | ------------------------------- | -------------------- | ---------------------------------- |
| **Impressions Counting** | HyperLogLog                     | Redis + Kafka        | Real-time, memory-efficient counts |
| **File Sync**            | Chunking + Append-only Metadata | Meta + Block Servers | Sync only deltas                   |
| **Rate Limiting**        | Sliding Window                  | Redis / In-memory    | Smooth control, fairness           |

---

**End of Notes – Algorithmic System Design I**
