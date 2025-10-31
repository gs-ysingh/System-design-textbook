# 🧩 Multi-Step Processes

## What Are Multi-Step Processes?

In real-world systems, a single user action (like "place order") often triggers many dependent steps — such as:

- Charge the user's payment
- Reserve inventory
- Create a shipping label
- Send a confirmation email
- Wait for human pickup

Each of these steps talks to different services, and any one of them can fail, timeout, or take hours/days to complete.

Building such flows reliably — with retries, crash recovery, and coordination — is what multi-step process handling is about.

## 🚨 The Core Problem

Simple services can handle "one read or write" easily (like a database).

But real applications involve:

- Many interdependent services
- Calls to flaky external systems
- Human involvement
- Crashes and restarts

If you try to handle this manually, your code becomes a fragile mess of retries, timeouts, and compensations (like issuing refunds).

If you try to handle this manually, your code becomes a fragile mess of retries, timeouts, and compensations (like issuing refunds).

## 🧠 Step-by-Step Solutions (from basic → advanced)

### 1. Single Server Orchestration

You write all steps sequentially in one service:

```javascript
processPayment();
reserveInventory();
shipOrder();
```

**Pros:**

- ✅ Easy to understand

**Cons:**

- ❌ Fails if the server crashes mid-way (e.g., charged money but didn't ship)
- ❌ Doesn't scale, no recovery mechanism

**Lesson:** Works only for simple flows. As soon as reliability or long duration is needed, it breaks.

### 2. Single Server + State Persistence

You start saving state between steps in a DB and use pub/sub for callbacks.

**Pros:**

- ✅ Survives server restarts

**Cons:**

- ❌ Becomes complicated — you're manually creating a distributed state machine
- ❌ Hard to handle partial failures (e.g., refund if shipping fails)

**Lesson:** Avoid reinventing workflow management by hand.

### 3. Event Sourcing

Instead of storing just the latest state, you store every event (like "OrderPlaced", "PaymentCharged", "InventoryReserved").

Workers consume these events and emit new ones.

**Pros:**

- ✅ Fault-tolerant (if one worker fails, another picks up)
- ✅ Scalable (just add more workers)
- ✅ Full audit trail

**Cons:**

- ❌ Requires significant infra (event logs, queues, monitoring)
- ❌ Debugging event flows can get tricky

**Lesson:** Great foundation, but still too much plumbing work.

### 4. Workflow Systems & Durable Execution

These systems automate what you've been building manually so far.

They:

- Save progress between steps
- Resume automatically after crashes
- Handle retries and compensations
- Keep full history and observability

There are two main approaches 👇

## 🏗️ Durable Execution Engines (Code-driven)

You write workflow logic as code, and the engine (like Temporal) ensures it's durable.

Example (Temporal):

```javascript
const result = await processPayment(order);
if (result.success) await reserveInventory(order);
```

Even if the machine crashes, Temporal replays the workflow from history and continues.

**Pros:**

- ✅ Survives restarts
- ✅ Activities are retried safely
- ✅ Code looks normal

**Used by:** Uber (Cadence/Temporal), Netflix, Stripe

## 🗂️ Managed Workflow Systems (Declarative)

You define the workflow using JSON/YAML, and the system (like AWS Step Functions) runs it for you.

**Pros:**

- ✅ No infra to manage
- ✅ Visual diagrams

**Cons:**

- ❌ Less flexible (JSON-heavy, limited logic)

**Best for:** AWS-based systems or simpler orchestrations.

## 🧰 Comparison

| System | Style | Pros | Cons |
|--------|-------|------|------|
| Temporal | Code | Powerful, reliable, long-running | Complex setup |
| AWS Step Functions | Declarative | Easy to use, no infra | Limited expressiveness |
| Airflow | DAG-based | Great for data pipelines | Not for real-time workflows |

**Interview tip:**
Say "I'd use Temporal for robust business workflows; Step Functions for lightweight AWS orchestration."

## 💡 When to Use Workflows (Interview Insight)

**Use workflows when:**

- You need multiple dependent steps
- Steps may fail and need compensation
- The process takes long (minutes to days)
- You need guaranteed completion or rollback

**Examples:**

- Payment systems (charge, refund, inventory)
- Human approval flows (loan processing, deliveries)

**When NOT to use:**

- Simple API or CRUD
- Single async job (e.g., resize image)
- Latency-sensitive user requests

## 🧭 Common Deep Dives in Interviews

### 1. Workflow Updates

"What if you change a workflow while thousands are running?"

- **Versioning:** Old flows run old code; new flows use new version.
- **Migration:** Introduce patches in code to switch logic safely.

### 2. Large Workflow State

Temporal replays all history, so keep payloads small and recreate long-running workflows periodically.

### 3. External Events / Human Waits

Use signals to pause workflow until an external event (e.g., signature received) — no need for polling.

### 4. Exactly-once Execution

Make activities idempotent — so retries don't cause double charges or duplicate emails.

## 🧩 Final Takeaway

Workflow systems solve the toughest parts of distributed systems:

- Reliable multi-step orchestration
- Fault-tolerant state management
- Automatic retries and compensations
- Auditability and observability

If you find yourself coding complex retry + state logic manually —
👉 **it's time to use a workflow system like Temporal.**

## 🏁 Summary Table

| Level | Approach | Good For | Weakness |
|-------|----------|----------|----------|
| 1️⃣ | Single server | Simple sequences | Fails on crash |
| 2️⃣ | Server + state | Persistent steps | Messy logic |
| 3️⃣ | Event sourcing | Scalable orchestration | Complex infra |
| 4️⃣ | Workflow engines | Long-running, reliable systems | Operational overhead |
