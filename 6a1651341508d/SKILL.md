---
id: 6a1651341508d
name: AI-Humanizer: Academic Writing
tags: [academic-writing, ai-detection, humanizer, cli, writing]
updated_at: 2026-05-27T03:10:23.065282Z
---

## `humanize` CLI — lxgicstudios

| Command | Example | Purpose |
|--------|--------------|----------|
| `humanize analyze` | `humanize analyze -f draft.md -V` | Detect AI markers & patterns |
| `humanize score` | `humanize score -f draft.md` | Risk score (0–100) |
| `humanize suggest` | `cat draft.md \| humanize suggest` | Prioritized fix list |
| `humanize transform` | `humanize transform -f draft.md > out.md` | Auto-apply fixes |
| `humanize watch` | `humanize watch ./papers --threshold 60` | Monitor dir for changes |
| `humanize config` | `humanize config` | Show config & API status |

**Key flags:** `-f <file>` · `-V` verbose · `-j` JSON · `-a` include GPTZero/Originality.ai · `-t <n>` threshold

## `humanizer` CLI — brandonwise

| Command | Example | Purpose |
|-------|------------|--------|
| `humanizer score` | `humanizer score < draft.txt` | Quick AI score |
| `humanizer analyze` | `humanizer analyze draft.md --verbose` | Full pattern detection |
| `humanizer suggest` | `humanizer suggest draft.md` | Improvement list |
| `humanizer humanize` | `humanizer humanize --autofix -f draft.md` | Guided rewrite + auto-fixes |
| `humanizer compare` | `humanizer compare --before v1.md --after v2.md` | Score delta |
| `humanizer report` | `humanizer report draft.md > report.md` | Full markdown report |
| `humanizer scan` | `humanizer scan ./papers --ext md,txt` | Batch folder analysis |

**Key flags:** `--verbose` · `--json` · `--autofix` · `--ignore-code` · `--fail-above <n>`

## Workflow (60–100 min per 500–1000 words)

| Stage | Action | Tool |
|----|--------|---------|
| 1. Baseline | Score before editing | `humanize score -f draft.md` |
| 2. Audit | Spot all AI markers | `humanizer analyze draft.md --verbose` |
| 3. Suggest | Get fix priorities | `humanize suggest -f draft.md` |
| 4. Voice | Contractions, asides, uneven paras | *(manual)* |
| 5. Transform | Apply safe auto-fixes | `humanize transform -f draft.md > v2.md` |
| 6. Cognition | Uncertainty markers, self-corrections | *(manual)* |
| 7. Compare | Confirm score improved | `humanizer compare --before v1.md --after v2.md` |

## Scoring Targets

| Score | Status |
|-------|---------|
| 0–25 | ✅ Human-like |
| 26–50 | ⚠️ Light AI — one more pass |
| 51–75 | ❌ Moderate — significant revision |
| 76–100 | ❌ Heavy — rewrite |

> Academic prose scores naturally higher. **< 40 is acceptable** for academic writing.

## What to Inject vs. Remove

| Remove (AI) | Inject (Human) |
|-------------|----------------|
| "delve", "leverage", "utilize" | "explore", "use" |
| Uniform 15–20 word sentences | Mix 5-word punches + 25-word elaborations |
| Perfectly balanced structure | Uneven paras, tangents |
| No contractions | *don't*, *wasn't*, *it's* throughout |
| Linear clean conclusion | Spiral reasoning, visible doubt |
| Abstract emotion ("I was sad") | Specific behavior ("I kept setting out two bowls") |
| — | Self-corrections: "Actually, that's not quite right…" |
| — | Uncertainty: "I suspect…", "I'm less sure about…" |

## Gotchas

- GPTZero free: 3 scans/day — use local `score` commands for iteration loops
- Grammarly will fight you; ignore ~50% of suggestions to preserve intentional imperfection
- `-a` flag on `humanize` calls external APIs (GPTZero/Originality.ai) — costs credits
- `--ignore-code` skips code blocks in technical/CS papers

TAGS: academic-writing, ai-detection, humanizer, cli, writing
