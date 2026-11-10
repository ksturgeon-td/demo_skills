# Fabric SQL — Error Diagnosis

The error text returned with a `failed` job is the source system's or Teradata's own
message. Use it directly — it identifies the specific problem.

| Symptom | Likely cause | Fix |
|---|---|---|
| Syntax error near `LIMIT`, `::`, `ILIKE` | Non-Teradata syntax in the outer query | Rewrite per §6 / `reference/dialect-rewrite.md` |
| Syntax error near a backticked or reserved remote name | Teradata is parsing remote syntax | Use `FOREIGN TABLE` with `@BEGIN_PASS_THRU` (§7 in SKILL.md) |
| Error 3706 "ORDER BY is not allowed in subqueries" from a pass-through body | The target (Teradata) received the body as a subquery | Move `ORDER BY` to the outer query, or add `TOP n` inside the body if a bounded remote ordering is intended (§7) |
| "`REPLACE`/`TRUNCATE`/`CALL`/... statements not allowed" on a `SELECT` that only calls a function or names a column | The read-only denylist matches the word anywhere in the statement, including function names and unquoted identifiers (§8 in SKILL.md) | Use `OREPLACE` instead of `REPLACE` (§6); double-quote reserved-word identifiers: `"call"` |
| Object does not exist (3807) or access denied on a table | Wrong foreign server, database or table; missing grants; unqualified reference resolved to the connector default database | Re-check with `get_schema_details`; qualify as `database.table@foreign_server`; never guess a server |
| Whole `SELECT *` fails | One column has an unsupported type | Name the supported columns only (§4 in SKILL.md) |
| Prefix form rejected | Engine older than 20.00.31.76, or the context is `FOREIGN TABLE` / `RETURNS` | Use the suffix form `database.table@foreign_server` |
| `GROUP BY` / `ORDER BY` / `DISTINCT` / join fails on a string column | Column arrived as `CLOB` | Cast in a derived table (`reference/data-types.md`) |
| Query is very slow or returns far more rows than expected | Filter or `TOP n` not pushed down | §3 in SKILL.md; use a pass-through query when the work must run remotely (§7) |
| Wrong rows on a date filter | Time zone or DATE comparison issue | §5 / `reference/data-types.md` — explicit casts and time zones, or pass-through |
| Truncated strings | Unbounded remote string into a bounded Teradata type | `CAST` or `RETURNS` with a larger length (`reference/data-types.md`) |
| Error 3798 on a `VARCHAR(64000) CHARACTER SET UNICODE` | Unicode chars occupy 2 bytes; 64,000 × 2 exceeds the 64,512-byte row limit | Use `VARCHAR(32000) CHARACTER SET UNICODE` or `VARCHAR(64000) CHARACTER SET LATIN` |

---

## Cleanup Note

Fabric creates temporary tables or views on the target database while a query runs and
cleans them up when it finishes. After an aborted query, an administrator may need to clean
up manually.
