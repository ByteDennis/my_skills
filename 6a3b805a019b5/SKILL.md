---
id: 6a3b805a019b5
name: dtrack: Oracle → Athena (row + col)
tags: [dtrack, oracle, athena, runbook, row, col, rss, public]
updated_at: 2026-06-24T12:12:36.874062Z
---

# dtrack: Oracle → Athena (row + col)

>Hands-on log comparison with Oracle on the left and Athena on the right. Key point: `date_type` typically differs between the two sides (Oracle `timestamp`/`date` ↔ Athena `string_dash`/`num`); dtrack normalizes both to the same date bucket before comparing.<

See [[dtrack: CLI runbook|6a3b8031de3c0]] for the base workflow; see [[Data Engineer|6a05342d3068b]] for cross-engine semantic pitfalls.

## Sample pair

`oracle_ts_vs_aws_str` from `testing/pairs.json` — Oracle `TIMESTAMP` (including 00:00:00) vs Athena `VARCHAR` in `YYYY-MM-DD`:

```json
"oracle_ts_vs_aws_str": {
  "left":  { "name": "ts_cust_daily", "table": "CUST_DAILY_VW", "source": "pcds",
             "conn_macro": "pcds", "date_col": "RPT_TS", "date_type": "timestamp" },
  "right": { "name": "str_cust_daily", "table": "cust_daily", "source": "aws",
             "conn_macro": "analytics_db", "date_col": "rpt_dt", "date_type": "string_dash" },
  "mode": "incremental", "fromDate": "2024-10-01", "toDate": "2024-12-31"
}
```

> [!NOTE] `source: "pcds"` vs `"oracle"`
> Both go through the Oracle extractor; `pcds` is a named macro. The Athena right side's `conn_macro` is the **Glue database name** (here `analytics_db`), and `table` is a table within it.

## Live env (`.env`)

```bash
# Oracle (left)
PCDS_USR=your_user
PCDS_PWD=...                 # conn_macro=pcds → looks up PCDS_PWD
# Athena (right)
AWS_DEFAULT_REGION=us-east-1
AWS_S3_WORK_GROUP=primary
AWS_S3_STAGING_DIR=s3://your-bucket/athena-output/
AWS_USR=... ; AWS_PWD=...
```
Run `dtrack doctor` first to confirm `PCDS_USR=set`, `pcds_pwd` is not MISSING, and all four AWS variables are present.

## Running (justfile)

```bash+ Recommended: run only this one pair
# Place the pair above in testing/oracle_athena.json, then:
just pair=oracle_athena e2e row          # row-count phase
just pair=oracle_athena e2e col-gen      # generate col SQL, pause to review
#   → review testing/csv/extract_col.aws.sql and extract_col.oracle.sql
just pair=oracle_athena e2e col-run      # execute → load → compare
# or run everything at once: just pair=oracle_athena e2e all
```

For raw command reference see "Full command sequence" in [[dtrack: CLI runbook|6a3b8031de3c0]].

## What to observe

### Row phase
- [ ] `extract_row.oracle.sql`: `RPT_TS` is normalized to day via `TRUNC(...)`/`CAST AS DATE`, window `2024-10-01..12-31`
- [ ] `extract_row.aws.sql`: `rpt_dt` (string `YYYY-MM-DD`) compared via `date_parse`/direct comparison, consistent window
- [ ] `compare_row.html`:
  - **Dates present on only one side** → extraction window or source data gap
  - **Same-day count mismatch** → real row-count difference; note the dates for focus in the col phase
- [ ] `run-sql` last line `N ok, M failed`; Athena failures commonly caused by staging dir / workgroup permissions

### Col phase
- [ ] Col SQL dates are **clipped to the intersection of dates confirmed present on both sides** (the iron rule in action)
- [ ] `compare_col.html` — look at each column for four signal types:

| Signal | Most likely cause |
|---|---|
| `n_total` / `n_missing` differ | Row count or NULL definition inconsistency |
| Only `length_avg` differs, `hash_sum` same | **LENGTH character/byte semantics** differ (see [[Data Engineer|6a05342d3068b]]) |
| `hash_sum` and `length_avg` **both** differ | NULL/sentinel mapping inconsistency |
| Only `hash_sum` differs | Values themselves differ → run [[dtrack: CSV → CSV + sample-hash|6a3b87c0905a7]] to locate |

## Oracle ↔ Athena-specific pitfalls

> [!WARNING] Asymmetric date types
> Oracle `TIMESTAMP` carries hours/minutes/seconds; the Athena side usually has plain date strings. `date_type` must be set accurately (`timestamp` vs `string_dash`) so dtrack can TRUNC both sides to day. Wrong values cause the entire table to appear "mismatched".

> [!CAUTION] Athena partitions
> If `rpt_dt` is a partition column, a WHERE clause that misses it will cause a full table scan — slow and expensive. Confirm that the WHERE in `extract_row.aws.sql` hits the partition column.

- **Empty string = NULL only holds in Oracle**: when `hash_sum`/`length_avg` both differ, suspect this first (Athena `''≠NULL`).
- **`SUBSTR(...,1,4000)` truncation**: long text on the Oracle side is truncated to 4000 characters for length calculation; if any values exceed that, `length_avg` will be skewed — confirm no column has values >4000 characters.
- **Hash byte consistency**: MD5 operates on bytes on both sides, so identical bytes always produce identical values; divergence comes only from pre-processing steps like TRIM/LENGTH/NULL handling (see the hash section in [[Data Engineer|6a05342d3068b]]).
