# Networking

Think of networking like sending a letter through a postal system. Each "layer" takes care of a different part of the process so you don't have to worry about every detail yourself.

## 1. Network Layer (Layer 3) — "The Postal Service"

**What it does:** Figures out how to get your data (like a letter) from your computer to another one across the internet.

**Protocol:** IP (Internet Protocol)

**Details:**

- Breaks your data into packets (small chunks)
- Adds the addresses (your IP and destination IP)
- Routes the packets through different paths to reach the destination
- Doesn't guarantee delivery — some packets can be lost, duplicated, or arrive out of order

👉 *Think of it like sending multiple envelopes — some might get delayed or lost.*

## 2. Transport Layer (Layer 4) — "The Delivery Options"

**What it does:** Makes sure your data actually gets there properly — or not, depending on which option you use.

### TCP (Transmission Control Protocol)

- Like registered mail — you first confirm the connection ("handshake"), then send the data
- Guarantees that everything arrives in order and without errors
- But it's slower because it checks and confirms each step

### UDP (User Datagram Protocol)

- Like sending postcards — no setup, no confirmation
- Faster, but if something gets lost or scrambled, too bad
- Great for video calls or games where speed matters more than perfection

## 3. Application Layer (Layer 7) — "The App That Uses the Network"

**What it does:** Provides the rules for how specific apps talk to each other.

**Examples:**

- **HTTP** – for web pages
- **DNS** – for finding website addresses
- **WebSocket/WebRTC** – for real-time communication (like chat or video calls)

👉 *This is what you, as a developer, usually work with — the layer closest to your app.*

## 🧩 How They Work Together (Example)

When you open a webpage:

1. **Application Layer (HTTP):** Your browser says, "Hey, I want to fetch this page."
2. **Transport Layer (TCP):** It creates a reliable connection to the web server.
3. **Network Layer (IP):** The request is split into packets, each routed across the internet.

The server responds, and the layers work in reverse to deliver the webpage to you.

---

## What Happens When You Type a URL in Your Browser?

When you type a URL (like `https://hellointerview.com`) in your browser and hit Enter, a lot happens behind the scenes. But it's just a series of layers working together to fetch that page for you.

Let's go through it in plain, real-world terms 👇

### 🌍 Step 1: DNS Resolution (Finding the Address)

When you type a website name, your computer doesn't understand "hellointerview.com" — it needs the IP address (like a phone number for a website).

So your computer asks:

> "Hey DNS, what's the IP address for hellointerview.com?"

DNS replies with something like:

> "It's 32.42.52.62."

Now your browser knows where to send the request.

**🧠 Analogy:** You look up someone's phone number in a phone book before calling them.

### 🔗 Step 2: TCP Handshake (Making the Connection)

Now that your computer knows where to go, it must set up a reliable connection with that server. This is done using something called the **TCP three-way handshake**.

It's like saying hello politely before you start talking 👋

- **SYN → Client:** "Hi, can I talk to you?"
- **SYN-ACK → Server:** "Sure! I'm ready."
- **ACK → Client:** "Great, let's start."

Now both sides agree to communicate reliably.

**🧠 Analogy:** It's like two people saying:
> "Hello?" → "Hi!" → "Can you hear me?"

before starting a phone conversation.

### 📬 Step 3: HTTP Request (Asking for the Page)

Now that the connection is ready, your browser sends a message like:

```http
GET / HTTP/1.1
Host: hellointerview.com
```

This means:

> "Please give me the homepage of this website."

**🧠 Analogy:** You tell your friend on the phone exactly what you want to talk about.

### ⚙️ Step 4: Server Processing

The server receives your request, finds the right file (like `index.html`), maybe runs some backend logic, and prepares a response.

**🧠 Analogy:** Your friend looks for the answer before replying.

### 📦 Step 5: HTTP Response (Getting the Data)

The server sends back a response like:

```http
HTTP/1.1 200 OK
Content-Type: text/html
```

followed by the HTML page content.

Your browser receives it, renders it, and displays the webpage you wanted 🎉

**🧠 Analogy:** Your friend answers your question.

### 🚪 Step 6: TCP Teardown (Saying Goodbye)

Once the page is fully sent, both sides close the connection politely — this is a four-step goodbye:

- **Client:** "I'm done talking" (FIN)
- **Server:** "Okay, got it" (ACK)
- **Server:** "I'm done too" (FIN)
- **Client:** "All right, bye!" (ACK)

**🧠 Analogy:** Ending a phone call with:
> "Okay, talk later." → "Bye!" → *click*

---

## ⚡ Important Takeaways for System Design

### 1. Each round trip adds latency

Every handshake (both TCP and DNS) means extra time before actual data moves.

- DNS lookup = one round trip
- TCP setup = another
- Then HTTP data transfer begins

That's why things like **connection reuse (HTTP Keep-Alive)** and **persistent connections** matter — they avoid doing this setup again and again.

### 2. TCP Connection = Shared State

Once connected, both client and server have to maintain connection state — memory, buffers, etc.

If you create a new connection for every request, it wastes resources.

That's why modern systems often:

- Reuse connections (Keep-Alive)
- Use persistent connections for realtime (like WebSockets)

---

## Load Balancers


---

## Load Balancers

A load balancer is like a traffic cop 🚦 standing in front of a group of servers. Its job is to decide which server should handle each incoming request — so no single server gets overloaded.

In big systems (like chat apps, e-commerce sites, or streaming platforms), you almost never have just one server. You have many servers, and a load balancer in front of them to split the work evenly.

### 🧱 Two Main Types of Load Balancers

There are two types, based on which networking layer they work at:

- **Layer 4 (Transport layer)** → Works at the TCP/UDP level
- **Layer 7 (Application layer)** → Works at the HTTP/content level

Let's make this easy to picture 👇

---

### ⚙️ Layer 4 (L4) Load Balancer — "The Fast Traffic Cop"

Operates at the **Transport Layer (TCP/UDP)**.

Makes decisions based on **IP address and port** — not the content of the request.

**🧠 Think of it like:**

> A toll booth worker who doesn't care what's inside the car, just checks the license plate and sends the car to a lane (server).

**⚡ What it does:**

- Forwards raw network traffic (TCP packets) directly to servers
- Keeps one persistent TCP connection between the client and a specific server
- Super fast and lightweight (because it doesn't read or modify the content)
- Doesn't understand HTTP or URLs — it only sees IPs and ports

**🪄 When to use it:**

- When performance and speed matter most
- When using persistent connections, like WebSockets (chat, live updates, gaming)

**✅ Example:**

> You connect to `chat.example.com` →  
> L4 load balancer picks Server 2 →  
> All your WebSocket messages go to that same server until you disconnect.

🧩 It's as if you're connected directly to that server.

---

### 🌐 Layer 7 (L7) Load Balancer — "The Smart Traffic Cop"

Operates at the **Application Layer (HTTP)**.

Understands the actual data inside your requests (like URLs, headers, cookies).

**🧠 Think of it like:**

> A toll booth worker who opens the car's trunk and checks what's inside to decide where to send it.

**⚡ What it does:**

- Reads the HTTP request content
- Can make intelligent routing decisions, like:
  - "If the URL starts with `/api/`, send it to the API servers."
  - "If the user's cookie says `user_id=123`, always send them to the same backend."
- Terminates the incoming client connection and creates a new connection to the backend
  - (So client and server are not directly connected — the load balancer sits in between.)

**⚙️ Trade-offs:**

- More flexible (can route based on request details)
- More CPU-intensive (since it reads and processes each request)
- Usually better for HTTP-based apps like websites, APIs, etc.

**✅ Example:**

> A request for `/api/users` → goes to API servers  
> A request for `/home` → goes to Frontend servers

🧩 It behaves a bit like an API Gateway.

