# Fabric SQL — Data Types, LOB Handling, and Dates

## Source-to-Teradata Type Mapping

The type reported by `get_schema_details` is the best available signal of the mapping, but
its form depends on the connector: Teradata-connector sources report Teradata HELP type
codes (`CV` = VARCHAR, `I8` = BIGINT, `DA` = DATE), and Generic JDBC sources may report the
source's own `TYPE_NAME` (for example Redshift `int4`, `int2`, `numeric`). When the
reported value is not a Teradata type you recognise, use this table to translate it before
writing casts.

| Source type | Exposed as |
|---|---|
| `BIGINT`, `INTEGER` | `BIGINT`, `INTEGER` |
| `TINYINT`, `SMALLINT` | `SMALLINT` |
| `BOOLEAN`, `BIT` | `BYTEINT` — compare with `= 1` / `= 0`; do not use `TRUE`/`FALSE` literals |
| `DECIMAL`, `NUMERIC` | `DECIMAL` / `NUMBER` |
| `REAL`, `FLOAT`, `DOUBLE` | `DOUBLE PRECISION` |
| `VARCHAR`, `NVARCHAR`, `ARRAY` | `VARCHAR` (Unicode) |
| `CLOB`, `NCLOB`; BigQuery `STRING`; Arrow UTF-8 strings | `CLOB` — treat as a LOB (see below). BigQuery `STRING` arrives as `VARCHAR` instead when the connector's LOB support is disabled; the reported type tells you which. |
| Arrow binary | `BLOB` |
| `DATE`, `TIME`, `TIMESTAMP` | `DATE`, `TIME`, `TIMESTAMP` |
| Teradata to Teradata | unchanged |

**Key limits:**
- Maximum `VARCHAR`: 64,000 characters (LATIN); **32,000 for UNICODE** (2 bytes per character
  against the 64,512-byte row limit). 32k through Generic JDBC.
- `BLOB`/`CLOB` limited to just under 2 GB.
- Standard Teradata result limits apply to the outer query — at most 2048 returned columns.

---

## LOB Columns: Cast Before Grouping, Ordering, or Joining

A column exposed as `CLOB`/`BLOB` fails in `GROUP BY`, `ORDER BY`, `DISTINCT`, and join
conditions. BigQuery `STRING` columns are the most common case — they arrive as `CLOB`.
Wrap the foreign table in a derived table that casts the needed columns to `VARCHAR(n)`:

```sql
-- FAILS AT RUNTIME — continent_name arrives as CLOB and cannot be grouped
SELECT c.continent_name, COUNT(*)
FROM bigquery_public_view_country.country_codes@bqfs c
GROUP BY c.continent_name;

-- VALID — cast LOB columns to VARCHAR inside a derived table, then use the casts
SELECT dt.country_name, COUNT(*)
FROM (
    SELECT CAST(country_name AS VARCHAR(500)) AS country_name,
           CAST(continent_name AS VARCHAR(500)) AS continent_name
    FROM bigquery_public_view_country.country_codes@bqfs
) dt
GROUP BY dt.country_name;
```

**Choose `n` to fit the actual data.** Casting a value longer than `n` errors or truncates
it, and truncated values that share the retained prefix collapse into the same group or join
key — wrong results, not just shorter strings. `VARCHAR(64000) CHARACTER SET UNICODE` fails
with error 3798. When no fitting `n` exists, do the grouping or join on the source instead
with a pass-through query (§7 in SKILL.md).

---

## Controlling Result Types

Use `CAST` in the select list to control the result type. Use a `RETURNS` clause to control
string length and character set (suffix form `database.table@foreign_server` only; not
supported on Teradata-to-Teradata links):

```sql
-- VALID — explicit VARCHAR length via RETURNS
SELECT c_vchar FROM demo.testtabl@fs1 RETURNS(c_vchar VARCHAR(200));
```

---

## Cast Both Sides Across Type Boundaries

When a join or filter compares columns whose exposed types differ, cast both sides to one
common type:

```sql
-- VALID — VARCHAR compared with what was a CLOB; both sides cast to VARCHAR(100)
SELECT p.player_name, c.alpha_3_code
FROM spider2_IPL.player@to103dbc p
JOIN bigquery_public_view_country.country_codes@bqfs c
    ON CAST(c.country_name AS VARCHAR(100)) = CAST(p.country_name AS VARCHAR(100));
```

---

## Dates, Times and Time Zones

- Fabric transfers all time values in UTC.
- **Use explicit time zones** (`TIMESTAMP WITH TIME ZONE`) when the initiating and remote
  systems have different time zone settings. An implicit timestamp is interpreted using
  system defaults — the data type, the `TimeDateWZControl` database flag, and the session
  time zone — and can be shifted or mismatched.
- A documented failure mode: a remote system at UTC+03:00 against an initiator with
  `TimeDateWZControl` disabled caused pushdown queries to fail because each system
  interpreted the same timestamp differently.
- Date literals in `WHERE` clauses are **not converted** to the remote system's time zone
  when it differs from the initiator's. Test date predicates; prefer explicit casts on
  both sides.
- If a date or timestamp filter returns suspicious results, cast both sides explicitly, or
  run the filter as a pass-through query (§7 in SKILL.md) in the remote engine's own date
  syntax.

```sql
-- VALID — explicit type and time zone on the filter literal
SELECT o.id, o.order_ts
FROM sales.orders@pg_sales o
WHERE o.order_ts >= TIMESTAMP '2025-01-01 00:00:00+00:00';
```
