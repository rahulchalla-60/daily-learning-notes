# Day 006 — System Design Goals & Key Buzzwords

---

## 1. Scalability

**"Can the system grow when demand increases?"**

Scalability is the ability of a system to handle increasing traffic, users, or data without significant performance degradation.

**Real-world analogy:**
```
Not Scalable:  1 chef  → 100 customers → chef overwhelmed, long wait
Scalable:     10 chefs → 100 customers → workload distributed, fast service
```

**In system design:**
```
Before:   Users → Single Server        ← crashes under load

After:    Users → Load Balancer → S1
                               → S2
                               → S3    ← traffic shared
```

### Types of Scaling

**Vertical Scaling (Scale Up)** — make one machine more powerful:
```
RAM:  4GB  → 32GB
CPU:  2    → 16 cores
```
Like upgrading your laptop. Simple, but has a hardware ceiling.

**Horizontal Scaling (Scale Out)** — add more machines:
```
Server1 | Server2 | Server3
```
Like adding more cash counters in a supermarket. Preferred for large-scale systems.

> Netflix scales by adding servers worldwide. Amazon scales horizontally during peak sales events.

---

## 2. Availability

**"Can users access the system when they need it?"**

Availability is the percentage of time a system remains operational and accessible.

```
Availability = Uptime / Total Time
```

**Real-world analogy:** Electricity supply — frequent power cuts = low availability.

**In system design:**
```
Single Server:         Users → Server       ← server dies = system down
Multiple Servers:      Users → Load Balancer → S1, S2   ← S1 dies, S2 takes over
```

### Availability Levels

| Availability | Downtime per Year | Common Name |
|---|---|---|
| 99% | ~3.65 days | Two nines |
| 99.9% | ~8.7 hours | Three nines |
| 99.99% | ~52 minutes | Four nines |
| 99.999% | ~5 minutes | **Five nines** |

> Most production systems target 99.99% or higher. Five nines is the gold standard.

---

## 3. Reliability

**"Does the system do the right thing every time?"**

Reliability means the system performs correctly and consistently — not just that it's up, but that it produces accurate results.

**Real-world analogy:**
```
ATM withdraws ₹5,000
Expected: cash dispensed + balance updated ✅
Unreliable: balance deducted but no cash given ❌
```

**In system design:**
```
Payment of ₹1,000
Reliable:   deducted once, merchant credited once ✅
Unreliable: deducted twice ❌
```

### Reliability vs Availability

| Scenario | Available? | Reliable? |
|---|---|---|
| Website loads but shows wrong data | ✅ Yes | ❌ No |
| Correct data but server is down | ❌ No | ✅ Yes |
| Loads correctly every time | ✅ Yes | ✅ Yes |

> A system can be available without being reliable. You want both.

---

## 4. Consistency

**"Do all users see the same data at the same time?"**

After a write operation, every subsequent read should return the latest value.

**Real-world analogy:**
```
Bank balance: ₹10,000
You withdraw: ₹5,000

Every ATM should now show: ₹5,000
Not: ATM1 → ₹5,000  and  ATM2 → ₹10,000  ← inconsistent ❌
```

### Strong Consistency

All users see the latest data **immediately** after a write.

- Data is never stale
- Higher latency (must wait for all nodes to sync)
- **Used in:** banking, payments, stock trading

### Eventual Consistency

Data becomes consistent **after a short delay** — replicas catch up over time.

```
You change your profile picture:
  Your phone      → new picture ✅
  Friend's phone  → old picture (for a few seconds) → new picture ✅
```

- Lower latency, higher availability
- Temporary staleness is acceptable
- **Used in:** social media, product catalogs, DNS

**Companies using eventual consistency:** Facebook, Instagram, Amazon

---

## 5. Fault Tolerance

**"Can the system survive failures?"**

Fault tolerance is the ability of a system to continue operating correctly even when some components fail.

**Real-world analogy:**
```
Airplane with 2 engines:
  One engine fails → plane still flies safely ✅
```

**In system design:**
```
Server1 ✅
Server2 ❌ crashes
Server3 ✅

Result: system continues running on Server1 and Server3
```

### How It's Achieved

- **Replication** — multiple copies of data across servers
- **Redundant servers** — backup instances ready to take over
- **Multiple databases** — no single DB is a single point of failure
- **Multiple regions** — if one data center fails, another takes over

---

## 6. No Single Point of Failure (No SPOF)

**"Is there any one component whose failure takes everything down?"**

A Single Point of Failure (SPOF) is any component that, if it fails, brings the entire system down. Designing for No SPOF means eliminating these bottlenecks.

**Real-world analogy:**
```
One bridge to an island:
  Bridge collapses → island completely isolated ❌
  Bridge = SPOF
```

**In system design:**
```
SPOF design:       Users → Single Server     ← server fails = all down ❌

No SPOF design:    Users → Load Balancer → S1
                                         → S2   ← S1 fails, S2 serves ✅
```

### The Chain Reaction

```
Replication
    ↓
Fault Tolerance
    ↓
High Availability
    ↓
No SPOF
```

Each layer builds on the previous one. Replication enables fault tolerance, which enables high availability, which eliminates SPOFs.

---

## 7. CAP Theorem

One of the most important concepts in distributed systems.

> Proposed by Eric Brewer. A distributed system can guarantee only **two of these three** properties simultaneously.

| Property | What It Means |
|---|---|
| **C — Consistency** | Every read returns the most recent write |
| **A — Availability** | Every request receives a response (may be stale) |
| **P — Partition Tolerance** | System keeps working despite network failures between nodes |

### What is a Network Partition?

A partition is when nodes in a distributed system can't communicate due to network failure:
```
Server A  ✗  Server B   ← connection lost
```

Real causes: cable cuts, router failures, data center outages. This **always** happens in the real world — so Partition Tolerance is non-negotiable.

> Since P is mandatory, the real choice is between **C** and **A** when a partition occurs.

---

### CP Systems — Consistency + Partition Tolerance

Sacrifices availability to maintain correctness.

**What happens during a partition:**
```
Server A  ✗  Server B
↓
Request rejected rather than returning potentially wrong data
```

**Best for:** Banking, payments, inventory — anywhere wrong data causes real harm.

**Example:** ATM network — if balance can't be verified, transaction is blocked rather than risking an incorrect withdrawal.

---

### AP Systems — Availability + Partition Tolerance

Sacrifices consistency to stay available.

**What happens during a partition:**
```
Phone A sees: 101 likes
Phone B sees:  99 likes
↓
Temporary mismatch accepted — service stays up
```

**Best for:** Social media, feeds, notifications — temporary staleness is acceptable.

**Example:** Instagram like counts, news feeds, product reviews.

---

### Why CA (Consistency + Availability without P) Is Impossible

A CA system would need to guarantee consistency and availability but assume the network never fails. In reality, network partitions always happen eventually — so CA is not a viable option for any real distributed system.

```
Real world = network failures always possible
Therefore  = P is always required
Choice     = CP or AP
```

---

## All Six Properties — Side by Side

| Property | Question It Answers | Example |
|---|---|---|
| Scalability | Can it grow with demand? | Add servers for more traffic |
| Availability | Can I always access it? | 99.99% uptime SLA |
| Reliability | Does it always work correctly? | Payment charged exactly once |
| Consistency | Do all users see the same data? | Bank balance is always accurate |
| Fault Tolerance | Can it survive failures? | Server crash doesn't take down the system |
| No SPOF | Is there a hidden kill switch? | Load balancer instead of one server |

---

## Quick Memory Trick

```
Scalability     → Can it GROW?
Availability    → Can I ACCESS it?
Reliability     → Can I TRUST it?
Consistency     → Do we all SEE the same thing?
Fault Tolerance → Can it SURVIVE failures?
No SPOF         → No single component can KILL it?
CAP             → When the network splits, choose C or A
```

---

## CAP Cheat Sheet

| System Type | Guarantees | Sacrifices | Use When |
|---|---|---|---|
| CP | Consistency + Partition Tolerance | Availability | Data accuracy is critical (banking) |
| AP | Availability + Partition Tolerance | Consistency | Always-on is critical (social, feeds) |
| CA | Not achievable in distributed systems | — | — |

---

## One-Line Formula

```
Reliable + Available + Consistent + Fault Tolerant + No SPOF + Scalable = Well-Designed System
```
