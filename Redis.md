# Redis — System Design Learning Notes

## 🧠 Overview

**Redis (Remote Dictionary Server)** is an **in-memory key–value store** known for its **extremely low latency** and **rich data structures**.  
It’s commonly used as a **cache**, **session store**, **message broker**, and **real-time analytics engine** in large-scale distributed systems.

---

## ⚙️ Core Features

- **In-memory data store** → microsecond response times  
- **Persistence options** → RDB snapshots or AOF (Append Only File)  
- **Advanced data types** → Strings, Hashes, Lists, Sets, Sorted Sets, Streams, HyperLogLog, Bitmaps  
- **Pub/Sub messaging system**  
- **Built-in replication and clustering**  
- **Atomic operations** and **Lua scripting support**

---

## 🧩 Common Use Cases

| Use Case | Description | Redis Data Type / Feature |
|-----------|--------------|----------------------------|
| **Caching** | Store frequently accessed data to reduce DB load | Strings / Hashes |
| **Session Store** | Maintain user sessions in distributed systems | Hashes |
| **Rate Limiting** | Control user or IP request frequency | Counters (INCR / EXPIRE) |
| **Task Queue** | Manage background jobs or events | Lists / Streams |
| **Pub/Sub Messaging** | Real-time notifications and inter-service communication | Pub/Sub |
| **Leaderboard / Rankings** | Store top-N players, likes, etc. | Sorted Sets |
| **Real-time Analytics** | Count events, track stats, metrics | Counters / Streams |
| **Geospatial Queries** | Find nearest users, delivery points | GEO commands |

---

## 🚦 When to Use Redis

Use Redis when:
- You need **fast reads/writes** (microseconds).
- Data can be **recomputed or refetched** if lost.
- You want **to offload load from your database**.
- You need **distributed coordination** (locks, counters, queues).

Avoid using Redis alone when:
- Data **must never be lost**.
- Dataset size **exceeds available memory**.
- You need **complex joins or queries** — use a database instead.

---

## 🧱 Architecture Layer Integration

```text
Client/App → API Server → Redis Cache → Primary Database
```

### Example Workflow

1. App requests data (e.g., user profile).
2. API checks Redis cache.
3. If **cache hit** → return from Redis.
4. If **cache miss** → fetch from DB, store in Redis with TTL.

---

## 🪄 Setup & Implementation

### 1. Run Redis

#### Local

```bash
brew install redis      # macOS
redis-server
```

#### Docker

```bash
docker run --name redis -p 6379:6379 -d redis
```

---

### 2. Connect in Node.js

```bash
npm install redis
```

```javascript
import { createClient } from 'redis';

const client = createClient();
client.on('error', err => console.error('Redis Client Error:', err));
await client.connect();
```

---

### 3. Basic Commands

```javascript
// Store with TTL (in seconds)
await client.set('user:123', JSON.stringify({ name: 'Yogesh', age: 37 }), { EX: 60 });

// Retrieve
const data = await client.get('user:123');
console.log(JSON.parse(data));

// Delete
await client.del('user:123');
```

---

## ⚡️ Real-world Patterns

### 🧠 1. Database Query Caching

```javascript
async function getUser(id) {
  const key = `user:${id}`;
  const cached = await client.get(key);
  
  if (cached) return JSON.parse(cached);
  
  const user = await db.query('SELECT * FROM users WHERE id = $1', [id]);
  await client.set(key, JSON.stringify(user), { EX: 300 }); // 5 min TTL
  return user;
}
```

---

### 🚦 2. API Rate Limiting

```javascript
async function isAllowed(userId) {
  const key = `rate:${userId}`;
  const limit = 100; // requests per minute
  const ttl = 60;    // seconds
  
  const count = await client.incr(key);
  if (count === 1) await client.expire(key, ttl);
  
  return count <= limit;
}
```

* * *

---

### 💬 3. Pub/Sub Example

**Publisher:**

```javascript
await client.publish('news', 'Hello Subscribers!');
```

**Subscriber:**

```javascript
await client.subscribe('news', (message) => {
  console.log('Received:', message);
});
```

---

### 🧾 4. Leaderboard

```javascript
await client.zAdd('game:leaderboard', [
  { score: 120, value: 'Alice' },
  { score: 90, value: 'Bob' },
]);

const topPlayers = await client.zRange('game:leaderboard', 0, -1, { REV: true });
console.log(topPlayers);
```

---



---

## 🧮 Redis Persistence Modes

* * *

### 🧾 4. Leaderboard

# 

---

## 🧮 Redis Persistence Modes

| Mode | Description | Use Case |
| --- | --- | --- |
| RDB (Snapshotting) | Saves DB state periodically | Fast recovery, good for backups |
| AOF (Append Only File) | Logs each write operation | Better durability |
| Hybrid (RDB + AOF) | Default in Redis ≥ 7 | Best of both worlds |

* * *

## 🧠 Key Design Best Practices

# 

1.  **Use TTLs for cache entries** → `EX` or `PEX` to prevent memory bloat.
    
2.  **Namespace your keys** → e.g., `user:123:profile`.
    
3.  **Avoid “hot keys”** → shard data or apply per-request caching.
    
4.  **Enable persistence** if needed → AOF for durability.
    
5.  **Use connection pooling** in high-concurrency apps.
    
6.  **Monitor performance** → `INFO`, `SLOWLOG`, `MONITOR` commands.
    
7.  **Eviction policy tuning:**
    
    *   `volatile-lru`, `allkeys-lru`, `volatile-ttl`, `noeviction`.
        


