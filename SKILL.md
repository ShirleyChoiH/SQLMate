---
name: sql-copilot-tutor
description: This skill should be used when a user needs help writing an SQL query, understanding a query's logic, learning SQL syntax, or getting a teaching-style breakdown of SQL concepts. Triggers on requests such as "write a SQL query for ...", "how do I count unique users per page", "explain this SQL", "what does GROUP BY do", "SQL 教程", "SQL 入门", or whenever a schema and a data question are provided together. Produces an optimized, dialect-aware SQL query along with a line-by-line explanation, output interpretation, caveats/edge cases, and a mini-classroom on the SQL concepts involved.
agent_created: true
---

# SQL Copilot & Tutor

## Overview

This skill turns a natural-language data question (with an optional schema) into a clean, runnable SQL query, then walks the user through it in plain language. Every response follows the same five-section format, in this fixed order: (1) the optimized SQL, (2) a line-by-line explanation, (3) what the result set actually means, (4) caveats and reminders, (5) a short lesson on the SQL concepts used. Use the same five-section shape whether the user wants to write a query, debug one, or learn SQL — only the depth and emphasis change.

Write for the user in front of you. Assume the reader is new to SQL unless they show otherwise. Choose plain words over technical ones. When a technical term is necessary, define it the first time it appears. Keep sentences short. Prefer one idea per sentence.

## When To Use This Skill

Use this skill whenever the user:

- Asks for an SQL query. Examples: "Write a query that finds...", "How do I get the top 5 customers by revenue?"
- Provides a schema and a data question at the same time. This is the most common trigger.
- Asks for an explanation of an existing SQL query. Example: "Explain what this query does."
- Wants to learn an SQL function or concept. Examples: "Teach me window functions", "What is the difference between WHERE and HAVING?"
- Needs help turning logic (pseudocode or a business rule) into SQL.
- Asks to optimize a slow or expensive query.

Do **not** use this skill for non-SQL data work such as pandas transformations or MongoDB aggregations. This skill is SQL only.

## Workflow

Always follow these five steps in order. Each step produces a labeled section in the final reply. Skip a section only when the user's intent makes it irrelevant. For example, a pure "explain this query" request may still use steps 2 to 4, but it will not need step 5's "concepts used" framing.

### Step 1 — Optimized SQL Query

Produce a single, runnable SQL block. Default to standard ANSI SQL that works across PostgreSQL, MySQL, SQLite, SQL Server, and BigQuery unless the user names a specific dialect. Follow these best practices:

- List columns explicitly in `SELECT`. Avoid `SELECT *` unless the user is exploring an unfamiliar table.
- Use short table aliases such as `u` or `pv`, and prefix every column with its alias.
- Place the most selective filter first inside `WHERE`.
- Apply aggregations through `GROUP BY` correctly. Never reference a non-aggregated column in `SELECT` without including it in the `GROUP BY`.
- Prefer `EXISTS` over `IN (SELECT ...)` for subqueries on large tables.
- Prefer explicit `JOIN ... ON` over comma joins.
- Add a `LIMIT` (or `TOP` / `FETCH FIRST`) when returning sample rows.
- Use CTEs (`WITH ... AS`) when the query has two or more logical stages. Keep them short, and name them for what they mean, not what they select.

### Step 2 — Line-by-Line Explanation

Walk through the query from top to bottom, clause by clause: `WITH` → `SELECT` → `FROM` → `JOIN` → `WHERE` → `GROUP BY` → `ORDER BY` → `LIMIT`. For each clause, include:

- A one-sentence description of what it does.
- The reason it is there, in plain language the user can follow.
- A short note on anything that beginners often find confusing. For example: "this `LEFT JOIN` keeps users with zero page_views, because those users still appear in the `users` table."

Use two to three sentences for complex lines such as CTEs, window functions, or conditional aggregations. Match the length of the explanation to the length of the query. A five-line query should not get a thirty-line explanation.

### Step 3 — Output Interpretation

Describe, in plain English, what each column in the result set means. Follow the order in which the columns appear in the query. Include:

- The unit or scale. Examples: counts of distinct users, US dollars, average minutes.
- One small example row that shows a plausible shape of the data.
- The shape of the output: one row per X, a single aggregate, or a list of rows.

### Step 4 — Caveats & Reminders

Always include a `### Caveats` block, even when it is short. Cover anything the user could misread:

- Metric semantics, such as average versus sum, or counts of distinct users versus counts of events. These numbers can diverge when one user does many things.
- Time-window assumptions, such as last 30 days, all time, or since signup.
- Whether rows with zero activity are kept or dropped. An `INNER JOIN` drops them silently; a `LEFT JOIN` keeps them.
- NULL handling. `COUNT(column)` ignores NULLs; `COUNT(*)` counts them.
- Cardinality warnings, such as a "top 10" over a tiny dataset.
- Performance notes only when they are material, for example a missing index or a large scan. Keep these short and actionable.

### Step 5 — Mini-Classroom

Include a short lesson on every non-trivial SQL construct used in the query. Format each concept as:

```
**`CONCEPT_NAME`** — one-sentence definition in plain language.
  - What it does to the rows.
  - When to reach for it, and when to avoid it.
  - One minimal example (3 to 5 lines).
```

Write each definition for a beginner. Open every concept with a one-sentence plain-language summary before introducing any syntax. Concepts commonly taught: `GROUP BY`, `DISTINCT`, `JOIN` (INNER/LEFT/RIGHT/FULL/CROSS), `EXISTS` vs `IN`, `CTE` (`WITH`), `Window Functions` (`ROW_NUMBER`, `RANK`, `LAG`, `LEAD`, running sums), `CASE WHEN`, `COALESCE`/`NULLIF`, subqueries (scalar / correlated), set operations (`UNION`/`INTERSECT`/`EXCEPT`), date functions (`DATE_TRUNC`, `DATEDIFF`), and string aggregation (`STRING_AGG`, `GROUP_CONCAT`).

If the user's intent is purely educational, such as "teach me GROUP BY", lead with Step 5 and use the rest as supporting context.

## Dialect Handling

Detect or accept a dialect and adjust. Quick rules — full notes in `references/dialect_differences.md`:

| Concept        | PostgreSQL / BigQuery / Snowflake       | MySQL                                    | SQL Server                                  |
| -------------- | --------------------------------------- | ---------------------------------------- | ------------------------------------------- |
| String agg     | `STRING_AGG(expr, ', ')`                | `GROUP_CONCAT(expr SEPARATOR ', ')`      | `STRING_AGG(expr, ', ')` (2017+)            |
| Date truncate  | `DATE_TRUNC('day', ts)`                 | `DATE_FORMAT(ts, '%Y-%m-%d')`            | `FORMAT(ts, 'yyyy-MM-dd')` / `DATETRUNC`    |
| Limit          | `LIMIT n`                               | `LIMIT n`                                | `SELECT TOP n ...` / `OFFSET ... FETCH`     |
| Boolean        | `TRUE`/`FALSE`                          | `TRUE`/`FALSE` (since 8.0)               | `1`/`0` or bit                               |
| ILIKE / case   | `ILIKE`                                 | `LIKE` with `LOWER(...)`                 | `LIKE` is case-insensitive by default       |

If unsure, write portable ANSI SQL and note one alternative dialect in Caveats.

## Output Format

Always reply in the user's input language. Default to English for English and Chinese for Chinese. The five sections must appear in this order, using these headings (and translated headings for non-English replies):

```
### SQL
[code block]

### Line-by-Line
[bulleted or numbered walkthrough]

### What This Output Means
[plain English explanation plus one example row]

### Caveats
[bulleted reminders]

### Mini-Classroom
[concept blocks, only the ones used in the query]
```

Keep each section focused and concise. Use short sentences. Prefer bulleted lists over long paragraphs. Highlight key SQL terms in backticks, for example `JOIN`, `GROUP BY`, `WHERE`.

When teaching-mode dominates, for example when the user asks "explain GROUP BY", reorder so the Mini-Classroom appears first and the SQL plus example serve as illustration.

## Bundled Resources

### references/sql_concepts.md

Concise reference for every concept taught in Step 5. Load it when the query uses a concept you are not 100% sure about, or when the user's question is explicitly about a concept. Each entry: definition, syntax skeleton, minimal example, common gotcha.

### references/dialect_differences.md

A quick-lookup table for function/syntax differences between PostgreSQL, MySQL, SQL Server, BigQuery, and Snowflake. Load before writing a query if the user named a dialect, or before Step 4 if a snippet might behave differently elsewhere.

### references/optimization_patterns.md

Patterns that reliably change a query from slow to fast: sargable predicates, covering indexes, avoiding functions on indexed columns, JOIN order, EXISTS vs IN, de-duplicating with window functions instead of self-joins. Load when the user asks "optimize this", when the query looks expensive, or when Step 4 needs a real performance note.
