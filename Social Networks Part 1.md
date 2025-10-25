# 🧠 Social Networks System Design (Instagram Case Study)

## 1. 🌍 Introduction & Core Principles
- Design progressively — start small, scale gradually.
- Avoid premature optimization.
- Assume every component can fail — design for HA/DR.
- Dig deep beyond diagrams — know implementation details.

---

## 2. 🧱 Instagram’s Day-Zero Tech Stack (2010)
| Layer | Technology | Purpose |
|--------|-------------|----------|
| Web Server | Nginx | Reverse proxy & load balancer |
| Backend | Django (Python) | Web framework |
| Async Jobs | Celery + RabbitMQ | Background processing |
| DB | PostgreSQL | Primary relational database |
| Cache | Memcached / Redis | Faster reads |
| Object Store | Cassandra | Photo storage |

> 3 engineers → 14M users in 1 year.

**Key Lessons:**
- Asynchronous-first mindset.
- Eventual consistency is fine.
- Stateless servers = easy scale.
- Use managed services today.

---

## 3. 🧩 Database Schema & Design Choices
**Users:** id (PK), email (unique), username (unique), first_name, last_name  
**Photos:** id, user_id, url, caption, created_at, (user_id, created_at) index  
**Tags:** photo_id, tag → later optimized as tag_id  
**Reactions:** photo_id, user_id, reaction_type (int)  
**Relations:** from_user_id, to_user_id, is_deleted (soft delete)

---

## 4. 🖼️ Image Upload Service

### Two Approaches
1. **Two-Step:** Upload image → get URL → create post with URL.  
   ✅ Lightweight servers.
2. **One-Step:** Upload base64 image in `/post`.  
   ⚠️ Doubles data transfer, memory-heavy.

### ✅ Best Practice — Signed URL Upload
- API returns signed S3 URL (5 min expiry).
- Client uploads directly to S3.
- Server stores URL only.

**Frontend Constraints:**
- Validate file type, size, format.
- Enforce MIME policies (S3 rejects invalid uploads).

---

## 5. 🔁 Updating Profile Photos
**Solutions:**
1. **Static Path:** `/profile/<user_id>.jpg` — overwrite old photo.
2. **Soft Link:** `/profile/<user_id>.jpg` → points to latest upload.

⚠️ Manage CDN cache invalidation (TTL/manual purge).

---

## 6. 🌐 CDN Integration
- Use Akamai / CloudFront / Cloudflare before S3.
- Caches static content regionally.
- On cache miss → fallback to S3 → cache again.

---

## 7. 🧮 Image Optimization
**Why:** Save data & improve UX on slow connections.

### Options
1. **Preprocessing:** Background resize into multiple resolutions.
2. **Dynamic Resizing:** Query param `?w=64` triggers Lambda-based resizing. Cached.
3. **Hybrid:** Precompute common sizes + dynamic fallback.

**Frontend logic:** Based on `navigator.connection`, screen DPI, and battery.

---

## 8. 🏷️ Hashtag Extraction Service
Flow:
1. Post published → Event sent to queue.
2. Consumer extracts hashtags via regex.
3. Stores photo_id, tag_id.

**Scaling Issue:** 2M posts/day × 8 tags/post = 16M writes/day.

**Optimizations:**
- Separate `tags(photo_id, word_id)` & `words(id, word)`.
- Batch inserts (batch get + insert).
- Precompute tag counts.

---

## 9. ⚙️ Event-Driven Architecture
**Problem:** Multiple services triggered on one event.

**Solution:** Event Bus (Kafka / Kinesis)
- One publisher → many subscribers.
- Retention: ~14 days.
- Parallel reads via partitions/shards.

✅ Decoupled, horizontally scalable.

---

## 10. 💬 Tag Popularity & Trending Feed
**Tag Store (DynamoDB/Cassandra):**
- Key: tag_id → `{count, popular_photos[]}`
- `count++` for every post.
- Popularity service updates top posts list.

**DynamoDB schema:** Partition key = tag, Sort key = created_at.

---

## 11. 🪣 Scaling & Cost Optimization
**Issue:** `count++` for each post = costly.

**Fixes:**
1. Micro-batch & flush in bulk.
2. Allow small lag (eventual consistency).
3. Use stream processors (Flink/Spark/Kinesis).

---

## 12. ❤️ Reaction Modeling
- Boolean → Integer `reaction_type`
- Dynamic reactions (Slack-like):
  - Store each as row (user_id, post_id, emoji_id)
  - Or use array in NoSQL (atomic add/remove)

---

## 13. 🧠 Key Takeaways
| Theme | Learning |
|--------|-----------|
| Design | Start simple, evolve |
| Backend | Async + stateless |
| DB | Batch writes, minimal indices |
| Storage | Signed URLs + CDN |
| Eventing | Kafka/Kinesis backbone |
| Data | Precompute + cache |
| Cost | Aggregate & relax constraints |
| Extensibility | Plan schema for future |

---
