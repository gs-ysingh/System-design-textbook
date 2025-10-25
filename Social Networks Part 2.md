# Social Networks II — Learnings Summary

## 🧮 1. Data Accuracy and Approximation in Large Systems

- **Batching Writes** reduces load on systems.  
  - Instead of incrementing (`count++`) for every event, batch updates (`count += k`) every few seconds or minutes.  
  - This sacrifices real-time accuracy slightly but saves massive IOPS and database costs.
- **Overcounting vs Undercounting:**  
  - If you update DB before committing to Kafka, failure may cause *overcounting*.  
  - If you batch and then commit, you risk *undercounting*, which is easier to retry.
- **Approximate Counting:**  
  - Use algorithms like **HyperLogLog** for space-efficient counting at scale.  
  - Ideal when precision isn’t crucial (e.g., celebrity likes/views).
- **User Bucketization:**  
  - Small creators → require accuracy  
  - Large creators → can tolerate approximations or sampling.  
  - Example: Instagram only sends notifications for smaller accounts; celebrities are bucketized.

---

## 🧠 2. Profile Picture Design (Gravatar-style System)

### Problem:
Design a system where one static URL always returns the latest user photo.

### Key Ideas:
- Each user has a **static URL**, e.g.  
  `https://gravitar.com/user123` → always returns the latest image.
- **Two storage approaches:**
  1. **Overwrite same S3 path:** `/images/profile/user123.jpg`
  2. **Symbolic link** or logical pointer to current image.
- **Challenges:**  
  - Users can upload different formats (`.jpg`, `.png`).  
  - Transparent backgrounds in PNG must be handled carefully.

### Data Model:
| Table | Columns |
|--------|----------|
| `photos` | `photo_id`, `user_id`, `photo_url`, `uploaded_at`, `is_deleted` |
| `users` | `user_id`, `email`, `active_photo_id (FK → photos)` |

- The *active* photo is determined by `active_photo_id` in `users`, not a boolean flag.

### Rendering Flow:
1. Browser requests → `https://g.com/arpit`
2. Server queries DB for active `photo_url`.
3. Reads image bytes from S3.
4. Responds with raw binary or Base64 + correct headers (`Content-Type: image/png`).

### Optimization:
- **CDN Layer (e.g., CloudFront):**  
  Cache image responses.  
  When user updates photo → **purge CDN cache** for that key.
- CDN can cache any HTTP response, not just images — even JSON for static APIs.

---

## 🧩 3. Tagging People in Photos

- Store **coordinates** of each tag as *relative ratios* instead of absolute pixels.
  - E.g., `x = 0.75, y = 0.60` instead of pixels.
  - Makes it resolution and format agnostic.
- For bounding boxes (like face rectangles):  
  - Store either `(x1, y1, x2, y2)` or `(x, y, width, height)`, normalized.

---

## 🔗 4. User Relationships (Follow/Following Graph)

### Twitter’s Approach (Based on FlockDB):

- Store **edges**: each follow = one edge.
  ```
  source_id | destination_id | state | position | meta
  ```
- **Indexes:**
  - PK: `(source_id, state, position)`
  - Unique index: `(source_id, destination_id)`
- Partition data **by source_id** to keep reads localized.

### Dual-Edge Optimization:
- For every “A follows B”, create **two entries**:
  - `A follows B`
  - `B is_followed_by A`
- This avoids multi-shard lookups.
- Query examples:
  - Followers → `WHERE source_id = A AND relation = 'is_followed_by'`
  - Following → `WHERE source_id = A AND relation = 'follows'`
- Pagination is done using **cursor on position**, not `LIMIT OFFSET`.

---

## 💬 5. “Newly Unread Messages” Counter

### Requirement:
Show how many *new* message threads (from unique people) a user has received, **not** total unread messages.

### Core Design:
- Store precomputed counts in **Redis**:
  ```
  key = user_id
  value = sorted_set of sender_ids with timestamps
  ```
- On message receive:
  - If sender not already in set → add (`ZADD` with timestamp)
- On opening inbox:
  - Purge sorted set (`DEL` or `ZREM`)
- On “/me” API call:
  - Return `ZCARD(sorted_set)` as the counter value.

### Optional:
- Store `(sender_id, name, timestamp)` to show tooltips:
  - “Sarthak, Arpit, and 2 others messaged you.”

### Real-time Behavior:
- WebSocket can notify in-app users immediately.
- Offline users rely on Redis when they reconnect.
- Redis TTL or cache invalidation keeps data fresh.

---

## 📊 6. Design Principles Learned

1. **Precomputation for Speed:**  
   Compute counts/states beforehand and store them in cache.
2. **Denormalize When Needed:**  
   Read-heavy systems should duplicate data for performance.
3. **Partition by Access Pattern:**  
   Always shard by the key most used in queries (e.g., `source_id`).
4. **Approximation Is Acceptable:**  
   Trade a little accuracy for scalability.
5. **Use CDN for Static Content:**  
   Push expensive operations away from backend APIs.
6. **Cache Strategically:**  
   Redis for hot data; use TTL and purging for freshness.
7. **Write Amplification Control:**  
   Batch updates and merge events before committing.
8. **Model Active Relationships Efficiently:**  
   Store both “follows” and “is_followed_by” edges for quick access.
9. **Normalized Coordinates:**  
   Always normalize when UI scaling is possible.
10. **System Extensibility:**  
    Design DB schemas that can evolve with minimal rework (e.g., “relations” instead of “follows”).
