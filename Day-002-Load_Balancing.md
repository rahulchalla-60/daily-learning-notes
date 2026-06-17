# Day 002 — Load Balancing

---

## What is Load Balancing?

Load Balancing is the process of **distributing incoming network traffic across multiple servers** so that no single server gets overwhelmed.

**Without a load balancer** — all traffic hits one server:
```
Users → Server
```

**With a load balancer** — traffic is spread evenly:
```
           ┌─────────┐
Users ───► │  Load   │
           │Balancer │
           └────┬────┘
                │
      ┌─────────┼─────────┐
      │         │         │
  Server1   Server2   Server3
```

---

## Why Do We Need It?

| Without Load Balancer | With Load Balancer |
|---|---|
| One server gets overloaded | Traffic is distributed evenly |
| High latency | Better performance |
| Poor user experience | High availability |
| Single point of failure | Fault tolerance & scalability |

---

## Key Benefits

**Improved Availability** — if one server goes down, others handle the traffic:
```
Server1 ❌  →  Traffic redirects to Server2 ✅ and Server3 ✅
```

**Better Performance** — 1000 requests split evenly:
```
Server1 → 333
Server2 → 333
Server3 → 334
```

**Scalability** — just add more servers as traffic grows:
```
Before:  LB → Server1

After:   LB → Server1, Server2, Server3, Server4
```

**Fault Tolerance** — system keeps running even when servers fail.

---

## Core Components

| Component | Role | Examples |
|---|---|---|
| **Client** | Sends requests | Browser, Mobile App |
| **Load Balancer** | Routes requests to servers | NGINX, HAProxy, AWS ELB |
| **Backend Servers** | Process requests & return responses | App servers |

---

## Load Balancing Algorithms

Algorithms decide **which server** gets the next request. They fall into two categories:

| Type | How it works |
|---|---|
| **Static** | Routes without checking server load |
| **Dynamic** | Routes based on current server state |

---

## Static Algorithms

### 1. Round Robin

Requests go to servers **one by one in sequence**, cycling through all servers equally.

```
R1 → S1
R2 → S2
R3 → S3
R4 → S1  ← cycle repeats
R5 → S2
R6 → S3
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Simple to implement | Ignores server capacity |
| Equal distribution | Ignores current load |
| Low overhead | Poor for heterogeneous servers |

**Best for:** Identical servers with similar request processing times.

---

### 2. Weighted Round Robin

Servers are assigned **weights based on their capacity**. Higher weight = more requests.

```
Server1 (weight=3) → gets 3 requests
Server2 (weight=2) → gets 2 requests
Server3 (weight=1) → gets 1 request

Cycle: S1 S1 S1 S2 S2 S3 → repeat
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Accounts for server capacity | Doesn't reflect real-time load |
| Better resource utilization | Requires manual weight tuning |

**Best for:** Servers with different hardware specs (e.g., S1=16 CPU, S2=8 CPU, S3=4 CPU).

---

### 3. Sticky Round Robin (Session Affinity)

A user is **always routed to the same server** after their first request.

```
User A → first hits Server1 → always goes to Server1
User B → first hits Server2 → always goes to Server2
User C → first hits Server3 → always goes to Server3
```

**Why it's needed:** Some apps store session data locally on the server. If a user switches servers, their session is lost.

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Maintains session consistency | Uneven traffic distribution |
| Easier session management | Risk of server overload |
| | Harder to scale |

**Best for:** Legacy apps and session-based systems.

---

### 4. Hash-Based Load Balancing

A **hash function** maps a request property to a specific server. The same input always goes to the same server.

```
Hash(UserID) % NumberOfServers = Server Index

Hash(101) % 3 = 2  →  User 101 always goes to Server2
Hash(102) % 3 = 0  →  User 102 always goes to Server1
```

**Common hash keys:** User ID, Client IP, Session ID, Cookie

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Consistent, predictable routing | Rehashing needed when servers are added/removed |
| Great cache locality | Can cause uneven distribution |

**Best for:** Distributed caching, CDN routing, user-specific workloads.

---

## Dynamic Algorithms

### 5. Least Connections

Routes the request to the server with the **fewest active connections** right now.

```
Current state:
  Server1 → 100 connections
  Server2 → 50  connections
  Server3 → 20  connections  ← selected

New request → Server3
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Adapts to real workload | Connection count ≠ actual load |
| Great for long-lived connections | Needs live monitoring |

**Best for:** Chat apps, WebSockets, streaming systems.

---

### 6. Least Time

Routes to the server with the **lowest response time**, factoring in both speed and active connections.

```
Server1 → 30 connections, 50ms response
Server2 → 50 connections, 20ms response  ← selected (fastest)
Server3 → 10 connections, 100ms response
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Best user experience | More complex to implement |
| Lowest latency | Requires continuous metric monitoring |
| Adaptive routing | |

**Best for:** High-performance systems and large-scale web apps.

---

## Health Checks

The load balancer **continuously pings servers** to check if they're alive.

```
LB
 ├── Server1 ✅  →  receives traffic
 ├── Server2 ❌  →  traffic blocked
 └── Server3 ✅  →  receives traffic
```

If a server fails its health check, it's removed from the rotation automatically. When it recovers, it's added back.

---

## Scaling Strategies

### Vertical Scaling — Scale Up

Upgrade the **existing server** with more CPU/RAM.

```
Before: CPU=4,  RAM=16GB
  ↓
After:  CPU=16, RAM=64GB
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Simple, no architecture changes | Has a hardware ceiling |
| Easy to manage | Expensive at high end |
| | Still a single point of failure |

**Best for:** Small apps, early-stage startups, monolithic systems.

---

### Horizontal Scaling — Scale Out

Add **more servers** to handle the load.

```
Before:       After:
  LB            LB
   |             │
 Server1    ┌────┼────┐
            S1   S2   S3
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Nearly unlimited scalability | More complex to manage |
| High availability & fault tolerance | Data synchronization challenges |
| Cost-flexible | Requires distributed system design |

**Best for:** Large-scale systems, cloud-native apps, microservices.

---

## Vertical vs Horizontal Scaling — Quick Comparison

| Feature | Vertical Scaling | Horizontal Scaling |
|---|---|---|
| Method | Upgrade one server | Add more servers |
| Cost | Expensive | Flexible |
| Complexity | Low | High |
| Availability | Lower | Higher |
| Fault Tolerance | Low | High |
| Scalability Limit | Hardware ceiling | Almost unlimited |
| Performance | Good initially | Better long-term |

---

## Algorithm Quick Reference

### Static (no real-time awareness)
| Algorithm | Key Idea |
|---|---|
| Round Robin | Cycle through servers equally |
| Weighted Round Robin | Cycle by capacity weight |
| Sticky Round Robin | Same user → same server always |
| Hash-Based | Hash of request property → fixed server |

### Dynamic (real-time aware)
| Algorithm | Key Idea |
|---|---|
| Least Connections | Route to server with fewest active connections |
| Least Time | Route to server with fastest response time |

---

## One-Line Formula

```
Load Balancer = Traffic Distribution + High Availability + Fault Tolerance + Scalability + Better Performance
```
