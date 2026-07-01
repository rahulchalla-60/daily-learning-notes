# Day 015 — Instagram News Feed: Capacity Estimation

---

## Why Estimate Capacity?

Before designing any system, you need concrete numbers to answer:

- How many servers do we need?
- How much storage is required?
- What are the hardware specs (CPU, RAM, SSD, Network)?
- Is the system read-heavy or write-heavy?
- How do we scale in the future?

> **Goal:** Build infrastructure that handles expected traffic without over-provisioning (wasting money) or under-provisioning (causing outages).

---

## Step 1 — Establish User Base

| Metric | Value |
|---|---|
| MAU (Monthly Active Users) | 2 Billion |
| DAU (Daily Active Users) | 500 Million |

> DAU is the foundation for almost every other calculation. All estimates below derive from it.

---

## Step 2 — Identify Traffic Pattern

**Is Instagram read-heavy or write-heavy?**

```
Feed views (reads):   Billions per day
Post uploads (writes): Millions per day
```

**Instagram is heavily read-heavy** — roughly 1000:1 read-to-write ratio.

This tells us to invest in:
- Read replicas for the database
- Aggressive caching (Redis)
- CDN for media delivery

---

## Step 3 — Storage Estimation

### Daily Post Volume

```
DAU                          = 500 Million
Users who upload a post/day  = 10% of DAU
Posts per day                = 500M × 10% = 50 Million posts/day
```

### Post Type Breakdown

| Post Type | Share | Average Size |
|---|---|---|
| Image | 70% | 2 MB |
| Carousel | 10% | 8 MB |
| Video | 20% | 15 MB |

### Weighted Average Post Size

```
Average size = (70% × 2MB) + (10% × 8MB) + (20% × 15MB)
             = 1.4 + 0.8 + 3.0
             = 5.2 MB per post
```

### Storage Required

| Timeframe | Calculation | Result |
|---|---|---|
| Per day | 50M posts × 5.2 MB | **~260 TB/day** |
| Per year | 260 TB × 365 | **~95 PB/year** |
| 10 years | 95 PB × 10 | **~0.95 EB** |

> Actual storage is lower after compression and content expiration policies, but this gives the upper bound for planning.

---

## Step 4 — Cache (Memory) Estimation

We don't cache everything — only the hot data (popular feeds, trending posts, active user profiles).

```
Daily media generated  = 260 TB
Cache 1% of that       = 260 TB × 1% = ~2.6 TB
```

**Cache requirement: ~2–3 TB**, distributed across a Redis cluster.

This covers the most-viewed ~1% of content which satisfies the majority of read requests (Pareto principle — 1% of content gets ~80% of views).

---

## Step 5 — Network / Bandwidth Estimation

### Ingress (Uploads — data coming IN)

```
Posts uploaded/day  = 50 Million
Average post size   = 5.2 MB
Daily ingress       = 50M × 5.2 MB = ~260 TB/day
Ingress per second  = 260 TB / 86,400 ≈ 3 GB/sec
```

### Egress (Feed Views — data going OUT)

```
DAU                        = 500 Million
Average posts viewed/day   = 100 per user
Average post size          = 5.2 MB
Daily egress               = 500M × 100 × 5.2 MB = ~260 PB/day
Egress per second          = 260 PB / 86,400 ≈ 3 TB/sec
```

### Read vs Write Traffic

| Traffic Type | Volume | Per Second |
|---|---|---|
| **Ingress** (uploads) | ~260 TB/day | ~3 GB/sec |
| **Egress** (feed views) | ~260 PB/day | ~3 TB/sec |
| **Ratio** | Read is ~1000× write | |

This is why CDN is non-negotiable — no single origin can serve 3 TB/sec.

---

## Step 6 — QPS Estimation

```
Feed requests/day  = DAU × feed refreshes per user
                   = 500M × 10 = 5 Billion/day
Average QPS        = 5B / 86,400 ≈ 57,870 QPS
Peak QPS           = ~3× average ≈ 175,000 QPS
```

> Always design for peak QPS, not average. Lunch hours, events, and viral content cause 3–5× spikes.

---

## All Numbers at a Glance

| Metric | Value |
|---|---|
| DAU | 500 Million |
| Posts uploaded/day | 50 Million |
| Average post size | 5.2 MB |
| Daily storage (ingress) | ~260 TB/day |
| 10-year storage | ~0.95 EB |
| Cache needed | ~2–3 TB |
| Daily feed traffic (egress) | ~260 PB/day |
| Egress bandwidth | ~3 TB/sec |
| Average QPS | ~58,000 |
| Peak QPS | ~175,000 |

---

## What These Numbers Tell Us

| Finding | Architecture Decision |
|---|---|
| 260 TB/day uploads | Object storage (S3) + compression |
| 260 PB/day egress | CDN is mandatory |
| 1000:1 read/write ratio | Read replicas + caching |
| ~3 TB/sec egress | Multi-region CDN deployment |
| 175K peak QPS | Horizontal scaling + load balancer |
| 0.95 EB over 10 years | Tiered storage + archival policies |

---

## Estimation Flow (Interview Walkthrough)

```
1. State DAU / MAU
        ↓
2. Identify read-heavy vs write-heavy
        ↓
3. Estimate daily posts (DAU × upload rate)
        ↓
4. Calculate average post size (weighted average)
        ↓
5. Daily storage = posts/day × avg size
        ↓
6. Project to 1 year and 10 years
        ↓
7. Cache = 1% of daily data
        ↓
8. Ingress = same as daily storage
        ↓
9. Egress = DAU × posts viewed × avg size
        ↓
10. QPS = requests/day ÷ 86,400  (then × 3 for peak)
        ↓
11. Draw architecture conclusions
```

---

## Key Rules to Remember

| Rule | Why It Matters |
|---|---|
| DAU drives all calculations | Everything derives from daily active users |
| Peak = 3–5× average QPS | Design for peak, not average |
| Egress >> Ingress | One upload is viewed thousands of times |
| Cache 1% of content | Covers ~80% of reads (hot content) |
| 86,400 seconds/day | Used to convert daily numbers to per-second rates |
| Plan for 10× growth | Avoid redesigning the system next year |
