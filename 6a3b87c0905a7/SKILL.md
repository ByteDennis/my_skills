---
id: 6a3b87c0905a7
name: dtrack: CSV → CSV + sample-hash
tags: [dtrack, csv, duckdb, sample-hash, hash, runbook, rss, public]
updated_at: 2026-06-24T11:22:43.562984Z
---

# dtrack: CSV → CSV + sample-hash

>抽样几天的数据两边都下载成 CSV 再对（csv→csv，DuckDB 直读），并用 `sample-hash` 按主键逐行核对后端算出的 hash16，定位 `hash_sum` 差异到底是真差异还是算法/语义问题。<

基座流程见 [[dtrack: CLI runbook]]；跨引擎哈希/长度语义见 [[Data Engineer]]。

## 为什么 csv → csv（而不是 csv → Athena）

原方案是左 CSV、右 Athena 用 CTE 重构"当前视图"。但数据是**抽样的几天**，Athena 端重构对不齐时间——所以**两边都下载成 CSV** 再比，时间对齐由"下的是同样那几天的快照"天然保证。CSV 源走 DuckDB `read_csv_auto` 直读，无需凭证。

## 样例 pair（两边 `source:"csv"`）

```json+ testing/csv_csv.json
{
  "pairs": {
    "left_csv_vs_right_csv": {
      "description": "抽样几天：左右两份下载的快照 CSV 对数。",
      "left":  { "name": "snap_left",  "table": "snap_left", "source": "csv",
                 "conn_macro": "/abs/path/left_snapshot.csv",
                 "date_col": "rpt_dt", "date_type": "string_dash" },
      "right": { "name": "snap_right", "table": "snap_right", "source": "csv",
                 "conn_macro": "/abs/path/right_snapshot.csv",
                 "date_col": "rpt_dt", "date_type": "string_dash" },
      "mode": "incremental", "fromDate": "", "toDate": ""
    }
  }
}
```

| 字段 | csv 源说明 |
|---|---|
| `source` | `"csv"`（两侧都是） |
| `conn_macro` | **CSV 文件绝对路径**——和其他 source 一样，定位符就放 `conn_macro`；DuckDB `read_csv_auto('<path>', header=True)` 直读 |
| `table` / `name` | 仅作逻辑名 / qname，不进 FROM |
| `date_col` / `date_type` | 仍用于分桶；抽样几天就让窗口覆盖那几天即可 |

> [!IMPORTANT] 没有单独的 `csv_path` 字段
> csv 源的文件路径**放在 `conn_macro`**（与 oracle macro / athena Glue db / databricks `catalog.schema` 同位）。旧的 `csv_path` 字段已移除——CLI、compare、web 配对表单都改成读 `conn_macro`。网页里 source 选 `csv` 时，直接在「Connection Macro」框填路径。

> [!NOTE] DuckDB 的日期类型推断
> `read_csv_auto` 会把 ISO 字符串（`2024-10-01`）**自动推断成原生 DATE**。dtrack 的 csv 分桶已用 `CAST(... AS DATE)` 兼容「VARCHAR 或 DATE」两种情况，所以 `date_type:"string_dash"` 不论被读成哪种都能跑。`YYYYMMDD` 紧凑型用 `string_compact`/`num`。

## 跑 csv → csv

```bash
just pair=csv_csv e2e all
# 或裸命令（见 [[dtrack: CLI runbook]]）：generate→run-sql→load→compare，--type row 再 col
```
- 列自动从 CSV 表头发现（`discover_csv_columns`，DuckDB `DESCRIBE` / pandas 兜底），所以 `match-columns` 直接能用——**无需**手写 `{qname}_columns.csv` 侧车（那是给 Athena CTE 那种无法自省的场景的）。
- 抽样几天 → 用 `--from-date/--to-date` 或 pair 的 `where_map` 把两边都框到同样的日期集。

### 观察什么
- [ ] `extract_row.csv.sql` 里 `FROM read_csv_auto('<你的路径>', header=True)`，路径对、`header=True`
- [ ] `compare_row` 两边日期/行数对齐（抽样的那几天都在）
- [ ] `compare_col` 逐列差异；`hash_sum` 有差就进下面的 sample-hash

## sample-hash：逐行核对哈希

`compare_col` 只给每列一个聚合 `hash_sum`。两边 `hash_sum` 不一致时，`sample-hash` 帮你**按主键抽 N 行、两边取同样的 key、后端各自算 16-bit hash16、逐格摆出来**，一眼看出是哪几格、是真值差还是空白/语义差。

> [!TIP] 这是通用工具，不止 csv→csv
> 支持 **aws / oracle / databricks / csv** 四种源；任何涉及 `hash_sum` 的 pair 都能用。建议在 col_gen **之前**先跑一遍存疑的 pair。

```bash
dtrack sample-hash project.db --config pairs.json \
  --key rpt_dt,cust_id          # 左侧主键；右侧经 col_map 解析；逗号=复合键
  [--pair left_csv_vs_right_csv] # 不给则跑所有未跳过的 pair
  [-n 50]                        # 每个 pair 抽样行数（默认 20）
  [--out output/sample_hash.xlsx]

# justfile：
just key=cust_id sample-hash                  # 所有 pair，n=20
just key=cust_id pair=csv_csv sample-hash -n 50
```

机制：**从左侧抽 N 个 key → 两边都按这批同样的 key 取行 → 每个 mapped 列后端算 `hash16` → 一行一条记录写进 Excel**（只出 `.xlsx`，不出 `.txt`）。

### 读 Excel（怎么判读）
- 每行 = 一个 key；每个 mapped 列有「左值 / 右值 / 左hash16 / 右hash16」。
- **左右 hash16 相等** → 该格一致。注意 hash 前会 `TRIM`，所以 `WEST` vs `WEST␣`（尾随空格）会判**一致**——证明算法没问题、差异来自空白。
- **hash16 不等** → 真实值差异（如 `300` vs `999`），直接定位到 key + 列。
- 一致的格子却在 `hash_sum` 上有差 → 多半是**碰撞**或**语义**（LENGTH 字节/字符、NULL/哨兵、`\xa0` 不间断空格）——对照 [[Data Engineer]] 的诊断规律。

> [!IMPORTANT] hash16 跨引擎等价
> 各平台 `int(md5(trim(value))[:4], 16)` 的等价表达式（Oracle/Trino/Spark/DuckDB 各自方言）数值一致，这正是 `hash_sum` 能跨引擎相加比较的基础。sample-hash 用的就是同一套表达式，所以它"算的"和 col 抽取"算的"是同一个 hash。

## csv / sample-hash 专属坑
- **路径**：`conn_macro`（CSV 路径）必须存在且可读；不存在时 discover 会打印 `conn_macro (csv path) missing or not found` 并跳过该表。
- **抽样对不齐**：两份 CSV 必须是**同样那几天**的快照，否则 row 阶段就会显示一堆"只在一侧"的日期。
- **复合主键**：`--key a,b` 顺序左侧；右侧名由 `col_map` 决定，先确认 `match-columns` 把 key 列也映上了。
- **16-bit 碰撞**：基数 K≈100 时碰撞概率约 7%（见 [[Data Engineer]]）——sample-hash 逐行看原值能直接区分"碰撞"与"真一致"。
