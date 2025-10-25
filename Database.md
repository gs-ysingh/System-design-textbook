# 🗄️ Relational Databases – Complete Learnings

## 1. Basics of Relational Databases

Relational databases store data as rows and columns—like digital ledgers (e.g., Excel sheets).

**Most popular systems:** MySQL, PostgreSQL, Oracle.

### ACID Guarantees

- **Atomicity** → Transaction is a single unit (all or none)
- **Consistency** → Data moves from one valid state to another (constraints must hold)
- **Isolation** → Concurrent transactions don't interfere
- **Durability** → Committed changes persist across crashes

## 2. Database Normalization

**Goal:** Reduce redundancy and improve integrity.

### Normal Forms

- **1NF:** Data is atomic—no CSV or arrays in a cell
- **2NF:** Remove partial dependencies  
- **3NF:** Every non-key field depends only on the primary key

> **Note:** 3NF is widely used, but at scale, denormalization is introduced for performance (joins become costly).

## 3. Indexes

**Purpose:** Speed up lookups.

**Trade-off:** Slower writes (each write updates indexes).

**Implementation:** Usually a B+ tree.

### Types

- **Primary Index:** Key + address to data
- **Clustered Index:** Key + full row data (faster reads, heavier writes)

### Best Practices

- Use few clustered indexes (≤3 per table)
- Rebuild or defragment indexes periodically
- Use `EXPLAIN` to analyze query plans

> **⚠️ Caution:** Too many indexes increase I/O during writes.

## 4. Transactions & ACID Deep Dive

Use transactions to group multiple SQL operations atomically:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

- **Commit log:** Ensures recovery post-failure
- **Check constraints:** Enforce business rules like `balance >= 0`

## 5. Replication

### Master–Replica Setup

- Writes go to **Master**
- Reads go to **Replicas**  
- Replicas stay updated via query or binlog replication

### Replication Strategies

- **Synchronous:** Safe but slower (writes wait for replica confirmation)
- **Asynchronous:** Faster but risk of data loss on master crash
- **Failover:** If master fails, promote a replica

## 6. Locking & Concurrency

Used to achieve **Isolation**.

### Types of Locks

- **Read (Shared) Lock:** Multiple reads allowed, no writes
- **Write (Exclusive) Lock:** No other reads/writes allowed

**Granularity:** Table, Row, Column, Page.

### Deadlocks

- Occur when transactions wait on each other
- DB engine detects and kills one transaction

### Isolation Levels

- `READ UNCOMMITTED`
- `READ COMMITTED`
- `REPEATABLE READ`
- `SERIALIZABLE`

### Locking SQL Syntax

```sql
SELECT * FROM users WHERE id = 1 FOR UPDATE;   -- Exclusive lock
SELECT * FROM users WHERE id = 1 FOR SHARE;    -- Read lock
```

### Skip/No-Wait Locks

- `FOR UPDATE NOWAIT`: Don't wait if row is locked
- `FOR UPDATE SKIP LOCKED`: Skip locked rows

Used in ticketing or queue systems: "Book seats except locked ones."

## 7. Scaling Relational Databases

### 7.1 Vertical vs Horizontal

- **Vertical scaling:** Bigger machine
- **Horizontal scaling:** Add more nodes

### 7.2 Scaling Approaches

- **Read replicas** → Handle high read volume
- **Data sharding** → Split data by key range (A–M, N–Z)
- **Multi-master setup** → Risky, potential conflicts

### 7.3 Common Issues

- Cross-shard joins are expensive
- Normalization limits scalability → prefer denormalization at scale
- Consistency constraints cannot span databases

## 8. Why RDBMS Don't Scale Linearly

- Enforcing ACID makes it hard to distribute
- Cross-table constraints and joins require co-location
- Data model wasn't built for horizontal scale
- Joins are costly → denormalization helps

## 9. Key-Value Store on SQL (SQL-backed KV Engine)

### Schema

| Column | Type | Purpose |
|--------|------|---------|
| `key` | VARCHAR | Primary Key |
| `value` | BLOB | Data (large binary/json) |
| `ttl` | INT | Expiration time (store absolute expiry time, not offset) |
| `ts` | DATETIME | Creation time |
| `is_deleted` | BOOL | Soft delete flag |

### TTL Design

- Store final expiration timestamp, not duration
- Use index on TTL for efficient cleanup
- Cleanup with cron + check expiry on read

### Soft Delete

- Mark `is_deleted = true`, then batch cleanup nightly
- Avoid constant B-tree rebalancing
- Optionally push deletes to buffer/queue for async cleanup

## 10. Scaling the KV Store

### Architecture Evolution

1. Single server + DB
2. Add read replicas
3. Shard data (A–M, N–Z)
4. Add load balancer → multiple app servers

**Problem:** Each app server connects to all DBs → too many open connections.

**Solution:** Add intelligent per-shard load balancers.

- LB1 handles A–M
- LB2 handles N–Z
- Each has own app + DB layer

**Real-world Example:** DynamoDB uses multi-hop load balancing.

## 11. Implementing TTL + Deletes

Cron job removes expired keys.

**On read:**

```sql
SELECT * FROM store WHERE key='k' AND ttl > NOW() AND is_deleted = FALSE;
```

Combine cron + runtime check for accuracy.

## 12. Pagination and Range Scans

Implement "scan all" via last-key pagination:

- User sends `last_key`
- System continues from that key
- When a shard is exhausted, load balancer forwards to the next shard

```json
{ "lastKey": "J" }
```

→ returns next 10 keys after "J"

Similar to DynamoDB's `ExclusiveStartKey`.

Avoid `LIMIT/OFFSET` for large scale scans.

## 13. Real-World Insights

- **DynamoDB** internally uses this sharded design
- **Google Firestore** supports JSON columns and range indexing
- **Netflix, Hotstar, Dream11** → use locking + caching to handle concurrency
- **Cache hit ratio at Netflix:** ~99.7%
- **Autocomplete in Google** → caches top 10 query results to avoid DB hits

## 14. Recommended Books & Resources

📘 **Head First Design Patterns** – for LLD

📗 **Designing Data-Intensive Applications** – for distributed system theory

📙 **Database Internals by Alex Petrov** – for SQL engine internals

## 15. Key Takeaways

- **Normalize for clarity** → **Denormalize for scale**
- **SQL guarantees correctness** → **NoSQL trades off for scalability**
- **Locks and indexes are powerful but double-edged**
- **Always measure query performance using `EXPLAIN`**
- **Cache, shard, and replicate intelligently**
- **Build for failure:** master crashes, deadlocks, and replication lag will happen
