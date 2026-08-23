# SQL Concepts Reference

Concise definitions, syntax, minimal example, and the most common gotcha for each concept. Load only the entries you need; do not dump all of them into the user-facing reply.

---

## GROUP BY

Collapses rows that share the same values in the listed columns into one output row per group. Aggregates then run *per group*, not over the whole table.

**Syntax**
```sql
SELECT group_col, AGG(expr) AS alias
FROM t
GROUP BY group_col;
```

**Common gotcha** — `SELECT` may list only (a) columns in `GROUP BY` and (b) aggregates. A non-aggregated column outside `GROUP BY` is an error in strict mode (PostgreSQL, BigQuery, Snowflake) and undefined garbage in MySQL with `ONLY_FULL_GROUP_BY` off.

---

## DISTINCT

Removes duplicate rows from the result set. `COUNT(DISTINCT col)` counts distinct non-null values; `COUNT(*)` counts rows (including duplicates and `NULL`).

**Gotcha** — `DISTINCT` applies to the *whole* select list, not a single column, unless wrapped in an aggregate: `SELECT DISTINCT a, b` ≠ `SELECT DISTINCT a, DISTINCT b`.

---

## JOINs

Combine rows from two tables by a predicate.

| Type       | Keeps rows where...                                   |
| ---------- | ------------------------------------------------------ |
| `INNER`    | predicate matches on **both** sides                   |
| `LEFT`     | all left rows; right side is `NULL` when unmatched    |
| `RIGHT`    | all right rows; left side is `NULL` when unmatched    |
| `FULL`     | all rows from both; missing side becomes `NULL`        |
| `CROSS`    | every combination (cartesian product)                 |
| `LATERAL`  | right side may reference left-side columns row-by-row |

**Gotcha** — `LEFT JOIN` plus a filter on the right table in `WHERE` quietly turns it into an `INNER JOIN`. Put right-side filters in `ON`, not `WHERE`, when you want unmatched left rows preserved.

---

## EXISTS vs IN

Both test membership. Prefer `EXISTS` for subqueries because it short-circuits and decouples the inner column list from the outer.

```sql
SELECT u.id FROM users u
WHERE EXISTS (SELECT 1 FROM page_views pv WHERE pv.user_id = u.id);
```

`IN` with a subquery can also misbehave with `NULL`: `NOT IN (subquery with NULL)` returns **zero** rows, not "rows whose value isn't in the list".

---

## CTE (WITH)

A named subquery scoped to one statement. Improves readability and lets you reference the same intermediate result multiple times.

```sql
WITH daily AS (
  SELECT user_id, DATE_TRUNC('day', ts) AS day FROM page_views
)
SELECT day, COUNT(DISTINCT user_id) FROM daily GROUP BY day;
```

**Gotcha** — In older PostgreSQL / SQL Server, a CTE is an *optimization fence* (materialized once). In PostgreSQL 12+ and BigQuery CTEs are inlined by default. Don't reuse the same CTE name twice in one statement.

---

## Window Functions

Run an aggregate (or ranking, or navigation) without collapsing rows. The result has the same row count as the input; a new column is added.

```sql
SELECT
  user_id,
  ts,
  ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY ts) AS nth_view,
  COUNT(*)    OVER (PARTITION BY user_id)                AS views_per_user
FROM page_views;
```

Useful window functions: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, `LEAD`, `SUM/AVG/COUNT ... OVER (...)`, `FIRST_VALUE`, `NTILE`.

**Gotcha** — `PARTITION BY` is *not* `GROUP BY`. It does not reduce rows; it only resets the window per group.

---

## CASE WHEN

Inline conditional. Two forms: simple (one expression per branch) and searched (boolean per branch).

```sql
SELECT
  page_name,
  CASE
    WHEN ts >= NOW() - INTERVAL '7 days' THEN 'recent'
    WHEN ts >= NOW() - INTERVAL '30 days' THEN 'month'
    ELSE 'old'
  END AS bucket,
  COUNT(DISTINCT user_id) AS users
FROM page_views
GROUP BY page_name, bucket;
```

Use for bucketing, custom sort orders, and conditional aggregation (`SUM(CASE WHEN is_new THEN 1 ELSE 0 END)`).

---

## COALESCE / NULLIF

- `COALESCE(a, b, c, ...)` — first non-null argument. The standard "default value".
- `NULLIF(a, b)` — returns `NULL` if `a = b`, otherwise `a`. Handy for dividing without `/0` errors: `amount / NULLIF(quantity, 0)`.

**Gotcha** — `COALESCE` and `NULLIF` are often confused with `ISNULL`/`NVL` (dialect-specific). Use `COALESCE` for portability; substitute in dialect-tuned snippets.

---

## Subqueries: Scalar and Correlated

- **Scalar** — returns one value. Use in `SELECT` or `WHERE col = (SELECT ...)`.
- **Correlated** — references the outer query; re-evaluated per outer row. Often a perf footgun on big tables.

**Gotcha** — `SELECT col = (SELECT ...)` returns `NULL` when the subquery returns more than one row, *without an error*.

---

## Set Operations

`UNION` (deduplicates), `UNION ALL` (keeps duplicates, faster), `INTERSECT`, `EXCEPT`/`MINUS`.

All require the same number of columns and compatible types. `INTERSECT`/`EXCEPT` remove duplicates silently — add `ALL` if you want to keep them.

---

## Date / Time Functions

| Goal                          | PostgreSQL              | MySQL                          | SQL Server             | BigQuery                |
| ----------------------------- | ----------------------- | ------------------------------ | ---------------------- | ----------------------- |
| Truncate to day               | `DATE_TRUNC('day', ts)` | `DATE_FORMAT(ts, '%Y-%m-%d')`  | `CAST(ts AS DATE)`     | `DATE_TRUNC(ts, DAY)`   |
| Difference in days            | `ts1 - ts2`             | `DATEDIFF(ts1, ts2)`           | `DATEDIFF(day,...)`    | `DATE_DIFF(DATE ts1, ts2)` |
| Add n days                    | `ts + INTERVAL '1 day'` | `DATE_ADD(ts, INTERVAL 1 DAY)` | `DATEADD(day, 1, ts)`  | `DATE_ADD(ts, INTERVAL 1 DAY)` |
| Now                           | `NOW()`                 | `NOW()`                        | `GETDATE()`            | `CURRENT_TIMESTAMP()`   |

---

## String Aggregation

Concatenate many rows into one cell.

- PostgreSQL / SQL Server / Snowflake: `STRING_AGG(expr, ', ' ORDER BY ...)`
- MySQL: `GROUP_CONCAT(expr SEPARATOR ', ')`
- BigQuery: `STRING_AGG(expr, ', ')`

**Gotcha** — `DISTINCT` inside `STRING_AGG` is supported in PostgreSQL but not all dialects; deduplicate in a CTE first when in doubt.
