---
title: "Bloom Indexes in PostgreSQL"
description: "How bloom filters work, and how PostgreSQL uses them to index many columns with a single small index, with worked bit-level examples and a hands-on benchmark."
date: "2026-10-29"
tags:
  - PostgreSQL
  - Indexing
  - Bloom Filter
  - Query Optimization
---

# Bloom Indexes in PostgreSQL

A B-tree index is excellent when we know exactly which columns our queries filter on. But what if queries can filter on *any combination* of many columns? Creating an index for every combination quickly becomes impractical. A **bloom index** takes a different approach: one compact index over many columns that can quickly rule out most non-matching rows. In this article we'll start from the basic idea of a bloom filter, see how PostgreSQL applies it, and then test it on a 10-million-row table.

## What is a bloom filter?

A bloom filter is a space-efficient data structure that answers one question: *is this value in the set?* It gives one of two answers:

- **"Definitely not present."** This answer is always correct.
- **"Possibly present."** The value might be in the set, or it might be a **false positive**, so it must be verified against the real data.

A bloom filter never produces false negatives. It trades a small chance of false positives for a very small memory footprint, which is what makes it useful on large datasets: most lookups for missing values can be answered without touching the actual data.

### How it works

A bloom filter is an array of bits (all `0` at the start) plus a few hash functions:

- **Adding a value:** run it through each hash function and set the resulting bit positions to `1`.
- **Checking a value:** compute the same positions. If **any** bit is `0`, the value is definitely absent. If **all** bits are `1`, it is possibly present.

### A real-world analogy

Consider a sign-up form that must check whether an email address is already taken. A bloom filter kept in front of the user database can answer instantly for most new addresses: if it says "definitely not present", the address is free and no database lookup is needed. Only a "possibly present" answer requires the slower, authoritative check against the real data. (This is an application-level use of bloom filters, shown here only to build intuition. PostgreSQL's bloom *index*, covered next, applies the same principle to table rows.)

### A concrete example

Let's keep a filter of known `user_id` values using a 16-bit array and 2 hash functions. (The bit positions below are made up for illustration; a real hash function would produce them.)

| Value added | Bits set |
|---|---|
| `101` | 1, 9 |
| `205` | 2, 9 |
| `309` | 3, 10 |

After adding all three, the array looks like this:

```text
position:  0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
bit:       0 1 1 1 0 0 0 0 0 1  1  0  0  0  0  0
```

Now let's look up some values:

| Lookup | Bits to check | Result |
|---|---|---|
| `5548` | 2, 11 | Bit 11 is `0`, so **definitely absent** |
| `205` | 2, 9 | Both are `1`, so **possibly present** (and it really is) |
| `777` | 1, 10 | Both are `1`, so **possibly present**, but `777` was never added. This is a **false positive** |

Bits are only ever set, never cleared, which is why a bloom filter can say "definitely not" but never "definitely yes".

## Bloom indexes in PostgreSQL

PostgreSQL ships a **bloom** index access method as the `bloom` extension (in the standard `contrib` modules). Instead of keeping one bloom filter for a whole set, it stores one small **signature** per table row.

- A **signature** is a fixed-length bit string built from the hashed values of all indexed columns of that row.
- Each index entry is simply `signature + location of the row in the table`.
- Unlike a B-tree, there is no sorted structure. To answer a query, PostgreSQL **reads through all the signatures** and keeps the rows whose signature could match.

Because a signature is a lossy summary, it can report false positives. So PostgreSQL always **rechecks** candidate rows against the actual column values in the table (the heap). This is why bloom index plans always use a **Bitmap Index Scan** followed by a **Bitmap Heap Scan** with a `Recheck Cond`: bloom indexes only support bitmap scans.

The signature length is a trade-off. A longer signature means fewer false positives and therefore fewer wasted heap visits, but a bigger index that takes longer to scan.

## The problem bloom indexes solve

Bloom indexes are useful when a table has **many columns** and queries filter on **arbitrary combinations** of them. Take a `transactions` table:

| transaction_id | user_id | country_code | status | payment_method | amount |
|---|---|---|---|---|---|
| TXN001 | 101 | NP | SUCCESS | WALLET | 500 |
| TXN002 | 205 | IN | FAILED | CARD | 1200 |
| TXN003 | 101 | NP | SUCCESS | CARD | 800 |
| TXN004 | 309 | US | PENDING | WALLET | 300 |

And these queries:

```sql
-- Query 1
SELECT * FROM transactions WHERE user_id = 101;

-- Query 2
SELECT * FROM transactions WHERE country_code = 'NP';

-- Query 3
SELECT * FROM transactions WHERE status = 'SUCCESS';

-- Query 4
SELECT * FROM transactions WHERE country_code = 'NP' AND status = 'SUCCESS';

-- Query 5
SELECT * FROM transactions WHERE user_id = 101 AND payment_method = 'WALLET';
```

With B-trees, one index per column helps the single-column queries:

```sql
CREATE INDEX idx_transactions_user_id ON transactions (user_id);
CREATE INDEX idx_transactions_country ON transactions (country_code);
CREATE INDEX idx_transactions_status  ON transactions (status);
CREATE INDEX idx_transactions_payment ON transactions (payment_method);
```

For the multi-column queries, composite indexes work best:

```sql
CREATE INDEX idx_country_status ON transactions (country_code, status);
CREATE INDEX idx_user_payment   ON transactions (user_id, payment_method);
```

That is six indexes for five queries. (You could trim this, since a composite index also serves queries on its *leading* column, so `idx_country_status` already covers Query 2. But the problem is not these five queries. It is what happens as the number of possible filters grows.)

The number of possible column combinations grows exponentially. For a table with 10 columns there are **45** possible two-column combinations alone, and **1,023** non-empty combinations in total. And for a B-tree, column *order* matters as well. Building and maintaining an index for each one would consume huge amounts of disk space and slow down every `INSERT` and `UPDATE`.

A single bloom index can replace all of them:

```sql
CREATE EXTENSION IF NOT EXISTS bloom;

CREATE INDEX idx_transactions_bloom
ON transactions USING bloom (user_id, country_code, status, payment_method);
```

This one index can help with an equality filter on **any subset** of those four columns, in any order.

## How a bloom index works, step by step

### Step 1: Hash each column value into a few bits

Each indexed column's value is hashed, and the hash selects a small number of bit positions in the signature (2 per column by default). Let's use a 16-bit signature and an index on `(user_id, country_code, status)`. (We leave out `payment_method` to keep the bit strings short.) The bits below are illustrative:

| Value | Bits set |
|---|---|
| `user_id = 101` | 1, 9 |
| `user_id = 205` | 2, 9 |
| `user_id = 309` | 3, 10 |
| `user_id = 450` | 1, 6 |
| `country = NP` | 4, 12 |
| `country = IN` | 5, 11 |
| `country = US` | 7, 15 |
| `country = GB` | 9, 15 |
| `status = SUCCESS` | 6, 13 |
| `status = FAILED` | 0, 14 |
| `status = PENDING` | 8, 12 |
| `status = REFUND` | 2, 13 |

### Step 2: Combine them into one signature per row

The bit patterns of all columns in a row are combined with a bitwise **OR** to make the row's signature. We've added a fifth row, `TXN005`, which will matter in a moment:

| Row | user_id | country | status | Signature |
|---|---|---|---|---|
| TXN001 | 101 | NP | SUCCESS | `01001010 01001100` |
| TXN002 | 205 | IN | FAILED | `10100100 01010010` |
| TXN003 | 101 | NP | SUCCESS | `01001010 01001100` |
| TXN004 | 309 | US | PENDING | `00010001 10101001` |
| TXN005 | 450 | GB | REFUND | `01100010 01000101` |

For example, TXN001 sets bits 1 and 9 (from `101`), 4 and 12 (from `NP`), and 6 and 13 (from `SUCCESS`), giving bits `{1, 4, 6, 9, 12, 13}`.

The index stores each signature next to the location of its table row:

```text
Bloom index: idx_transactions_bloom

Signature of TXN001  →  tuple location of TXN001
Signature of TXN002  →  tuple location of TXN002
...
```

### Step 3: Use the signatures to answer a query

```sql
SELECT * FROM transactions
WHERE user_id = 101 AND status = 'SUCCESS';
```

1. **Hash the search values.** `101` needs bits 1, 9 and `SUCCESS` needs bits 6, 13. The required pattern is `01000010 01000100`.
2. **Test every signature.** A row is a candidate only if it has *all* the required bits set, i.e. `signature AND required = required`.
3. **Reject the rest.** A signature missing even one required bit cannot match.
4. **Recheck candidates in the table.** Fetch the candidate rows and verify the actual conditions.

Applied to our rows:

| Row | Bits in signature | Required bits {1, 6, 9, 13} | Outcome |
|---|---|---|---|
| TXN001 | 1, 4, 6, 9, 12, 13 | all present | Candidate, **real match** |
| TXN002 | 0, 2, 5, 9, 11, 14 | 1, 6, 13 missing | Rejected without touching the table |
| TXN003 | 1, 4, 6, 9, 12, 13 | all present | Candidate, **real match** |
| TXN004 | 3, 7, 8, 10, 12, 15 | none present | Rejected without touching the table |
| TXN005 | 1, 2, 6, 9, 13, 15 | all present | Candidate, but the recheck finds `user_id = 450` and `status = REFUND`, so it is discarded (**false positive**) |

Two of the five rows were eliminated using only the compact signatures. TXN005 shows why the recheck is essential: its values collided with the required bits purely by chance.

## Limitations

1. **Equality only.** Bloom indexes support only the `=` operator. There is no help for `<`, `>`, `BETWEEN`, `LIKE`, or sorting.
2. **Rechecks cost time.** Matching signatures must be verified against the table, so false positives add extra heap visits.
3. **The whole index is scanned.** There is no tree to navigate, so the index is read in full for each query. It is much smaller than the table, but the work still grows with the number of rows.
4. **No `UNIQUE` indexes and no searching for `NULL`.**
5. **Limited data types out of the box.** The `bloom` extension provides operator classes for `integer` (`int4`) and `text`. Other types need additional operator classes.
6. **Bitmap scans only.** The planner can never use a bloom index for a plain Index Scan or Index Only Scan.

## Creating a bloom index

```sql
CREATE INDEX bloomidx ON tbloom USING bloom (i1, i2, i3)
WITH (length=80, col1=2, col2=2, col3=4);
```

The storage parameters are:

| Parameter | Meaning | Default | Limit |
|---|---|---|---|
| `length` | Signature length in bits per row (rounded up to a multiple of 16) | 80 | up to 4096 |
| `col1`...`col32` | Number of bits set for each indexed column | 2 | up to 4095 |

An index can cover up to **32 columns**. The statement above creates an 80-bit signature where `i1` and `i2` set 2 bits each (the default, so they could be omitted) and `i3` sets 4 bits. Giving a column more bits makes matches on that column more selective, at the cost of filling the signature faster.

### Why the signature length and the number of filtered columns matter

Here is a back-of-the-envelope estimate (not a measurement) for a 6-column index with `length=80` and 2 bits per column. Each row sets at most 12 of the 80 bits, so a given bit is set in roughly 14% of signatures. A query filtering on **two** columns needs 4 specific bits, so about `0.14⁴ ≈ 0.04%` of unrelated rows pass the signature test anyway. On a 10-million-row table, that is roughly **4,000** useless heap visits. Doubling the signature to `length=160` drops this to about 280.

The same arithmetic shows why bloom indexes shine when queries filter on **several columns at once**. A filter on a *single* column needs only 2 bits, and about 2% of unrelated rows (around 200,000 of 10 million) will pass, which is far less selective.

## Hands-on: bloom vs. sequential scan vs. B-tree

### Setup

Create a table of 10 million rows with six random integer columns:

```sql
CREATE TABLE tbloom AS
SELECT
    (random() * 1000000)::int AS i1,
    (random() * 1000000)::int AS i2,
    (random() * 1000000)::int AS i3,
    (random() * 1000000)::int AS i4,
    (random() * 1000000)::int AS i5,
    (random() * 1000000)::int AS i6
FROM generate_series(1, 10000000);

```

Each value occurs about 10 times per column (10 million rows over 1 million possible values), so a condition on two columns is extremely selective. Most likely the query below returns zero rows, which is fine for timing. To test with a guaranteed hit, first run `SELECT i2, i5 FROM tbloom LIMIT 1;` and use those values.

### Without an index

```sql
EXPLAIN ANALYZE
SELECT * FROM tbloom WHERE i2 = 898732 AND i5 = 123451;
```

With no index available, PostgreSQL has to read the whole table and test every row. The plan has this shape (on a table this size it may show up as `Gather` with a `Parallel Seq Scan`):

```text
Seq Scan on tbloom  (...)
  Filter: ((i2 = 898732) AND (i5 = 123451))
  Rows Removed by Filter: ~10,000,000
Execution Time: 2189.410 ms
```

On my machine this took about **2,189 ms**.

### With a bloom index

```sql
CREATE EXTENSION IF NOT EXISTS bloom;

CREATE INDEX bloomidx ON tbloom USING bloom (i1, i2, i3, i4, i5, i6);
-- took about 5.5 seconds on my machine
```

Run the same query again:

```sql
EXPLAIN ANALYZE
SELECT * FROM tbloom WHERE i2 = 898732 AND i5 = 123451;
```

The plan now uses the bloom index:

```text
Bitmap Heap Scan on tbloom  (...)
  Recheck Cond: ((i2 = 898732) AND (i5 = 123451))
  Rows Removed by Index Recheck: ...   -- the false positives
  Heap Blocks: exact=...
  ->  Bitmap Index Scan on bloomidx  (...)
        Index Cond: ((i2 = 898732) AND (i5 = 123451))
```

Here `Bitmap Index Scan` reads the signatures and builds the bitmap of candidate rows, and `Bitmap Heap Scan` fetches them and rechecks the real values. The number reported on the `Rows Removed by Index Recheck` line is the count of false positives. It is the practical measure of how well the signature size suits your data. Note also that this one index also serves a query on *any other* combination of the six columns, for example `WHERE i1 = ... AND i4 = ... AND i6 = ...`.

Before moving on, record the index size:

```sql
SELECT pg_size_pretty(pg_relation_size('tbloom'))    AS table_size,
       pg_size_pretty(pg_relation_size('bloomidx'))  AS bloom_index_size;
```

### With B-tree indexes instead

To get the same flexibility with B-trees you would create **six** single-column indexes. Drop the bloom index first so the planner can't choose it:

```sql
DROP INDEX bloomidx;

CREATE INDEX btreeidx1 ON tbloom (i1);
CREATE INDEX btreeidx2 ON tbloom (i2);
CREATE INDEX btreeidx3 ON tbloom (i3);
CREATE INDEX btreeidx4 ON tbloom (i4);
CREATE INDEX btreeidx5 ON tbloom (i5);
CREATE INDEX btreeidx6 ON tbloom (i6);

EXPLAIN ANALYZE
SELECT * FROM tbloom WHERE i2 = 898732 AND i5 = 123451;
```

Since each value matches only about 10 rows, the planner will typically either combine two indexes (`BitmapAnd` of `btreeidx2` and `btreeidx5`) or use one and filter the rest. Either way, the query should be faster than with the bloom index, because a B-tree jumps straight to the matching entries instead of scanning every signature.

Then compare the cost in disk space:

```sql
SELECT indexrelid::regclass AS index_name,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_index
WHERE indrelid = 'tbloom'::regclass;
```

Each B-tree on 10 million integers typically takes a couple of hundred MB, so six of them add up to a large multiple of the single bloom index. They also slow down every write, since each insert must update six indexes. That is why this trade-off usually isn't worth making for ad hoc, many-column filtering.

## Choosing the right approach

| Approach | Query speed | Index size | Flexibility | Write overhead |
|---|---|---|---|---|
| **Sequential scan** (no index) | Slowest (2,189 ms in our test) | None | Any query | None |
| **One B-tree per column** | Fastest | Large (one index per column) | Any single-column filter, combined with bitmap `AND` | High |
| **Composite B-tree** | Fastest for the matching column order | Medium per index | Only queries using the leading columns | Medium to high |
| **Bloom index** | Much faster than a seq scan, slower than a matching B-tree | Small | Equality on **any combination** of the indexed columns | Low |

## Conclusion

A bloom index is a trade-off, not a replacement for B-trees. It stores a compact, lossy signature for each row, which lets PostgreSQL discard most non-matching rows by comparing a few bits and verify only the survivors against the table. In return for a small risk of false positives and an index that must be scanned in full, you get one small index that supports equality filters on any combination of up to 32 columns, which would otherwise take a great many B-trees.

Reach for a bloom index when your table is wide, the filters are ad hoc equality conditions on varying columns, and the cost of many B-trees (disk space and write speed) is a concern. Prefer B-trees when queries hit a known column or column order, when you need range conditions or sorting, or when you need the absolute fastest lookups. As always, confirm the choice with `EXPLAIN ANALYZE` on your own data, and tune `length` and the per-column bits until `Rows Removed by Index Recheck` is acceptably small.
