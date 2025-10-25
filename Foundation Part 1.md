# 🧱 Foundational Topics in System Design

Comprehensive learnings consolidated from the session **“Foundational Topics in System Design.”**  
Structured for long-term reference, study, and interview preparation.

---

## 1. What is System Design?

**Definition:**  
System design is the process of **translating product requirements into a working system** made up of multiple interconnected components.

It applies to:
- A small module or service (microservice)
- A large distributed architecture (e.g., Instagram, YouTube)
- A complete product ecosystem

**Goal:**  
To build a reliable, scalable, extensible, and maintainable system that satisfies both business and engineering constraints.

---

## 2. Layers of System Design

### 2.1 Architectural Design (High-Level)
- Bird’s-eye view of how macro-components connect.  
- Example: `Client → Load Balancer → API Servers → Cache → Database`
- Focuses on **structure**, **data flow**, and **integration**.

### 2.2 Logical Design (Business Logic & Extensibility)
- Defines **algorithms**, **rules**, and **flows** that govern the system.  
- Examples:
  - Tinder ensures a user doesn’t see the same profile twice.
  - Banking systems maintain transaction integrity.
- Key Principle: **Extensibility**
  - Use abstractions (DAO, service layer) to decouple components.
  - Enables changing DB providers or cloud infra with minimal impact.

### 2.3 Physical Design (Infrastructure & Deployment)
- Deals with real-world constraints:
  - Cloud configuration (AWS, Azure, GCP)
  - Instance sizing, IOPS, disk types
  - Backup/restore mechanisms
  - Auto-scaling and cost optimization
- Requires **capacity planning** and understanding of hardware trade-offs.

---

## 3. Balancing Business and Engineering

- Sometimes **business priorities** override ideal engineering.
  - Example: Launching a critical feature quickly even if it creates technical debt.
- Avoid **premature optimization**.
- Focus on delivering value early, but **design for extensibility** later.

---

## 4. Scoping a System

Every design must define:
- **Functional Scope:** End-user features.
- **Non-Functional Scope:** Performance, availability, consistency, security.

### Ask Critical Questions
- “What if traffic increases 10x?”
- “What happens if one data center fails?”
- “Can we migrate to a new DB easily?”

### Seek Clarifications
Ambiguity kills design quality.  
Clarify early between SQL vs NoSQL, queue vs stream, or which consistency guarantees matter most.

---

## 5. Design Approaches

| Approach | Description | Common Usage |
|-----------|--------------|--------------|
| **Bottom-Up** | Build core components first, assemble into system. | Large companies (Google, Microsoft) |
| **Incremental** | Build MVP → iterate & evolve. | Startups, fast-moving teams |

**Golden Rule:**  
> Start small and build on top — but design at least two hops ahead.

---

## 6. Six Foundational Pillars of System Design

These six dimensions appear in *every* system:

| # | Pillar | Description |
|---|---------|-------------|
| 1 | **Database** | Persistent storage layer |
| 2 | **Caching** | Speed up reads using temporary storage |
| 3 | **Scaling** | Vertical (bigger machines) vs Horizontal (more machines) |
| 4 | **Delegation** | Offload tasks via queues, workers, cron jobs |
| 5 | **Concurrency** | Manage threads, locks, semaphores safely |
| 6 | **Communication** | How components talk — HTTP, gRPC, WebSockets, TCP, UDP |

> 💡 If any one pillar is weak, it becomes your bottleneck.

---

## 7. Communication Choices

Not everything is HTTP:

- **HTTP/REST** – Simple, widely supported  
- **gRPC** – Binary protocol, fast, type-safe  
- **WebSockets** – Real-time, bidirectional  
- **TCP/UDP** – Low-level transport for custom protocols  

The choice directly affects **latency**, **reliability**, and **scalability**.

---

## 8. Database Families & Use Cases

| Type | Example | When to Use |
|------|----------|-------------|
| **Relational (SQL)** | MySQL, Postgres | Structured data, transactions |
| **NoSQL** | MongoDB, DynamoDB | Flexibility, scalability |
| **In-Memory** | Redis, Memcached | Caching, real-time counters |
| **Graph DB** | Neo4j | Relationship-heavy data |
| **Time-Series DB** | InfluxDB, Prometheus | Metrics, telemetry |
| **Columnar DB** | ClickHouse | Analytics, OLAP workloads |
| **Blob Storage** | S3, GCS | Media, large files |
| **Search DB** | ElasticSearch | Full-text search |

> ⚡ Modern systems use a mix (polyglot persistence).  
> Example: SQL for business data + Redis for caching + S3 for images + Elastic for search.

---

## 9. Case Studies & Real Systems

| System | Key Learning |
|--------|---------------|
| **Instagram** | 14M users, 3 engineers → Monolith on AWS Django |
| **WhatsApp** | Low-latency messaging for billions |
| **YouTube** | Massive scale, rapid feature rollout (e.g., Shorts) |
| **Amazon** | Predictive analytics and reliability at extreme scale |
| **Dream11 / Zerodha** | Heavy reliance on AWS and infra design |
| **Unacademy** | Security breach → full infra rotated in 3 days |

> Real-world lesson: Security posture and recoverability are as critical as scale.

---

## 10. Practical Example – Designing a Blog System

### 10.1 Requirements
- Single-user blog  
- Multiple posts with tags  
- Search by keyword  
- Ordered by publish time  
- 1M concurrent readers  
- 99.9999% uptime  
- Objective performance goal: API < 500 ms  

---

### 10.2 Architecture Overview

```text
Client
  │
  ▼
Load Balancer
  │
  ▼
API Server ──────────────► Cache (Redis)
  │
  ├──► Database (Postgres / MySQL)
  │
  ├──► Search Index (ElasticSearch)
  │
  └──► Object Store (S3)
```

### 10.3 Database Schema

#### 🧩 Users Table

| Column | Description |
|--------|--------------|
| id | Unique identifier for the user |
| name | User's name |
| bio | Short user biography |

#### 🧩 Blogs Table

| Column | Description |
|--------|--------------|
| id | Unique identifier for the blog |
| title | Title of the blog post |
| user_id | Author reference (foreign key to Users) |
| published_at | Timestamp when published |
| body | Main content of the blog |
| excerpt | Short preview or summary |
| slug | SEO-friendly URL version of the title |
| is_deleted | Soft delete flag (true = deleted) |
| updated_at | Last modification timestamp |

**Notes:**
- `slug` → SEO-friendly URL version of title.  
- `excerpt` → One-line teaser for the post list.  
- `is_deleted` → Supports soft delete and recovery.  

---

### 10.4 Deletion Strategy

| Type | Behavior | Pros | Cons |
|------|-----------|------|------|
| **Hard Delete** | Physically removes data | Saves space | Irreversible, fragments DB pages |
| **Soft Delete** | Marks row as deleted | Recoverable, easy audit | Slightly more app logic |

> 🕒 **Tip:** Run batch cleanup jobs (nightly) to purge soft-deleted data safely during low-traffic hours.

---

### 10.5 Tags Schema

#### 🧩 Tables

| Table | Columns |
|--------|----------|
| **tags** | id, name |
| **blogs_tags** | blog_id, tag_id |

**Notes:**
- Integer foreign keys are faster than string matches.  
- Enables tag renames without rewriting blog rows.  

---

### 10.6 Caching

- Cache **hot posts**, **tag pages**, and **search results**.  
- Example: Redis or Memcached layer in front of DB.  
- Reduces repeated reads and brings **P95 latency under 100 ms**.

---

### 10.7 Delegation (Background Work)

Move **slow or non-critical tasks** out of the user request path:

- Update search index after post creation  
- Generate image thumbnails  
- Send notification emails  

✅ Keeps user-facing APIs consistently fast (< 10 ms).  

---

### 10.8 Scaling

- **Horizontal scaling:** Add more API servers.  
- **Vertical scaling:** Increase machine resources temporarily.  
- **Read replicas:** Serve heavy read load from followers.  
- **Auto-scaling policies:** e.g., “Add 1 node per 200 RPS.”  

---

### 10.9 Concurrency

- Use **locks/mutexes** for shared counters.  
- Design **idempotent writes** to avoid duplicates.  
- Protect **critical sections** when updating shared state.  

---

### 10.10 Communication Layer

| Use Case | Recommended Protocol |
|-----------|----------------------|
| **Public clients** | REST APIs (JSON over HTTPS) |
| **Internal services** | gRPC or message queues |
| **Live updates** | WebSockets for real-time feeds |

---

### 10.11 Physical Considerations

- Plan **backups**, **restores**, and **region failover**.  
- Use **monitoring** (Prometheus + Grafana).  
- Configure **auto-scaling thresholds** carefully.  
- Include **alerting** for latency, CPU, and error spikes.  

---

## 11. Design Best Practices

✅ Define scope early (functional + non-functional).  
✅ Keep requirements **objective**, not vague.  
✅ Ask **critical questions** to clarify expectations.  
✅ Choose **communication protocols** deliberately.  
✅ **Start small**, then scale thoughtfully.  
✅ Document **assumptions** and **trade-offs**.  
✅ Avoid tight **vendor coupling**.  
✅ Prioritize **security** and **observability**.  

---

## 12. System Design Checklist

| Category | Guiding Question |
|-----------|------------------|
| **Requirements** | Are functional & non-functional goals clear? |
| **Data** | Which DB type fits the workload? |
| **Caching** | What needs to be cached? TTL strategy? |
| **Scaling** | Vertical or horizontal plan defined? |
| **Delegation** | Which tasks can be offloaded? |
| **Concurrency** | Shared data or race conditions handled? |
| **Communication** | Chosen protocol & failure handling? |
| **Failure Recovery** | Backup, restore, retry strategy? |
| **Security** | Auth, rate limiting, encryption included? |

---
