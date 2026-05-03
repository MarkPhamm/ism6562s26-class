# Week 3 — `EXPLAIN` & Indexing TLDR

A condensed companion to the Week 3 lab. Three things to learn:

1. **How to read `EXPLAIN ANALYZE`** — the only honest answer to "is my query slow?"
2. **How the plan changes when you add an index** — the whole point of indexing.
3. **The tradeoff** — indexes speed up reads, slow down writes, cost disk.

All examples use the `lab_orders` table from the in-class walkthrough (100 K rows, no indexes beyond the PK).

---

## 1. TLDR

| Question | Answer |
| --- | --- |
| How do I know if a query is slow? | `EXPLAIN ANALYZE the_query;` → look at `Execution Time` and `Rows Removed by Filter`. |
| When should I add an index? | When the plan shows a `Seq Scan` on a big table whose `Filter:` discards >90% of rows. |
| What's the catch? | Every `INSERT` / `UPDATE` / `DELETE` must update every index on the table. Indexes also take disk space. |
| How do I confirm the index helps? | Re-run `EXPLAIN ANALYZE` after creating it. If the plan still says `Seq Scan`, the planner thinks the index isn't worth it — trust it. |

---

## 2. `EXPLAIN ANALYZE` Cheat Sheet

### Two flavors

```sql
EXPLAIN q;            -- estimated plan, doesn't run the query
EXPLAIN ANALYZE q;    -- runs the query, prints estimates + real timings
```

> ⚠ `ANALYZE` actually executes. For destructive queries, wrap in `BEGIN; ... ROLLBACK;`.

### Anatomy of one plan line

```text
Seq Scan on lab_orders  (cost=0.00..2084.00  rows=18  width=27)
                        (actual time=0.028..11.482  rows=21  loops=1)
  Filter: (customer_id = 42)
  Rows Removed by Filter: 99979
```

| Piece | Meaning |
| --- | --- |
| `Seq Scan` | Node type — read every row of the table top to bottom. |
| `cost=0.00..2084.00` | Planner's startup..total estimate, in arbitrary cost units (**not ms**). |
| `rows=18` | **Estimated** rows out of this node. |
| `actual time=0.028..11.482` | **Real** start..end ms for this node, **per loop**. |
| `rows=21 loops=1` | Real row count, real loop count. Multiply `actual time × loops` for total time. |
| `Filter:` | Condition applied **after** rows are read — wasted work. |
| `Rows Removed by Filter: 99979` | The smoking gun: 99,979 rows were read just to be thrown away. |

### Read plans bottom-up

Indented children execute first; parents consume their output.

```text
Limit                          ← 4. final 10 rows returned to client
  Sort                         ← 3. sort by total DESC, keep top 10
    HashAggregate              ← 2. group by customer_id, SUM(amount)
      Seq Scan on lab_orders   ← 1. read & filter the table
```

### Node types you'll meet this week

| Node | When you see it |
| --- | --- |
| **Seq Scan** | No usable index, or filter not selective enough. |
| **Index Scan** | Selective filter on an indexed column (returns a small fraction). |
| **Bitmap Index Scan + Bitmap Heap Scan** | Medium-selective range/list — Postgres builds a bitmap, then reads the heap in physical order. Very common for `BETWEEN` on indexed columns. |
| **Index Only Scan** | All needed columns live in the index (covering index). Heap not touched. |
| **Sort** | `ORDER BY` without a matching index. |
| **HashAggregate / GroupAggregate** | `GROUP BY` / aggregates. |
| **Hash Join / Nested Loop / Merge Join** | Three join algorithms. |

### Quick-read checklist

- `Seq Scan` + huge `Rows Removed by Filter` → **index opportunity**.
- Estimated `rows=` very different from actual `rows=` → **stale stats**, run `ANALYZE table_name;`.
- Inner side of `Nested Loop` with `loops=N` and a `Seq Scan` → that scan runs N times — **index the join column**.
- `Sort Method: external merge Disk:` → sort spilled to disk; consider a matching ordered index or raise `work_mem`.

---

## 3. The Key Plan Diff — Before vs. After an Index

The single most important picture in this lab: **same query, plan changes, time drops 165×.**

```sql
-- query
SELECT * FROM lab_orders WHERE customer_id = 42;
```

### Before — no index on `customer_id`

```text
Seq Scan on lab_orders  (cost=0.00..2084.00 rows=18 width=27)
                        (actual time=0.028..11.482 rows=21 loops=1)
  Filter: (customer_id = 42)
  Rows Removed by Filter: 99979
Execution Time: 11.513 ms
```

Postgres reads all 100,000 rows and discards 99,979.

### After — `CREATE INDEX idx_lab_orders_cust ON lab_orders (customer_id);`

```text
Index Scan using idx_lab_orders_cust on lab_orders  (cost=0.29..72.49 rows=18 width=27)
                                                    (actual time=0.025..0.048 rows=21 loops=1)
  Index Cond: (customer_id = 42)
Execution Time: 0.069 ms
```

Postgres walks the B-tree directly to the 21 matching rows and reads only those.

### What actually changed

| Metric | Before (Seq Scan) | After (Index Scan) |
| --- | --- | --- |
| Node type | `Seq Scan` | `Index Scan` |
| `Filter:` line | present | gone (replaced by `Index Cond:`) |
| `Rows Removed by Filter` | 99,979 | line absent — nothing was removed |
| Total cost | 0.00..2084.00 | 0.29..72.49 |
| Execution time | ~11.5 ms | ~0.07 ms |
| Speedup | — | **~165×** |

Why: without the index, work is **O(N)** — 100,000 row reads. With it, work is **O(log N)** — ~3–4 B-tree levels plus 21 row fetches. The bigger the table, the wider this gap.

### Plan diff for a range query

`WHERE order_date BETWEEN '2025-06-01' AND '2025-06-30'` returns ~8,219 of 500,000 rows (~1.6%) — too many for a plain `Index Scan`, too few for a `Seq Scan`. Postgres picks a **bitmap** plan after `CREATE INDEX idx_lab_orders_date ON lab_orders (order_date);`:

```text
Bitmap Heap Scan on lab_orders  (… rows=8219 loops=1)
  Recheck Cond: ((order_date >= '2025-06-01') AND (order_date <= '2025-06-30'))
  Heap Blocks: exact=164
  ->  Bitmap Index Scan on idx_lab_orders_date  (… rows=8219)
        Index Cond: (...)
Execution Time: 6.945 ms       (was 42.378 ms before the index — ~6× faster)
```

### The two scans broken down

**Bitmap Index Scan** (cheap, ~1.2 ms)
Walks `idx_lab_orders_date` looking for entries in your date range. Doesn't return rows — returns a **bitmap** saying *"rows you want live in pages X, Y, Z…"* (164 pages here).

**Bitmap Heap Scan** (the heavier work, ~2.6 ms more)
Takes that bitmap, reads those 164 pages from the actual table file **in physical disk order** (sequential I/O = fast), then walks each page's rows and applies `Recheck Cond:` to drop non-matching rows.

### Why this combo instead of a plain Index Scan?

A plain `Index Scan` jumps to each matching row one at a time → **random I/O**. With 8,219 matches, that's painful. The bitmap pair trades a small recheck cost for **sequential page reads**.

Postgres picks bitmap when selectivity is **mid-range**: too many rows for a direct `Index Scan`, too few for a full `Seq Scan`.

```text
   selectivity →  very few rows ─────────────────► most rows
   plan         →  Index Scan    Bitmap Heap Scan    Seq Scan
                  (random I/O    (sequential I/O    (no index)
                   per row)       on matched pages)
```

### Why does `Recheck Cond:` exist?

If the bitmap gets too big to fit in `work_mem`, Postgres compresses it to **lossy mode** where each bit represents a *whole page* instead of a single row. That means *some rows on a marked page might not actually match*, so the heap-scan step has to re-evaluate the predicate to drop them. With an exact (non-lossy) bitmap the recheck is essentially free; with a lossy one it's the price of memory savings.

### Selectivity drives the gain

Speedup is smaller here (6× vs 165×) because the filter returns ~1.6% of the table rather than ~0.02%. **Selectivity is the single biggest predictor of how much an index will help.**

> **Side note — Redshift / columnar engines don't have this.**
> Columnar engines store data by column with **zone maps** (per-block min/max metadata). They prune blocks *at scan time* without a separate index structure, so you don't see Bitmap Index Scan / Bitmap Heap Scan plans there. Different mechanism, same goal: skip the data you don't need.

---

## 4. The Tradeoff — Reads Get Faster, Writes Get Slower

Indexes are not free. Every `INSERT`/`UPDATE`/`DELETE` must update every index on the table. The lab's bulk-insert experiment makes this concrete:

| Approach | Time (200 K-row insert) |
| --- | --- |
| Insert **WITH** index already on the table | **~1,842 ms** |
| `DROP INDEX` → insert → `CREATE INDEX` | **~1,300 ms** (987 ms insert + 313 ms build) |
| Savings | ~30% |

The index-rebuild path wins because building an index in one shot is sort-and-build (one pass) — way cheaper than incremental B-tree maintenance per row.

> **Production pattern (used in the midterm ETL):** for big bulk loads, drop indexes, load, then recreate.

### Other costs to remember

| Cost | Notes |
| --- | --- |
| **Disk space** | A B-tree is typically 20–60% the size of the table. |
| **Write amplification** | 5 indexes ≈ 5× the per-row write work. |
| **Vacuum / bloat** | More indexes = more maintenance. |
| **Plan time** | Marginal, but more options for the planner to consider. |

### When indexes **don't** help (planner will skip them)

- **Tiny table** (`locations`, 10 rows) — Seq Scan reads one disk page.
- **Low-selectivity filter** — `WHERE is_active = TRUE` matches 95% of rows.
- **Function on column** — `WHERE LOWER(building) = 'library'` doesn't match a plain index on `building`. Use an expression index: `CREATE INDEX … ON locations(LOWER(building));`.
- **`LIKE '%foo%'`** — leading wildcard, B-tree can't help.
- **Wrong leading column** — `(a, b)` index doesn't help bare `WHERE b = …`.

If `EXPLAIN` shows `Seq Scan` after you added an index, the planner usually has a reason. Investigate selectivity before forcing it.

---

## 5. Index Types in One Page

```sql
-- single column
CREATE INDEX idx_orders_cust ON lab_orders (customer_id);

-- composite (leftmost-prefix rule: helps WHERE a=… and WHERE a=… AND b=…,
-- but NOT bare WHERE b=…)
CREATE INDEX idx_orders_cust_date ON lab_orders (customer_id, order_date);

-- partial (smaller; only used when the WHERE clause matches)
CREATE INDEX idx_orders_recent ON lab_orders (customer_id)
  WHERE order_date >= '2026-01-01';

-- expression (matches a function used in the query)
CREATE INDEX idx_orders_lower_product ON lab_orders (LOWER(product));

-- covering (extra columns in the leaf → enables Index Only Scan)
CREATE INDEX idx_orders_cust_inc ON lab_orders (customer_id) INCLUDE (amount);

-- unique (also enforces uniqueness)
CREATE UNIQUE INDEX idx_orders_pk_uniq ON lab_orders (order_id);

-- non-blocking on prod (slower to build, doesn't lock writes)
CREATE INDEX CONCURRENTLY idx_orders_date ON lab_orders (order_date);
```

### Composite-index column order rules

1. **Equality columns first**, range columns last.
2. **Most selective column first** when in doubt.
3. Put the `ORDER BY` column last to skip the `Sort` node.

---

## 6. The Workflow

```text
   ┌──────────────────────────────────────────────────────────┐
   │ 1. EXPLAIN ANALYZE the slow query                        │
   │ 2. Find the bottleneck node (high time × loops, big      │
   │    "Rows Removed by Filter")                             │
   │ 3. Hypothesize an index that would change the plan       │
   │ 4. CREATE INDEX                                          │
   │ 5. Re-run EXPLAIN ANALYZE — did the plan change?         │
   │    Did wall time actually drop?                          │
   │ 6. If not, DROP INDEX. Cost without benefit.             │
   └──────────────────────────────────────────────────────────┘
```

### Reference commands

```sql
EXPLAIN q;                              -- estimated plan
EXPLAIN ANALYZE q;                      -- run + real timings
EXPLAIN (ANALYZE, BUFFERS) q;           -- + I/O breakdown

\d lab_orders                           -- columns + indexes
SELECT indexname FROM pg_indexes WHERE tablename = 'lab_orders';
SELECT pg_size_pretty(pg_relation_size('idx_lab_orders_cust'));

-- which indexes are unused since stats were last reset?
SELECT indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0;

ANALYZE lab_orders;                     -- refresh planner stats
DROP INDEX idx_lab_orders_cust;
```

---

## Takeaways

1. **`EXPLAIN ANALYZE` is the only honest answer.** Estimates lie; "optimization" without measurement is guessing.
2. **`Seq Scan` + huge `Rows Removed by Filter` = index opportunity.** That single line is the smoke alarm.
3. **An index changes the *plan*, not just the time.** The Filter line disappears, replaced by `Index Cond:`.
4. **The index speedup scales with selectivity.** 0.02% selectivity → 165×. 1.6% → 6×. 90% → 0× (planner ignores the index).
5. **Reads get faster, writes get slower.** Drop indexes for bulk loads, recreate after.
6. **Always re-run `EXPLAIN ANALYZE` after creating an index** to confirm the plan changed *and* the wall time actually dropped. If not, drop it.
