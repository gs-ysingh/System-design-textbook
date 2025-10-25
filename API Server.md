# API Server and Caching Layers

## 1. What is an API Server?
An **API Server** is the backend component that:
- Receives client requests (from web, mobile, or other services).
- Applies **business logic** and validations.
- Interacts with databases, caches, and other systems.
- Returns structured responses (usually JSON or GraphQL).

### Common Responsibilities:
- Authentication & Authorization
- Request Validation
- Business Logic Execution
- Data Fetching & Transformation
- Response Caching & Logging
- Error Handling & Observability

---

## 2. API Server Storage Layers

### 🔹 In-Memory Storage
- Data is stored directly in the server’s **RAM**.
- Access is **ultra-fast** but **non-persistent**.
- Used for **short-lived**, frequently accessed data.

**Examples:**
- Recent API responses or hot data (LRU cache).
- Rate limiting counters (`user:123 → 5 requests/sec`).
- Authentication session tokens.

**Pros:**
- Fastest possible access (no disk or network call).
- Reduces database load.

**Cons:**
- Data lost on restart or crash.
- Not shared between multiple server instances.

---

### 🔹 On-Disk Storage
- Data stored on **local SSD/HDD** of the API server.
- Survives process restarts (if on same machine).
- Slower than in-memory but faster than remote DBs.

**Examples:**
- Precomputed responses (e.g., `product-1.json`).
- Cached blog posts or search results.
- Large data files too big for memory.

**Pros:**
- Persistent across restarts (on same machine).
- Reduces repeated DB/API calls.

**Cons:**
- Not shared between multiple servers.
- Lost in containerized or ephemeral environments.
- Can consume disk space quickly.

---

## 3. Example: Disk Cache in Node.js

```js
import express from "express";
import fs from "fs";
import path from "path";
import fetch from "node-fetch";

const app = express();
const CACHE_DIR = path.join(process.cwd(), "cache");
if (!fs.existsSync(CACHE_DIR)) fs.mkdirSync(CACHE_DIR);

function getFromDiskCache(key) {
  const filePath = path.join(CACHE_DIR, `${key}.json`);
  if (fs.existsSync(filePath)) {
    const data = fs.readFileSync(filePath, "utf-8");
    return JSON.parse(data);
  }
  return null;
}

function saveToDiskCache(key, data) {
  const filePath = path.join(CACHE_DIR, `${key}.json`);
  fs.writeFileSync(filePath, JSON.stringify(data));
}

async function fetchProductFromDB(id) {
  console.log("Fetching from DB...");
  const response = await fetch(`https://fakestoreapi.com/products/${id}`);
  return response.json();
}

app.get("/product/:id", async (req, res) => {
  const id = req.params.id;
  const cacheKey = `product-${id}`;
  let product = getFromDiskCache(cacheKey);

  if (product) {
    console.log("✅ Served from disk cache");
    return res.json(product);
  }

  product = await fetchProductFromDB(id);
  saveToDiskCache(cacheKey, product);

  console.log("🗄️ Saved to disk cache");
  res.json(product);
});

app.listen(3000, () => console.log("API server running on port 3000"));
```

**Behavior:**

1. Checks if cached file exists → returns it.
2. If not, fetches data → stores on disk → returns it next time.
    

---

## 4. What Happens on Server Failure

### 🧠 In-Memory Cache

- Data **wiped immediately** when process restarts.
- Cache reinitializes empty (cold start).

### 💾 Disk Cache

- Data **remains** if server restarts on same machine.
- Lost if deployed in containers (ephemeral storage).
- Not shared among multiple instances.

### ☁️ Shared Cache (Recommended)

Use a **distributed and persistent** cache:

- **Redis / Memcached** → Fast, shared, survives restarts.
- **Database Materialized Views** → Precomputed results.
- **Object Storage (S3, GCS, Azure Blob)** → Persistent file cache.
- **CDN (CloudFront, Akamai, Fastly)** → Persistent HTTP response cache.

---

## 5. Real-World Caching Strategy

| Layer | Type | Scope | Use Case | Persistence |
| --- | --- | --- | --- | --- |
| Client Cache | In-memory / LocalStorage | Per user | Avoid hitting API for repeated queries | ✅ |
| API Server In-Memory | RAM (LRU) | Per instance | Hot data, rate limits | ❌ |
| API Server On-Disk | Local SSD | Per instance | Medium-term cache, precomputed responses | ⚠️ |
| Central Cache (Redis) | Distributed | Shared | Fast shared cache | ✅ |
| CDN / Reverse Proxy | Edge | Global | Full HTTP responses | ✅ |

---

## 6. Best Practice Architecture

```text
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────┐
│ API Gateway or Load Balancer    │
└───────────────┬─────────────────┘
                │
                ▼
        ┌───────────────┐
        │  API Server   │
        └───────┬───────┘
                │
        ┌───────┼───────────────┐
        │       │               │
        ▼       ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐
│ In-memory   │ │ Disk cache  │ │ Redis cache         │
│ cache       │ │ (semi-      │ │ (shared,           │
│ (fast,      │ │ persistent) │ │ distributed)       │
│ local)      │ │             │ │                    │
└─────────────┘ └─────────────┘ └─────────────────────┘
                                          │
                                          ▼
                        ┌─────────────────────────────────────┐
                        │ Database / Object Store /           │
                        │ External APIs                       │
                        └─────────────────────────────────────┘
```

- Use **multi-layered caching** (memory → Redis → DB).
- Keep API server **stateless** if deployed in scalable environments.
- For persistence beyond restarts, use **Redis or external storage**.