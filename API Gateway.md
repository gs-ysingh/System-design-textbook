# 🧭 API Gateway, Serverless & Lambda — Complete Learning Notes

## 🔹 1. Cache-Control Header

**Purpose:** Tells browsers/CDNs how to cache responses.

| Directive | Meaning | Example |
| --- | --- | --- |
| no-store | Never cache the response. | Sensitive data (auth, banking). |
| no-cache | Cache locally but always revalidate. | Dynamic API responses. |
| public | Anyone (browser, CDN) can cache. | Static content, images. |
| private | Only the browser can cache. | User-specific content. |
| max-age=<sec> | Time (in sec) resource stays fresh. | max-age=3600 → 1 hour. |
| s-maxage=<sec> | Like max-age but for shared caches (CDNs). | s-maxage=300. |
| must-revalidate | Once expired, must revalidate. | Ensures freshness. |
| immutable | Resource will not change. | Versioned JS/CSS files. |
| stale-while-revalidate=<sec> | Serve stale while fetching fresh. | stale-while-revalidate=30. |

**Examples:**

```http
Cache-Control: public, max-age=31536000, immutable
Cache-Control: no-cache
Cache-Control: private, no-store
```

* * *

## 🔹 2. ETags (Entity Tags)

**Purpose:** Validate cached content efficiently.

### How It Works

1. Server sends ETag:
   
   `ETag: "abc123"`
   
2. Client stores ETag and sends:
   
   `If-None-Match: "abc123"`
   
3. Server compares:
   
   * Same → `304 Not Modified`
   * Different → new content + new ETag
        

### Types

* **Strong ETag:** `"abc123"` → byte-for-byte identical.
* **Weak ETag:** `W/"abc123"` → semantically identical.
    

**Benefit:** Saves bandwidth, speeds up validation.

* * *

## 🔹 3. What Is API Gateway

An **entry point** for all client traffic into your backend or microservices.

### Responsibilities

* Routing (`/orders` → Order Service)
* Authentication & Authorization (JWT, API Keys, OAuth2)
* Rate limiting & throttling
* Request/response transformation
* Logging & monitoring
* Security (CORS, TLS, WAF)
    

* * *

## 🔹 4. How API Gateway Works

* **Configuration file/UI** defines routes, limits, caching, etc.
* **Runtime service** enforces these rules — runs on AWS infra (managed) or your own servers (self-hosted gateways like Kong, NGINX).
    

* * *

## 🔹 5. AWS API Gateway Setup (Step-by-Step)

1. **Create API**
   
   * HTTP API (lightweight) or REST API (full-featured).

2. **Define Routes**
   
   * `/users`, `/orders` with `GET`, `POST`, etc.

3. **Integrate Backends**
   
   * Lambda function
   * HTTP backend (ECS, EC2, external API)
   * Mock (for testing)

4. **Configure Security**
   
   * CORS
   * Auth (Cognito, JWT, API keys)

5. **Enable Throttling / Rate Limits**
   
   * Protect backend (e.g., 100 req/sec).

6. **Enable Caching**
   
   * Per-method caching with TTL.

7. **Logging & Monitoring**
   
   * CloudWatch + X-Ray tracing.

8. **Deploy**
   
   * Create **stage** (`dev`, `prod`), get URL.

9. **Custom Domain (Optional)**
   
   * `api.mycompany.com` via Route 53 + ACM.
        

* * *

## 🔹 6. Self-Hosted Gateway Example

### NGINX

```nginx
server {
  listen 443 ssl;
  server_name api.example.com;
  
  location /orders/ {
    proxy_pass http://order-service/;
    proxy_set_header Host $host;
    proxy_cache my_cache;
    proxy_cache_valid 200 1m;
  }
}
```

### Kong

```yaml
services:
- name: order-service
  url: http://order-service:8080
  routes:
  - paths:
    - /orders

plugins:
- name: rate-limiting
  config:
    minute: 100
    policy: local
```

* * *

## 🔹 7. What Is Serverless

Serverless means **you don’t manage servers** — the cloud provider runs and scales them for you.

### Benefits

* Auto-scaling to zero
* Pay per request
* No infra management
* Event-driven execution
* Fast iteration

### When to Avoid

* Long-running or persistent workloads
* High constant traffic
* Complex networking needs
    

* * *

## 🔹 8. AWS Lambda

**AWS Lambda** = Serverless compute that runs your code on demand.

### Key Points

* Supports Node.js, Python, Java, Go, C# …
* Max duration = 15 min
* Stateless per invocation
* Triggered by events (API Gateway, S3, DynamoDB, CloudWatch, etc.)
    

**Example:**

```javascript
exports.handler = async (event) => {
  console.log("Event:", event);
  return {
    statusCode: 200,
    body: JSON.stringify({ message: "Hello from Lambda!" }),
  };
};
```

* * *

## 🔹 9. Lambda in Microservice Architecture

### Option 1 — Direct Routing

`Client → API Gateway → Order Service (ECS/EC2) Client → API Gateway → Product Service`

### Option 2 — Lambda as Adapter

`Client → API Gateway → Lambda (Orders) → Order Service Client → API Gateway → Lambda (Products) → Product Service`

Each Lambda acts as a lightweight controller:

```javascript
exports.handler = async (event) => {
  const orderId = event.pathParameters.id;
  const resp = await fetch(`http://orders-service.internal/api/orders/${orderId}`);
  const data = await resp.json();
  return { statusCode: 200, body: JSON.stringify(data) };
};
```

### Hybrid Pattern (common)

* API Gateway → Lambda → microservice for **external traffic**
* Direct service-to-service calls internally (no Lambda)
    

* * *

## 🔹 10. Summary of Choices

| Layer | Recommended Option | Reason |
| --- | --- | --- |
| API Gateway | AWS API Gateway (managed) | Easy, scales, integrates with Lambda |
| Compute | Lambda (serverless) or ECS/K8s | Lambda for events; containers for long-running services |
| Cache | Gateway caching + CloudFront + ETags | Reduces load and latency |
| Auth | Cognito / JWT / custom authorizer | Secure endpoints |
| Monitoring | CloudWatch + X-Ray | Full visibility |

* * *

✅ **Core takeaway:**

> Use API Gateway + Lambda when you want a fully managed, event-driven entry layer.  
> Use direct routing to microservices when you need lower latency or already run containerized services.  
> Combine both for hybrid architectures — serverless at the edge, microservices in the core.