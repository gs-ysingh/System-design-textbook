# 🧠 Distributed ID Generators — Complete Learnings Summary

## 1. 💡 Core Idea

ID generation is the process of creating **globally unique identifiers (IDs)** for database entities.
It becomes complex at **scale**, especially across distributed systems where multiple machines, threads, and time synchronization come into play.

---

## 2. 🏰 Common ID Generation Strategies

| Strategy                 | Description                            | Pros                                     | Cons                                            |
| ------------------------ | -------------------------------------- | ---------------------------------------- | ----------------------------------------------- |
| **Auto-increment (SQL)** | Database auto-generates sequential IDs | Simple, monotonic                        | Not scalable across distributed nodes           |
| **UUID / GUID**          | 128-bit random identifier              | Globally unique, no central coordination | Large size, non-sequential, poor index locality |
| **Timestamp-based**      | Use Unix epoch (ms, µs, ns)            | Naturally increasing                     | Collisions possible if two requests in same ms  |
| **Hash-based (MD5/SHA)** | Hash of content or attributes          | Consistent, unique for same data         | Collisions (rare), expensive                    |
| **Composite**            | Combine (machine_id + time + counter)  | High uniqueness, flexible                | Slightly complex                                |

---

## 3. ⚙️ Evolution of Better ID Design

### Step-by-step refinement

1. **Use current time** → unique but can collide in same ms.
2. **Add machine/thread ID** → avoids cross-thread/machine collision.
3. **Add counter** → avoids same-ms collision.
4. **Persist counter** → solves reboot-reset issue.
5. **Batch write counter to disk (every N ops)** → reduces I/O cost.

---

## 4. 🤎 Key Design Insights

* **Time should be MSB (most significant bits)** — makes IDs sortable and roughly monotonic.
* **Clocks across machines drift**, so strict monotonicity is hard without centralization.
* **Centralized ID Service** ensures order but adds latency and a single point of failure.
* **Distributed generation** improves availability but loses strict ordering.

---

## 5. 🛡️ Real-world Approaches

### 🌄 Flickr

* Two ID servers behind a load balancer.
* One generates **odd**, another **even** IDs.
* Nearly monotonic without collision.
* **Pros:** Simple, distributed.
* **Cons:** Not strictly sequential.

---

### 🕊️ Twitter Snowflake

**Structure (64-bit):**

```
41 bits → timestamp (epoch ms)
10 bits → machine ID
12 bits → sequence number
```

* Uses **custom epoch** to save bits.
* Sortable by time and unique per machine.
* Enables **pagination via ID**, not limit-offset.

**Advantages:**

* Efficient storage (64-bit)
* Roughly monotonic, sortable
* No central bottleneck

**Trade-offs:**

* Minor clock skew issues.

Also adopted by **Discord** and **Instagram (Sonicflake)**.

---

### 📸 Instagram – Database-based Snowflake

* Implemented Snowflake **inside Postgres** using stored procedures.
* Custom epoch (2011), 13 bits for shard, 10 bits for sequence.
* Table auto-generates ID via SQL function:

```sql
CREATE FUNCTION next_id() RETURNS bigint AS $$
BEGIN
  RETURN (timestamp << 23) | (shard_id << 10) | sequence_id;
END;
$$ LANGUAGE plpgsql;
```

**Pros:**

* No external service.
* Database-level scalability.
* Logical + physical sharding ready.

**Cons:**

* Tightly coupled to DB implementation.

---

### 🖊️ Discord (Sonyflake)

* Forked Twitter's Snowflake.
* Epoch set to **2015**.
* Implemented in Go.

---

## 6. ⚡ Pagination Optimization (ID vs Offset)

| Approach                          | Performance             | Comment                |
| --------------------------------- | ----------------------- | ---------------------- |
| `LIMIT + OFFSET`                  | Slows with depth (O(N)) | Must scan skipped rows |
| `WHERE id > last_seen_id LIMIT N` | Constant                | Leverages ordered IDs  |

### Benchmark

* `LIMIT/OFFSET` → response time grows with each page.
* `WHERE id > x` → constant latency even at millions of rows.

---

## 7. 🔐 Other Learnings

* **Bloom Filters** for fast existence checks (e.g., email availability).

  * Probabilistic (no false negatives, rare false positives).
  * Used in GitHub usernames, Medium deduplication.

* **Vector Clocks / Lamport Clocks**

  * Logical ordering of distributed events.
  * Expensive; needs gossip.

* **Trade-offs**

  | Type        | Pros                      | Cons                          |
  | ----------- | ------------------------- | ----------------------------- |
  | Centralized | Strict monotonicity       | Latency, SPOF                 |
  | Distributed | Scalability, availability | Clock drift, non-strict order |

---

## 8. 💡 Observations

* **Twitter** scaled by moving ID gen logic to app servers.
* **Pagination on IDs** powers infinite scroll.
* **Amazon** prefetches ID batches (e.g., 10K per app server).
* **Instagram** DB-based; **Discord** Go-based.

---

## 9. 🤟 Design Guidelines

1. IDs must be **globally unique** and ideally **monotonic**.
2. Combine **time + machine ID + counter** for scalable uniqueness.
3. Prioritize **distributed (Snowflake)** for scale, **central** for consistency.
4. Keep **time on the left (MSB)**.
5. Avoid per-call I/O; **batch writes**.
6. **Paginate via IDs**, not offsets.
7. Use **Bloom filters** for fast lookups.

---

## 10. 🔄 Comparison Table

| Feature        | Central Server | Snowflake  | Instagram (DB) |
| -------------- | -------------- | ---------- | -------------- |
| Monotonic      | ✅ Strict       | ⚠️ Approx. | ✅ Strict       |
| Distributed    | ❌              | ✅          | ⚠️ Partial     |
| Complexity     | Medium         | Medium     | High           |
| Latency        | High           | Low        | Low            |
| Scalability    | Limited        | Excellent  | Good           |
| Failure Impact | High           | Low        | Medium         |

---

**Summary:**

> Monotonic, unique, scalable IDs are the foundation of distributed systems.
> Twitter’s Snowflake design became the gold standard, balancing order, scalability, and cost.
