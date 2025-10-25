# 🧠 Caching Layers in a Modern System

Caching is one of the most powerful and misunderstood performance tools in system design.  
This guide summarizes **all caching layers** from client → backend, how to implement them, and when each is useful.

---

## 🧩 Overview Diagram

```text
Client/App Cache
        ↓
CDN / Edge Cache
        ↓
API Gateway Cache
        ↓
Load Balancer / Reverse Proxy Cache
        ↓
API Server (In-Memory)
        ↓
API Server (On-Disk)
        ↓
Centralized Cache (Redis/Memcached)
        ↓
Materialized Views (DB-side)
        ↓
Primary Database
```

---

## 1. 🧍 Client / App Cache

**Purpose:**  
Store recent API responses or UI data locally so the user doesn’t hit backend servers repeatedly.

**Common in:** Almost every web or mobile app.

**Implementation:**

- **Web:** Browser cache headers (`Cache-Control`, `ETag`), `localStorage`, `IndexedDB`.
- **Mobile:** SQLite / Room DB for offline persistence.

**Example (React / JS):**

```javascript
const cached = localStorage.getItem(`user:${id}`);
if (cached) return JSON.parse(cached);

const res = await fetch(`/api/user/${id}`);
const data = await res.json();
localStorage.setItem(`user:${id}`, JSON.stringify(data));
```

---

## 2. 🌐 CDN / Edge Cache

**Purpose:**  
Serve static and semi-static assets from global edge locations to minimize latency.

**Common in:** All production web apps.

**Implementation:**

- Use providers like **Cloudflare**, **Akamai**, **AWS CloudFront**, or **Fastly**.
- Configure caching headers (`Cache-Control`, `Surrogate-Control`).
    

**Example:**

```http
Cache-Control: public, max-age=86400
```

**Tip:** Purge cached assets on deployment to avoid stale JS/CSS.

---

## 3. 🚪 API Gateway Cache

**Purpose:**  
Cache full API responses at the gateway layer — before requests even hit backend servers.

**Common in:** Microservice or cloud-native architectures.

**Implementation:**

- **AWS API Gateway:** Enable caching per endpoint (TTL configurable).
- **Kong / NGINX Gateway:** Use built-in caching plugin.

**Benefits:**

- Reduces backend load.
- Handles authentication, rate limiting, versioning, and response caching centrally.

---

## 4. ⚖️ Load Balancer / Reverse Proxy Cache

**Purpose:**  
Distribute traffic across multiple backend servers and optionally cache responses at L7.

**Common in:** Medium → large systems.

**Implementation:**

- **Nginx / HAProxy / Envoy** support response caching.
**Example (Nginx):**

```nginx
proxy_cache_path /var/cache/nginx keys_zone=my_cache:10m;

server {
  location /api/ {
    proxy_cache my_cache;
    proxy_pass http://api_upstream;
    proxy_cache_valid 200 30s;
  }
}
```

**Benefits:**

- Prevents identical requests from hitting the app layer.
- Handles TLS termination and load distribution.

---

## 5. 💾 API Server (In-Memory Cache)

**Purpose:**  
Avoid recomputing or re-fetching hot data within the same API instance.

**Common in:** Most apps with hot endpoints.

**Implementation:**

- Use in-process caches like **LRU Cache**, **Guava**, or **functools.lru_cache**.
    

**Example (Node.js):**

```javascript
import LRU from "lru-cache";
const cache = new LRU({ max: 1000 });

async function getUser(id) {
  if (cache.has(id)) return cache.get(id);
  const user = await db.users.findById(id);
  cache.set(id, user);
  return user;
}
```

**Trade-off:** Each server instance has its own cache → can become stale if other instances update data.

---

## 6. 🗂️ API Server (On-Disk Cache)

**Purpose:**  
Pre-store large, semi-static responses on local disk instead of recomputing or fetching from DB.

**Common in:** Heavy content or analytics APIs.

**Implementation:**

```javascript
import fs from "fs/promises";

async function getBlog(id) {
  const path = `/tmp/blog_${id}.json`;
  try {
    return JSON.parse(await fs.readFile(path, "utf8"));
  } catch {
    const blog = await db.blogs.findById(id);
    await fs.writeFile(path, JSON.stringify(blog));
    return blog;
  }
}
```

**Pros:**

- Larger capacity than in-memory.
- SSD seek is often faster than a network round-trip.

**Cons:**

- Local-only (each instance has its own copy).
- Risk of stale data.

---

## 7. 🧱 Centralized Cache (Redis / Memcached)

**Purpose:**  
Reduce load on DB by caching hot data across all API instances.

**Common in:** Any medium-to-large-scale app.

**Implementation (Redis):**

```javascript
import Redis from "ioredis";
const redis = new Redis();

async function getPost(id) {
  const key = `post:${id}`;
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);
  
  const post = await db.posts.findById(id);
  await redis.set(key, JSON.stringify(post), "EX", 60);
  return post;
}
```

**Patterns:**

- Cache-aside (read-through)
- Write-through / Write-back
- TTL + versioned keys for invalidation

**Benefits:**

- Shared across all app servers
- Extremely low latency (sub-ms lookups)

---

## 8. 🧮 Materialized Views (DB-side)

**Purpose:**  
Precompute expensive queries (joins, aggregates) and store results in a table-like view.

**Common in:** Reporting, analytics, dashboards.

**Implementation (Postgres):**

```sql
CREATE MATERIALIZED VIEW active_users AS
SELECT id, name FROM users 
WHERE last_login > now() - interval '7 days';

-- Refresh periodically
REFRESH MATERIALIZED VIEW active_users;
```

**Benefits:**

- Dramatically faster reads for heavy analytical queries.
- Can be refreshed on schedule or event-based.

---

## 9. 🏛️ Primary Database

**Purpose:**  
Single source of truth.  
All other caches eventually derive from here.

**Implementation:** SQL or NoSQL depending on data model.

---

## 🪜 Which Layers Are Typically Present?

| Scale | Typical Layers | Notes |
| --- | --- | --- |
| MVP / Small App | Client + CDN + DB | Simple & cost-effective |
| Medium Scale | + Redis + In-Memory Cache | Handle 10–100× load |
| Enterprise Scale | + API Gateway + On-Disk + Materialized Views | Handle millions of requests per second |

---

## ⚙️ How to Add Layers Incrementally

1. **Start simple:** Client cache → CDN → DB.
2. **Add Redis** when DB reads dominate.
3. **Add in-memory LRU** for ultra-hot keys.
4. **Add gateway caching** when microservices multiply.
5. **Add materialized views** for analytics or reporting workloads.

---

## 🔍 Comparison Summary

| Layer | Optimizes | Cache Scope | Typical Tool |
| --- | --- | --- | --- |
| Client/App | Network latency | Per-user | Browser, SQLite |
| CDN/Edge | Global latency | Global edge | Cloudflare, Akamai |
| API Gateway | Multi-service latency | Per-endpoint | AWS API Gateway, Kong |
| Load Balancer/Proxy | Traffic distribution + cache | Between client & servers | Nginx, HAProxy |
| API Server (In-Memory) | Hot keys | Per-instance | LRU, Guava |
| API Server (On-Disk) | Precomputed data | Per-instance | Local SSD |
| Centralized Cache | DB offload | Cluster-shared | Redis, Memcached |
| Materialized Views | Heavy joins | DB-layer | Postgres, BigQuery |