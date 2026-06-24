---
id: 6a3b87c0905a7
name: dtrack: CSV → CSV + sample-hash
tags: [dtrack, csv, duckdb, sample-hash, hash, runbook, rss, public]
updated_at: 2026-06-24T12:12:37.201611Z
---

# dtrack: CSV → CSV + sample-hash

>Download a few sampled days of data from both sides as CSV, then reconcile them (csv→csv, read directly with DuckDB), and use `sample-hash` to verify the backend-computed hash16 row-by-row against primary keys — pinpointing whether `hash_sum` discrepancies are real differences or algorithm/semantic issues.<

Base workflow: [[dtrack: CLI runbook|6a3b8031de3c0]]; cross-engine hash/length semantics: [[Data Engineer|6a05342d3068b]].

## Why csv → csv (instead of csv → Athena)

The original plan was left CSV, right Athena reconstructing the "current view" via CTE. But the data is **a few sampled days**, and Athena-side reconstruction cannot align time — so **both sides are downloaded as CSV** and then compared; time alignment is naturally guaranteed by "both sides downloaded the same snapshot days". The CSV source is read directly via DuckDB `read_csv_auto`, no credentials needed.

## Sample pair (both sides `source:"csv"`)

```json+ testing/csv_csv.json
{
  "pairs": {
    "left_csv_vs_right_csv": {
      "description": "A few sampled days: row-count comparison of two downloaded snapshot CSVs.",
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

| Field | csv source description |
|---|---|
| `source` | `"csv"` (both sides) |
| `conn_macro` | **Absolute path to the CSV file** — same as other sources, the locator goes in `conn_macro`; DuckDB `read_csv_auto('<path>', header=True)` reads it directly |
| `table` / `name` | Used as logical name / qname only, not in FROM |
| `date_col` / `date_type` | Still used for bucketing; for sampled days just let the window cover those days |

> [!IMPORTANT] There is no separate `csv_path` field
> The file path for a csv source **goes in `conn_macro`** (same position as oracle macro / athena Glue db / databricks `catalog.schema`). The old `csv_path` field has been removed — the CLI, compare, and web pairing form all now read `conn_macro`. In the web UI when source is `csv`, enter the path directly in the "Connection Macro" field.

> [!NOTE] DuckDB date type inference
> `read_csv_auto` will **automatically infer** ISO strings (`2024-10-01`) as native DATE. dtrack's csv bucketing already uses `CAST(... AS DATE)` to handle both VARCHAR and DATE, so `date_type:"string_dash"` works regardless of which type it gets read as. For `YYYYMMDD` compact format, use `string_compact`/`num`.

## Running csv → csv

```bash
just pair=csv_csv e2e all
# or bare commands (see [[dtrack: CLI runbook|6a3b8031de3c0]]): generate→run-sql→load→compare, --type row then col
```
- Columns are discovered automatically from the CSV header (`discover_csv_columns`, DuckDB `DESCRIBE` / pandas fallback), so `match-columns` works out of the box — **no need** to manually write a `{qname}_columns.csv` sidecar (that is for scenarios like Athena CTE that cannot introspect).
- Sampled days → use `--from-date/--to-date` or the pair's `where_map` to restrict both sides to the same date set.

### What to observe
- [ ] `extract_row.csv.sql` has `FROM read_csv_auto('<your path>', header=True)`, path is correct, `header=True`
- [ ] `compare_row` date/row counts are aligned on both sides (all sampled days are present)
- [ ] `compare_col` per-column diffs; if `hash_sum` differs, proceed to sample-hash below

## sample-hash: row-by-row hash verification

`compare_col` gives only one aggregate `hash_sum` per column. When both sides have an inconsistent `hash_sum`, `sample-hash` helps you **sample N rows by primary key, pull the same keys from both sides, have the backend compute hash16 for each, and lay them out cell-by-cell** — making it easy to see which cells differ and whether it is a real value difference or a whitespace/semantic issue.

> [!TIP] This is a general-purpose tool, not just for csv→csv
> Supports all four source types: **aws / oracle / databricks / csv**; usable for any pair involving `hash_sum`. Recommended to run on any suspicious pair before col_gen.

```bash
dtrack sample-hash project.db --config pairs.json \
  --key rpt_dt,cust_id          # left-side primary key; right side resolved via col_map; comma = composite key
  [--pair left_csv_vs_right_csv] # if omitted, runs all non-skipped pairs
  [-n 50]                        # number of sampled rows per pair (default 20)
  [--out output/sample_hash.xlsx]

# justfile:
just key=cust_id sample-hash                  # all pairs, n=20
just key=cust_id pair=csv_csv sample-hash -n 50
```

Mechanism: **sample N keys from the left side → fetch rows for that same set of keys from both sides → backend computes `hash16` for each mapped column → write one record per row into Excel** (only `.xlsx` output, no `.txt`).

### Reading the Excel (how to interpret)
- Each row = one key; each mapped column has "left value / right value / left hash16 / right hash16".
- **Left and right hash16 equal** → that cell is consistent. Note that hash computation applies `TRIM` first, so `WEST` vs `WEST␣` (trailing space) will be judged **consistent** — proving the algorithm is fine and the difference comes from whitespace.
- **hash16 not equal** → real value difference (e.g. `300` vs `999`), directly located to key + column.
- Cells that are consistent yet show a `hash_sum` difference → likely **collision** or **semantics** (LENGTH bytes/characters, NULL/sentinel, `\xa0` non-breaking space) — refer to the diagnostic patterns in [[Data Engineer|6a05342d3068b]].

> [!IMPORTANT] hash16 cross-engine equivalence
> The equivalent expression of `int(md5(trim(value))[:4], 16)` across platforms (Oracle/Trino/Spark/DuckDB each in their own dialect) produces the same numeric value — this is precisely the foundation that allows `hash_sum` to be summed and compared across engines. sample-hash uses the same set of expressions, so what it "computes" is the same hash as what the col extraction "computes".

## csv / sample-hash specific pitfalls
- **Path**: `conn_macro` (CSV path) must exist and be readable; if missing, discover will print `conn_macro (csv path) missing or not found` and skip that table.
- **Sampled data misalignment**: Both CSVs must be snapshots of **the same days**; otherwise the row stage will show a flood of dates present on only one side.
- **Composite primary key**: `--key a,b` order is left-side; right-side names are determined by `col_map` — confirm that `match-columns` has also mapped the key columns.
- **16-bit collision**: With cardinality K≈100, collision probability is about 7% (see [[Data Engineer|6a05342d3068b]]) — sample-hash lets you inspect raw values row-by-row to directly distinguish "collision" from "true match".
