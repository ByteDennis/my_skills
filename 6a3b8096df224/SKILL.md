---
id: 6a3b8096df224
name: dtrack: Oracle → Databricks (row + col)
tags: [dtrack, oracle, databricks, runbook, row, col, rss, public]
updated_at: 2026-06-24T12:12:37.020921Z
---

# dtrack: Oracle → Databricks (row + col)

>Left Oracle, right Databricks (Spark SQL) reconciliation log. Two unique points: `conn_macro` is `catalog.schema`; connection uses OAuth (`DB_PROFILE`+`DB_HTTPS`) + reuses the Athena proxy. The col stage borrows directly from Athena's SQL template (Spark and Trino dialects are close enough).<

Base pipeline see [[dtrack: CLI runbook|6a3b8031de3c0]]; cross-engine semantic gotchas see [[Data Engineer|6a05342d3068b]].

## Sample pair

There is currently **no** oracle→databricks pair in the repo (only the mock fixture `databricks_pos_daily`). To test, create one as shown below — left Oracle `pb30` `POS_DAILY`, right Databricks `pos_daily`:

```json+ testing/oracle_databricks.json
{
  "pairs": {
    "oracle_vs_databricks_pos": {
      "description": "Oracle DATE POS_DAILY vs Databricks pos_daily (day granularity).",
      "left":  { "name": "pos_daily", "table": "POS_DAILY", "source": "oracle",
                 "conn_macro": "pb30", "date_col": "POS_DT", "date_type": "date" },
      "right": { "name": "pos_daily", "table": "pos_daily", "source": "databricks",
                 "conn_macro": "main.analytics", "date_col": "pos_dt", "date_type": "date" },
      "mode": "incremental", "fromDate": "2024-10-01", "toDate": "2024-12-31"
    }
  }
}
```

> [!IMPORTANT] `conn_macro` = `catalog.schema`
> Databricks `conn_macro` must be dot-separated `catalog.schema` (e.g. `main.analytics`), ultimately resolving to `{catalog}.{schema}.{table}`.
> - Provide only a schema (`analytics`) → dtrack prepends `DB_DEFAULT_CATALOG` to form `{DB_DEFAULT_CATALOG}.analytics.pos_daily`.
> - Leave it entirely empty → falls back to the warehouse's default catalog.schema.

## Real env (`.env`)

```bash
# Oracle (left)
PCDS_USR=your_user
PB30_PWD=...                 # conn_macro=pb30 → looks up PB30_PWD
# Databricks (right)
DB_PROFILE=your_profile      # OAuth profile for databricks-sdk Config
DB_HTTPS=/sql/1.0/warehouses/xxxxxxxx   # http_path for SQL warehouse
DB_DEFAULT_CATALOG=main      # optional; prepended when conn_macro provides only a schema
# Proxy: reuses the Athena setup (HTTPS_PROXY only, no AWS token refresh)
AWS_USR=... ; AWS_PWD=... ; AWS_HOST=proxy.host
```

> [!NOTE] Connection internals (from `databricks.py`)
> `databricks_connect` obtains a Bearer token via `Config(profile=DB_PROFILE).authenticate()`, resolves the host from the profile, and sets `http_path=DB_HTTPS`. If `AWS_USR/PWD/HOST` are all present, sets `HTTPS_PROXY=http://usr:pwd@host:8080`. Does **not** run the AWS token dance — that was causing intermittent failures in the col_gen runner. Missing `DB_PROFILE`/`DB_HTTPS` raises `RuntimeError` immediately.

## Running (justfile)

```bash
just pair=oracle_databricks e2e row
just pair=oracle_databricks e2e col-gen     # review testing/csv/extract_col.databricks.sql
just pair=oracle_databricks e2e col-run
# all at once: just pair=oracle_databricks e2e all
```
Mock dry-run (validate the pipeline without a real database): set `DTRACK_MOCK=testing/mock` in `.env` — it copies `databricks_pos_daily_row.csv` as the result.

## What to observe

### Row stage
- [ ] `extract_row.databricks.sql`: table name is `main.analytics.pos_daily` (catalog.schema correctly assembled)
- [ ] `pos_dt` (Spark `DATE`) bucketed by day, window `2024-10-01..12-31`
- [ ] `run-sql` succeeds; on failure first check whether `DB_PROFILE`/`DB_HTTPS` is missing or the proxy is unreachable
- [ ] `compare_row.html`: same as Oracle→Athena — look for missing dates / count discrepancies

### Col stage
- [ ] col SQL uses the **Athena/Trino template** (`date_parse`/`date_trunc`/`approx_count_distinct`/`SUBSTR`) — Spark compatible
- [ ] Hash expression should be Databricks dialect 16-bit MD5:
  ```sql
  CAST(conv(SUBSTR(LOWER(md5(TRIM(col))), 1, 4), 16, 10) AS BIGINT)
  ```
  Numerically equivalent to Oracle's `int(md5(trim(value))[:4],16)` — this is the key to cross-engine `hash_sum` matching.
- [ ] `compare_col.html` four signal categories interpreted the same as [[dtrack: Oracle → Athena (row + col)|6a3b805a019b5]]

## Oracle ↔ Databricks specific gotchas

> [!WARNING] Timezone loss
> Databricks `TIMESTAMP` **has no timezone**, Oracle `TIMESTAMP WITH TIME ZONE` does. Comparing cross-timezone timestamps directly will drift. Impact on day-granularity reconciliation is small (both TRUNC to day), but if `date_col` contains time components and the two sides have different timezones, they can be off by one day — prefer pure `date`/normalize to day.

> [!CAUTION] Dialect divergence fallback
> The col stage borrows the Athena template; the vast majority of functions are available in Spark. If a particular expression is not recognized by Spark, attach a `vintage_transform` override on the pair to replace that SQL (see the header comment in `databricks.py`).

- **Empty string ≠ NULL** (same as Athena, opposite to Oracle) → when `hash_sum`+`length_avg` diverge together, check NULL handling first.
- **Catalog permissions**: `pos_daily` lives in `main.analytics` but the warehouse defaults to a different catalog → you must put the catalog in `conn_macro` or set `DB_DEFAULT_CATALOG`, otherwise "table not found".
- Suspect hashing: run [[dtrack: CSV → CSV + sample-hash|6a3b87c0905a7]] (supports fetching the same rows by key from the databricks side and computing hash16 cell-by-cell for verification).
