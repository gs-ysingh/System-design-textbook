# Scaling Reads (Detailed System Design Explanation)

## Overview

Scaling reads deals with handling the exponential growth of read traffic as user volume increases. In most large systems, read operations vastly outnumber writes — often by 10:1, 100:1, or more.

**Examples:**
- One tweet → thousands of reads
- One Instagram post → millions of views
- One YouTube upload → billions of video plays

As the read-to-write ratio widens, databases struggle under load due to hardware and I/O limits. Read scaling ensures systems can serve high-volume reads efficiently while maintaining acceptable latency and consistency.

## 1. The Problem

### 1.1 Read-Write Imbalance

For every new piece of content (a write), thousands of users request to view it (reads).

- Instagram feed load → 100+ DB reads for one screen
- YouTube metadata → billions of reads vs. millions of writes
- The challenge: reads grow exponentially while writes grow linearly.

### 1.2 Hardware Boundaries

Even optimized queries hit physical limits:

- CPU cores have finite throughput
- Memory can only hold so much working data
- Disk I/O (even SSDs) caps read speed

Once the database reaches these limits, performance degrades regardless of query efficiency.

## 2. Approaches to Scale Reads

Read scaling follows a natural progression:

1. Optimize within the database
2. Scale the database horizontally
3. Add caching layers

Each stage targets a different bottleneck and adds operational complexity.

## 3. Optimize Within the Database

### 3.1 Indexing

Indexes accelerate queries by maintaining an ordered lookup table.

- Converts O(n) scans → O(log n) lookups.
- **B-tree indexes** for range queries
- **Hash indexes** for equality lookups
- **Full-text or spatial indexes** for specialized data

**Example:**

```sql
CREATE INDEX idx_users_email ON users(email);
```

Now lookups like:

```sql
SELECT * FROM users WHERE email = 'a@example.com';
```

use index scans instead of full table scans.

**Indexes should exist on:**

- Filter columns (`WHERE`)
- Join keys (`ON`)
- Sorting/grouping fields (`ORDER BY`, `GROUP BY`)

**Trade-off:** Slightly slower writes, but dramatically faster reads. Modern databases handle this overhead efficiently.

### 3.2 Denormalization

**Normalize for writes; denormalize for reads.**

Normalization reduces redundancy but increases join complexity.

**Example:**

**Normalized schema:**

- Users table
- Orders table
- Products table

Querying requires multiple joins:

```sql
SELECT u.name, o.date, p.name
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id;
```

**Denormalized approach:**

- Combine into an `order_summary` table duplicating user and product details.

Now reads are single-table queries:

```sql
SELECT user_name, order_date, product_name
FROM order_summary WHERE order_id = 12345;
```

**Trade-off:**

- Redundant data storage
- Extra complexity during updates
- Huge read performance gain

**Use when** reads are far more frequent than writes.

### 3.3 Materialized Views

Precompute and store expensive aggregations.

Instead of:

```sql
SELECT AVG(rating) FROM reviews WHERE product_id = 42;
```

Create:

```sql
CREATE MATERIALIZED VIEW product_ratings AS
SELECT product_id, AVG(rating) as avg_rating FROM reviews GROUP BY product_id;
```

The database serves precomputed results directly from the view — ideal for analytics and dashboards.

### 3.4 Hardware Upgrades (Vertical Scaling)

- Move from HDD → SSD
- Increase RAM to keep more data in memory
- Add CPU cores for more concurrency

Quick win for moderate scale but limited scalability.

## 4. Scale Database Horizontally

When one database instance cannot handle read traffic, distribute it across multiple servers.

### 4.1 Read Replicas (Leader-Follower Replication)

- **Leader (Primary):** Handles all writes
- **Followers (Replicas):** Handle read traffic
- **Replication:** Synchronous (consistent but slower) or Asynchronous (faster but may lag)

Clients route reads to replicas:

```sql
SELECT * FROM user_profiles WHERE id = 123 → Replica
INSERT INTO user_profiles ... → Primary
```

**Advantages:**

- Distributes read load
- Provides redundancy and failover

**Challenges:**

- Replication lag → users may not see their latest writes immediately
- Handling eventual consistency in replicas

**Typical threshold:** systems exceeding 50k–100k read QPS start using replicas.

### 4.2 Database Sharding

Sharding partitions data across multiple databases.

#### Horizontal (Range) Sharding

Split records by key range, e.g.:

- DB1 → users A–M
- DB2 → users N–Z

#### Functional Sharding

Split by domain:

- User data → user DB
- Product data → product DB

#### Geographic Sharding

Store data close to users:

- US traffic → US DB
- EU traffic → EU DB

**Pros:**

- Smaller dataset per shard → faster queries
- Parallel processing

**Cons:**

- High operational complexity
- Rebalancing and cross-shard joins are difficult

Usually adopted at massive scale.

## 5. Add External Caching Layers

Caching provides the largest read performance gain and reduces load on the database.

### 5.1 Application-Level Caching

In-memory caches like Redis or Memcached store pre-fetched or precomputed data.

**Flow:**

1. App checks cache for key
2. If found → return cached data (sub-millisecond)
3. If not → query DB → store in cache

**Example (pseudo-code):**

```javascript
if (cache.has(userId)) return cache.get(userId);
const user = db.query("SELECT * FROM users WHERE id = ?", userId);
cache.set(userId, user, TTL=300);
```

**Common invalidation strategies:**

- **TTL-based:** Automatically expire after fixed duration
- **Write-through:** Update cache during writes
- **Write-behind:** Asynchronously update cache
- **Tagged invalidation:** Group related cache entries
- **Versioned keys:** Include version in cache key (safe and atomic)

### 5.2 CDN and Edge Caching

CDNs (Akamai, Cloudflare, Fastly) store static and semi-dynamic content closer to users.

**Use for:**

- Product pages
- Public profiles
- Images, videos, API responses

**Advantages:**

- Global edge nodes → minimal latency (10–20ms)
- Offloads 80–90% of origin server load

**Limitations:**

- Not useful for personalized data
- Complex invalidation across regions

## 6. Common Operational Issues

### 6.1 Cache Stampede

When a popular cache entry expires, thousands of requests hit DB simultaneously.

**Solutions:**

- **Distributed locks:** One process rebuilds cache, others wait
- **Probabilistic early refresh:** Randomly refresh before TTL
- **Background refresh:** Scheduled rebuilds before expiration

### 6.2 Hot Key Problem

Millions of users request the same key (e.g., viral tweet).

**Solutions:**

- **Request coalescing:** Batch concurrent requests into one backend call
- **Key fan-out:** Store identical data under multiple keys (e.g., `feed:123:1`, `feed:123:2`)

### 6.3 Cache Invalidation (Consistency Problem)

When data changes, cached versions must be updated or invalidated.

#### Best practice: Cache key versioning

On write:

1. Increment version in DB
2. Write cache under new version key
3. Old cache naturally becomes obsolete

**Example:**

```text
event:123:v42  → before update
event:123:v43  → after update
```

**Advantages:**

- No race conditions
- No explicit invalidation needed
- Safe for concurrent updates

## 7. Real-World Examples

| System | Read Scaling Strategy |
|--------|----------------------|
| Bitly | Redis + CDN cache (short→long URL), no TTL |
| Ticketmaster | Cache event details; live seat availability from master DB |
| News Feed | Precompute feeds; cache timelines; paginate |
| YouTube | Cache video metadata + CDN for thumbnails |
| Instagram | Feed caching; image CDN; precomputed profiles |

## 8. When to Use / When Not To

✅ **Use when:**

- Application is read-dominant
- You expect high QPS (>50k–100k reads/sec)
- Most data changes infrequently
- You can tolerate eventual consistency

❌ **Avoid or delay when:**

- System is write-heavy (e.g., GPS updates, chat apps)
- User base < few thousand
- Data must be strongly consistent (e.g., financial transactions)
- Real-time collaborative systems (Google Docs)

## 9. Summary Table

| Stage | Technique | Goal | Trade-offs |
|-------|-----------|------|------------|
| 1 | Indexing | Fast lookups | Slower writes |
| 2 | Denormalization | Fewer joins | Data redundancy |
| 3 | Materialized Views | Precomputed results | Stale if not refreshed |
| 4 | Read Replicas | Load distribution | Possible lag |
| 5 | Sharding | Dataset partitioning | Complexity |
| 6 | App Caching | Fast reads | Stale data risk |
| 7 | CDN/Edge Cache | Global latency reduction | Complex invalidation |

## 10. Conclusion

Scaling reads is one of the most frequent system design challenges. The fundamental progression is:

> **Optimize → Replicate → Cache**

Start with database tuning (indexes, denormalization), then distribute load (replicas, sharding), and finally add caching (Redis, CDN). Every step trades simplicity for performance, so choose based on scale and consistency needs.
