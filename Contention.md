# Dealing with Contention

## Overview

**Contention** occurs when multiple processes compete for the same resource simultaneously (e.g., booking the last concert ticket, bidding on an auction). Without proper handling, you get race conditions, double-bookings, and inconsistent state.

## The Problem: Race Conditions

### Example: Concert Ticket Booking
- **Scenario**: 1 seat left, Alice and Bob both click "Buy Now" simultaneously
- **What happens**:
  1. Both read "1 seat available"
  2. Both check availability (1 ≥ 1) ✓
  3. Both proceed to payment
  4. Both get charged, seat count → -1
  5. Both receive same seat number → conflict!

### Root Cause
- Reading and writing aren't atomic
- Gap between "check state" and "update state" creates opportunity for race conditions
- Problem scales with concurrent users (10,000+ users amplifies conflicts)

## Solutions

Solutions progress from simple to complex: single-database → coordination mechanisms → distributed systems.
### Single Node Solutions

#### 1. Atomicity (Transactions)

**Concept**: Group of operations either all succeed or all fail (no partial completion).

**Example**: Money transfer
```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE user_id = 'alice';
UPDATE accounts SET balance = balance + 100 WHERE user_id = 'bob';
COMMIT;
```

**Ticket purchase example**:
```sql
BEGIN TRANSACTION;
UPDATE concerts SET available_seats = available_seats - 1 WHERE concert_id = 'weeknd_tour';
INSERT INTO tickets (user_id, concert_id, seat_number) VALUES ('user123', 'weeknd_tour', 'A15');
COMMIT;
```

**Limitation**: Atomicity alone doesn't prevent concurrent transactions from reading the same data. Alice and Bob can both see "1 seat available" and both succeed, overselling the seat.

---

#### 2. Pessimistic Locking

**Concept**: Prevent conflicts by acquiring locks upfront. Assumes conflicts will happen.

**Example with explicit row locks**:

```sql
BEGIN TRANSACTION;
-- Lock the row first to prevent race conditions
SELECT available_seats FROM concerts
WHERE concert_id = 'weeknd_tour'
FOR UPDATE;

-- Now safely update
UPDATE concerts SET available_seats = available_seats - 1
WHERE concert_id = 'weeknd_tour';

INSERT INTO tickets (user_id, concert_id, seat_number)
VALUES ('user123', 'weeknd_tour', 'A15');
COMMIT;
```

**How it works**: `FOR UPDATE` acquires an exclusive lock. When Alice runs this, Bob's transaction blocks at the SELECT until Alice commits.

**Performance tips**:
- Lock as few rows as possible
- Hold locks for minimal time
- Lock entire tables kills concurrency

---

#### 3. Isolation Levels

**Concept**: Control how much concurrent transactions can see each other's changes.

**Standard isolation levels**:

- **READ UNCOMMITTED**: Can see uncommitted changes (rarely used)
- **READ COMMITTED**: See only committed changes (PostgreSQL default)
- **REPEATABLE READ**: Consistent reads within transaction (MySQL default)
- **SERIALIZABLE**: Strongest isolation, transactions appear sequential

**Example**:

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
UPDATE concerts SET available_seats = available_seats - 1
WHERE concert_id = 'weeknd_tour';
INSERT INTO tickets (user_id, concert_id, seat_number)
VALUES ('user123', 'weeknd_tour', 'A15');
COMMIT;
```

**Tradeoff**: SERIALIZABLE automatically detects conflicts but is more expensive than explicit locks. Use explicit locks when you know exactly what to lock.

---

#### 4. Optimistic Concurrency Control (OCC)

**Concept**: Assume conflicts are rare, detect them after they occur. Better performance under low contention.

**Pattern**: Use a version number (or existing data) to check if record changed.

**Example with version column**:

```sql
-- Alice reads: 1 seat, version 42
-- Bob reads: 1 seat, version 42

-- Alice updates first:
UPDATE concerts
SET available_seats = available_seats - 1, version = version + 1
WHERE concert_id = 'weeknd_tour' AND version = 42;
-- Success! seats = 0, version = 43

-- Bob tries:
UPDATE concerts
SET available_seats = available_seats - 1, version = version + 1
WHERE concert_id = 'weeknd_tour' AND version = 42;
-- Fails! 0 rows affected (version is now 43)
```

**Using existing data as version** (no separate version column):

```sql
-- Use seat count itself as the "version"
UPDATE concerts
SET available_seats = available_seats - 1
WHERE concert_id = 'weeknd_tour' AND available_seats = 1;
```

**Other examples**:
- Auctions: use current highest bid
- Bank transfers: use account balance
- Inventory: use stock count

**Watch out for ABA problem**: Value changes A→B→A, making it look unchanged. Use monotonically increasing values (counts, timestamps) to avoid this.

**When to use**: Low contention scenarios where conflicts are rare. Occasional retry is worth avoiding lock overhead.

---

### Multiple Node Solutions (Distributed)

**Rule**: Keep data in a single database as long as possible! Only use distributed coordination when you've outgrown single-node capacity.

When you need to coordinate across multiple databases:

#### 1. Two-Phase Commit (2PC)

**Concept**: Coordinator manages transaction across multiple database participants.

**Example**: Bank transfer (Alice in DB A, Bob in DB B)

**Phase 1 - Prepare**:

```sql
-- Database A
BEGIN TRANSACTION;
SELECT balance FROM accounts WHERE user_id = 'alice' FOR UPDATE;
UPDATE accounts SET balance = balance - 100 WHERE user_id = 'alice';
-- Transaction stays open

-- Database B
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance + 100 WHERE user_id = 'bob';
-- Transaction stays open
```

**Phase 2 - Commit/Abort**:

- If both prepare successfully → Coordinator tells both to COMMIT
- If either fails → Coordinator tells both to ROLLBACK

**Critical requirement**: Coordinator must write to persistent log before sending decisions. Without this, coordinator crashes leave participants uncertain.

**Problems**:

- Keeping transactions open across network calls is dangerous
- Coordinator crash leaves databases in limbo
- Locks held during network calls blocks other operations
- Slow/unavailable database blocks entire transfer

**When to use**: Only when you absolutely need atomicity across systems and can tolerate complexity.

---

#### 2. Distributed Locks

**Concept**: Ensure only one process can work on a resource at a time across entire system.

**Technologies**:

- **Redis with TTL**: Fast, automatic expiration, but single point of failure

  ```bash
  SET lock:alice_bob "transfer123" EX 30 NX
  ```

- **Database columns**: Use existing DB, slower, need cleanup jobs

  ```sql
  UPDATE accounts SET locked_until = NOW() + INTERVAL '30 seconds'
  WHERE user_id IN ('alice', 'bob') AND locked_until < NOW()
  ```

- **ZooKeeper/etcd**: Purpose-built, robust, handles failures well, but operationally complex

**User experience pattern**: Create intermediate "reserved" states

- Ticketmaster: "available" → "reserved" (10 min) → "sold"
- Uber: driver "available" → "pending_request" → "assigned"
- E-commerce: item → "in cart" (temp hold) → "purchased"

**Benefit**: Shrinks contention window from minutes to milliseconds.

**When to use**: User-facing flows, simpler than 2PC, need temporary exclusive access.

---

#### 3. Saga Pattern

**Concept**: Break operation into sequence of independent steps, each with compensation (undo).

**Example**: Bank transfer

1. Debit Alice's account (DB A) → COMMIT immediately
2. Credit Bob's account (DB B) → COMMIT immediately
3. Send confirmations

If step fails → compensate previous steps (credit Alice back)

**Tradeoff**:

- ✅ No long-running transactions, no coordinator crashes leaving limbo
- ❌ Temporary inconsistency (Alice debited but Bob not yet credited)
- Handle by showing "pending" states in UI

**When to use**: Need resilience over strict consistency, can handle eventual consistency.

---

## Choosing the Right Approach

| Approach | Use When | Avoid When | Latency | Complexity |
|----------|----------|------------|---------|------------|
| **Pessimistic Locking** | High contention, critical consistency, single DB | Low contention, high throughput | Low | Low |
| **SERIALIZABLE Isolation** | Auto conflict detection, can't identify specific locks | Performance critical, high contention | Medium | Low |
| **Optimistic Concurrency** | Low contention, high read/write ratio | High contention, can't tolerate retries | Low (no conflicts) | Medium |
| **Distributed Transactions** | Must have atomicity across systems | High availability needs, performance critical | High | Very High |
| **Distributed Locks** | User-facing flows, need reservations | No alternatives, purely technical | Low | Medium |

**Decision tree**:

1. **Can you keep data in single database?** → Yes: Use pessimistic or optimistic
   - High contention? → Pessimistic locking (FOR UPDATE)
   - Low contention? → Optimistic concurrency control
   
2. **Multiple databases, must be atomic?** → Distributed transactions
   - Need strong consistency? → 2PC
   - Can handle eventual consistency? → Sagas
   
3. **User experience matters?** → Distributed locks with reservations

**Default choice**: Start with pessimistic locking in single database. Simple, predictable, improvable later.

---

## When to Use in Interviews

**Recognition signals** (proactively identify these):

- Multiple users competing for limited resources (tickets, auctions, inventory)
- Prevent double-booking/double-charging (payments, reservations, scheduling)
- Data consistency under high concurrency (balances, inventory, collaborative editing)
- Race conditions in distributed systems

**Common interview scenarios**:

- **Online Auctions**: Use OCC (current bid as version), distributed locks for "bidding ends soon"
- **Ticketmaster**: Distributed locks with seat reservations (10-min expiration) for better UX
- **Banking/Payments**: Sagas for resilience, mention 2PC if interviewer pushes for strict consistency
- **Ride Sharing**: Distributed locks (driver status "pending_request"), cache with TTL or DB with cleanup
- **Flash Sales/Inventory**: Mix of OCC (stock count as version) + distributed locks (cart holds)
- **Yelp Reviews**: OCC (rating + review count as version) for concurrent review updates

**Pro tip**: Don't wait to be asked! Identify contention early and propose solutions.

---

## Common Deep Dives

### 1. Preventing Deadlocks

**Problem**: Transaction A locks Alice then waits for Bob, Transaction B locks Bob then waits for Alice → circular wait.

**Solution - Ordered locking**: Always acquire locks in consistent order

```python
# Always lock users in ascending ID order
user_ids = sorted([alice_id, bob_id])
for user_id in user_ids:
    lock_account(user_id)
```

**Fallback**: Set transaction timeouts so deadlocks auto-abort and retry.

---

### 2. Coordinator Crashes in 2PC

**Problem**: Coordinator crashes between prepare and commit → databases stuck with open transactions holding locks.

**Solution**:

- Coordinator failover with persistent logs
- New coordinator reads logs and completes transactions
- Production systems use transaction managers (e.g., Java JTA)

**Why Sagas are better**: No locks across network calls, coordinator crash just pauses progress.

---

### 3. ABA Problem

**Problem**: Value changes A→B→A, optimistic check sees same value but missed transitions.

**Example**: Restaurant rating 4.0 → 4.1 → 4.0 (after 2 reviews). Using rating as version misses that reviews happened.

**Solution**: Use monotonically increasing values

```sql
-- Use review_count instead of avg_rating
UPDATE restaurants
SET avg_rating = 4.1, review_count = review_count + 1
WHERE restaurant_id = 'pizza' AND review_count = 100;
```

**Alternatives**: Explicit version column, timestamps, database-provided versions (DynamoDB conditional writes).

---

### 4. Hot Partition / Celebrity Problem

**Problem**: Everyone wants same resource (celebrity follow, Taylor Swift tickets, rare collectible).

**Why normal scaling fails**:

- Can't shard one resource across databases
- Load balancers just distribute requests to servers competing for same DB row
- Read replicas don't help (bottleneck is writes)

**Solutions**:

1. **Change the problem**:
   - Multiple identical items instead of one
   - Make it eventually consistent (social media likes/follows)

2. **Queue-based serialization**:
   - Dedicate queue to hot resource
   - Single worker processes sequentially
   - Eliminates contention, adds latency
   - Better than entire system grinding to halt

---

## Key Takeaways

1. **Stay in single database as long as possible**
   - Modern databases handle terabytes and thousands of connections
   - Single-DB solutions are simpler, faster, more reliable
   
2. **Match approach to contention level**:
   - High contention → Pessimistic locking
   - Low contention → Optimistic concurrency
   
3. **Distributed coordination is expensive**:
   - Only use when truly necessary
   - Sagas > 2PC for most use cases
   
4. **User experience > technical purity**:
   - Reservations/holds prevent user frustration
   - "Pending" states make inconsistency acceptable
   
5. **Always consider**:
   - Deadlock prevention (ordered locking)
   - Failure scenarios (coordinator crashes)
   - Edge cases (ABA problem, hot partitions)

**Default approach**: Pessimistic locking in single database. Simplest solution that works is usually right.
