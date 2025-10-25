# High Throughput Systems II — Learnings Summary

## 🧩 Part 1: Log-Structured Merge (LSM) Trees

### 1. Background: From Bitcask to LSM Trees
- **Bitcask** was fast due to:
  - Append-only writes → high write throughput (no random writes)
  - In-memory key index → near O(1) lookups  
- **Limitation:** All keys must fit in memory. Not scalable.

### 2. The Goal
To make Bitcask *better*, not just faster — by removing its limitation of keeping all keys in memory.

### 3. Step-by-Step Evolution

#### (a) Buffering Writes
- Buffer writes in memory and flush periodically or when full.
- Benefit: fewer disk I/O → faster writes.
- Risk: possible data loss before flush.

#### (b) Handling Data Loss
- **Write-Ahead Log (Commit Log):**  
  Every operation (`put`, `delete`) is logged to an append-only file.  
  On restart → rebuild in-memory state; after flush → truncate log.

### 4. From Bitcask to Memtables
- Introduce **memtables** (sorted trees like Red-Black Tree).
  - Store in sorted order in memory.
  - When full → flush to disk as immutable sorted files.

### 5. Flushing Strategy
- **Option 1:** Append to same file → huge file, GC-like pauses.  
- **Option 2:** Create new file for each flush → smaller & faster.

### 6. Read Path
1. Check Memtable (latest data).
2. Check disk files (most recent first).

### 7. Introducing Sorted String Tables (SSTables)
- Each flush = **SSTable** = sorted key-value pairs + index.
- Supports binary search due to sorted keys.

### 8. Compaction and Merging
- Merge old SSTables → remove outdated keys.
- Saves disk space and optimizes reads.

### 9. Optimizing Reads — Bloom Filters
- Problem: Need to check all SSTables for missing keys.
- **Solution: Bloom Filter**  
  - Tells if key *definitely doesn’t exist* (saves disk I/O).  
  - One Bloom filter per SSTable.

### 10. Handling Data Loss
- Use commit logs for durability.
- After flush → clear log.
- On crash → replay log to rebuild.

### 11. Real Systems Using LSM Trees
- Used in **LevelDB**, **RocksDB**, **Cassandra**, **HBase**.
- Advantages:
  - High write throughput
  - Bounded memory
  - Efficient reads via compaction + Bloom filters

---

## 🎬 Part 2: YouTube-Like High Throughput Video Pipeline

### 1. Upload Flow
1. User uploads → API returns **signed URL**.
2. Browser uploads directly to blob storage.
3. User edits metadata (title, tags, etc.) in parallel.
4. Upload completion triggers API call → mark uploaded.
5. DB entry has `status: draft` → becomes `published` post-upload.

### 2. Handling Upload Failures
- Mark as `draft` or run cleanup jobs.
- Timeout jobs remove abandoned uploads.

### 3. Improving Upload Speed — Chunking
- Split file into chunks → upload in parallel threads.
- Retry only failed chunks.
- Avoid re-uploading whole file.

### 4. Transcoding Pipeline
- Raw → multiple formats/resolutions (240p–1080p, H.264/HEVC).
- Enables **adaptive bitrate streaming (ABR)**.

### 5. Implementation
- **FFmpeg** handles transcoding:  
  ```bash
  ffmpeg -i input.mp4 -vf scale=1280:720 output_720p.mp4
  ```
- Steps: download → process → upload → cleanup.

### 6. Workflow Management — Apache Airflow (DAG)
- Orchestrates multi-step pipelines.
- DAG nodes = processing units.
- Benefits: parallel execution, retries, visualization.

### 7. Event Backbone — Kafka
- Decouples systems via asynchronous events.
- Example events:
  - `VIDEO_UPLOADED`
  - `VIDEO_TRANSCODED`
  - `VIDEO_VIEWED`

### 8. CDN Decider & Caching
- Hot/viral videos → proactively transcode + cache.
- Long-tail/low-view → on-demand transcoding only.

### 9. Trending Service
- Consumes “view” events.
- Ranks trending content & pre-caches it.

### 10. Analytics & Metrics
- Frontend sends playback events.
- Backend logs API calls + system metrics.
- Both streamed to warehouse for analytics.

### 11. Live Streaming Notes
- Uses **UDP/WebRTC (TURN/STUN)** for low-latency.
- Transcoding chunk-by-chunk with small lag (~5 min typical).

### 12. Additional Topics
- **Convention over Configuration:** e.g., `user_id.jpg` naming.
- **Security:** DRM, watermarking, encryption at rest.
- **Adaptive UX:** tech should serve UX, not limit it.
- **Parallelism:** multi-core, high concurrency = essential.
- **Cost Control:** selective transcoding/CDN caching.

---

## 🏁 Key Takeaways

| Concept | Purpose | Core Benefit |
|----------|----------|---------------|
| **LSM Tree** | Append + flush model | High write throughput |
| **Memtable + SSTable** | Sorted in-memory + immutable files | Efficient writes |
| **Bloom Filter** | Probabilistic existence check | Skip unnecessary disk reads |
| **Commit Log** | Crash recovery | Durability |
| **Apache Airflow** | Workflow orchestration | High parallelism |
| **FFmpeg** | Video transcoding | Efficient, reliable |
| **Kafka** | Event backbone | Scalability |
| **CDN Decider** | Smart caching | Cost efficiency |
| **Adaptive Bitrate** | Dynamic resolution | Smooth UX |
| **Chunk Uploads** | Multipart transfer | Resilient, fast uploads |

---

*End of High Throughput Systems II Summary*
