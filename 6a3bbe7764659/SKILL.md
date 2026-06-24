---
id: 6a3bbe7764659
name: dtrack: maintenance / develop codebase map
tags: [dtrack, maintenance, architecture, dev, rss, public]
updated_at: 2026-06-24T11:27:25.847704Z
---

# dtrack: maintenance / develop codebase map

>想改/扩 dtrack 时的导航图：模块各管什么、抽取调度链怎么走、加一个新平台 / 新统计 / 新日期格式分别动哪几处、测试与 mock 怎么跑。<

配套运行手册 [[dtrack: CLI runbook]]；内部细节（playground 多语句、列映射规则实现）见 [[dtrack: general knowledge]]。

## 模块速查（`dtrack/`）

| 模块 | 职责 |
|---|---|
| `cli.py` | argparse 子命令 → `cmd_*` 处理器（每个 CLI 命令一个） |
| `extract.py` | **抽取调度**：按 source 分组、分发到各平台抽取器 / 运行器 |
| `platforms/base.py` | 跨平台共享：`build_col_sql`、日期范围/过滤、`resolve_date_format`、`quote_ident`、run-sql 骨架、mock 拷贝、SQL 结果缓存 |
| `platforms/<src>.py` | 各源的抽取器 + 连接 + `run_*_sql_file`（见下表） |
| `compare.py` | `compare-row/col` 逻辑、`apply_column_rules`、`sample_hash` |
| `csv_compare.py` | 按主键的**字符串精确** CSV↔CSV 比对（`dtype=str`，`""`≠NaN，`"10"`≠`"10.0"`）——区别于 stats 型 compare-col |
| `db.py` | SQLite `project.db`：行/列结果表、`col_map`/`col_rules`/`col_filter`、`_column_meta` 的存取 |
| `config.py` | `load_unified_config` / `get_all_tables_from_unified`（pairs.json → 内部表列表） |
| `constants.py` | `STATS_PRESETS`、`DATE_TYPE_FORMATS`、CSV 表头、SQL 标记注释 |
| `loader.py` | 把 `{qname}_row.csv` / `_col.csv` 灌进 db |
| `html_export.py` · `excel.py` | 生成 HTML / Excel 报告 |
| `stats.py` · `date_utils.py` | 列统计计算 / 日期解析+vintage 分桶工具 |
| `logmgr.py` · `interact.py` | 统一日志（run.log）/ CLI 交互提示 |

## 平台层（`dtrack/platforms/`）

| source | 抽取器（gen） | 运行器（run-sql） | 连接/方言 |
|---|---|---|---|
| `oracle`/`pcds` | `extract_oracle` | `run_oracle_sql_file` | oracledb，macro→service+`<MACRO>_PWD` |
| `aws` | `extract_aws` | `run_sql_file` | pyathena，Glue db |
| `databricks` | `extract_databricks` | `run_databricks_sql_file` | databricks-sql OAuth，`catalog.schema` |
| `hadoop` | `extract_hadoop` | `run_hadoop_sql_file` | Hive（SASL 串行握手） |
| `csv` | `extract_csv` | `run_csv_sql_file` | DuckDB 进程内，`conn_macro`=文件路径 |
| `sas` | `gen_sas` | （在 SAS 服务器上跑 `.sas`） | hadoop_sas / sas_libname |

col 抽取大多走共享 `base.build_col_sql(dialect, …)`，每个方言一套 `ops`（oracle/hadoop/duckdb/databricks）。Databricks col 借用 Athena/Trino 模板（Spark 方言够近）。

## 抽取调度链（`extract.py`）

```text
load_unified_config → get_all_tables_from_unified
   → classify_tables_by_platform(tables)   # 按 source 分桶：oracle/aws/databricks/hadoop/csv/sas
generate:  extract_all(..., gen_only=True) # 每个非空桶调 extract_<src>(gen_only) → 写 extract_{type}.<platform>.sql
run-sql:   run_sql_all(...)                # 按 _RUNNERS 找 runner 执行；DTRACK_MOCK/<P>_MOCK 命中则拷 fixtures
```
`_RUNNERS = {platform: (module, fn, mock_env_var)}`。加平台要同时改：`classify_tables_by_platform` 的桶、`get_platform_extractor`、`extract_all` 分支、`_RUNNERS`、`run_sql_all` 的平台循环元组。

## 改动配方

=== "加一个新平台"
    1. `platforms/<x>.py`：`extract_<x>(…, gen_only)`、`run_<x>_sql_file`、`_query_<x>`。
    2. `extract.py`：`classify_tables_by_platform` 加桶 + source 元组；`get_platform_extractor` 加分支；`extract_all` 加分支；`_RUNNERS` 加项；`run_sql_all` 平台循环加名。
    3. col 走 `base.build_col_sql` 的话，给 base 加该方言的 `ops` + 可移植 hash16 表达式。
    4. mock fixture + 测试。

=== "加一项统计"
    - 14 列 schema (`COL_CSV_HEADERS`) **固定不变**；不在 preset 里的统计在 CSV 里出 `''`。
    - 在 `STATS_PRESETS` 把该统计加到对应 preset；在 `build_col_sql` 各方言生成它的 SQL 表达式。
    - 见 [[dtrack: 列匹配 / stats preset]]。

=== "加一个日期格式"
    - `constants.py`：`DATE_TYPE_FORMATS` 加 `date_type→格式`，`DATE_TYPE_TO_CANONICAL_DTYPE` 加 `date_type→DATE/TIMESTAMP/NUMBER/VARCHAR2`。
    - 见 [[dtrack: date_type · vintage · 增量窗口]]。

## 测试 + mock

```bash
just test                      # pytest（238 passed / 134 skipped 是基线）
just test-cov                  # 带覆盖率；--lf 只重跑上次失败
just gen-mock                  # 重生成 e2e mock fixtures → testing/mock
```
- **Mock 模式**：`DTRACK_MOCK=<dir>`（一把梭）或 `DTRACK_<PLATFORM>_MOCK`；run-sql 命中就把 `<dir>/{qname}_{type}.csv` 拷成结果，不连真库。
- e2e 测试夹具目录 `DTRACK_E2E_MOCK_DIR`（justfile 设 `testing/mock`）；`testing/generate_mock_data.py` 造数据。
- csv 源测试见 `tests/csv/`（`helpers.py` 造 `source:csv` 配置——路径放 `conn_macro`）。

## web vs CLI

同一套抽取/对比内核；web（`dtrack/web/app.py` + `static/*.js` + `templates/*.html`）是 GUI 壳：playground 多语句执行、col_mapping 列注释页、pairs 配对表单都只在 web。CLI 与 web 指同一 `workdir`（`project.db` + `pairs.json`）即操作同一项目。详见 [[dtrack: general knowledge]]。
