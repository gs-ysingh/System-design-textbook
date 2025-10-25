# 🧠 Distributed Systems – Learnings Summary

## 1. Introduction
Distributed Systems are the backbone of modern scalable applications.  
They consist of **multiple components running on multiple machines** that work together as one coherent system.  
**Key challenge:** communication over unreliable networks (latency, partitions, clock sync).

### Why Distributed Systems Matter
- **Scalability:** Horizontal scaling (add more machines instead of upgrading one).
- **Reliability:** If one machine fails, others can continue.
- **Performance:** Parallel processing across nodes.
- **Drawback:** Increased latency and reduced observability.

---

## 2. Common Failures and Circuit Breakers

### Cascading Failures
- One service slowdown or crash can lead to others queuing and failing.
- Example: DB latency → API slow → queue backlog → service crash.

### Circuit Breaker Pattern
- Prevents repeated failing calls to downstream services.
- If dependent service is unhealthy → “break the circuit” → fail fast or fallback.
- Once circuit breaker is ON, requests are blocked for a configured duration, allowing system recovery.

**Example:**  
If Profile API fails → Communication API stops making requests → reduces pressure → system stabilizes.

---

## 3. Load Balancer (LB)

### Purpose
- Distributes client traffic across multiple backend servers.
- Prevents single point of failure and enables horizontal scalability.

### Architecture
```
Client → Load Balancer → Multiple Identical Servers
```

### Load Balancer Placement
- **External Load Balancer:** Internet-facing (entry point to system).
- **Internal Load Balancer:** Within a VPC (between microservices, databases, or caches).

### Types
1. **Software LB** – Nginx, HAProxy (application-level).
2. **Hardware LB** – Physical devices used in data centers.
3. **L4 vs L7 LB:**
   - L4 → Operates at transport layer (TCP/UDP packets).
   - L7 → Operates at application layer (HTTP headers, URLs, cookies).

---

## 4. Load Balancing Algorithms

| Algorithm | Description | Use Case |
|------------|--------------|----------|
| **Round Robin** | Sequentially distributes requests | Equal request sizes |
| **Weighted Round Robin** | Some servers get more requests | Different server capacities |
| **Least Connections** | Chooses server with fewest active connections | Varying request duration |
| **IP Hash / URL Hash** | Based on client IP or URL prefix | Sticky sessions / routing by service |
| **Consistent Hashing** | Key-based distribution | Scalable cache systems |

---

## 5. Implementing a Load Balancer

### Core Logic
1. **Accept TCP connection** from client.
2. **Choose backend** using algorithm.
3. **Create TCP connection** from LB to backend.
4. **Pipe data** (bi-directional copy).
5. **Handle concurrency** using threads or goroutines.

**Go Implementation Highlights:**
- `io.Copy` used for bidirectional TCP forwarding.
- Each request handled in its own goroutine.
- Supports dynamic backend addition and algorithm switching.

---

## 6. Scaling Load Balancers

- **Load balancers themselves can be load balanced!**
- Done using **DNS-based round-robin routing** (e.g., Route53):
  - `lb.company.com` → multiple IPs (LB1, LB2, LB3, …)
- Each LB handles millions of TCP connections.
- Avoids single point of failure.

---

## 7. Supporting Components

| Component | Function |
|------------|-----------|
| **Heartbeat Service** | Continuously checks backend health. |
| **Trigger System** | Publishes events (e.g., server down) via Kafka/Kinesis. |
| **Stats Engine** | Exports CPU, memory, latency metrics (e.g., Prometheus + Grafana). |

### Prometheus
- Pull-based time-series DB.
- Fetches metrics from servers via `/metrics` endpoint.
- Grafana visualizes these metrics.

---

## 8. Distributed Locks & Remote Locks

### Why Locks?
- Ensure **coordination** and **data consistency** across multiple distributed nodes.
- Used to prevent **simultaneous updates** or **duplicate processing**.

### Central Lock Manager
Acts like a mutex for distributed systems and must support:
- **Atomicity**
- **Isolation**
- **TTL** (to prevent deadlocks)

### Example Use Cases
- Booking systems (avoid double booking)
- Distributed cron jobs
- Kafka/SQS consumers (exactly-once message processing)
- MongoDB cross-shard transactions

---

## 9. Implementing Remote Locks (Using Redis)

### Redis Lock
Each lock represented by a key.

**SET key value NX PX ttl**
- NX → only if key doesn’t exist.
- PX → expire after TTL (ms).

Delete lock only if held by same client.

### Distributed Locks (Redlock Algorithm)
- Use **5 independent Redis nodes** (no replication).
- Client must acquire lock on **majority (3/5)** nodes.
- Prevents single-point failure and ensures consistency.

---

## 10. Other Examples

- **Aurora Read Load Balancer:** Internal LB managing read replicas.
- **API Gateway vs Load Balancer:**
  - LB handles identical backends.
  - API Gateway routes heterogeneous requests (`/payment` → PaymentSvc, `/profile` → ProfileSvc).
  - API Gateway adds caching, authentication, throttling, and routing.

---

## 11. Best Practices

- Assume every dependency can fail.
- Use circuit breakers and retries smartly.
- Instrument observability early (metrics, logs, traces).
- Remove single points of failure.
- Implement TTLs to prevent deadlocks.
- Design for graceful degradation (fallbacks, defaults).

---

## 12. Recommended Readings / Assignments

1. **Build:**
   - A mini load balancer (like the Go example).
   - Simulated distributed lock system.
2. **Read:**
   - *Chubby Lock Service for Loosely Coupled Systems* (Google Paper)
   - *Martin Kleppmann – Distributed Locks with Redis*
   - *Maglev: Google’s Load Balancer Paper*
3. **Explore:**
   - Prometheus & Grafana dashboards
   - Implement a distributed cron job
   - Implement Redlock algorithm
