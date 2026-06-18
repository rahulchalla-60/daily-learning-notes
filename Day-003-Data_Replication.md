# Day 003 — Data Replication

---

## What is Data Replication?

Replication is the process of **maintaining multiple copies of the same data** across different database servers.

**Without replication** — one database handles everything:
```
Application → Single Database
```

**With replication** — data is copied across multiple servers:
```
              Primary
                 │
     ┌───────────┼───────────┐
     │           │           │
 Replica 1   Replica 2   Replica 3
```

All replicas hold identical data. The primary handles writes; replicas handle reads.

---

## Why Do We Need It?

A single database becomes a bottleneck as traffic grows:

- **High read volume** → DB gets saturated
- **Single point of failure** → one crash = full outage
- **No disaster recovery** → data loss risk
- **Limited availability** → any maintenance = downtime

Replication solves all four of these.

---

## Replication vs Sharding — Quick Distinction

| | Replication | Sharding |
|---|---|---|
| What happens to data | Copied to all nodes | Split across nodes |
| Primary purpose | Scale reads | Scale writes & storage |
| Each node holds | Same data | Different data |

> Use both together for massive systems — sharding scales writes, replication scales reads.

---

## Core Architecture

### Writes — always go to Primary
```
Application → Primary Database
```

### Reads — distributed across Replicas
```
Application
     │
Load Balancer
     │
┌────┼────┐
R1   R2   R3
```

---

## How Replication Works (Step by Step)

```
1. App sends a write to Primary
         ↓
2. Primary stores the change locally
         ↓
3. Primary writes change to Replication Log
   (Binary Log / WAL / Transaction Log)
         ↓
4. Replicas pull from the log and apply changes
         ↓
5. All nodes eventually hold the same data
```

---

## Primary-Replica Responsibilities

| Node | Operations | Use Case |
|---|---|---|
| **Primary** | INSERT, UPDATE, DELETE | All write traffic |
| **Replica** | SELECT | Read traffic, analytics, reporting |

**Real-world load split example:**
```
1000 total requests
  ├── Primary    →  100 writes
  ├── Replica 1  →  300 reads
  ├── Replica 2  →  300 reads
  └── Replica 3  →  300 reads
```

---

## Types of Replication

### 1. Synchronous Replication

Primary **waits for all replicas to confirm** before responding to the client.

```
Client → Write → Primary
                    │
             ┌──────┴──────┐
         Replica 1      Replica 2
             │              │
          Confirm        Confirm
                    │
             Success to Client
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Strong consistency — all replicas up to date | Higher write latency |
| Zero data loss | Slower under network delays |

**Best for:** Banking, payments, anything requiring strict accuracy.

---

### 2. Asynchronous Replication

Primary **responds to the client immediately**, then propagates changes to replicas in the background.

```
Client → Write → Primary → Success to Client
                               │
                          (later...)
                          Replicas updated
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Fast writes, high throughput | Replication lag — replicas may be behind |
| Better overall performance | Risk of data loss if primary crashes mid-replication |

**Best for:** Social media feeds, catalogs, analytics — workloads tolerating slight staleness.

> Most production systems use asynchronous replication by default.

---

## Replication Lag

The time gap between a write on the primary and when it appears on a replica.

```
Primary updated at:  10:00:00
Replica updated at:  10:00:03
                     ──────────
Replication lag:     3 seconds
```

During this window, a user reading from a replica may see **stale data**. This is **eventual consistency** — replicas will catch up, just not instantly.

---

## Replication Topologies

### 1. Single Leader (Primary-Replica)

The most common model. One primary handles all writes; replicas serve reads.

```
       Primary
          │
┌─────────┼─────────┐
│         │         │
R1        R2        R3
```

**Best for:** Most standard web applications.

---

### 2. Multi-Leader (Multi-Primary)

Multiple nodes accept writes and sync changes with each other.

```
Primary A ◄──► Primary B
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| No single point of failure for writes | Write conflicts when same row updated simultaneously |
| Low-latency writes in each region | Complex conflict resolution logic |

**Best for:** Multi-region systems where users need fast local writes.

**Conflict example:**
```
Server A updates balance → 100
Server B updates balance → 200
          ↓
Which value wins? Must be resolved.
```

---

### 3. Leaderless Replication

No designated primary. Writes and reads go to multiple nodes simultaneously.

```
App → writes to multiple nodes at once
App → reads from multiple nodes, picks latest value
```

**Examples:** Apache Cassandra, Amazon DynamoDB

**Best for:** Systems maximizing availability and partition tolerance.

---

## Failover

When the primary fails, a replica is **automatically promoted** to become the new primary.

```
Before:                  After Primary Fails:
  Primary ✅                  Replica 1 ✅
      │                      (New Primary)
  Replica 1                       │
  Replica 2                   Replica 2
```

**Failover steps:**
1. Health check detects primary is down
2. Leader election runs (Raft or Paxos algorithm)
3. Best replica is promoted to primary
4. Write traffic is redirected
5. Other replicas sync from new primary

> Automated failover is built into AWS RDS, MongoDB Atlas, Google Cloud SQL, and most managed databases.

---

## Consistency Models

| Model | Behaviour | Example |
|---|---|---|
| **Strong Consistency** | All reads always see the latest write | Banking, payments |
| **Eventual Consistency** | Replicas catch up over time | Social feeds, likes, views |
| **Read-Your-Writes** | You always see your own recent writes | Profile updates |

---

## CAP Theorem & Replication

Distributed systems can only fully guarantee **two of three** properties:

| Property | Meaning |
|---|---|
| **Consistency (C)** | Every read returns the most recent write |
| **Availability (A)** | Every request receives a response |
| **Partition Tolerance (P)** | System works despite network failures |

- **Synchronous replication** → favors Consistency
- **Asynchronous replication** → favors Availability

---

## Read Scaling with Replicas

**Without replication:**
```
1000 reads/sec → Single DB  ← overwhelmed
```

**With 4 read replicas:**
```
1000 reads/sec
  ├── Replica 1 → 250 reads
  ├── Replica 2 → 250 reads
  ├── Replica 3 → 250 reads
  └── Replica 4 → 250 reads
```

Add more replicas as read traffic grows — scales linearly.

---

## High Availability via Replication

| Scenario | Without Replication | With Replication |
|---|---|---|
| Primary crashes | ❌ Full outage | ✅ Replica takes over |
| Maintenance window | ❌ Downtime | ✅ Replicas serve traffic |
| Region failure | ❌ Data inaccessible | ✅ Other regions take over |

---

## Replication in Real Databases

| Database | Replication Type |
|---|---|
| MySQL | Async primary-replica (binary log) |
| PostgreSQL | Streaming replication (WAL) |
| MongoDB | Replica sets (1 primary + N secondaries) |
| Cassandra | Leaderless, configurable replication factor |
| DynamoDB | Multi-region leaderless |
| Redis | Primary-replica with Sentinel for failover |

**MongoDB Replica Set:**
```
    Primary
       │
  ┌────┴────┐
  S1        S2    ← Secondaries replicate from Primary
```

---

## Replication + Sharding Together

The standard pattern for large-scale production systems:

```
         Shard 1              Shard 2              Shard 3
        /       \            /       \            /       \
   Replica   Replica    Replica   Replica    Replica   Replica
```

| Layer | Purpose |
|---|---|
| Sharding | Distributes writes and storage across servers |
| Replication | Scales reads and provides fault tolerance per shard |

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
Database Router
  │
┌──────────────────────────────┐
│                              │
Primary DB               Read Replicas
(writes)               R1 | R2 | R3
```

---

## Pros & Cons Summary

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Scales read traffic linearly | Replication lag causes stale reads |
| No single point of failure | Increased storage cost |
| Disaster recovery built-in | Failover logic adds complexity |
| Read from geographically nearest node | Write conflicts in multi-leader setups |

---

## Replication Metrics to Monitor

| Metric | What It Tells You |
|---|---|
| **Replication Lag** | How far replicas are behind the primary |
| **Read Throughput** | Reads per second across all replicas |
| **Write Throughput** | Writes per second on the primary |
| **Replica Health** | Are all replicas online and in sync |

---

## Quick Reference Cheat Sheet

| Concept | One-Line Summary |
|---|---|
| Primary | Handles all writes |
| Replica | Serves reads, acts as backup |
| Read Replica | Read-only replica dedicated to SELECT traffic |
| Replication Lag | Delay between primary write and replica update |
| Failover | Replica promoted when primary goes down |
| Leader Election | Raft/Paxos picks the new primary automatically |
| Synchronous | Wait for replica ACK — consistent, slower |
| Asynchronous | Fire and forget — fast, eventual consistency |
| Circular Replication | Rarely used; each node replicates to the next |

---

## One-Line Formula

```
Replication = Multiple Data Copies + Read Scalability + High Availability + Fault Tolerance + Disaster Recovery
```

---

## The Golden Rule for Interviews

```
Replication  →  Scale Reads
Sharding     →  Scale Writes
Both         →  Scale Everything
```
