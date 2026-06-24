---
id: 6a3b8031de3c0
name: dtrack: CLI runbook
tags: [dtrack, cli, runbook, data-engineering, rss, public]
updated_at: 2026-06-24T11:25:07.078690Z
---

# dtrack: CLI runbook

>真机跑 dtrack 的统一心智模型：generate → run-sql → load → compare，分 row / col 两阶段。所有平台共用同一套命令，只是 `.env` 和 pair 配置不同。<

本卡是基座，场景卡都从这里展开：
[[dtrack: Oracle → Athena (row + col)]] · [[dtrack: Oracle → Databricks (row + col)]] · [[dtrack: CSV → CSV + sample-hash]]
深入：[[dtrack: date_type · vintage · 增量窗口]] · [[dtrack: 列匹配 / stats preset]] · [[dtrack: 故障排查 / 常见错误]] · [[dtrack: 维护 / 开发 codebase 地图]]
跨引擎语义坑见 [[Data Engineer]]；Web/代码内部见 [[dtrack: general knowledge]]。

## 心智模型：4 步 × 2 阶段

dtrack 把"对数"拆成 **row（行数对齐）** 和 **col（列统计对齐）** 两个阶段，每阶段都是同样的 4 步：

| 步骤 | 命令 | 作用 | 产物 |
|---|---|---|---|
| ① 生成 | `dtrack generate … --type row` | 把 pair 翻成各平台 SQL，**不执行** | `csv/extract_row.<platform>.sql` |
| ② 执行 | `dtrack run-sql … --type row` | 在真库上跑上面的 SQL | `csv/{qname}_row.csv` |
| ③ 入库 | `dtrack load-row … csv/` | CSV → SQLite `project.db` | `_row_comparison` 表 |
| ④ 对比 | `dtrack compare-row …` | 左右 join，出报告 | `output/compare_row.html` |

> [!IMPORTANT] 顺序铁律：**row 必须先于 col**
> col 的抽取 SQL 会被裁剪到「row-compare 认定两边都齐的日期」。所以 `compare-row` 没跑过，col 阶段无据可依。`just e2e all` 已强制这个顺序；单独跑 `col*` 前必须先有一次 `e2e row`。

`generate` 和 `run-sql` 分家是刻意的：先生成 SQL 让你**肉眼审一遍**（尤其 col 的 CTE / 哈希表达式），确认无误再 `run-sql` 真正打到库上。

## 一次完整的 row + col（裸 dtrack 命令）

```bash+ 完整命令序列（把 DB/CFG/DIR 换成你的）
DB=project.db; CFG=pairs.json; DIR=.        # DIR 下会有 csv/ sas/ output/

dtrack init $DB                              # 首次：建库（已存在可跳过）

# ---------- ROW ----------
dtrack generate $DB --config $CFG --outdir $DIR --type row
dtrack run-sql  $DB --config $CFG --outdir $DIR --type row
dtrack load-columns $DB --config $CFG --columns-dir $DIR/csv   # 顺带抓列/类型元数据
dtrack load-row $DB $DIR/csv --config $CFG
dtrack compare-row $DB --config $CFG --html $DIR/output/compare_row.html -y

# ---------- COL ----------
dtrack match-columns $DB --config $CFG --yes                   # 自动匹配左右列 → col_map
dtrack generate $DB --config $CFG --outdir $DIR --type col     # 审 csv/extract_col.*.sql
dtrack run-sql  $DB --config $CFG --outdir $DIR --type col
dtrack load-col $DB $DIR/csv --config $CFG
dtrack compare-col $DB --config $CFG --html $DIR/output/compare_col.html -y
```

> [!TIP] 列注释 / 时间映射在 **Web 的 col_mapping 页**做，不在 CLI。CLI 的 `match-columns` 只做自动匹配 + 规则匹配（见 [[dtrack: general knowledge]] 的 Column Mapping Rules）。

## justfile 捷径（推荐日常用）

justfile 把上面 11 条命令包成带 **skip-cache** 的 recipe，并统一日志到 `output/run.log`。

| just 命令 | 等价于 |
|---|---|
| `just e2e row` | generate+run-sql(row) → discover → load-row → compare-row |
| `just e2e col-gen` | match-columns → generate(col)（**停在这，给你审 SQL**） |
| `just e2e col-run` | run-sql(col) → load-col → compare-col |
| `just e2e col` | col-gen 接 col-run（不停） |
| `just e2e all` | row 然后 col（默认） |
| `just force=1 e2e all` | 忽略 skip-cache，所有贵阶段重跑 |
| `just pair=NAME e2e all` | 只跑 `testing/NAME.json` 这一组；报告/缓存按 `_NAME` 命名隔离 |
| `just e2e-clean [state\|data\|all]` | 清缓存 / 连 csv,sas / 连 db,output |
| `just logs` | tail `output/run.log` |
| `just doctor` | 体检：解析到的 env 文件 + Oracle macro + AWS 变量 |

抽取参数透传给底层 `generate`，挂在 **col-gen** 上（因为它们烘进 col SQL）：
```bash
just pair=month e2e col-gen --vintage month --preset detailed
# --vintage day|week|month|quarter|year|all   分桶粒度
# --preset quick|standard|detailed            列统计深度
# --from-date / --to-date YYYY-MM-DD          增量窗口
# --stats-override n_unique=on,hash_sum=off   单项统计开关
```

> [!NOTE] skip-cache 机制
> 贵阶段（row 抽取 / discover / col 抽取）成功后在 `.state/` 落一个 `.done` 哨兵，存在即跳过。便宜步骤（load / compare / match）每次都重跑，所以即使 DB 被重置，报告也会用缓存的 CSV 重新填好。强制重抽：`just force=1 …` 或 `just e2e-clean`。

## 上真机前：先 `dtrack doctor`

`doctor` 不连库，只打印**解析到了哪个 .env、哪些变量 set/MISSING**。这是排错第一步。

各平台需要的 `.env`（用 `dtrack --env /path/to/.env` 或 `DTRACK_ENV_FILE` 覆盖路径）：

=== "Oracle"
    ```bash
    PCDS_USR=your_user            # 所有 Oracle 连接共用的用户名
    PCDS_PWD=...                  # macro=pcds 的密码
    PB30_PWD=...                  # 每个 conn_macro 一个 <MACRO>_PWD
    # 可选：LDAP_DSN（不设则按 service name 直连）
    # 可选：DTRACK_ORACLE_MACROS=pb40:svc_x,pb50:svc_y  追加内置表没有的 macro
    ```
    doctor 会列出 `pb30 -> service (builtin)` 和 `pb30_pwd=set/MISSING`。

=== "Athena"
    ```bash
    AWS_DEFAULT_REGION=us-east-1
    AWS_S3_WORK_GROUP=primary
    AWS_S3_STAGING_DIR=s3://your-bucket/athena-output/
    AWS_USR=... ; AWS_PWD=...     # 走公司代理 + 取 token
    # 可选：AWS_HOST（代理主机）AWS_TOKEN_URL AWS_ARN_URL
    ```

=== "Databricks"
    ```bash
    DB_PROFILE=your_profile       # databricks-sdk Config 的 profile（OAuth）
    DB_HTTPS=/sql/1.0/warehouses/xxxx   # SQL warehouse 的 http_path
    DB_DEFAULT_CATALOG=main       # 可选；pair 的 conn_macro 只给 schema 时前置它
    # 代理复用 Athena 的 AWS_USR / AWS_PWD / AWS_HOST（仅设 HTTPS_PROXY，不做 AWS token）
    ```

=== "CSV / Mock"
    ```bash
    # CSV 源无需凭证；文件绝对路径写在 pair 的 conn_macro 里（DuckDB read_csv_auto 直读）
    # Mock 模式（不连真库，拷贝 fixtures 当结果）：
    DTRACK_MOCK=/path/to/mock              # 一把梭，覆盖所有平台
    # 或单平台：DTRACK_ORACLE_MOCK / DTRACK_ATHENA_MOCK / DTRACK_DATABRICKS_MOCK / DTRACK_CSV_MOCK
    ```

## pair 配置（pairs.json）骨架

```json+ 一个 pair 的最小结构
{
  "pairs": {
    "my_pair_name": {
      "description": "人话说明这对在比什么",
      "left":  { "name": "短名", "table": "SCHEMA_TABLE", "source": "oracle",
                 "conn_macro": "pb30", "date_col": "POS_DT", "date_type": "date" },
      "right": { "name": "短名", "table": "tbl", "source": "aws",
                 "conn_macro": "warehouse_db", "date_col": "int_date", "date_type": "num" },
      "mode": "incremental", "fromDate": "2024-10-01", "toDate": "2024-12-31"
    }
  }
}
```

关键字段：

| 字段 | 含义 | 取值 |
|---|---|---|
| `source` | 用哪个抽取器 | `oracle`/`pcds` · `aws` · `databricks` · `hadoop` · `csv` · `sas` |
| `conn_macro` | 连接标识 / 定位符 | Oracle=macro名；Athena=Glue db；Databricks=`catalog.schema`；CSV=**文件绝对路径** |
| `date_col` | 用于分桶/增量的日期列 | 列名 |
| `date_type` | 日期物理类型 | `date` `timestamp` `string_dash`(YYYY-MM-DD) `string_compact`(YYYYMMDD) `num`(YYYYMMDD/YYYYMM) `datetime`(SAS) |
| `processed` | 右/左侧预处理 SQL（CTE） | 见 CTE 场景卡 |
| `col_map` | `{左列: 右列}` | match-columns 生成，也可手写 |

## 通用观察清单（每次真机跑都过一遍）

- [ ] `dtrack doctor`：目标平台的凭证全部 `set`，没有 `MISSING`
- [ ] row `generate` 后**先读** `csv/extract_row.<platform>.sql`，确认表名/日期过滤对
- [ ] `run-sql` 末尾打印 `N ok, M failed`；M>0 时看 `FAIL <name>: <error>`（也在 `output/run.log`）
- [ ] `compare-row` 报告里关注「只在一侧出现的日期」「同日期 count 不一致」
- [ ] 进 col 前确认 row 报告已生成（铁律）
- [ ] col `generate` 后审 `extract_col.*.sql`：哈希/长度表达式、CTE、日期是否被裁到 row 认定的交集
- [ ] `compare-col` 关注 `hash_sum` / `length_avg` 差异 → 多半是语义不一致而非数据差异（诊断见 [[Data Engineer]]）
- [ ] 怀疑哈希算错时跑 [[dtrack: CSV → CSV + sample-hash]] 逐行核对
