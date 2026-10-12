---
title: "Join Optimization in PostgreSQL: Nested Loop, Hash Join, and Merge Join"
description: "Understand how PostgreSQL chooses join algorithms, read EXPLAIN ANALYZE plans, and identify practical opportunities to improve join performance."
date: "2026-10-12"
tags:
  - PostgreSQL
  - Query Optimization
  - Performance
  - EXPLAIN ANALYZE
---

# Join Optimization in PostgreSQL

When we think about joins, we often think first about logical join types such as `INNER JOIN`, `LEFT JOIN`, `RIGHT JOIN`, `FULL JOIN`, and `CROSS JOIN`. These describe **which rows belong in the result**. Under the hood, PostgreSQL must also choose a physical algorithm to produce that result. Its main join algorithms are **Nested Loop Join**, **Hash Join**, and **Merge Join**.

The query planner estimates the cost of possible execution plans and chooses the plan it expects to be cheapest. The choice depends on factors such as table sizes, filter selectivity, available indexes, the order of input rows, statistics, memory settings, and the columns being joined. An index does not guarantee an index scan or a particular join method.

This article uses `customers` and `transactions` tables to demonstrate the three algorithms and how to read their plans. Exact plans and timings can differ between PostgreSQL versions, data distributions, hardware, statistics, and configuration.

## 1. Prepare the practice tables

The following script creates 10,000 customers and 1.8 million transactions. Run it in a test database. It assumes these tables do not already exist; the `DROP TABLE` statements remove existing tables with these names, so omit them if you need to preserve your data.

```sql
DROP TABLE IF EXISTS transactions;
DROP TABLE IF EXISTS customers;

CREATE TABLE customers (
    user_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL
);

INSERT INTO customers (customer_name)
SELECT 'Customer ' || gs
FROM generate_series(1, 10000) AS gs;

CREATE TABLE transactions (
    transaction_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL,
    amount NUMERIC(12, 2) NOT NULL,
    created_on TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    modified_on TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_transactions_customer
        FOREIGN KEY (user_id)
        REFERENCES customers (user_id)
);

INSERT INTO transactions (user_id, amount, created_on, modified_on)
SELECT
    (floor(random() * 10000) + 1)::BIGINT,
    round((random() * 100000)::NUMERIC, 2),
    TIMESTAMP '2025-01-01' + random() * INTERVAL '364 days',
    TIMESTAMP '2025-01-01' + random() * INTERVAL '364 days'
FROM generate_series(1, 1800000);

ANALYZE customers;
ANALYZE transactions;
```

The foreign key ensures every transaction references an existing customer. The `customers.user_id` primary key automatically creates a unique B-tree index. At this point, there is no separate index on `transactions.user_id`.

## 2. Start with a Hash Join

Run the following query:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.customer_name, t.amount
FROM transactions AS t
JOIN customers AS c
    ON c.user_id = t.user_id;
```

A typical plan for data of this shape may resemble:

```text
Hash Join  (cost=289.00..38017.00 rows=1800000 width=21)
           (actual time=2.140..409.871 rows=1800000 loops=1)
  Hash Cond: (t.user_id = c.user_id)
  ->  Seq Scan on transactions t
        (cost=0.00..33001.00 rows=1800000 width=12)
        (actual time=0.014..75.907 rows=1800000 loops=1)
  ->  Hash
        (cost=164.00..164.00 rows=10000 width=21)
        (actual time=2.112..2.114 rows=10000 loops=1)
        Buckets: 16384  Batches: 1  Memory Usage: 675kB
        ->  Seq Scan on customers c
              (cost=0.00..164.00 rows=10000 width=21)
              (actual time=0.007..0.866 rows=10000 loops=1)
Planning Time: 0.207 ms
Execution Time: 460.712 ms
```

These values are an example from a particular run, not expected constants. Read the plan as a tree: scan nodes supply rows, and nodes above them perform operations such as hashing and joining. The executor's exact work order depends on the node. For a Hash Join, PostgreSQL first builds the hash table from the `Hash` child, then scans the other input and probes that table.

### How the Hash Join executes

1. **Scan `customers`.** PostgreSQL reads the 10,000 customer rows. The estimated and actual row counts match in this example.
2. **Build the hash table.** PostgreSQL hashes the customer join keys (`user_id`) into an in-memory hash table. It must also retain enough information from each row to return the requested customer columns.
3. **Scan `transactions`.** PostgreSQL reads the 1.8 million transaction rows.
4. **Probe and match.** For each transaction, PostgreSQL hashes `t.user_id`, locates the relevant bucket, checks actual key equality, and emits the joined row when a match is found.

A Hash Join is often a good choice for large equality joins when scanning the inputs and building a hash table is cheaper than performing many index lookups or sorting both inputs. It does not require inputs to arrive pre-sorted.

### Understanding `Buckets`, `Batches`, and `Memory Usage`

```text
Buckets: 16384  Batches: 1  Memory Usage: 675kB
```

- **Buckets** are slots in the hash table used to organize hash entries. Hash collisions are possible, so PostgreSQL also checks the actual join key for equality.
- **Batches** indicate how many batches are used to process the hash join. `Batches: 1` means the hash table fit in memory for this execution. More than one batch can mean PostgreSQL had to partition the hash work; batches may involve temporary-file I/O.
- **Memory Usage** reports the memory used by the hash table, approximately 675 KB in this example.

You can experiment with a smaller `work_mem` setting:

```sql
SHOW work_mem;
SET work_mem = '64kB';

EXPLAIN (ANALYZE, BUFFERS)
SELECT c.customer_name, t.amount
FROM transactions AS t
JOIN customers AS c
    ON c.user_id = t.user_id;

RESET work_mem;
```

With less memory available, PostgreSQL may use more batches and temporary I/O, which can make the query slower. The exact bucket and batch counts vary by data and plan. `work_mem` is a per-operation limit rather than a single memory budget shared by the whole server, so changing it globally without workload testing can increase total memory consumption under concurrency.

### Why sequential scans are reasonable here

The query returns essentially every transaction, so reading the transaction table through a sequential scan can be cheaper than looking up a large fraction of its rows through an index and then fetching heap pages. Although `customers.user_id` has a primary-key index, PostgreSQL can reasonably choose a sequential scan of `customers` because it needs all customer rows to build the hash table.

Do not add an index solely because a column appears in a join condition. First check whether the query filters out most rows, whether the index can reduce heap access, and whether the planner already has an efficient plan.

## 3. Merge Join: processing ordered inputs

A Merge Join needs its inputs ordered by the merge join key. PostgreSQL can obtain this order from a suitable B-tree index or by adding explicit `Sort` nodes. Once both inputs are ordered, the algorithm advances through them, matching equal keys and moving past keys that cannot match the current row. It can handle duplicate keys and return all matching combinations.

The `customers.user_id` primary-key index can provide rows in `user_id` order. Add an index to the transaction table:

```sql
CREATE INDEX idx_transactions_user_id
ON transactions (user_id);

ANALYZE transactions;
```

Then inspect a join whose output is ordered by the join key:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.customer_name, t.amount
FROM customers AS c
JOIN transactions AS t
    ON c.user_id = t.user_id
ORDER BY c.user_id;
```

A possible Merge Join plan is:

```text
Merge Join
  Merge Cond: (c.user_id = t.user_id)
  ->  Index Scan using customers_pkey on customers c
  ->  Index Scan using idx_transactions_user_id on transactions t
```

This is an illustration, not a guarantee. Because the query reads all transactions, PostgreSQL might prefer a Hash Join, sequential scans, and a separate sort instead. Maintaining an index also has storage and write costs.

To study a Merge Join specifically, you can temporarily discourage the other join methods in your own test session:

```sql
SET enable_hashjoin = off;
SET enable_nestloop = off;

EXPLAIN (ANALYZE, BUFFERS)
SELECT c.customer_name, t.amount
FROM customers AS c
JOIN transactions AS t
    ON c.user_id = t.user_id
ORDER BY c.user_id;

RESET enable_hashjoin;
RESET enable_nestloop;
```

These settings are for learning and diagnosis, not a production tuning shortcut. Inspect the resulting plan to see whether PostgreSQL uses index scans, explicit sorts, or both to supply ordered inputs.

Merge Join is often attractive when both sides are already ordered, when indexes can supply the order efficiently, or when the result also needs that ordering. It is not automatically best for large joins: sorting large inputs can consume CPU, memory, and temporary disk space. It is also inaccurate to say that Merge Join is simply the preferred method for range predicates; PostgreSQL must have a supported merge-join condition, and the planner compares costs for the actual query.

## 4. Nested Loop Join: a small outer result and efficient lookups

A Nested Loop takes each row from its **outer** input and searches the **inner** input for matching rows. If the inner side has a useful index, PostgreSQL can perform a targeted lookup for each outer row. Without an efficient inner access path, repeated inner scans can become expensive.

The index created above makes the following query a good candidate for a Nested Loop:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.user_id, t.amount
FROM customers AS c
JOIN transactions AS t
    ON c.user_id = t.user_id
WHERE c.user_id = 5548;
```

A possible plan is:

```text
Nested Loop
  ->  Index Only Scan using customers_pkey on customers c
        Index Cond: (user_id = 5548)
  ->  Bitmap Heap Scan on transactions t
        Recheck Cond: (user_id = 5548)
        ->  Bitmap Index Scan using idx_transactions_user_id
              Index Cond: (user_id = 5548)
```

The execution flow is:

1. The outer index-only scan looks up customer `5548`. An index-only scan can avoid heap visits when the visibility map confirms that the relevant heap pages are all-visible; `Heap Fetches: 0` is possible but not guaranteed.
2. For that customer row, the inner bitmap index scan finds index entries for transactions with `user_id = 5548`.
3. The bitmap heap scan fetches those transaction rows from the table heap. It may be preferred over a plain index scan when several matching rows or heap pages are involved.
4. The Nested Loop combines the customer row with each matching transaction row and returns the requested columns.

For the example plan analyzed during this lab, PostgreSQL estimated 180 matching transactions but returned 162. The query finished in approximately 0.303 ms in that run. Timings vary with cache state, storage, and other workload activity.

Nested Loop is often effective when a filter reduces the outer side to a small number of rows and the inner side can be searched efficiently with an index. It can be a poor choice when the outer side contains many rows and each iteration scans a large inner relation. In `EXPLAIN ANALYZE`, check `loops` on the inner node: a high loop count can indicate repeated work. The `actual rows` value is generally an average per loop when `loops` is greater than one.

## 5. How to read `EXPLAIN (ANALYZE, BUFFERS)`

Use this form while comparing alternatives:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

Important fields:

| Field | Meaning |
|---|---|
| `cost=startup..total` | Planner estimates in cost units, not milliseconds. The total cost includes the estimated cost of child nodes. |
| `rows` | Estimated rows emitted by the node, not necessarily the number of rows it scans. |
| `width` | Estimated average output-row width in bytes. |
| `actual time=start..end` | Measured milliseconds to produce the first row and finish producing rows for that node. Node times include child work; do not add every node's time together. |
| `actual rows` | Actual rows emitted. Compare with the estimate to find cardinality-estimation errors. |
| `loops` | Number of times the node ran. Interpret `actual rows` together with this value. |
| `Buffers: shared hit` | Requested shared-buffer pages were already in PostgreSQL's shared buffers. |
| `Buffers: shared read` | Pages had to be read into PostgreSQL shared buffers. This does not necessarily mean a physical disk read, because the operating system may have cached the data. |
| `Heap Fetches` | Heap visits made during an index-only scan because visibility could not be confirmed from the visibility map alone. |
| `Batches` | For a hash operation, the number of batches used. More than one can indicate the hash operation did not fit in memory as a single batch. |
| `Planning Time` / `Execution Time` | Time spent planning and executing the statement, respectively. |

Plan trees are usually easiest to understand from the leaf scan nodes upward. However, the executor does not blindly run every line in simple bottom-to-top order. For example, a Hash Join builds its hash side before scanning the other input; a Nested Loop executes the outer branch first and then evaluates the inner branch for each outer row.

## 6. A practical join-optimization workflow

1. **Measure first:** capture `EXPLAIN (ANALYZE, BUFFERS)` for the real query and representative parameters.
2. **Check row estimates:** large gaps between estimated and actual rows can lead the planner toward the wrong strategy. Run `ANALYZE` after significant data changes and investigate skew or correlated columns if estimates remain poor.
3. **Find where rows are reduced:** selective filters applied early can reduce the amount of data that enters the join. PostgreSQL normally pushes safe filters down where appropriate.
4. **Check index usefulness:** indexes are especially valuable for selective filters and Nested Loop inner lookups. They are not automatically beneficial when most of a table must be read.
5. **Inspect repeated work:** examine `loops`, buffer activity, and the cost of inner scans in Nested Loop plans.
6. **Inspect sorting and spilling:** for Merge Join, check whether large inputs are sorted; for Hash Join, check `Batches` and temporary I/O.
7. **Change one thing at a time:** test a justified index or statistics change, run the same query again, and compare elapsed time, buffers, rows, and plan shape. Avoid relying on a single run because caching and system load affect timings.

### Quick comparison

| Join method | Often useful when | Main risk |
|---|---|---|
| Nested Loop | The outer input is small and the inner lookup is cheap, often through an index | Repeated inner scans or lookups become expensive when the outer result is large |
| Hash Join | Large equality joins with unsorted inputs and a hash table that is reasonably sized | A large hash table can require multiple batches and temporary I/O |
| Merge Join | Both inputs can be provided in join-key order at a reasonable cost | Sorting large inputs can be expensive; indexes may add write and storage overhead |

## Conclusion

PostgreSQL does not choose a join method simply because a table is small, an index exists, or the input is sorted. It estimates the cost of the available plans and selects the strategy it expects to perform best. Nested Loop is often effective for a small outer result with efficient inner lookups; Hash Join is often effective for large equality joins over unsorted data; and Merge Join can be attractive when both inputs can be read in join-key order without excessive sorting.

The best optimization is therefore not to force a particular join algorithm, but to understand the work represented by the plan. Use `EXPLAIN (ANALYZE, BUFFERS)` to compare estimated and actual rows, repeated loops, memory use, buffer activity, and execution time. Make a change only when the measurements point to a specific problem, then test again with representative data and workload conditions.
