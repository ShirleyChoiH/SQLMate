# SQL Optimization Patterns

Load this reference when the user says "optimize this", when the query looks expensive, or when `Caveats` needs a real performance note. Each pattern comes with the **anti-pattern**, the **fix**, and **why it works**.

---

## 1. Sargable Predicates

A predicate is *sargable* if the database can use an index on the column being filtered.

**Anti-pattern**
```sql
WHERE UPPER(email) = 'USER@EXAMPLE.COM'
WHERE DATE(created_at) = '2026-01-15'
WHERE YEAR(created_at) = 2026
```

**Fix**
```sql
WHERE email = 'user@example.com'           -- store canonical case, or use functional index on UPPER
WHERE created_at >= '2026-01-15' AND created_at < '2026-01-16'
WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'
```

**Why** — wrapping a column in a function forces a full table scan; the index cannot be used.

---

## 2. Prefer EXISTS to IN for Subqueries

**Anti-pattern**
```sql
SELECT u.* FROM users u
WHERE u.id IN (SELECT user_id FROM page_views WHERE ts >= NOW() - INTERVAL '30 days');
```

**Fix**
```sql
SELECT u.* FROM users u
WHERE EXISTS (
  SELECT 1 FROM page_views pv
  WHERE pv.user_id = u.id AND pv.ts >= NOW() - INTERVAL '30 days'
);
```

**Why** — `IN` materializes the subquery and compares each outer row to the full set; `EXISTS` short-circuits on the first match per outer row.

---

## 3. Avoid SELECT *

**Anti-pattern** — `SELECT *` returns columns you don't use, blocks covering-index strategies, and breaks code when the schema changes.

**Fix** — name the columns you need. With column-store warehouses (BigQuery, Snowflake) this can cut scanned data by 5–100×.

---

## 4. Filter Early, JOIN Late

Push `WHERE` filters as close to the source as possible. Modern optimizers usually do this for you, but CTEs with `WHERE`s downstream can confuse older planners.

**Pattern**
```sql
WITH recent AS (
  SELECT user_id, page_name, ts
  FROM page_views
  WHERE ts >= NOW() - INTERVAL '30 days'   -- filter first
)
SELECT r.page_name, COUNT(DISTINCT r.user_id) AS users
FROM recent r
GROUP BY r.page_name;
```

---

## 5. Window Functions Instead of Self-Joins for De-Dup

**Anti-pattern**
```sql
SELECT * FROM events e1
JOIN events e2 ON e1.user_id = e2.user_id AND e1.ts < e2.ts;
```

**Fix**
```sql
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY ts DESC) AS rn
  FROM events
) ranked WHERE rn = 1;
```

`ROW_NUMBER() = 1` is one pass and parallelizable; the self-join squares the input.

---

## 6. Use Covering Indexes for Hot Queries

A covering index includes every column the query needs (filter + `SELECT`), so the database never has to visit the heap.

```sql
CREATE INDEX idx_pv_user_recent
  ON page_views (user_id, ts DESC)
  INCLUDE (page_name);     -- SQL Server / PostgreSQL 11+
```

---

## 7. Count Distinct: HyperLogLog vs Approximate vs Exact

| Need                              | Cheapest option                              |
| --------------------------------- | -------------------------------------------- |
| Exact count                       | `COUNT(DISTINCT col)`                        |
| Approx, ±2%                       | `APPROX_COUNT_DISTINCT(col)` (BigQuery / Snowflake) |
| HyperLogLog                       | PostgreSQL `hll` extension / Snowflake `HLL_ACCUMULATE` |

`COUNT(DISTINCT)` is one of the most expensive operations on big tables; switch to approximate counts when the user only needs a magnitude.

---

## 8. JOIN Order Matters on Small Engines

On MySQL / PostgreSQL the planner usually reorders joins. On very large joins, controlling order with CTEs (in PostgreSQL 12+, CTEs are inlined by default) or `/*+ LEADING */` optimizer hints in MySQL/Oracle can beat the planner. Don't reach for hints unless you have measured that the planner is wrong.

---

## 9. Window Function Frames

A window function with no `ROWS BETWEEN` clause uses `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` by default, which is usually fine. On large partitions, narrowing the frame makes the function dramatically cheaper:

```sql
SUM(amount) OVER (
  PARTITION BY user_id
  ORDER BY ts
  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
) AS running_total
```

Prefer `ROWS` (row-count based, fast) over `RANGE` (value based, scans equal values).

---

## 10. Batch Deletes and Updates

`DELETE FROM huge_table WHERE condition` may lock the table for minutes. Break it up:

```sql
DELETE FROM logs WHERE ts < '2025-01-01' LIMIT 10000;  -- MySQL
-- or, in PostgreSQL:
DELETE FROM logs WHERE ctid IN (
  SELECT ctid FROM logs WHERE ts < '2025-01-01' LIMIT 10000
);
```

---

## Cardinality and Cost Smells to Flag in Caveats

- `SELECT COUNT(*) FROM huge_table` without a `WHERE` — full scan.
- `DISTINCT` on every column of a wide table — single-row hash per row.
- Correlated subquery in `SELECT` of a huge outer table — re-evaluated per row.
- `LIKE '%foo'` (leading wildcard) — cannot use index.
- Multiple OR-ed inequalities on the same indexed column — usually a UNION is cheaper.
- `UNION` (without `ALL`) over large `UNION ALL` would-have-been results — sort/dedupe cost.
