---
id: 6a3bbe7764659
name: dtrack: maintenance / develop codebase map
tags: [dtrack, maintenance, architecture, dev, rss, public]
updated_at: 2026-06-24T12:12:37.763204Z
---

# dtrack: maintenance / develop codebase map

>Navigation map for modifying or extending dtrack: what each module owns, how the extract dispatch chain works, which files to touch when adding a new platform / new statistic / new date format, and how to run tests and mocks.<

Companion runbook: [[dtrack: CLI runbook|6a3b8031de3c0]]; internal details (playground multi-statement execution, column-mapping rule implementation) are in [[dtrack: general knowledge|6a3a7b11b439b]].

## Module quick reference (`dtrack/`)

| Module | Responsibility |
|---|---|
| `cli.py` | argparse subcommands → `cmd_*` handlers (one per CLI command) |
| `extract.py` | **Extract dispatch**: group by source, fan out to per-platform extractors / runners |
| `platforms/base.py` | Cross-platform shared code: `build_col_sql`, date-range/filtering, `resolve_date_format`, `quote_ident`, run-sql skeleton, mock copy, SQL result cache |
| `platforms/<src>.py` | Per-source extractor + connection + `run_*_sql_file` (see table below) |
| `compare.py` | `compare-row/col` logic, `apply_column_rules`, `sample_hash` |
| `csv_compare.py` | **Exact-string** CSV↔CSV comparison by primary key (`dtype=str`, `""`≠NaN, `"10"`≠`"10.0"`) — distinct from the stats-based compare-col |
| `db.py` | SQLite `project.db`: row/col result tables, storage and retrieval for `col_map`/`col_rules`/`col_filter`/`_column_meta` |
| `config.py` | `load_unified_config` / `get_all_tables_from_unified` (pairs.json → internal table list) |
| `constants.py` | `STATS_PRESETS`, `DATE_TYPE_FORMATS`, CSV headers, SQL marker comments |
| `loader.py` | Load `{qname}_row.csv` / `_col.csv` into the db |
| `html_export.py` · `excel.py` | Generate HTML / Excel reports |
| `stats.py` · `date_utils.py` | Column statistics calculation / date parsing + vintage bucketing utilities |
| `logmgr.py` · `interact.py` | Unified logging (run.log) / CLI interactive prompts |

## Platform layer (`dtrack/platforms/`)

| source | Extractor (gen) | Runner (run-sql) | Connection / dialect |
|---|---|---|---|
| `oracle`/`pcds` | `extract_oracle` | `run_oracle_sql_file` | oracledb, macro→service+`<MACRO>_PWD` |
| `aws` | `extract_aws` | `run_sql_file` | pyathena, Glue db |
| `databricks` | `extract_databricks` | `run_databricks_sql_file` | databricks-sql OAuth, `catalog.schema` |
| `hadoop` | `extract_hadoop` | `run_hadoop_sql_file` | Hive (SASL serial handshake) |
| `csv` | `extract_csv` | `run_csv_sql_file` | DuckDB in-process, `conn_macro`=file path |
| `sas` | `gen_sas` | (runs `.sas` on the SAS server) | hadoop_sas / sas_libname |

Col extraction mostly goes through shared `base.build_col_sql(dialect, …)`, with one set of `ops` per dialect (oracle/hadoop/duckdb/databricks). Databricks col borrows the Athena/Trino template (Spark dialect is close enough).

## Extract dispatch chain (`extract.py`)

```text
load_unified_config → get_all_tables_from_unified
   → classify_tables_by_platform(tables)   # bucket by source: oracle/aws/databricks/hadoop/csv/sas
generate:  extract_all(..., gen_only=True) # each non-empty bucket calls extract_<src>(gen_only) → writes extract_{type}.<platform>.sql
run-sql:   run_sql_all(...)                # looks up runner via _RUNNERS; if DTRACK_MOCK/<P>_MOCK matches, copies fixtures instead
```
`_RUNNERS = {platform: (module, fn, mock_env_var)}`. To add a platform, update all of: the bucket in `classify_tables_by_platform`, `get_platform_extractor`, the branch in `extract_all`, `_RUNNERS`, and the platform loop tuple in `run_sql_all`.

## Change recipes

=== "Add a new platform"
    1. `platforms/<x>.py`: `extract_<x>(…, gen_only)`, `run_<x>_sql_file`, `_query_<x>`.
    2. `extract.py`: add bucket + source tuple in `classify_tables_by_platform`; add branch in `get_platform_extractor`; add branch in `extract_all`; add entry to `_RUNNERS`; add name to the platform loop in `run_sql_all`.
    3. If col goes through `base.build_col_sql`, add that dialect's `ops` + a portable hash16 expression to base.
    4. Mock fixture + tests.

=== "Add a new statistic"
    - The 14-column schema (`COL_CSV_HEADERS`) is **fixed**; statistics not in a preset appear as `''` in the CSV.
    - Add the statistic to the appropriate preset in `STATS_PRESETS`; add its SQL expression for each dialect in `build_col_sql`.
    - See [[dtrack: column match / stats preset|6a3bbe7736cb9]].

=== "Add a new date format"
    - `constants.py`: add `date_type→format` to `DATE_TYPE_FORMATS`, add `date_type→DATE/TIMESTAMP/NUMBER/VARCHAR2` to `DATE_TYPE_TO_CANONICAL_DTYPE`.
    - See [[dtrack: date_type · vintage · config|6a3bbe771bede]].

## Tests + mock

```bash
just test                      # pytest (238 passed / 134 skipped is the baseline)
just test-cov                  # with coverage; --lf to rerun only last failures
just gen-mock                  # regenerate e2e mock fixtures → testing/mock
```
- **Mock mode**: `DTRACK_MOCK=<dir>` (blanket override) or `DTRACK_<PLATFORM>_MOCK`; when run-sql matches, it copies `<dir>/{qname}_{type}.csv` as the result instead of hitting a real database.
- e2e test fixture directory `DTRACK_E2E_MOCK_DIR` (set to `testing/mock` in the justfile); `testing/generate_mock_data.py` generates the data.
- CSV source tests are in `tests/csv/` (`helpers.py` builds a `source:csv` config with the path in `conn_macro`).

## web vs CLI

Same extract/compare core; web (`dtrack/web/app.py` + `static/*.js` + `templates/*.html`) is the GUI shell: playground multi-statement execution, the col_mapping annotation page, and the pairs pairing form all exist only in web. CLI and web point to the same `workdir` (`project.db` + `pairs.json`), so they operate on the same project. See [[dtrack: general knowledge|6a3a7b11b439b]].
