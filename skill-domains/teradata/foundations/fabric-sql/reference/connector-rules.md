# Fabric SQL — Connector-Specific Rules

## BigQuery

- Cross-project references (`project.dataset.table@fs`) are not supported — use a
  pass-through query (§7 in SKILL.md).
- Federated data sources are read-only.
- `STRING` arrives as `CLOB` (see `reference/data-types.md` for the LOB cast pattern).
  BigQuery-side `BLOB`/`CLOB` values are further limited to 100 MB.

## Generic JDBC (for example Amazon Redshift)

- Types map through JDBC metadata; maximum `VARCHAR` is 32k.
- User-defined types are not supported.

## Arrow Flight SQL (for example Dremio)

- **Import-only** — no remote joins.
- A single response message is limited to about 2 GB: filter, project fewer columns, or
  page the request.
- Three-part names need the catalog/schema portion quoted (`"catalog.schema".table`);
  names containing a dot are not supported.

## Hive / Spark SQL

- Spark has no `CHAR`/`VARCHAR` — string columns arrive as `STRING`.
- The Spark connector does not support ACID or transactional tables.
- Import is not supported on character columns whose table character set is not Latin or
  UTF-16.

## Teradata to Teradata

- All operations push down.
- Maximum `VARCHAR`: 64,000 characters (LATIN) / 32,000 (UNICODE).
- `WITH` clauses **are** supported inside `FOREIGN TABLE` pass-through bodies on
  Teradata-to-Teradata links (unlike other connectors).
