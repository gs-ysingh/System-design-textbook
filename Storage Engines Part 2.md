# Storage Engines II — Learnings Summary

## 🧠 Storage Engine Concepts and Learnings

### 1. Understanding MySQL Storage Engines
- A **storage engine** defines *how* data is stored, indexed, and retrieved.
- **InnoDB** (default) → Disk-based, supports transactions and persistence.  
- **MEMORY Engine** → In-memory, zero persistence, ultra-fast for cache-like operations.
- You can switch storage engines in MySQL using `ALTER TABLE ... ENGINE = MEMORY;`
- Great use case: implement cache on top of MySQL without losing SQL query support.

**Takeaway:**  
By changing the MySQL storage engine, you can turn it into a high-performance in-memory cache that still supports SQL queries, indexing, and atomic operations.

---

### 2. Real-world Example: Key-Value Store on MySQL
- Built a **key-value database** on MySQL supporting JSON storage and secondary indexes.
- Sharded the data manually using **consistent hashing**.
- Needed **identical schemas** for cache and DB to make switching seamless.
- Each DB shard (`M1`, `M2`, `M3`) had a corresponding cache (`C1`, `C2`, `C3`).
- Implemented `get/put/delete` consistency between MySQL (persistent) and MEMORY engine (cache).

**Takeaway:**  
Use **identical schemas** and **storage-engine abstraction** to maintain transparent query behavior between cache and persistent layers.

---

## 📘 Designing a Word Dictionary (File-based Storage)

### Problem:
Build a 1 TB word dictionary *without* using a traditional DB.

### Step-by-step evolution of design:
1. **Naive Approach – Flat Files**
   - Store as `word,meaning` (CSV).
   - Inefficient because lookups require full file scans.

2. **Partitioned Approach**
   - Split by first letter: `a.txt`, `b.txt`, etc.
   - Faster lookups, but scaling to 1 TB causes inefficiency.

3. **Index + Data Blocks (B-Tree inspired)**
   - Create an **index file** mapping ranges of words → disk blocks.
   - Each block holds sorted key-value pairs.
   - Unit of read/write = **disk block size** (4 KB or 8 KB).

4. **Variable-length Data Problem**
   - Store `word + pointer` instead of `word + meaning`.
   - Pointers hold (filename, offset) → retrieves full meaning.

5. **Making It Portable**
   - Combine all files into one **.dat** file + fixed **header**.
   - Header stores offsets and lengths of sections (index/data).
   - Enables one-click copy, easy recovery, and OS-independent portability.

**Takeaway:**  
Every database or file format (e.g., Redis RDB, JPG, RocksDB) follows a similar **Header + Index + Data** pattern for efficient, portable storage.

---

## 💾 Indexing and Query Optimization

### Building a Custom Index:
- Create an `index.dat` file storing:  
  `word | start_offset | length`
- Load it in memory (≈2.6 MB for 1 TB dictionary).
- During query:
  - Lookup word in memory.
  - Fetch from S3 using byte-range read (`Range: bytes=a-b`).

**Result:**  
- Random access (O(1)) for reads.  
- Sequential writes (O(1)) for updates.

---

## ⚙️ Handling Updates and Consistency

### Dictionary Updates (Weekly)
- Use **merge-sort merge** between old dictionary and change log.
- Generate new dictionary + index.
- Upload to S3.

### Preventing Garbage Data
- Avoid in-place overwrite.
- Version with epoch suffix (`index.1.dat`, `dict.1.dat`).
- Maintain a `meta.json` pointer to current version.
- Rolling update servers gracefully.

**Result:**  
Users get **stale but correct** data (never corrupted).

---

## 🔢 Compression and Optimization
- Apply **Huffman compression** to save ~40%.
- Remove redundant separators.
- Reduce `index.dat` from 2.6 MB → ~2.1 MB.

---

## 🧩 Log-Structured Storage Engines

### Core Principle
- Writes are **append-only**, no random writes or seeks.
- Reads can seek to exact offsets.
- Deletes are marked (not erased).
- Enables extremely **high write throughput**.

### How It Works
Each record: `[CRC][timestamp][key_size][value_size][key][value]`

- **CRC (checksum)** → detects incomplete writes.
- **Put/Delete:** append-only.
- **Get:** sequential scan (optimized with in-memory index).

### Optimizing Reads
- Maintain hash map: `{ key → (file_id, offset, entry_size) }`
- Each `get`: one hash lookup + one disk seek.

**Result:** O(1) read & write operations.

---

## ⚡ Performance and Reliability

| Operation | Complexity | Notes |
|------------|-------------|-------|
| `put` | O(1) | Sequential append |
| `get` | O(1) | Hash lookup + 1 disk seek |
| `delete` | O(1) | Append special marker |
| `merge/compact` | O(N) | Periodic cleanup |

**Benefits:**
- Predictable performance.  
- Easy crash recovery (CRC).  
- Excellent IO efficiency.  
- Simplified backups (copy files).

---

## 🧱 Real-world Implementations

| System | Based On | Notes |
|--------|-----------|-------|
| **Bitcask** | Log-structured KV store | Used in **Riak** |
| **Riak** | Distributed DB | Used by **Uber** |
| **Kafka** | Log-structured messaging | Sequential append |
| **LevelDB / RocksDB** | LSM-tree | Log-based |
| **Redis RDB** | Snapshot file | Header + Index + Data |

---

## 🔍 Key Takeaways
1. **Changing storage engines** changes persistence/performance.  
2. **Indexes turn sequential stores into O(1) systems.**  
3. **Appending-only writes** = predictable performance.  
4. **Merging/Compaction** = database garbage collection.  
5. **Header + Index + Data** = universal design pattern.  
6. **Always define constraints early** — they drive design decisions.

---
