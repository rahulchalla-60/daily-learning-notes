# Day 003 — Data Sharding

---

## What is Data Sharding?

Sharding is a **horizontal database scaling technique** where a large database is split into smaller, independent pieces called **shards**. Each shard holds a subset of the data and lives on its own server.

**Without sharding** — one database holds everything:
```
Single Database
├── All Users
├── All Orders
├── All Messages
└── All Products
```

**With sharding** — data is partitioned across servers:
```
Shard 1 → Users A–F
Shard 2 → Users G–M
Shard 3 → Users N–Z
```

---

## Why Do We Need Sharding?

As an application grows, a single database becomes a bottleneck:

- More users → more storage needed
- More requests → CPU and memory get saturated
- Larger tables → queries slow down
- Single server → single point of failure

Sharding solves this by spreading data and load across multiple servers.

---

## Vertical Scaling vs Sharding

**Vertical Scaling** — upgrade the same server:
```
Before: CPU=4,  RAM=16GB,  Storage=1TB
After:  CPU=32, RAM=128GB, Storage=10TB
```

**Horizontal Scaling (Sharding)** — add more servers:
```
DB1 | DB2 | DB3 | DB4
```

| Feature | Vertical Scaling | Horizontal Scaling (Sharding) |
|---|---|---|
| Implementation | Easy, no app changes | Complex |
| Cost | Expensive at scale | More flexible |
| Growth Limit | Hardware ceiling | Almost unlimited |
| Fault Tolerance | Single point of failure | Better fault isolation |

> For large-scale systems, sharding is the preferred long-term strategy.

---

## Basic Sharding Architecture

```
Application
     │
Shard Router        ← decides where data lives
     │
┌────┼────┐
│    │    │
S1   S2   S3
```

The **Shard Router** intercepts every query and forwards it to the correct shard. The application doesn't need to know which shard holds what — the router handles it.

---

## Key Terminology

| Term | Definition |
|---|---|
| **Shard** | A smaller partition of the full dataset |
| **Shard Key** | The field used to determine which shard stores a record |
| **Shard Router** | The component that routes queries to the right shard |
| **Hotspot** | When one shard receives disproportionately more traffic |
| **Re-sharding** | The process of redistributing data when adding new shards |

---

## Sharding Strategies

### 1. Range-Based Sharding

Data is divided into **continuous ranges** based on the shard key.

```
Shard 1 → user_id 1 – 1,000,000
Shard 2 → user_id 1,000,001 – 2,000,000
Shard 3 → user_id 2,000,001 – 3,000,000
```

Query routing is simple — the router checks the range:
```sql
SELECT * FROM users WHERE user_id = 1500
-- Router: 1500 is in Shard 1
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Simple to implement | Uneven distribution over time |
| Efficient range queries | Hotspot risk (new users always hit the latest shard) |

**Best for:** Time-series data, logs, ordered datasets.

---

### 2. Hash-Based Sharding

A **hash function** maps the shard key to a shard index. This is the most widely used strategy.

```
Shard = Hash(shard_key) % number_of_shards

Hash(1001) % 3 = 2  →  Store in Shard 2
Hash(1002) % 3 = 0  →  Store in Shard 0
Hash(1003) % 3 = 1  →  Store in Shard 1
```

```
Application
     │
Hash Function
     │
┌────┼────┐
S1   S2   S3
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Even distribution | Re-sharding is expensive |
| Avoids hotspots | Range queries span multiple shards |
| Good scalability | |

**Best for:** User data, session data, most general-purpose sharding.

---

### 3. Directory-Based Sharding

A **lookup table** explicitly maps each key to a shard. The most flexible strategy.

```
Lookup Table
────────────────
User 101  →  Shard 1
User 102  →  Shard 3
User 103  →  Shard 2
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Very flexible | Extra lookup on every request |
| Easy to migrate individual records | Lookup table becomes a critical bottleneck |

**Best for:** Systems that need custom or irregular data placement.

---

### 4. Geographic Sharding

Data is stored based on the **user's geographic region**.

```
India users   →  India DB   (Mumbai region)
USA users     →  USA DB     (Virginia region)
Europe users  →  Europe DB  (Frankfurt region)
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Lower latency for regional users | Cross-region queries are expensive |
| Supports data residency compliance | Uneven shard sizes by population |

**Best for:** Global applications, GDPR/data residency requirements.

---

## Choosing a Good Shard Key

The shard key is one of the most critical design decisions in sharding. A bad choice causes hotspots and uneven distribution.

**A good shard key:**
- Distributes data evenly across shards
- Avoids hotspots
- Is frequently used in queries
- Scales predictably

| ✅ Good Shard Keys | ❌ Bad Shard Keys |
|---|---|
| `user_id` | `country` (uneven population) |
| `customer_id` | `status` (most records are "active") |
| `account_id` | `gender` (only 2 values) |

---

## The Hotspot Problem

A hotspot occurs when one shard receives far more traffic than the others.

```
Shard 1 →   5M users  (normal load)
Shard 2 →   5M users  (normal load)
Shard 3 → 100M users  ← overloaded ❌
```

**Consequences:** Slow queries, high latency, server overload — essentially the same problem sharding was meant to solve.

**Prevention:** Choose a high-cardinality shard key (like `user_id`) and use hash-based sharding.

---

## Re-Sharding

When existing shards fill up or become unbalanced, you need to add new shards and redistribute data.

```
Before: Shard1, Shard2, Shard3
After:  Shard1, Shard2, Shard3, Shard4, Shard5
```

**The challenge:** With simple hash-based sharding (`Hash(key) % N`), adding one shard changes almost every mapping — forcing a massive data migration.

---

## Consistent Hashing

Consistent hashing solves the re-sharding problem by minimizing data movement when shards are added or removed.

**How it works:** Servers and data keys are placed on a **virtual hash ring**. Each key is assigned to the nearest server clockwise on the ring.

```
        Shard A
       /       \
      /         \
Shard C         Shard B
```

When a new shard is added, **only the data between it and its neighbor** moves — not everything.

**Used in:** Apache Cassandra, Amazon DynamoDB, Redis Cluster, most modern distributed caches.

| Simple Hashing | Consistent Hashing |
|---|---|
| Adding 1 shard → most data moves | Adding 1 shard → only ~1/N data moves |
| Expensive re-sharding | Minimal data migration |

---

## Query Routing

The application needs to know which shard holds the requested data. There are two approaches:

**Application-Level Routing** — the app determines the shard itself:
```
Application → compute shard → query DB directly
```

**Proxy-Based Routing** — a proxy layer handles routing transparently:
```
Application → Shard Router (Vitess, ProxySQL) → correct shard
```

> Proxy-based routing is preferred for large systems — it keeps routing logic out of application code.

---

## Challenges with Sharding

### Cross-Shard Joins
Simple in a single database:
```sql
SELECT * FROM users JOIN orders ON users.id = orders.user_id
```

In a sharded system, `users` and `orders` may be on different shards — the join becomes expensive or impossible at the DB level.

**Solutions:**
- Avoid cross-shard joins by co-locating related data on the same shard
- Denormalize data (store copies where needed)
- Perform joins at the application layer

---

### Distributed Transactions
A money transfer between two accounts on different shards requires a transaction spanning two databases.

```
Account A (Shard 1) → debit $100
Account B (Shard 4) → credit $100
```

**Solutions:**
- **Two-Phase Commit (2PC)** — ensures atomicity but adds latency
- **Saga Pattern** — breaks the transaction into steps with compensating rollbacks

---

## Sharding + Replication Together

Most production systems combine both for maximum resilience:

```
         Shard 1              Shard 2              Shard 3
        /       \            /       \            /       \
   Replica   Replica    Replica   Replica    Replica   Replica
```

| Technique | Purpose |
|---|---|
| Sharding | Scales **writes** and **storage** |
| Replication | Scales **reads** and improves **availability** |

---

## Sharding vs Replication

| Feature | Sharding | Replication |
|---|---|---|
| Goal | Scale data & writes | Scale reads |
| Data | Split across nodes | Copied to each node |
| Write Scaling | ✅ Excellent | ⚠️ Limited |
| Read Scaling | ⚠️ Partial | ✅ Excellent |
| Complexity | High | Medium |
| Storage | Different data per node | Same data on all nodes |

---

## Real-World Usage

| Company | How They Use Sharding |
|---|---|
| Instagram | Shards user data across PostgreSQL databases |
| Facebook | Shards user info and social graph |
| YouTube | Distributes video metadata across shards |
| Uber | Shards ride and location data |
| Amazon | Shards customer and order data |

---

## Full Large-Scale Architecture

```
Users
  │
CDN
  │
Load Balancer
  │
Application Servers
  │
Cache (Redis)
  │
Shard Router
  │
┌──────────────┬──────────────┐
│              │              │
Shard 1      Shard 2       Shard 3
│              │              │
Replicas     Replicas      Replicas
```

---

## Pros & Cons Summary

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Near-unlimited scalability | Significantly more complex architecture |
| Higher write throughput | Cross-shard joins are hard |
| Better storage capacity | Distributed transactions are complex |
| Fault isolation per shard | Re-sharding requires data migration |
| Smaller datasets = faster queries | More operational overhead |

---

## Sharding Strategy Cheat Sheet

| Strategy | Routing Method | Best For | Watch Out For |
|---|---|---|---|
| Range-Based | Key range lookup | Time-series, ordered data | Hotspots on latest range |
| Hash-Based | Hash % N | General-purpose, user data | Expensive re-sharding |
| Directory-Based | Lookup table | Custom placement needs | Table becomes a bottleneck |
| Geographic | Region mapping | Global apps, compliance | Cross-region queries |

---

## One-Line Formula

```
Sharding = Horizontal DB Partitioning + Distributed Storage + Write Scalability + Reduced Bottlenecks
```
