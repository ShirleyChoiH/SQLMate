# SQL Dialect Differences

Quick lookup for choosing syntax across the major dialects. Pick the row matching the dialect the user named; if none is named, default to PostgreSQL/ANSI SQL and add a one-line portability note in `Caveats`.

---

## SELECT TOP / LIMIT

| Dialect                | Syntax                                              |
| ---------------------- | --------------------------------------------------- |
| PostgreSQL, MySQL      | `SELECT ... LIMIT n OFFSET m;`                     |
| SQLite                 | `SELECT ... LIMIT n OFFSET m;`                     |
| BigQuery               | `SELECT ... LIMIT n;`                              |
| Snowflake              | `SELECT ... LIMIT n;`                              |
| SQL Server             | `SELECT TOP n ...` (legacy) or `SELECT ... ORDER BY ... OFFSET m ROWS FETCH NEXT n ROWS ONLY;` (ANSI) |
| Oracle                 | `SELECT ... FETCH FIRST n ROWS ONLY;`              |

---

## String Aggregation

| Dialect       | Function                                                       |
| ------------- | --------------------------------------------------------------- |
| PostgreSQL    | `STRING_AGG(expr, ', ' ORDER BY col)`                          |
| SQL Server    | `STRING_AGG(expr, ', ')` (2017+)                                |
| MySQL         | `GROUP_CONCAT(expr SEPARATOR ', ')`                             |
| BigQuery      | `STRING_AGG(expr, ', ' ORDER BY col)`                          |
| Snowflake     | `LISTAGG(expr, ', ') WITHIN GROUP (ORDER BY col)`              |
| Oracle        | `LISTAGG(expr, ', ') WITHIN GROUP (ORDER BY col)`              |

---

## Date / Time Functions

| Goal              | PostgreSQL             | MySQL                          | SQL Server                                  | BigQuery                 |
| ----------------- | ---------------------- | ------------------------------ | ------------------------------------------- | ------------------------ |
| Now               | `NOW()` / `CURRENT_TIMESTAMP` | `NOW()`                  | `GETDATE()` / `SYSDATETIME()`               | `CURRENT_TIMESTAMP()`    |
| Truncate to day   | `DATE_TRUNC('day', ts)`| `DATE(ts)` / `DATE_FORMAT(ts, '%Y-%m-%d')` | `CAST(ts AS DATE)` / `FORMAT(ts, 'yyyy-MM-dd')` | `DATE_TRUNC(ts, DAY)`   |
| Date diff (days)  | `ts1::date - ts2::date` | `DATEDIFF(ts1, ts2)`         | `DATEDIFF(day, ts2, ts1)`                   | `DATE_DIFF(ts1, ts2, DAY)` |
| Add n days        | `ts + INTERVAL '1 day'`| `DATE_ADD(ts, INTERVAL 1 DAY)` | `DATEADD(day, n, ts)`                       | `DATE_ADD(ts, INTERVAL n DAY)` |
| Year / month extract | `EXTRACT(YEAR FROM ts)` | `YEAR(ts)`, `MONTH(ts)`     | `YEAR(ts)`, `MONTH(ts)`                     | `EXTRACT(YEAR FROM ts)` |

---

## Booleans and Truthiness

| Dialect       | Boolean literal      | Truthiness for `WHERE`                |
| ------------- | -------------------- | ------------------------------------- |
| PostgreSQL    | `TRUE` / `FALSE`     | explicit boolean expression required  |
| MySQL 8+      | `TRUE` / `FALSE`     | numbers coerce to bool; `0` = false    |
| SQL Server    | no native boolean    | use `bit`: `WHERE is_active = 1`      |
| BigQuery      | `TRUE` / `FALSE`     | explicit boolean expression required  |
| SQLite        | `0` / `1`            | `0` is false, anything else true      |

---

## String Matching and Regex

| Dialect       | Case-insensitive LIKE | Regex                              |
| ------------- | --------------------- | ---------------------------------- |
| PostgreSQL    | `ILIKE`               | `~` / `~*` (case-insens) / `REGEXP_MATCHES` |
| MySQL          | `LIKE` is case-insensitive by default (depends on collation) | `REGEXP` / `RLIKE` |
| SQL Server    | `LIKE` is case-insensitive by default (depends on collation) | `LIKE` with `%[...]%` patterns |
| BigQuery      | `LOWER(col) LIKE LOWER(pattern)` | `REGEXP_CONTAINS(col, pattern)` |
| SQLite        | `LIKE` is case-insensitive for ASCII by default | `REGEXP` (no built-in; needs extension) |

---

## Identifier Quoting

| Dialect       | Quotes identifiers | Notes                              |
| ------------- | ------------------ | ---------------------------------- |
| PostgreSQL    | `"column"`          | case-sensitive                      |
| MySQL          | `` `column` ``        | case-sensitive on Linux, insensitive on macOS/Windows |
| SQL Server    | `[column]` or `"column"` | case-insensitive              |
| BigQuery      | `` `column` ``         | backticks                          |
| Standard ANSI | `"column"`          | preferred for portability            |

---

## NULL Handling Differences

- `CONCAT` skips `NULL` in MySQL and BigQuery but yields `NULL` in PostgreSQL.
- `NULLIF(a, 0)` is universal; use it before division.
- `DISTINCT` treats all `NULL`s as equal (one `NULL` survives). Universal.
- Ordered-set aggregates (`PERCENTILE_CONT`) exist in PostgreSQL and SQL Server; in MySQL there is no direct equivalent — compute in app.

---

## Pagination

| Dialect       | Best practice                                  |
| ------------- | ---------------------------------------------- |
| PostgreSQL    | `LIMIT n OFFSET m` (deep offsets are slow — keyset pagination preferred) |
| MySQL          | `LIMIT n OFFSET m`                            |
| SQL Server    | `ORDER BY ... OFFSET m ROWS FETCH NEXT n ROWS ONLY` |
| BigQuery      | `LIMIT n OFFSET m`; BigQuery also supports cursor-based pagination via `WHERE ts > last_ts` |
| Snowflake     | `LIMIT n OFFSET m`                             |

---

## When The User Says "Just SQL"

Default to PostgreSQL/ANSI SQL syntax that runs in: PostgreSQL, SQLite, BigQuery, and Snowflake with no changes, and in MySQL/SQL Server with small tweaks. Note the most likely tweaks in `Caveats`:

- `STRING_AGG` → `GROUP_CONCAT` on MySQL.
- `LIMIT n` → `SELECT TOP n` on SQL Server.
- `TRUE`/`FALSE` → `1`/`0` on SQL Server.
- `EXTRACT(YEAR FROM ts)` → `YEAR(ts)` on MySQL/SQL Server.
