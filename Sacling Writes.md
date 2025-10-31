# Scaling Writes — Simplified

## Goal
Learn how to handle high write volumes when a single database or server becomes a bottleneck.

---

## 💡 The Challenge

Writes become a bottleneck faster than reads because each write modifies data. As systems scale (e.g., millions of posts or transactions per second), disk I/O, CPU, and network limits are quickly reached.

Interviewers often test how you identify and fix these bottlenecks.

---

## ⚙️ Core Strategies to Scale Writes

There are four main strategies to scale writes effectively:

1. **Vertical Scaling and Write Optimization**
2. **Sharding and Partitioning**
3. **Queues and Load Shedding**
4. **Batching and Hierarchical Aggregation**

---

## 1️⃣ Vertical Scaling & Write Optimization

### Vertical Scaling
Before adding complexity, ensure the single server is fully optimized.

- Upgrade hardware (faster disks, more cores, higher network bandwidth)
- Do quick math to confirm hardware limits are actually reached

### Database Optimization
Choose or tune the database for write-heavy workloads.

**Examples:**
- **Cassandra**: Append-only writes → high throughput, slower reads
- **Time-series DBs** (e.g., InfluxDB): Optimized for sequential writes
- **Column stores** (e.g., ClickHouse): Efficient batched writes

**Tuning Tips:**
- Remove unnecessary indexes and constraints
- Tune WAL (e.g., group commits in PostgreSQL)
- Disable expensive triggers or full-text indexing during heavy writes

> **Trade-off**: Write optimization usually slows down reads — choose based on your workload balance.

---

## 2️⃣ Sharding and Partitioning

When one server can't keep up, split the load across multiple servers.

### Horizontal Sharding
Distribute data across shards by a key (e.g., `userID`).

- **Good keys** spread data evenly
- **Bad keys** (e.g., country) cause hot spots

### Vertical Partitioning
Split data by column or feature type.

**Example for a social app:**
- **Post content** → read-heavy
- **Engagement metrics** → write-heavy
- **Analytics/events** → append-only

Each can live in different databases optimized for its access pattern.

---

## 3️⃣ Handling Bursts — Queues & Load Shedding

### Write Queues (Kafka, SQS)
- Absorb bursts by decoupling acceptance from processing
- Database writes happen steadily while queues smooth spikes
- **Trade-off**: eventual consistency and processing delay
- Use queues for temporary bursts, not constant overload

### Load Shedding
When overwhelmed, drop less-critical writes.

**Examples:**
- Skip redundant location updates (Uber, Strava)
- Drop low-priority analytics during spikes

> **Better to lose non-essential data than crash the system.**

---

## 4️⃣ Batching & Hierarchical Aggregation

### Batching
Combine multiple small writes into one to reduce overhead.

- **Application layer**: group multiple writes before sending to DB
- **Intermediate layer**: "Like batcher" aggregates 100 likes → 1 write
- **DB layer**: configure flush intervals (e.g., Redis flush every 100ms)

### Hierarchical Aggregation
Aggregate data in stages instead of writing every single event.

**Example:**
- Live comments → aggregate likes/comments per node → send batched updates upstream
- Reduces fan-in/fan-out load dramatically

---

## 🧠 Interview Applications

| Scenario | Techniques |
|----------|-----------|
| **Instagram / Strava** | Sharding by user, vertical partitioning (posts, metrics, analytics) |
| **News Feed** | Partition by user; batching celebrity posts |
| **Search / Analytics** | Partition + batching for pre-processing |
| **Live Comments** | Hierarchical aggregation to handle millions of writes |

---
## 🚫 When Not to Scale Writes

**Don't over-engineer.**

- Always estimate load first — use back-of-envelope math
- Each technique adds complexity or trade-offs (latency, consistency, cost)

---

## 🔍 Common Deep Dives

### 1. Resharding
- Use dual-write migration: write to old + new shards until migration completes

### 2. Hot Keys
- Split keys (e.g., `post1Likes-0`, `post1Likes-1`, etc.) to distribute load
- Or dynamically create sub-keys for hot data
- Readers aggregate counts across sub-keys

---

## ✅ Key Takeaways

1. **Scale up first, then scale out**
2. **Choose the right database** for your write pattern
3. **Shard and partition** to distribute load evenly
4. **Use queues** for short-term bursts, shed load if needed
5. **Batch and aggregate** to reduce write frequency
