# 🧠 Storage Engines — Part I: Distributed Cache Deep Dive

## 1️⃣ Foundational Concepts
- A **distributed cache** is simply a cache spread across multiple nodes.
- The cache can be:
  - **Distributed + Replicated:** Each node has the same data (like master-slave DB replication).
  - **Distributed + Sharded:** Each node holds a unique subset of data (typical design).
- Storage difference:
  - **Cache:** Data in memory.
  - **Database:** Data persisted on disk.

---

## 2️⃣ Evolution of a Cache System

| Stage | Architecture | Problem | Solution |
|-------|--------------|----------|-----------|
| **Day 0** | One API server with in-memory hashmap | Out of RAM | Vertical scaling |
| **Day 1** | Bigger machine with more RAM | Too many client requests | Add load balancer + multiple API servers |
| **Day 2** | Multiple API servers each holding its own cache | Data inconsistency across servers | Separate **Storage Node** |
| **Day 3** | API servers (stateless) + central storage node | Single point of failure in storage | Replication + eventual horizontal scale |

**Key takeaway:**  
Separate **storage** (RAM-heavy, fault-tolerant) from **API layer** (CPU/network-heavy, stateless).

---

## 3️⃣ Cache Internals

### Basic Operations
- Minimum supported ops: `GET`, `PUT`, `DELETE`
- Redis-like systems just expose advanced data structures over this foundation.

### Storage Implementation
- Internal data structure: **Hash Table**
- Each key can have a **type** (e.g., list, tree, graph, trie, etc.)
- Use **polymorphism/inheritance** in low-level design to handle different data types under one API.

---

## 4️⃣ Scaling Strategy

### When to Scale
- **Vertical scaling** hits RAM or CPU limits.
- **Horizontal scaling** = add more storage nodes and shard data.

### Sharding Model
```
node_index = hash(key) % N
```
But this naive approach has problems (explained later under Consistent Hashing).

---

## 5️⃣ Cache Eviction Policies

| Algorithm | Description | Use Case |
|------------|--------------|-----------|
| **LRU (Least Recently Used)** | Evict least recently accessed key | News Feed (recent > old) |
| **LFU (Least Frequently Used)** | Evict least accessed key | Wikipedia (popular pages stay) |
| **Random Eviction** | Evict random key | Good when cache hit ratio > 80–90% |

💡 **Insight:** Random eviction gives similar performance to LRU/LFU when dataset fits well in cache and saves memory by avoiding complex data structures.

---

## 6️⃣ Expiry / TTL Management

- Each key stores an **absolute expiration time**, not relative.
- Use a **Min-Heap / Priority Queue** to keep keys sorted by expiry time.
- The top of the heap always holds the next key to expire.
- Background thread removes expired keys periodically.
- Optimization: Use **Linked List** if all keys have fixed TTL (faster insertion/deletion).

---

## 7️⃣ Concurrency Control in Cache

### Common Problem
- Two writes on same key → data race.

### Approaches

| Method | Description | Example / Notes |
|---------|-------------|----------------|
| **Locks (Pessimistic)** | Wait to acquire lock before write | Ensures serializable updates, lower throughput |
| **Optimistic Locking** | Assume no conflict; use conditional update | “Update value where current_value = X” |
| **Single-threaded Design** | No concurrency issues | Redis, Node.js — each command atomic |

**Optimistic locking rule:**  
> “Update what you read.”  
If another writer changes the data between read and write, retry.

**Compare-and-swap (CAS)** operations implement optimistic locking efficiently.

---

## 8️⃣ Using MySQL as a Cache

Challenge:  
Can we implement an in-memory–like cache on MySQL with similar throughput?

Idea:  
- Use **large buffer pool** or **in-memory table engines**.
- Requires tuning and custom indexing — still slower than pure RAM cache.
- Good for durability but not true “cache” latency.

---

## 9️⃣ Consistent Hashing

### Problem with Naive Hashing
If hash = `f(key) % N`,  
Adding/removing a node changes **N**, forcing **massive data reshuffling**.

### Solution
**Consistent Hashing Ring**
- Represent hash space as a ring (0–max_hash).
- Each storage node is assigned a hash position.
- A key belongs to the **next node clockwise** from its hash value.
- When adding/removing nodes → **only nearby range** redistributes → **minimal data movement**.

### Data Movement Example
- Old hash: mod 2 → keys split across 2 nodes.
- Add new node → mod 3 → ~50% data moves (bad).
- Consistent hashing → only few keys move to new node (good).

### Implementation Options
- **Binary Search on Sorted Array** of node hashes → find “next-right” node.
- Maintain arrays:
  - `nodes[]` – list of servers  
  - `keys[]` – sorted hash positions
- Use binary search to find correct node index for a key.

### Advanced Optimizations
- **Virtual Nodes:** multiple hash points per node → better balance.
- **Weighted Hashing:** assign weight-based range (used by GitHub LB).

---

## 🔁 Handling Node Add/Remove

| Action | Effect | Handling |
|--------|---------|----------|
| **Add node** | Only nearby key range moves | Copy subset from nearest node or allow temporary cache misses |
| **Remove node** | Neighbor node takes over | Cache misses until re-populated or copy data before removal |
| **Crash recovery** | Load from RDB dump (Redis) | Warm start from disk |

---

## 🔐 High Availability

| Method | Description |
|--------|--------------|
| **Read Replicas** | Hot/warm standby cache servers |
| **Cross-DC Replication** | For critical infra |
| **RDB Dump / Snapshot** | Periodic disk dump to preload cache after restart |

💡 **Tip:** Always let some traffic hit DB (e.g., 30%) even when cache-hit ratio is high — prevents DB under-provisioning.

---

## 🔄 Distributed Hash Table (DHT)

- Core of decentralized systems (BitTorrent, DNS, peer-to-peer networks).
- Maps (key → node) in a **distributed, fault-tolerant** way.
- Each node knows some peers and forwards requests if it doesn’t own the key.
- Used in: BitTorrent, blockchain routing, distributed file systems.

---

## 🔍 Real-World Analogies
- **Load Balancer** = Bank’s reception (routes you to correct counter)
- **Consistent Hashing Ring** = Circular counter system where each person handles a segment.
- **Cache Eviction** = Cleaning old files from your desk drawer.
- **Pessimistic Locking** = Wait for your turn.
- **Optimistic Locking** = Assume no one interferes; retry if they do.

---

## 🧩 Assignments / Thought Experiments
1. **Implement Word Dictionary (Get/Put/Delete)** without traditional DB — your own mini storage engine.
2. **Build Cache using MySQL** but try achieving Redis-like throughput.
3. **Read / Implement:**  
   - DynamoDB research paper (consistent hashing)  
   - LFU O(1) implementation paper  
   - BitTorrent routing / Gossip Protocols  
