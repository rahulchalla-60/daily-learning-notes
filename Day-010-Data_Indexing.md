# Day 010 — Data Indexing

---

## What is an Index?

An index is a **separate data structure that helps the database find rows quickly** without scanning every row in the table.

**Analogy:** A book's index at the back.
```
Without index:  Flip every page looking for "Caching"... page 1, 2, 3... 1000
With index:     Caching → Page 542  (jump directly)
```

Database indexing works exactly the same way.

---

## Why Indexing Matters

Imagine a Users table with 10 million rows:

```sql
SELECT * FROM users WHERE id = 5000000;
```

**Without index — Full Table Scan:**
```
Check row 1... row 2... row 3... → row 5,000,000
Time complexity: O(n)
```

**With index — Direct Lookup:**
```
Find 5,000,000 in index → jump to row
Time complexity: O(log n)
```

At 10M rows, the difference is thousands of row checks vs ~20 comparisons.

---

## Does the Database Create Indexes Automatically?

| Column Type | Auto-Indexed? |
|---|---|
| Primary Key | ✅ Always |
| UNIQUE constraint | ✅ Usually |
| Foreign Keys | ⚠️ Depends on DB |
| Regular columns (`name`, `city`) | ❌ Never — you must create manually |

```sql
-- Auto-indexed (primary key)
CREATE TABLE users (
  id INT PRIMARY KEY,   -- index created automatically
  name VARCHAR(50)      -- no index
);

-- Manual index
CREATE INDEX idx_name ON users(name);
```

---

## How Indexes Work Internally

### The Data Structure: B-Tree (and B+ Tree)

Most databases (MySQL InnoDB, PostgreSQL, Oracle, SQL Server) use **B+ Trees** as the index structure.

**Why not a simple sorted array?**
Inserting into a sorted array at 10M records requires shifting millions of elements. B-Trees solve this.

**B-Tree search example:**
```
Find value 70:

              [50]
            /      \
         [20]      [80]
        /    \    /    \
     [10]  [30] [70]  [90]

50 → go right → 80 → go left → 70 ✅
Only 3 comparisons for millions of records
```

**Why B-Trees are fast on disk:**

| Tree Type | Records | Height | Disk Reads |
|---|---|---|---|
| Binary Tree | 1 million | ~20 levels | ~20 reads |
| B-Tree | 1 million | ~3–4 levels | ~3–4 reads |

Disk access is slow — fewer levels = fewer disk reads = much faster.

---

### B+ Tree — What Most Databases Actually Use

B+ Trees add one critical feature: **leaf nodes are linked together**.

```
           [30, 60]
          /    |    \
       [10]  [40]  [80]

Leaf layer (all data, linked):
10 ↔ 20 ↔ 30 ↔ 40 ↔ 50 ↔ 60 ↔ 80
```

This makes **range queries** extremely efficient:

```sql
SELECT * FROM users WHERE age BETWEEN 20 AND 30;
-- Index finds 20, then follows linked leaves: 21, 22, 23... 30
-- No jumping around — sequential traversal
```

---

## Types of Indexes

### Clustered Index

The table's actual data rows are **physically stored in index order**.

```
Index key → actual data (they're the same thing)
```

- Usually the Primary Key
- Only **one** per table (data can only be sorted one way)
- Fastest for primary key lookups

### Non-Clustered Index

A **separate structure** that stores the index key + a pointer to the actual row.

```
Index key → pointer → actual row data
```

- Can have **many** per table
- Slightly slower than clustered (one extra lookup)
- Used for all columns other than the primary key

---

## Composite Indexes

When queries filter on multiple columns, a composite index covers both:

```sql
-- Query filtering on city AND age
SELECT * FROM users WHERE city = 'Hyderabad' AND age = 25;

-- Bad: two separate indexes
CREATE INDEX idx_city ON users(city);
CREATE INDEX idx_age  ON users(age);

-- Good: one composite index
CREATE INDEX idx_city_age ON users(city, age);
```

### The Leftmost Prefix Rule

A composite index on `(city, age)` only helps queries that **start with the leftmost column**:

```sql
WHERE city = 'Hyderabad'              ✅ uses index
WHERE city = 'Hyderabad' AND age = 25 ✅ uses index
WHERE age = 25                        ❌ index NOT used (city is first)
```

> Design composite indexes to match your most common query patterns, leftmost column first.

---

## Query Optimization With Indexes

**Point lookup:**
```sql
SELECT * FROM users WHERE email = 'abc@gmail.com';

Without index: O(n) — scan every row
With index:    O(log n) — search index tree
```

**Sorted results:**
```sql
SELECT * FROM users ORDER BY name;

Without index: read all rows → sort in memory → return
With index:    index is already sorted → traverse directly
```

**JOINs:**
```sql
SELECT * FROM orders JOIN users ON orders.user_id = users.id;

Without index on user_id: nested loop scan (very slow)
With index on user_id:    fast index lookup per join
```

---

## The Cost of Indexes

Indexes make reads faster but have real costs on writes:

### Storage
```
Users table:  100 GB
Indexes:       30 GB
Total:        130 GB
```

### Write Overhead

Every INSERT, UPDATE, or DELETE now has extra work:

```
INSERT a row:
  1. Insert data row
  2. Insert index entry
  3. Rebalance B-Tree

UPDATE an indexed column:
  1. Update data row
  2. Remove old index entry
  3. Insert new index entry
  4. Rebalance

DELETE a row:
  1. Delete data row
  2. Delete index entry
  3. Rebalance
```

More indexes = slower writes. This is the core trade-off.

---

## When to Create an Index

| ✅ Create index on | Reason |
|---|---|
| Primary keys | Automatically done, essential |
| Foreign keys | Speeds up JOINs |
| Columns in `WHERE` clauses | `WHERE email = ?` |
| Columns in `ORDER BY` | Avoids in-memory sort |
| Columns used in JOINs | Critical for JOIN performance |

## When NOT to Create an Index

| ❌ Avoid indexing | Reason |
|---|---|
| Small tables (< few thousand rows) | Full scan is cheaper than index overhead |
| Columns updated very frequently | `last_seen_at` updated every second = constant rebalancing |
| Low-cardinality columns | `gender` has only 2 values — index barely helps |
| Write-heavy tables | Each write hits every index |

---

## System Design Perspective

```
100 rows      → indexes optional
1,000 rows    → indexes helpful
1 million rows  → indexes important
100 million rows → indexes critical
```

Without indexes at scale:
- Query latency spikes
- CPU and disk I/O spike
- Database becomes the bottleneck
- No amount of caching fully compensates

---

## Summary — Benefits vs Costs

| Benefit (Read side) | Cost (Write side) |
|---|---|
| Fast point lookups O(log n) | Extra storage per index |
| Fast range queries | Slower INSERT |
| Fast ORDER BY | Slower UPDATE |
| Fast JOINs | Slower DELETE |

> **The trade-off is almost always worth it for read-heavy systems.** Most large systems (e-commerce, social media, search) do far more reads than writes — trading a bit of write performance for massive read gains is the right call.

---

## Quick Reference Cheat Sheet

| Concept | Summary |
|---|---|
| Index | Separate structure for fast row lookup |
| Full Table Scan | O(n) — checks every row, no index |
| B+ Tree | Most common index structure, O(log n) search |
| Clustered Index | Data stored in index order (usually primary key) |
| Non-Clustered Index | Separate structure with pointer to data |
| Composite Index | Index on multiple columns |
| Leftmost Prefix Rule | Composite index only works if query starts with leftmost column |
| Cardinality | Number of unique values — high cardinality = good index candidate |

---

## One-Line Formula

```
Index = Fast Reads (O log n) + Extra Storage + Slower Writes
     → Worth it for read-heavy systems at scale
```
