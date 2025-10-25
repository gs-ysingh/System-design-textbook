# Non-Relational Databases (NoSQL) - Learning Summary

## 1. Real-World Incident — Security & Infrastructure Learnings

### Incident Overview
An AWS account was compromised when access keys from a Mac laptop were leaked.

### Impact
- Destructive actions across 700 EC2 instances
- 20+ databases affected

### Response
- Rotated all secrets, database passwords, and instances
- Implemented VPN within 15 minutes for secure access
- Learned importance of endpoint security, even on macOS

### Outcome
- Led to forming a dedicated SRE team
- Migrated infrastructure to Kubernetes with a security-first mindset
- 90% infra migrated to Kubernetes
- Running cost slightly higher but justified by manageability

## 2. NoSQL Fundamentals

### Definition
- **"NoSQL" ≠ "No SQL"** — stands for Non-Relational Databases

### Key Characteristics
- **Schema**: Often called schemaless, but there's still an underlying structure
- **Flexibility**: Some fields can exist in certain documents and be absent in others

### Primary Differences from SQL

- No foreign keys or joins
- High scalability via horizontal sharding
- Eventual consistency (as per the CAP theorem)

> **Key Principle**: Choose databases based on use case, not hype

## 3. CAP Theorem Recap

You can have only **two** out of the following three at a time:

- **Consistency**
- **Availability**
- **Partition tolerance**

> NoSQL generally sacrifices strong consistency for availability and scalability.

## 4. Types of NoSQL Databases

| Type | Description | Examples |
|------|-------------|----------|
| **Document DB** | JSON/Avro/Parquet documents; query inside values | MongoDB, Elasticsearch |
| **Key-Value Store** | Simple K-V access, often in-memory | Redis, DynamoDB |
| **Wide Column Store** | 2-D K-V store; schema can vary per row | Cassandra, HBase, Redshift |
| **Graph DB** | Nodes + Edges for relationships | Neo4j, Dgraph, Amazon Neptune |

## 5. Document Databases

### Characteristics

- Store JSON-like structures
- Allow partial updates (unlike typical K-V stores)
- Support secondary indexes on nested fields

### When to Use

Use when flexible schema, partial updates, and nested querying are needed.

## 6. Columnar Databases — For Analytics

### How It Works

- Data stored by **columns**, not rows
- Ideal for aggregations and analytics
- Avoids reading unnecessary columns → faster queries
- Enables 95–96% compression due to homogeneous column data

### Example Comparison

- **Row store**: scans entire rows even for single column
- **Column store**: reads only selected columns → lower disk I/O

### Best For

- **Analytics workloads (OLAP)**, not transaction systems (OLTP)
- Examples: Redshift, ClickHouse, BigQuery

## 7. Graph Databases

### Overview

Store nodes, edges, and relationships.

### When to Use

- **Use only when graph algorithms are required** (e.g., shortest path, degree of connection)
- **Don't use** for simple relationships like "A follows B"

### Good Use Cases

- Social graphs (LinkedIn degree of separation)
- Recommendation systems (collaborative filtering)

### Recommended Resource

**Dgraph** - open-source, Go-based; clean and simple architecture

## 8. Why NoSQL Scales

### Core Scaling Principles

- **Horizontal scaling** via partitioning/sharding
- SQL scales **vertically** → hardware-limited
- Data is **denormalized** to eliminate cross-table joins

### Trade-offs

- **Higher redundancy**
- **Weaker constraints**

### Partition Strategies

1. **Hash partitioning**: based on hash(key)
2. **Range partitioning**: e.g., a–m → shard1, n–z → shard2
3. **Fixed partitioning**: static allocation (e.g., per customer or region)

### Multi-Tenant Example

Enterprise customers may demand data isolation → fixed partitioning per tenant

## 9. Handling Relationships (Joins) in NoSQL

### Challenge

- NoSQL doesn't support native joins
- Use multiple selects + merge on the application layer

### Two Approaches

1. **Normalized**: Separate documents, multiple queries
2. **Denormalized**: Embed related documents directly (preferred for read-heavy systems)

### When to Denormalize

- Read-heavy systems where full document access is frequent
- Writes are infrequent or updates are small

## 10. N+1 Query Problem

### Problem Description

Occurs when each parent query triggers multiple child queries.

### Common Occurrences

- ORMs (Django, GraphQL)

### Mitigation Strategies

- **Prefetching**
- **DataLoader pattern**
- **Caching results**

## 11. SQL vs NoSQL — When to Use What

| Choose SQL When | Choose NoSQL When |
|-----------------|-------------------|
| Strong consistency & transactions (ACID) | Eventual consistency is fine |
| Complex queries / aggregations | Key-based or range lookups |
| Schema-driven data | Flexible schema or dynamic attributes |
| Analytics / joins needed | High-scale, simple reads & writes |
| Relational integrity | Denormalized, distributed data |

> **Recommendation**: Start with SQL for new systems; move to NoSQL for scale-out needs.

## 12. Sharding Essentials

### Types of Partitioning

- **Horizontal partitioning**: split by data ranges
- **Vertical partitioning**: split by tables (microservices)

### Challenges

- Cross-shard joins difficult
- Data rebalancing on shard change
- Referential integrity lost

## 13. Real-Time Databases (Firestore-like)

### Concept

Cloud-hosted DB that syncs updates in real time across clients.

### Use Cases

- Collaborative apps
- Chat applications
- Dashboards

### Mechanism

- Clients connected via WebSockets
- Each update → broadcast to all connected clients

### Example Flow

```
Client A updates → API → DB
                ↓
         Redis Pub/Sub → notify API servers
                ↓
    WebSocket broadcast → Clients B, C, D update instantly
```

### Key Technologies

- **Socket.IO** for WebSockets
- **Redis Pub/Sub** for inter-server communication
- **MongoDB / Firestore** for persistence

## 14. Scaling WebSockets

### Challenge

Multiple API servers → need inter-server messaging.

### Solution

- All servers subscribe to Redis Pub/Sub
- Any message published → Redis propagates to all subscribers instantly
- Each server broadcasts updates to its local WebSocket clients

### Benefits

- Servers act independently yet stay synced
- Works for chat, comments, live analytics

## 15. Handling Failures & Ordering

### Ordering Strategy

Use timestamps or sequence numbers.

### Handling Conflicts (Multi-write)

- **"Last write wins"** approach
- **Show merge conflicts** (like Git)

### Failure Recovery

- **Redis down**: Clients periodically poll DB to fetch missed messages
- **Reconnect jitter**: Add random delays (1–10s) to prevent "thundering herd"

## 16. Key Takeaways

- **NoSQL is about trade-offs** — flexibility & scale over strict consistency
- **Understand data access patterns first**, then pick DB
- **Database selection guide**:
  - Columnar DB → Analytics
  - Graph DB → Relationships
  - Key-Value → Speed
  - Document DB → Flexibility
- **Redis Pub/Sub** is a crucial building block for real-time scalable systems
- **Always architect for failure recovery, ordering, and conflict handling**
