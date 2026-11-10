---
name: fabric-sql
description: Translates ANSI SQL to Teradata Fabric SQL — routes queries across foreign servers (PostgreSQL, Oracle, BigQuery, Hive, Teradata, and more). Covers table addressing, pushdown rules, pass-through queries, the read-only validation gate, and execution lifecycle.
---

# ANSI SQL to Fabric SQL

## Purpose

Generate **Fabric SQL**: Teradata SQL in which every remote table reference names the
**foreign server** that holds the table. A foreign server is a named route to one data
source (PostgreSQL, Oracle, BigQuery, Hive, another Teradata system, and so on).

Content is derived from the *Teradata Fabric — Installation and User Guide 3.11*
(B035-5991-086K) and validated against a live fabric. Follow the rules exactly. Every rule
has a worked example. Example statements are labelled:

- **VALID** — passes the service's validation gate and runs successfully.
- **REJECTED** — refused by the service before execution. Never submit these.
- **FAILS AT RUNTIME** — passes validation but errors on the engine. Do not submit these.

> **Status note.** `execute_fabric_sql`, `get_fabric_sql_results` and `cancel_fabric_sql`
> run your query and return its rows (§10). `get_schema_details` is not implemented yet: it
> returns `not_implemented` and no table matches, so until it is, take table references
> from your user rather than guessing them.

**Tool naming.** Tools are named here in bare form (for example `execute_fabric_sql`). When
you reach this service through a gateway, every tool name carries a prefix of the form
`{targetName}___{toolName}` — three underscores, for example
`fabric-mcp___execute_fabric_sql`. Call the prefixed names exactly as the gateway's
`tools/list` advertises them. The SQL rules below are identical either way.

---

## Activation

Invoke this skill when a user asks to:
- Query data through a Teradata Fabric service
- Write or fix Fabric SQL targeting foreign servers
- Translate existing SQL (PostgreSQL, Oracle, BigQuery, etc.) to run through Fabric
- Understand why a Fabric SQL query failed or returned unexpected results

---

## Response Style

Default to plain English. Many users asking about Fabric queries are not Teradata SQL experts.

**Opening each interaction:**
Briefly explain in one sentence what you are about to do before generating any SQL. After
producing a query, summarize what it does and any important caveats before showing the SQL.

**Technical depth:**
Ask once, early: *"Would you like me to explain the SQL as we go, or just show you the final query?"*
- **Show SQL detail** when: the user asks for it, something fails and you need to debug, or
  the user is stepping through a translation manually.
- **Show final query only** otherwise.

**When in doubt:** assume the user wants a working query, not a SQL tutorial.

---

## 1. Workflow

1. **Find where each table lives.** Call the `get_schema_details` tool with a
   natural-language question. Each match returns `foreignServer`, `database`, `table`, a
   ready-to-use `tableReference` in the form `database.table@foreign_server`, and the
   columns with their reported types (how to read them: §4). Never guess a foreign server.
2. **Rewrite each table reference** with its foreign server (§2).
3. **Check pushdown** (§3) so work stays on the remote engine.
4. **Check data types, dates and time zones** (§4, §5) and **rewrite dialect differences**
   (§6). Use a pass-through query when you need source-native syntax (§7).
5. **Submit** (§8–§10). If the query fails, read the returned error text and fix the
   specific problem (§12).

---

## 2. Addressing remote tables

Two equivalent forms reference a remote table through a foreign server:

| Form | Syntax | Notes |
|---|---|---|
| Suffix | `<database>.<table>@<foreign_server>` | Works on all supported engine versions. **Required** inside `FOREIGN TABLE` pass-through queries and `RETURNS` clauses. This is the form `get_schema_details` returns — use it. |
| Prefix | `<foreign_server>.<database>.<table>` | Equivalent, but requires Teradata Database 20.00.31.76 or later. |

Rules:

- Always qualify with the database: `database.table@foreign_server`, not bare
  `table@foreign_server`. Without a database, the current session database or the
  connector's default database is used; if the table is not there the query fails with
  error 3807. Do not rely on the default.
- Do not mix the suffix and prefix forms for the same table in one query.
- Joins across different foreign servers are supported in one statement.
- A table on the local Teradata system takes no suffix: `database.table`.
- Views can be built on remote tables.

```sql
-- ANSI
SELECT c.name, o.total
FROM sales.customers c
JOIN sales.orders o ON o.customer_id = c.id
WHERE o.total > 1000;

-- VALID — both tables live behind foreign server pg_sales (suffix form)
SELECT c.name, o.total
FROM sales.customers@pg_sales c
JOIN sales.orders@pg_sales o ON o.customer_id = c.id
WHERE o.total > 1000;

-- VALID — identical query in prefix form
SELECT c.name, o.total
FROM pg_sales.sales.customers c
JOIN pg_sales.sales.orders o ON o.customer_id = c.id
WHERE o.total > 1000;

-- FAILS AT RUNTIME — no database qualifier; resolves against the connector default
-- database and raises error 3807 when orders is not in it
SELECT * FROM orders@pg_sales;
```

---

## 3. Pushdown: what runs on the source and what runs locally

The initiating Teradata system (the Compute Engine) plans the query and pushes each filter
and supported function to the remote engine when everything that operation touches lives on
that one foreign server. Anything it cannot push down is evaluated on the Compute Engine
**after the data has been transferred** — slower and more costly.

Operations pushed down, per connector:

| Target connector | Pushed down |
|---|---|
| Teradata | All operations |
| Hive | Eq, Ne, Gt, Ge, Lt, Le, And, Or, Not, Between, In, NotIn, Like, IsNull, Case, Abs, Extract, Ln, Log, Sqrt, Concat, Lower, Upper, Plus, Minus, Mult, Div |
| Spark SQL | As Hive, without Extract, Concat, Lower and Upper |
| Oracle | As Hive, without Log |
| BigQuery, Generic JDBC (for example Redshift), Arrow Flight SQL | The comparison, logical, arithmetic and string operations above, plus Cast, Mod, Substring and Trim |

Practical rules:

1. Put selective filters in the `WHERE` clause directly on the remote table's own columns,
   using only the operations listed for that connector, so they travel with the query.
2. Select only the columns you need. Avoid `SELECT *` on remote tables (see also §4).
3. Avoid wrapping join or filter columns in function calls when an unwrapped form exists —
   a wrapped column can block pushdown and force local evaluation over transferred rows.
4. Aggregations over **multiple remote tables** are not pushed to the remote system, even
   when the tables sit on the same source. Filter each side first where you can.
5. Remote joins (Remote Table Optimization) require **inner joins with equality predicates**.
6. Arrow Flight SQL connectors are import-only: remote joins are not supported through them.
7. `TOP n` pushdown is not supported on most connectors — a row limit on a remote table may
   still pull the full result before truncating. When the limit must run on the remote
   engine, use a pass-through query (§7).

```sql
-- VALID and fully pushed: filter and join execute on pg_sales; only results move
SELECT c.name, o.total
FROM sales.customers@pg_sales c
JOIN sales.orders@pg_sales o ON o.customer_id = c.id
WHERE o.total > 1000;

-- VALID but split: each side is filtered at its source; the join runs on the
-- Compute Engine. The selective predicates sit on each remote table's own columns.
SELECT c.name, o.total
FROM sales.customers@pg_sales c
JOIN sales.orders@ora_sales o ON o.customer_id = c.id
WHERE c.region = 'EU'
  AND o.total > 1000;
```

---

## 4. Data types across source systems

Every source type is mapped to a Teradata type when exposed through a foreign server.
Key rules:

- Data is converted when it moves between systems — unbounded remote strings imported into
  a bounded Teradata `VARCHAR` are **truncated**.
- **One unsupported column fails the whole `SELECT *`.** Name the columns you need.
- **LOB columns (`CLOB`/`BLOB`) cannot be used in `GROUP BY`, `ORDER BY`, `DISTINCT`, or
  join conditions.** BigQuery `STRING` columns are the most common case — they arrive as
  `CLOB`. Cast them to `VARCHAR(n)` inside a derived table first.
- Probe with a small `SELECT ... SAMPLE 1` when unsure how a value actually arrives.
- Control result types with `CAST` in the select list, and string length/character set
  with a `RETURNS` clause (suffix form only).

Load `reference/data-types.md` for the full source-to-Teradata type mapping table,
LOB cast patterns, and cross-type comparison examples.

---

## 5. Dates, times and time zones

- Fabric transfers all time values in UTC.
- **Use explicit time zones** (`TIMESTAMP WITH TIME ZONE`) when the initiating and remote
  systems have different time zone settings — an implicit timestamp can be shifted or
  mismatched depending on `TimeDateWZControl` and session settings.
- Date literals in `WHERE` clauses are **not converted** to the remote system's time zone —
  test date predicates; prefer explicit casts on both sides.

Load `reference/data-types.md` for date filter examples and known timezone failure modes.

---

## 6. Dialect rewrite: ANSI and other-dialect constructs to Teradata

The statement runs on Teradata, so the outer query must be Teradata SQL even though the
data is remote. The most critical rewrites:

| ANSI / other-dialect | Fabric SQL | Why it matters |
|---|---|---|
| `SELECT ... LIMIT n` | `SELECT TOP n ...` | `LIMIT` is not Teradata syntax |
| `x::type` casts | `CAST(x AS type)` | PostgreSQL cast syntax is rejected |
| `REPLACE(s, a, b)` | `OREPLACE(s, a, b)` | `REPLACE` hits the read-only denylist (§8) |
| `LIMIT n OFFSET m` | `ROW_NUMBER() OVER (...) QUALIFY` | `TOP` cannot skip rows |

Load `reference/dialect-rewrite.md` for the full rewrite table, semantics that change
results (case sensitivity, trailing spaces), and additional examples.

---

## 7. Pass-through queries: `FOREIGN TABLE`

Use `FOREIGN TABLE` to send SQL to the remote engine unparsed, written in **that engine's
dialect**. Use it when you need source-native syntax (a remote `LIMIT`, source-specific
functions) or identifiers that Teradata cannot parse.

```sql
-- VALID — the inner query is Hive SQL, sent to hive1 as written
SELECT *
FROM FOREIGN TABLE (
       SELECT make, model
       FROM default.cardata
       WHERE make = 'Buick'
     )@hive1 AS dt;

-- VALID — same query with a remote LIMIT, executed on Redshift
SELECT *
FROM FOREIGN TABLE (
       SELECT order_id, amount
       FROM sales.orders
       WHERE order_date >= DATE '2025-01-01'
       LIMIT 100
     )@redshift1 AS dt;

-- VALID — @BEGIN_PASS_THRU/@END_PASS_THRU stops Teradata parsing a remote reserved
-- word that needs backticks
SELECT *
FROM FOREIGN TABLE (@BEGIN_PASS_THRU
       SELECT `current`, `name`
       FROM testdb.testrk
     @END_PASS_THRU)@hive1 AS dt;
```

Rules:

- The `@foreign_server` **suffix** goes after the closing parenthesis. The prefix form is
  not allowed here.
- A `WITH` clause cannot be used **inside** the pass-through query, except on
  Teradata-to-Teradata links.
- The pass-through body executes **as a subquery on the target**. Constructs the target
  forbids in a subquery fail even inside `@BEGIN_PASS_THRU` markers — on a Teradata
  target, a bare `ORDER BY` in the body is rejected (error 3706). `ORDER BY` combined with
  `TOP n` is allowed; for unbounded ordering, apply `ORDER BY` in the outer query instead.
- Qualify table names with the catalog or schema as the remote engine expects.
- The service's read-only gate (§8) scans the whole statement, including the pass-through
  body: a write statement hidden inside `FOREIGN TABLE` is still rejected.

---

## 8. What this service accepts — the validation gate

> Items 1–3 are checked before anything runs: a statement that fails them is rejected with
> `invalid_sql` and never reaches the engine.

`execute_fabric_sql` accepts a statement only if all of the following hold:

1. **Single statement.** One statement per call. A single trailing `;` is optional and
   tolerated; any other semicolon is rejected. Comments are allowed; a word such as
   `UPDATE` inside a string literal or comment is not mistaken for a statement.
2. **Read-only.** The statement must begin with `SELECT`, `SEL`, or `WITH`. `INSERT`,
   `UPDATE`, `DELETE`, `MERGE`, `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `REPLACE`, `GRANT`,
   `REVOKE`, `CALL`, `EXEC`/`EXECUTE`, `READ_NOS`, and `WRITE_NOS` are rejected wherever
   they appear — including inside a `FOREIGN TABLE` pass-through body, as a function call
   (`REPLACE(x, a, b)` → use `OREPLACE`), or as an unquoted identifier (double-quote it:
   `"update"`). String literals and comments are masked before the scan.
3. **`EXPLAIN` and `SHOW` are not accepted** — the allowlist admits only `SELECT`/`SEL`/
   `WITH`. Validate by reading the error text of a failed execution (§12).
4. **Authorization is the database's, not the service's.** The query executes as *you* and
   can read only what your database user has been granted.

```sql
-- REJECTED — two statements
SELECT id FROM sales.orders@pg_sales; SELECT id FROM sales.customers@pg_sales;

-- REJECTED — not read-only
DELETE FROM sales.orders@pg_sales WHERE total < 0;

-- VALID — WITH is accepted, comments are allowed, trailing semicolon optional
WITH big_orders AS (
    SELECT id, total FROM sales.orders@pg_sales WHERE total > 1000
)
SELECT COUNT(*) AS n FROM big_orders; -- one statement, one query
```

---

## 9. `ORDER BY`, `TOP` and `SAMPLE` run exactly as written

The service runs your statement as written: nothing is removed, rewritten or re-applied.
A trailing `ORDER BY` orders the result across every batch. Without an `ORDER BY`, row
order is whatever the engine produces and can differ between runs. Add one whenever order
matters, and always with `TOP n`.

```sql
-- VALID — rows arrive ordered by total, highest first, across all batches
SELECT id, total FROM sales.orders@pg_sales ORDER BY total DESC;

-- VALID — TOP with ORDER BY returns the 10 largest orders
SELECT TOP 10 id, total FROM sales.orders@pg_sales ORDER BY total DESC;
```

---

## 10. Execution lifecycle — submit, fetch every batch, cancel or release

Submit with `execute_fabric_sql(sql)`. The call **waits** up to `timeoutSeconds` (default
600, maximum 780) for the query to finish. Keep `timeoutSeconds` **below both your client's
and the gateway's request deadlines**, with room for response delivery. The two outcomes:

- **Completed in time** → the response *is* the first batch:
  `{ taskId, status: "completed", columns, rows, batchRowCount, totalRowCount,
  nextBatchToken, expiresAt }`, in your `ORDER BY` order.
- **Not finished at the timeout** → the query is cancelled and the call fails with
  `query_timeout`. Narrow the query or raise `timeoutSeconds`, and run it again.

Results are **batched** — never assume one response holds the whole result:

1. If `nextBatchToken` is **null**, you have every row. If it is **non-null**, more rows
   exist — call `get_fabric_sql_results(taskId, batchToken: <token>)` and repeat until
   `nextBatchToken` comes back null.
2. Re-presenting the previous token replays the last batch (retry-safe). Any other token
   fails with `invalid_batch_token`.
3. `batchSize` is optional (default 1000, max 10000) on both submit and fetch.
4. `cancel_fabric_sql(taskId)` releases a result early.

Results are held for a sliding TTL (15 minutes by default, 4 hours maximum via
`resultTtlSeconds` at submit); every `get_fabric_sql_results` call restarts it.

---

## 11. Connector-specific rules

Load `reference/connector-rules.md` for rules specific to BigQuery, Generic JDBC
(Redshift), Arrow Flight SQL (Dremio), Hive/Spark, and Teradata-to-Teradata links.

---

## 12. Reading and acting on errors

The error text returned with a `failed` job is the source system's or Teradata's own
message — read it directly, it identifies the specific problem.

Load `reference/error-guide.md` for the full error diagnosis table (symptom → likely
cause → fix).

---

## 13. Column classifications

> Enforcement note: these flags are not yet applied. Results currently carry no
> `sensitive` labels and `visible` flags are not honoured — treat every returned column as
> unclassified.

Schema results may mark a column `sensitive: true`. Avoid `SELECT *`, mask it when
displaying, and keep it out of logs. Columns marked not visible are never returned at all.

---

## 14. Pre-submit checklist

1. Every remote table reference is `database.table@foreign_server` from
   `get_schema_details` — no invented server names, no bare `table@server` (§2).
2. Cross-server joins give each table its own server; selective filters sit on each remote
   table's own columns and use operations that push down for that connector (§3).
3. LOB-typed columns are cast to `VARCHAR(n)` in a derived table before grouping, ordering,
   deduplicating, or joining; cross-type comparisons cast both sides (§4).
4. No `LIMIT`, `::`, `ILIKE`, `REPLACE()`, or other non-Teradata syntax outside a
   `FOREIGN TABLE` pass-through body (§6, §7).
5. Date and timestamp filters use explicit types and time zones (§5).
6. One statement, beginning with `SELECT`, `SEL`, or `WITH`; at most one trailing `;`; no
   `EXPLAIN` (§8).
7. When row order matters, the statement ends with an `ORDER BY`, as with every `TOP n` (§9).
8. After submit: fetch every batch until `nextBatchToken` is null; cancel or release if
   unneeded (§10).

---

## Reference Files

| Need | Load this file |
|------|----------------|
| Source-to-Teradata type mapping, LOB cast patterns, date/timezone examples | `reference/data-types.md` |
| Full ANSI → Teradata rewrite table, case sensitivity, trailing spaces | `reference/dialect-rewrite.md` |
| Connector-specific rules (BigQuery, Redshift, Dremio, Hive/Spark, Teradata) | `reference/connector-rules.md` |
| Error diagnosis table (symptom → cause → fix) | `reference/error-guide.md` |
