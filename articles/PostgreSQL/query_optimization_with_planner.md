---
title: "Understanding Query Optimization with the Query Planner in PostgreSQL"
description: "How to read EXPLAIN ANALYZE and understand why PostgreSQL picks a sequential, index, bitmap or parallel scan, with a hands-on example."
date: "2026-10-12"
tags:
  - PostgreSQL
  - Query Optimization
  - Indexes
  - EXPLAIN ANALYZE
---

# Understanding Query Optimization with the Query Planner in PostgreSQL

Query optimization is the process of finding the most efficient way for a database to execute a SQL query. A good way to see how a query performs is to look at the steps the database takes to fetch the data. PostgreSQL exposes these steps through its **query planner/optimizer**, which you can inspect with `EXPLAIN` and `EXPLAIN ANALYZE`.

Take this simple query:

```sql
SELECT COUNT(*)
FROM transactions
WHERE transaction_id = 'trans_id';
```

PostgreSQL can retrieve the matching rows in several ways:

1. **Sequential Scan**: read the entire table.
2. **Index Scan**: use an index on `transaction_id` to jump straight to the row.
3. **Bitmap Index Scan + Bitmap Heap Scan**: use an index to collect the locations of many matching rows, then fetch them in bulk.
4. **Parallel Scan**: split the work among multiple worker processes. This is a modifier that can be applied to the scans above (Parallel Seq Scan, Parallel Index Scan, Parallel Bitmap Heap Scan).

The optimizer estimates the **cost** of each possible approach and picks the one it believes will be fastest. To see this in action, let's build a table and experiment.

## Setting up the data

```sql
CREATE TABLE transactions (
    transaction_id VARCHAR(50) PRIMARY KEY,
    user_id        INTEGER NOT NULL,
    amount         NUMERIC(12, 2) NOT NULL,
    created_on     DATE NOT NULL,
    modified_on    DATE NOT NULL
);
```

Now add 100,000 rows of random data:

```sql
INSERT INTO transactions (transaction_id, user_id, amount, created_on, modified_on)
SELECT
    'TXN-' || LPAD(gs::TEXT, 10, '0'),
    (RANDOM() * 9999 + 1)::INTEGER,
    ROUND((RANDOM() * 100000)::NUMERIC, 2),
    DATE '2025-01-01' + (RANDOM() * 364)::INTEGER,
    DATE '2025-01-01' + (RANDOM() * 364)::INTEGER
FROM generate_series(1, 100000) AS gs;

-- Refresh planner statistics so estimates are accurate
ANALYZE transactions;
```

When a table is created with a primary key, PostgreSQL automatically creates a unique index for it. Verify it with:

```sql
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'transactions';
```

This lists one index, `transactions_pkey`, along with its definition (`CREATE UNIQUE INDEX ... USING btree (transaction_id)`). Note that only `transaction_id` is indexed. Nothing helps us filter by `user_id` or `created_on` yet.

## Reading a Sequential Scan plan

Let's fetch one user's transactions before a given date and see how the planner executes it:

```sql
EXPLAIN ANALYZE
SELECT * FROM transactions
WHERE user_id = 5548 AND created_on < '2025-09-28';
```

A plan like this is produced (your numbers will differ slightly):

```text
Seq Scan on transactions  (cost=0.00..2334.00 rows=7 width=35) (actual time=0.016..5.917 rows=6.00 loops=1)
  Filter: ((created_on < '2025-09-28'::date) AND (user_id = 5548))
  Rows Removed by Filter: 99994
  Buffers: shared hit=834
Planning Time: 0.079 ms
Execution Time: 5.934 ms
```

Let's go through it piece by piece.

### 1. The node and its estimates

No index exists on the columns in the `WHERE` clause, so PostgreSQL falls back to a **sequential scan**, as expected.

```text
cost=0.00..2334.00
      │        │
      │        └── estimated total cost
      └─────────── estimated startup cost
```

- **Startup cost (0.00)**: the work needed before the first row can be returned. A sequential scan can start reading the first page immediately, so this is zero.
- **Total cost (2334.00)**: the optimizer's estimate for the whole node. It is expressed in *abstract cost units*, **not milliseconds**. For a sequential scan it is roughly `pages × seq_page_cost + rows × cpu_tuple_cost + rows × cpu_operator_cost × number_of_filters`. With the default settings that is `834 × 1.0 + 100000 × 0.01 + 100000 × 0.0025 × 2 = 2334`.
- **rows=7**: the planner's *estimate* of how many rows this node will return.
- **width=35**: the estimated average size, in bytes, of each returned row.

```text
actual time=0.016..5.917
            │       │
            │       └── time until the node finished producing all rows (ms)
            └────────── time until the first row was produced (ms)
```

- **rows=6.00**: the *actual* number of rows produced after filtering. Recent PostgreSQL versions print this with decimals because it is an average per loop.
- **loops=1**: the node was executed once.

Notice that the estimate (7) and the actual (6) are close. A big gap between them is one of the most useful signs that statistics are stale (run `ANALYZE`) or that the planner is misjudging the data.

### 2. Filter

```text
Filter: ((created_on < '2025-09-28'::date) AND (user_id = 5548))
```

The scan applies the conditions to every row it reads. `::date` is PostgreSQL's type-cast syntax.

### 3. Rows Removed by Filter

```text
Rows Removed by Filter: 99994
```

The table has 100,000 rows and only 6 matched, so the other 99,994 were read and then discarded. This is the cost of having no index: almost all the work is wasted.

### 4. Buffers

```text
Buffers: shared hit=834
```

PostgreSQL stores table data in **pages** (8 KB each by default). The query touched 834 pages, and all of them were already in PostgreSQL's **shared buffers** (memory), so they are counted as `shared hit`. Pages that are not in the cache must be loaded from disk (or the OS cache) and appear as `shared read`, which is typically much slower. (`Buffers` is shown with `EXPLAIN (ANALYZE, BUFFERS)`, and in newer PostgreSQL versions it is included in `ANALYZE` by default.)

### 5. Planning Time and Execution Time

- **Planning Time: 0.079 ms**: time spent building the execution plan.
- **Execution Time: 5.934 ms**: total time spent running the query.

The execution time is slightly more than the node's finishing time (5.917 ms). The small gap is executor start-up and shutdown overhead.

## Adding an index and comparing

Create a composite index on `user_id` and `created_on`:

```sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_created_on
ON transactions (user_id, created_on);

ANALYZE transactions;
```

> `CONCURRENTLY` builds the index without blocking writes to the table. It is slower and cannot be run inside a transaction block, but it is the safe choice on a live system.

Run the query again:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM transactions
WHERE user_id = 5548 AND created_on < '2025-09-28';
```

The plan now has this shape (read it from the innermost, most indented node outward):

```text
Bitmap Heap Scan on transactions  (...) (actual time=... rows=6 loops=1)
  Recheck Cond: ((user_id = 5548) AND (created_on < '2025-09-28'::date))
  Heap Blocks: exact=...
  Buffers: shared hit=...
  ->  Bitmap Index Scan on idx_users_created_on  (...)
        Index Cond: ((user_id = 5548) AND (created_on < '2025-09-28'::date))
Planning Time: ...
Execution Time: 0.213 ms
```

Compared with the sequential scan:

| | Seq Scan | Index-based plan |
|---|---|---|
| Execution time | 5.934 ms | 0.213 ms |
| Pages touched | 834 (the whole table) | a handful |
| Planning time | lower | slightly higher (more options to evaluate) |
| `shared read` | none (already cached) | none (already cached) |

The planning time grows a little because the optimizer now has more candidate plans to evaluate, but the execution time drops by more than 95%. The exact node PostgreSQL chooses (plain Index Scan or Bitmap scan) depends on how many rows it expects to match, which brings us to the different kinds of index-based scans.

## How a Bitmap scan works

PostgreSQL stores table data in 8 KB pages (also called the **heap**). A table of roughly 100 MB therefore occupies about 12,800 pages. An index stores each indexed key along with a **TID (Tuple ID)** that points to the row's location, conceptually `(page number, tuple position)`. For example, `user_id = 5548 → (120, 5)` means the row lives on heap page 120 at position 5.

- **Bitmap Index Scan** searches the index and builds an in-memory bitmap of the heap locations that match.
- **Bitmap Heap Scan** uses that bitmap to visit only the marked pages and fetch the rows.

Suppose `user_id = 5548` matches row 5 on page 120 and row 2 on page 125. The bitmap records `(120,5)` and `(125,2)`. The Bitmap Heap Scan then goes directly to pages 120 and 125 instead of reading all 12,800 pages. Because the bitmap is sorted by page, each page is also read only once, in physical order, which is friendlier to the disk than random jumps.

In short:
1. PostgreSQL scans the index and marks the matching locations in a bitmap.
2. It visits the table and reads the marked pages in bulk.
3. It filters out non-matching rows within those pages.
4. It **rechecks** the condition (`Recheck Cond`). If the bitmap grows too large for `work_mem`, it becomes *lossy* (tracking whole pages instead of exact rows), and the recheck is what guarantees correctness.

Bitmaps can also be **combined**. If you had two separate single-column indexes, one on `user_id` and one on `created_on`, PostgreSQL could build one bitmap from each, `AND` them together, and fetch only the rows in the result (`BitmapAnd`). `OR` conditions work the same way with `BitmapOr`. In our example a single composite index already covers both conditions, so one bitmap is enough.

## Index Scan vs. Bitmap Index Scan vs. Index-Only Scan

**Index Scan** is the best choice when only a small number of rows match. PostgreSQL walks the index and fetches each matching row from the heap one at a time. For example, looking up one transaction by its primary key narrows the result to a single row:

```sql
SELECT * FROM transactions WHERE transaction_id = 'TXN-0000100000';
```

**Index-Only Scan** is a special case. If *every* column the query needs is stored in the index, PostgreSQL does not need to touch the heap at all (as long as the table's visibility map is up to date, which `VACUUM` maintains). This is usually the fastest option.

**Bitmap Index Scan** is used when the query matches more rows than a plain Index Scan handles well, but far fewer than the whole table. A query like this one, which matches many rows for a user, is a typical candidate:

```sql
SELECT transaction_id, user_id, amount
FROM transactions
WHERE user_id = 2413 AND created_on > '2025-01-01';
```

The rule of thumb: few rows → Index Scan, a moderate number → Bitmap Scan, a large share of the table → Sequential Scan.

## Covering indexes with non-key columns (`INCLUDE`)

Non-key columns are extra fields stored in the index leaf pages. They are **not** used for searching or sorting, only to avoid heap lookups. This reduces I/O by letting the query be answered from the index alone:

```sql
CREATE INDEX user_transaction_amount_idx
ON transactions (user_id) INCLUDE (amount);
```

A query like this one can now be answered with an Index-Only Scan:

```sql
SELECT user_id, amount FROM transactions WHERE user_id = 1244;
```

The same applies to aggregates, such as the total spent by a user (`SELECT SUM(amount) ... WHERE user_id = 1244`), which can be computed without visiting the table. The trade-off is a larger index and slightly slower writes, so include only the columns you really need.

## Parallel Scan

PostgreSQL can use multiple worker processes to scan different portions of a table or index at the same time instead of having a single process do all the work. It considers this only when the query is expensive enough that the expected gain outweighs the overhead of:

- starting and using worker processes,
- dividing the work,
- coordinating the workers, and
- collecting their results.

That is why our 100,000-row table (about 6.5 MB, under the default `min_parallel_table_scan_size` of 8 MB) was scanned by a single process. Parallel scans pay off on large tables, typically in analytical queries that read a big portion of the data. They are rarely useful for small, selective lookups.

## Conclusion

`EXPLAIN ANALYZE` turns the query planner from a black box into something you can reason about. Compare the **estimated** rows with the **actual** rows, check `Rows Removed by Filter` and `Buffers` to see how much work was wasted, and treat `cost` as a relative number, never as time. In our experiment, a single composite index took the query from scanning all 834 pages in about 5.9 ms to touching only a few pages in about 0.2 ms.

No scan strategy is best in every situation. The planner chooses based on how many rows it expects to match and on the indexes available:

| Strategy | Best when | Strength | Weakness |
|---|---|---|---|
| **Sequential Scan** | A large share of the table is needed, or the table is small | No index needed; fast sequential I/O | Reads everything, even if few rows match |
| **Index Scan** | Very few rows match (e.g. primary key lookup) | Direct jump to rows; supports ordered results | Random heap access gets expensive as matches grow |
| **Index-Only Scan** | All needed columns are in the index (e.g. via `INCLUDE`) | Skips the heap entirely | Needs a larger index and an up-to-date visibility map |
| **Bitmap Index + Heap Scan** | A moderate number of rows match, or several indexes must be combined | Reads each page once, in order; combines conditions with `AND`/`OR` | Does not return rows in index order; has a bitmap build step |
| **Parallel Scan** | Large tables and expensive queries | Uses multiple CPUs | Start-up and coordination overhead; wasteful for small queries |

Keep in mind that adding an index is a trade-off: every index uses disk space and slows down inserts and updates. Index the columns your real queries filter on, keep statistics fresh with `ANALYZE`, and always verify with `EXPLAIN ANALYZE` instead of guessing.