# Fabric SQL — Dialect Rewrite Reference

The outer query must be Teradata SQL even though the data is remote. Rewrite these
constructs before submitting.

## Rewrite Table

| ANSI / other-dialect construct | Fabric SQL |
|---|---|
| `SELECT ... LIMIT n` | `SELECT TOP n ...` (`LIMIT` is not Teradata syntax; see the pushdown caution in §3 of SKILL.md) |
| `x::type` casts | `CAST(x AS type)` |
| `ILIKE` | `LOWER(x) LIKE LOWER(pattern)` |
| `DATE_TRUNC('month', d)` | `TRUNC(d, 'MM')` |
| `DATEADD` / `DATEDIFF` | Date arithmetic (`d + 30`, `d1 - d2` gives days), `ADD_MONTHS(d, n)`, or `INTERVAL` expressions |
| `NOW()` | `CURRENT_TIMESTAMP` |
| `IFNULL(x, y)`, `NVL(x, y)` | `COALESCE(x, y)` |
| `IIF(c, a, b)` | `CASE WHEN c THEN a ELSE b END` |
| `LEN(x)` (SQL Server) | `CHARACTER_LENGTH(RTRIM(x))` — `LEN` excludes trailing spaces, `CHARACTER_LENGTH` counts them; `RTRIM` preserves `LEN` semantics |
| `CHARINDEX(s, x)` | `POSITION(s IN x)` |
| `TRUE` / `FALSE` literals | `1` / `0` (booleans arrive as `BYTEINT`, see `reference/data-types.md`) |
| `CAST(x AS VARCHAR)` with no length | Always give a length: `CAST(x AS VARCHAR(100))` |
| `REPLACE(s, a, b)` string replacement | `OREPLACE(s, a, b)` — **critical**: `REPLACE` is on the read-only denylist (§8 of SKILL.md) and is rejected even as a function call |
| `LIMIT n OFFSET m` with nonzero `m` | `TOP` cannot skip rows — use `ROW_NUMBER() OVER (ORDER BY ...)` filtered with `QUALIFY` to the requested window |

**Unchanged:** `\|\|` string concatenation, `DATE 'YYYY-MM-DD'` literals, ANSI window
functions, `UNION` / `INTERSECT` / `EXCEPT` (`MINUS` is also available), CTEs (`WITH`).

---

## Semantics That Change Results

**Case sensitivity.** Teradata mode compares strings case-insensitively by default; many
remote engines compare case-sensitively. The same predicate can therefore match different
rows depending on where it runs. Use `UPPER(x) = UPPER(y)` when the comparison must be
case-insensitive on both sides.

**Trailing spaces.** Teradata pads the shorter operand when comparing strings. Trim or cast
values when matching across systems.

**Teradata-only clauses** (`QUALIFY`, `SAMPLE`, `TOP`) are valid in the outer query — they
are evaluated on the Compute Engine — but they are not portable to a non-Teradata source,
so do not place them where they must execute remotely.

---

## Examples

```sql
-- ANSI (PostgreSQL-style)
SELECT order_id, amount
FROM sales.orders
WHERE order_date >= DATE '2025-01-01'
LIMIT 100;

-- VALID — LIMIT rewritten to TOP, reference routed to redshift1 (Generic JDBC)
SELECT TOP 100 order_id, amount
FROM sales.orders@redshift1
WHERE order_date >= DATE '2025-01-01';

-- ANSI — second page of 25 rows (LIMIT 25 OFFSET 25)
SELECT order_id, amount
FROM sales.orders
ORDER BY order_id
LIMIT 25 OFFSET 25;

-- VALID — nonzero offset needs ROW_NUMBER + QUALIFY; TOP cannot skip rows.
-- The ORDER BY inside OVER() makes the window deterministic; keep the trailing
-- ORDER BY for presentation order.
SELECT order_id, amount
FROM sales.orders@redshift1
QUALIFY ROW_NUMBER() OVER (ORDER BY order_id) BETWEEN 26 AND 50
ORDER BY order_id;
```
