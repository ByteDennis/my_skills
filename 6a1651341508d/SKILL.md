---
id: 6a1651341508d
name: AI-Humanizer
tags: [academic-writing, ai-detection, humanizer, cli, writing]
updated_at: 2026-06-01T04:16:15.279971Z
---

## `humanize` CLI 

| 命令 | 示例 | 用途 |
|-----------|---------------------|----------|
| `humanize analyze` | `humanize analyze -f draft.md -V` | 检测 AI 标记与模式 |
| `humanize score` | `humanize score -f draft.md` | 风险评分（0–100） |
| `humanize suggest` | `cat draft.md \| humanize suggest` | 优先修改建议列表 |
| `humanize transform` | `humanize transform -f draft.md > out.md` | 自动应用修复 |
| `humanize watch` | `humanize watch ./papers --threshold 60` | 监听目录变更 |
| `humanize config` | `humanize config` | 查看配置与 API 状态 |

**常用参数：** `-f <file>` · `-V` 详细输出 · `-j` JSON 格式 · `-a` 调用 GPTZero/Originality.ai · `-t <n>` 阈值

## `humanizer` CLI 

| 命令 | 示例 | 用途 |
|-------|------------|--------|
| `humanizer score` | `humanizer score < draft.txt` | 快速 AI 评分 |
| `humanizer analyze` | `humanizer analyze draft.md --verbose` | 完整模式检测 |
| `humanizer suggest` | `humanizer suggest draft.md` | 改进建议列表 |
| `humanizer humanize` | `humanizer humanize --autofix -f draft.md` | 引导式重写 + 自动修复 |
| `humanizer compare` | `humanizer compare --before v1.md --after v2.md` | 对比两版本分差 |
| `humanizer report` | `humanizer report draft.md > report.md` | 输出完整 Markdown 报告 |
| `humanizer scan` | `humanizer scan ./papers --ext md,txt` | 批量扫描文件夹 |

**常用参数：** `--verbose` · `--json` · `--autofix` · `--ignore-code` · `--fail-above <n>`

## 工作流（每 500–1000 字约 60–100 分钟）

| 阶段 | 操作 | 工具 |
|----|--------|---------|
| 1. 基线 | 编辑前先打分 | `humanize score -f draft.md` |
| 2. 审查 | 检出所有 AI 标记 | `humanizer analyze draft.md --verbose` |
| 3. 建议 | 获取修改优先级 | `humanize suggest -f draft.md` |
| 4. 语气注入 | 添加缩略语、旁白、段落长短变化 | *(手动)* |
| 5. 自动修复 | 应用安全的自动修改 | `humanize transform -f draft.md > v2.md` |
| 6. 认知纹理 | 加入不确定语气、自我纠正 | *(手动)* |
| 7. 对比确认 | 确认分数改善 | `humanizer compare --before v1.md --after v2.md` |

## 评分目标

| 分数 | 状态 |
|-------|---------|
| 0–25 | ✅ 接近人类写作 |
| 26–50 | ⚠️ 轻度 AI 痕迹——再修一轮 |
| 51–75 | ❌ 中度——需要较大修改 |
| 76–100 | ❌ 严重——需重写 |

> 学术文体天然偏正式，分数会偏高。学术写作以 **< 40 为合格目标**。

## 应删除 vs. 应注入

| 删除（AI 特征） | 注入（人类特征） |
|-------------|----------------|
| "delve"、"leverage"、"utilize" | 改用 "explore"、"use" |
| 句长整齐划一（15–20词） | 混用短句（5词）与长句（25词） |
| 段落结构过于对称 | 段落长短不均，允许旁支 |
| 完全没有缩略语 | 全文穿插 *don't*、*wasn't*、*it's* |
| 线性推进至整洁结论 | 螺旋式推理，保留明显的思考痕迹 |
| 抽象情绪（"I was sad"） | 具体行为（"I kept setting out two bowls"） |
| — | 自我纠正："Actually, that's not quite right…" |
| — | 不确定语气："I suspect…"、"I'm less sure about…" |

## 注意事项

- GPTZero 免费版每天限 3 次——日常迭代用本地 `score` 命令代替
- Grammarly 会与你"对着干"；保留约 50% 的不完美，无需全部采纳
- `humanize` 的 `-a` 参数会调用外部 API（GPTZero/Originality.ai），会消耗额度
- `--ignore-code` 可跳过技术/CS 论文中的代码块

TAGS: academic-writing, ai-detection, humanizer, cli, writing
