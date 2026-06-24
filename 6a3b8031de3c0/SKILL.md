---
id: 6a3b8031de3c0
name: dtrack: CLI runbook
tags: [dtrack, cli, runbook, data-engineering, rss, public]
updated_at: 2026-06-24T12:12:36.694936Z
---

# dtrack: CLI runbook

>Unified mental model for running dtrack on real machines: generate → run-sql → load → compare, split into two phases: row / col. All platforms share the same commands; only the `.env` and pair config differ.<

This card is the foundation; scenario cards all branch from here:
[[dtrack: Oracle → Athena (row + col)|6a3b805a019b5]] · [[dtrack: Oracle → Databricks (row + col)|6a3b8096df224]] · [[dtrack: CSV → CSV + sample-hash|6a3b87c0905a7]]
Deep dives: [[dtrack: date_type · vintage · config|6a3bbe771bede]] · [[dtrack: column match / stats preset|6a3bbe7736cb9]] · [[dtrack: troubleshoot|6a3bbe7770c4a]] · [[dtrack: maintenance / develop codebase map|6a3bbe7764659]]
Cross-engine semantic pitfalls: see [[Data Engineer|6a05342d3068b]]; web/code internals: see [[dtrack: general knowledge|6a3a7b11b439b]].

## Mental model: 4 steps × 2 phases

dtrack splits "data reconciliation" into **row (row-count alignment)** and **col (column-stats alignment)** phases. Each phase follows the same 4 steps:

| Step | Command | Purpose | Output |
|---|---|---|---|
| ① Generate | `dtrack generate … --type row` | Translate pair into platform SQL, **no execution** | `csv/extract_row.<platform>.sql` |
| ② Execute | `dtrack run-sql … --type row` | Run the SQL against the real database | `csv/{qname}_row.csv` |
| ③ Load | `dtrack load-row … csv/` | CSV → SQLite `project.db` | `_row_comparison` table |
| ④ Compare | `dtrack compare-row …` | Left-right join, produce report | `output/compare_row.html` |

> [!IMPORTANT] Hard ordering rule: **row must precede col**
> The col extraction SQL is trimmed to "dates that row-compare deemed present on both sides." So without a completed `compare-row`, the col phase has no basis to work from. `just e2e all` enforces this order; before running `col*` standalone, a prior `e2e row` run is required.

`generate` and `run-sql` are intentionally separate: generating SQL first lets you **review it by eye** (especially col's CTEs / hash expressions) and confirm correctness before `run-sql` actually hits the database.

## A full row + col run (bare dtrack commands)

```bash+ Full command sequence (replace DB/CFG/DIR with your values)
DB=project.db; CFG=pairs.json; DIR=.        # DIR will contain csv/ sas/ output/

dtrack init $DB                              # First time: create DB (skip if already exists)

# ---------- ROW ----------
dtrack generate $DB --config $CFG --outdir $DIR --type row
dtrack run-sql  $DB --config $CFG --outdir $DIR --type row
dtrack load-columns $DB --config $CFG --columns-dir $DIR/csv   # Also fetches column/type metadata
dtrack load-row $DB $DIR/csv --config $CFG
dtrack compare-row $DB --config $CFG --html $DIR/output/compare_row.html -y

# ---------- COL ----------
dtrack match-columns $DB --config $CFG --yes                   # Auto-match left/right columns → col_map
dtrack generate $DB --config $CFG --outdir $DIR --type col     # Review csv/extract_col.*.sql
dtrack run-sql  $DB --config $CFG --outdir $DIR --type col
dtrack load-col $DB $DIR/csv --config $CFG
dtrack compare-col $DB --config $CFG --html $DIR/output/compare_col.html -y
```

> [!TIP] Column annotations / time mappings are done in the **web UI's col_mapping page**, not in the CLI. The CLI's `match-columns` only performs auto-matching + rule-based matching (see [[dtrack: general knowledge|6a3a7b11b439b]] Column Mapping Rules).

## justfile shortcuts (recommended for daily use)

The justfile wraps the 11 commands above into recipes with **skip-cache** support and consolidates logs to `output/run.log`.

| just command | Equivalent to |
|---|---|
| `just e2e row` | generate+run-sql(row) → discover → load-row → compare-row |
| `just e2e col-gen` | match-columns → generate(col) (**stops here so you can review the SQL**) |
| `just e2e col-run` | run-sql(col) → load-col → compare-col |
| `just e2e col` | col-gen followed by col-run (no pause) |
| `just e2e all` | row then col (default) |
| `just force=1 e2e all` | Ignore skip-cache, re-run all expensive stages |
| `just pair=NAME e2e all` | Run only the `testing/NAME.json` group; reports/cache namespaced with `_NAME` |
| `just e2e-clean [state\|data\|all]` | Clear cache / also csv,sas / also db,output |
| `just logs` | tail `output/run.log` |
| `just doctor` | Health check: resolved env file + Oracle macros + AWS variables |

Extraction parameters are forwarded to the underlying `generate`, attached to **col-gen** (since they get baked into the col SQL):
```bash
just pair=month e2e col-gen --vintage month --preset detailed
# --vintage day|week|month|quarter|year|all   bucketing granularity
# --preset quick|standard|detailed            column stats depth
# --from-date / --to-date YYYY-MM-DD          incremental window
# --stats-override n_unique=on,hash_sum=off   per-stat toggle
```

> [!NOTE] skip-cache mechanism
> After an expensive stage (row extraction / discover / col extraction) succeeds, a `.done` sentinel is written to `.state/`. If it exists, the stage is skipped. Cheap steps (load / compare / match) always re-run, so even if the DB is reset the report will be rebuilt from cached CSVs. To force a re-run: `just force=1 …` or `just e2e-clean`.

## Before hitting a real machine: run `dtrack doctor`

`doctor` does not connect to any database; it only prints **which `.env` was resolved and which variables are set/MISSING**. This is the first step in troubleshooting.

Required `.env` per platform (override path with `dtrack --env /path/to/.env` or `DTRACK_ENV_FILE`):

=== "Oracle"
    ```bash
    PCDS_USR=your_user            # Shared username for all Oracle connections
    PCDS_PWD=...                  # Password for macro=pcds
    PB30_PWD=...                  # One <MACRO>_PWD per conn_macro
    # Optional: LDAP_DSN (if unset, connects directly via service name)
    # Optional: DTRACK_ORACLE_MACROS=pb40:svc_x,pb50:svc_y  append macros not in the built-in table
    ```
    doctor will list `pb30 -> service (builtin)` and `pb30_pwd=set/MISSING`.

=== "Athena"
    ```bash
    AWS_DEFAULT_REGION=us-east-1
    AWS_S3_WORK_GROUP=primary
    AWS_S3_STAGING_DIR=s3://your-bucket/athena-output/
    AWS_USR=... ; AWS_PWD=...     # Used for corporate proxy + token retrieval
    # Optional: AWS_HOST (proxy host) AWS_TOKEN_URL AWS_ARN_URL
    ```

=== "Databricks"
    ```bash
    DB_PROFILE=your_profile       # databricks-sdk Config profile (OAuth)
    DB_HTTPS=/sql/1.0/warehouses/xxxx   # SQL warehouse http_path
    DB_DEFAULT_CATALOG=main       # Optional; prepended when conn_macro only provides schema
    # Proxy reuses Athena's AWS_USR / AWS_PWD / AWS_HOST (sets HTTPS_PROXY only, no AWS token)
    ```

=== "CSV / Mock"
    ```bash
    # CSV source requires no credentials; absolute file path goes in the pair's conn_macro (DuckDB read_csv_auto reads it directly)
    # Mock mode (no real DB connection, copies fixtures as results):
    DTRACK_MOCK=/path/to/mock              # All-in-one, overrides all platforms
    # Or per-platform: DTRACK_ORACLE_MOCK / DTRACK_ATHENA_MOCK / DTRACK_DATABRICKS_MOCK / DTRACK_CSV_MOCK
    ```

## pair config (pairs.json) skeleton

```json+ Minimal structure for one pair
{
  "pairs": {
    "my_pair_name": {
      "description": "Plain-language description of what this pair is comparing",
      "left":  { "name": "short name", "table": "SCHEMA_TABLE", "source": "oracle",
                 "conn_macro": "pb30", "date_col": "POS_DT", "date_type": "date" },
      "right": { "name": "short name", "table": "tbl", "source": "aws",
                 "conn_macro": "warehouse_db", "date_col": "int_date", "date_type": "num" },
      "mode": "incremental", "fromDate": "2024-10-01", "toDate": "2024-12-31"
    }
  }
}
```

Key fields:

| Field | Meaning | Values |
|---|---|---|
| `source` | Which extractor to use | `oracle`/`pcds` · `aws` · `databricks` · `hadoop` · `csv` · `sas` |
| `conn_macro` | Connection identifier / locator | Oracle=macro name; Athena=Glue db; Databricks=`catalog.schema`; CSV=**absolute file path** |
| `date_col` | Date column used for bucketing/incrementals | Column name |
| `date_type` | Physical date type | `date` `timestamp` `string_dash`(YYYY-MM-DD) `string_compact`(YYYYMMDD) `num`(YYYYMMDD/YYYYMM) `datetime`(SAS) |
| `processed` | Pre-processing SQL for right/left side (CTE) | See CTE scenario card |
| `col_map` | `{left_col: right_col}` | Generated by match-columns, can also be written by hand |

## General observation checklist (review on every real-machine run)

- [ ] `dtrack doctor`: all credentials for the target platform are `set`, none `MISSING`
- [ ] After row `generate`, **read** `csv/extract_row.<platform>.sql` first — confirm table names and date filters are correct
- [ ] `run-sql` prints `N ok, M failed` at the end; if M>0, check `FAIL <name>: <error>` (also in `output/run.log`)
- [ ] In the `compare-row` report, pay attention to "dates that appear on only one side" and "same-date count mismatches"
- [ ] Before entering col phase, confirm the row report has been generated (hard rule)
- [ ] After col `generate`, review `extract_col.*.sql`: hash/length expressions, CTEs, whether dates are trimmed to the intersection determined by row
- [ ] In `compare-col`, focus on `hash_sum` / `length_avg` discrepancies → usually semantic inconsistency rather than data difference (diagnostics in [[Data Engineer|6a05342d3068b]])
- [ ] When a hash looks wrong, run [[dtrack: CSV → CSV + sample-hash|6a3b87c0905a7]] to verify row by row
