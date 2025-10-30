# Consistent Hashing

## 🧩 The Problem It Solves

When you use WebSockets or Server-Sent Events (SSE), each client maintains a persistent connection to a backend server.

Now imagine you have multiple backend servers — say 10 WebSocket servers — behind a load balancer.

### ❓ Challenge

Let's say:

- User A connects to Server 2
- User B connects to Server 7

Now, when A sends a message to B, Server 2 needs to deliver it to Server 7, because B is connected there.

**But — how does Server 2 know that B is connected to Server 7?** 🤔

If there are millions of users and many servers, you need a way to quickly find out which server owns which user connection.

## ⚙️ The Solution: Consistent Hashing

Instead of randomly assigning users to servers, we **deterministically map** each user to a specific server using a hash function.

That way:

- Each user always connects to the same logical server
- Every server knows which users it is responsible for
- Other services can calculate which server to contact for a given user

## 🧮 Step-by-Step: How It Works

Let's take an example: We have 4 servers and many users.

**Servers:** S0, S1, S2, S3  
**Users:** U1, U2, U3, ...

### 1️⃣ Hash the User ID

Each user ID is hashed to a number:

```text
hash(U1) → 235
hash(U2) → 722
hash(U3) → 121
```

We then map this hash to a server index.

### 2️⃣ Simple Modulo Hashing (basic version)

Use `hash(userId) % N` where `N` = number of servers.

**Example:**

```text
hash(U1) % 4 = 3  → S3
hash(U2) % 4 = 2  → S2
hash(U3) % 4 = 1  → S1
```

So:

- U1 connects to Server 3
- U2 connects to Server 2
- U3 connects to Server 1

### 3️⃣ Client Connection Flow

When a client connects:

1. The load balancer may initially send it to any server
2. That server computes `hash(userId) % N`
3. If the client should be handled by a different server:
   - It redirects the client to the correct server (via a redirect or message)
   - The correct server establishes the persistent WebSocket/SSE connection
4. That server adds the user to its in-memory connection map:

   ```javascript
   userToSocket[userId] = socketInstance
   ```

### 4️⃣ Sending a Message

When A sends a message to B:

1. The message is received by Server(A)
2. Server(A) computes `hash(B) % N` → determines Server(B)
3. Server(A) forwards the message to Server(B)
4. Server(B) looks up B's connection in its in-memory map and sends the message to B's WebSocket

🔁 **So every message hop looks like:**

```text
User A  ──>  Server(A)
Server(A) ──> Server(B)
Server(B) ──>  User B
```

### 5️⃣ Coordination

Each server needs to know:

- How many servers there are (N)
- Their unique index or ID
- The mapping rules

This is handled by a coordination service, like:

- **ZooKeeper**
- **etcd**
- **Consul**

They store the metadata:

```json
{
  "servers": [
    { "id": 0, "address": "ws-server-0" },
    { "id": 1, "address": "ws-server-1" },
    { "id": 2, "address": "ws-server-2" },
    { "id": 3, "address": "ws-server-3" }
  ]
}
```

Servers use this info to compute where each user should go.

## 💥 The Problem with Simple Hashing

If you use `hash(user) % N` and add/remove a server, your N changes → almost every user gets remapped to a new server. 😱

That means:

- All users would disconnect and reconnect
- Huge churn
- Not scalable

## 🌀 The Fix: Consistent Hashing

Consistent hashing solves this by mapping servers and users onto a hash ring.

## 🧭 How Consistent Hashing Works

Imagine a ring with values from 0 → 2³² (hash space).

**Hash all your servers into this ring:**

```text
hash(S0) = 200
hash(S1) = 700
hash(S2) = 1100
hash(S3) = 1600
```

**Hash the user ID into the same ring:**

```text
hash(U1) = 750
```

**Assign each user to the next server clockwise in the ring.**

So:

- U1 (750) belongs to S2 (1100)
- U2 (1200) belongs to S3 (1600)
- U3 (50) belongs to S0 (200)

### 🧩 When a New Server is Added

Suppose we add S4 = 900.

**Only users between S1 (700) and S4 (900) need to move to S4.**  
→ Everyone else stays put.

That's the beauty of consistent hashing — **minimal disruption when scaling**.

## 🧱 Architecture Diagram

```text
                         ┌──────────────┐
                         │ ZooKeeper /  │
                         │ etcd Cluster │
                         └──────┬───────┘
                                │
    ┌───────────────┬───────────┴───────────────┬──────────────┐
    │               │                           │              │
┌─────────┐     ┌─────────┐               ┌─────────┐     ┌─────────┐
│ Server0 │     │ Server1 │               │ Server2 │     │ Server3 │
│ (hash 200)│   │ (700)   │               │ (1100)  │     │ (1600)  │
└─────────┘     └─────────┘               └─────────┘     └─────────┘
     ↑               ↑                         ↑              ↑
     │               │                         │              │
   U3(50)          U1(750)                 U2(1200)        (wraps around)
```

## 🧩 How Message Delivery Works in Practice

Let's say:

- User A (hash=250) → Server 0
- User B (hash=980) → Server 2

When A sends to B:

1. Server 0 computes `hash(B)` → 980 → Server 2
2. Server 0 forwards message to Server 2 (e.g., via gRPC or internal HTTP)
3. Server 2 finds B's WebSocket connection and sends it

## ⚙️ What Happens on Scaling

When adding or removing a server:

1. ZooKeeper updates the server list
2. Servers recompute hash ring
3. Only affected users reconnect
4. Gradual migration possible

You can also do graceful rebalancing, e.g.:

1. Register new server in ring
2. Slowly transfer users in affected range
3. Tear down old connections
