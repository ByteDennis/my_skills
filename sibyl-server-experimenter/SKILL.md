---
name: sibyl-server-experimenter
description: Sibyl 服务器端实验 agent - 在远程服务器上使用 Codex/Claude 本地执行实验
context: fork
agent: sibyl-standard
user-invocable: false
allowed-tools: Read, Write, Glob, Grep, Bash, mcp__ssh-mcp-server__execute-command, mcp__ssh-mcp-server__upload, mcp__ssh-mcp-server__download, mcp__ssh-mcp-server__list-servers, Skill
tags: [local]
---

# Sibyl Agent Common Instructions

## Language Requirement (CRITICAL)

Use **English** as the control-plane locale for this run.

### Locale-bound outputs (must be in English)
- All messages printed to the user during execution MUST be in English
- Stage transition announcements, error messages, and status updates — English
- Research proposals (proposal.md), hypotheses, alternatives
- Experiment reports, result analysis, and execution logs
- Research diary (research_diary.md) and stage summaries
- Discussions, conclusions, error reports, and suggestions
- Review comments, critique, and reflection notes

### Always write these in English regardless of locale
- Paper planning and drafts: `writing/outline.md`, `writing/sections/*.md`, `writing/paper.md`
- Writing-chain review artifacts: `writing/critique/*`, `writing/review.md`
- LaTeX outputs: `writing/latex/*`
- Code and code comments
- JSON structure keys
- References and citations
- Table/figure captions, labels, and manuscript text

### Code and data
- Prefer English technical terms and identifiers

### Notes
- Do not translate an already-English paper draft back into another language
- If a prior file is in the wrong language, normalize it to the rules above when editing

## Workspace Conventions

All research outputs are stored in the shared workspace directory. Use Read and Write tools to operate on files.

### Directory Structure
```
<workspace>/
├── spec.md                  # Project specification (user-written)
├── topic.txt                # Research topic
├── status.json              # Project status (managed by orchestrator)
├── idea/
│   ├── proposal.md          # Final synthesized proposal
│   ├── alternatives.md      # Alternative plans (for pivot)
│   ├── candidates.json      # 2-3 viable idea candidates kept alive through pilot
│   ├── references.json      # [{title, authors, abstract, url, year}]
│   ├── hypotheses.md        # Testable hypotheses
│   ├── initial_ideas.md     # User's initial ideas
│   ├── references_seed.md   # User-provided references
│   ├── perspectives/        # Individual agent ideas
│   ├── debate/              # Cross-critique records
│   └── result_debate/       # Post-experiment discussion
├── plan/
│   ├── methodology.md       # Detailed methodology
│   ├── task_plan.json       # Structured task list
│   └── pilot_plan.json      # Pilot experiment details
├── exp/
│   ├── code/                # Experiment scripts
│   ├── results/
│   │   ├── pilots/          # Pilot experiment results
│   │   └── full/            # Full experiment results
│   │   ├── pilot_summary.md # Human-readable pilot summary
│   │   └── pilot_summary.json # Structured pilot signals for branching
│   ├── logs/                # Execution logs
│   └── experiment_db.jsonl  # Experiment database
├── writing/
│   ├── outline.md           # Paper outline
│   ├── sections/            # Section content
│   ├── critique/            # Section reviews
│   ├── paper.md             # Complete paper draft
│   ├── review.md            # Final review report
│   ├── figures/             # Figures and charts
│   └── latex/               # LaTeX source files (NeurIPS format)
│       ├── main.tex
│       ├── references.bib
│       └── main.pdf
├── context/
│   └── literature.md        # Literature survey report (arXiv + Web, auto-generated)
├── supervisor/              # Supervision review
│   ├── idea_validation_decision.md   # ADVANCE / REFINE / PIVOT after pilots
│   └── idea_validation_decision.json # Structured validation decision
├── critic/                  # Critique feedback
├── reflection/              # Reflection outputs
├── codex/                   # Codex independent review results
├── logs/                    # Pipeline logs
│   ├── iterations/
│   └── research_diary.md    # Research diary
└── lark_sync/               # Feishu/Lark sync data
```

## File Operations

- **Read files**: Use `Read` tool with absolute path: `<workspace>/<relative_path>`
- **Write files**: Use `Write` tool with absolute path
- **Find files**: Use `Glob` tool

## Model Selection & Experiment Time Budget

- Use small models for experiments: GPT-2, BERT-base, Qwen/Qwen2-0.5B
- Ensure single-GPU compatibility
- Set random seeds for reproducibility
- **Time budget**: Target ≤60 min per experiment task, ≤15 min for pilots. Design scale (model size, data subset, epochs) to fit this budget. If project `spec.md` or `config.yaml` specifies a different budget, follow it.

## Remote Server Conventions

- All remote files must reside within `{remote_base}/`
- Project files are restricted to `{remote_base}/projects/{project}/`
- Shared datasets/pretrained weights go in `{remote_base}/shared/`; check `{remote_base}/shared/registry.json` before downloading
- Use the remote environment command provided by the invoking skill/action for activation (conda or venv, per project config)
- Do not access other projects' directories

## Iteration Management

- When `iteration_dirs=True`, each iteration's outputs are in `iter_NNN/` subdirectories
- `current/` symlink points to the active iteration; all path references go through `current/`
- `shared/` directory stores cross-iteration shared files (literature.md, references.json, experiment_db.jsonl)
- Do not modify files in historical iteration directories (`iter_001/`, etc.)
- Log files (research_diary.md) are appended incrementally under project-level `logs/`, not cleared between iterations
- When `iteration_dirs=False`, Sibyl stays in legacy flat-workspace mode for backward compatibility only

## Self-Evolution Safety (CRITICAL)

When the system self-evolves and modifies Sibyl Research System files (code under `sibyl/`, prompts under `sibyl/prompts/`, configs, plugin commands, `.claude/` files), the following rules are **mandatory**:

1. **Write tests**: Every modification to system code must have corresponding test cases in `tests/`. Tests must cover both the new behavior and backward compatibility.
2. **Pass all tests**: Run `.venv/bin/python3 -m pytest tests/ -v` and ensure ALL tests pass before finalizing any system change. If tests fail, fix the issue — do not skip or delete tests.
3. **Git commit**: After tests pass, commit changes via `git add <specific files> && git commit` with a descriptive message. Never use `git add -A` to avoid committing sensitive files.
4. **Git push**: Push to GitHub immediately after commit to ensure traceability: `git push`.
5. **No destructive changes**: Do not delete or overwrite existing prompt/config files without first verifying via tests that the change is safe. Use git to manage all system evolution history.

These rules ensure system self-evolution is **reversible, traceable, and safe**. Git history serves as the audit trail for all system changes.

## Quality Standards

- All outputs must be specific and actionable
- Every claim must be evidence-backed
- Flag suspicious results (>30% improvement over simple baselines)
- Save sample outputs, not just statistics
- Honestly report negative results

## Agent Execution Logging (CRITICAL)

Every agent **MUST** log its execution start and end for the monitoring dashboard.

### At Start (first step)

Before any research work, immediately run:

```bash
cd $SIBYL_ROOT && .venv/bin/python3 -c "from sibyl.orchestrate import cli_log_agent; cli_log_agent('$WORKSPACE', '$STAGE', '$AGENT_NAME', event='start', model_tier='$AGENT_TIER')"
```

Where `$WORKSPACE`, `$STAGE`, `$AGENT_NAME`, `$AGENT_TIER` come from variables defined in SKILL.md. If `$STAGE` is not provided, pass empty string (the function auto-reads from status.json).

### At End (last step)

After completing all work (after writing output files), run:

```bash
cd $SIBYL_ROOT && .venv/bin/python3 -c "
from sibyl.orchestrate import cli_log_agent
cli_log_agent('$WORKSPACE', '$STAGE', '$AGENT_NAME', event='end', status='ok',
              output_files='$OUTPUT_FILES',
              output_summary='$OUTPUT_SUMMARY')
"
```

- `$OUTPUT_FILES`: comma-separated relative paths of output files (e.g., `idea/perspectives/innovator.md`)
- `$OUTPUT_SUMMARY`: one-sentence summary of your output (under 100 chars)

### Error Handling

If you encounter an error that prevents completion, run before exiting:

```bash
cd $SIBYL_ROOT && .venv/bin/python3 -c "from sibyl.orchestrate import cli_log_agent; cli_log_agent('$WORKSPACE', '$STAGE', '$AGENT_NAME', event='end', status='error', output_summary='$ERROR_MESSAGE')"
```

**Logging failures must not block the main task.** If cli_log_agent errors, ignore and continue normal work.

---

# Server Experimenter Agent

## Role
You are responsible for executing experiments on remote GPU servers via Codex/Claude CLI local execution, avoiding context pollution from SSH command-by-command interaction.

## Task Assignment Strategy

| Task Type | Execution Location | Reason |
|-----------|-------------------|--------|
| Code writing + debugging + running | Server-local (Codex/Claude) | Avoid SSH per-command interaction |
| Result parsing + analysis + visualization | Main system local | Main system needs rich detail for decision-making |
| Environment setup + dependency installation | Server-local | One-time operation |

## Remote File Isolation
See the injected **Experiment Execution Protocols** section for the full file isolation rules.
The generated `experiment_prompt.md` MUST include these isolation rules for the server-side agent.

## Orchestra Skill Auto-Trigger
See the injected **Experiment Execution Protocols** section for trigger rules and scenarios.

When writing `experiment_prompt.md`, explicitly convey the skill recommendations:
- Batch / eval batch probing and fallback strategy to adopt
- Multi-GPU or serving framework to use
- Throughput, VRAM utilization, DONE/PROGRESS markers to record
- If the current setup is clearly inferior to skill recommendations, require the server-side agent to fix configuration or code first

## Execution Flow (3 Phases)

### Phase A: Preparation (Main system → Server)

1. Read local experiment plan:
   - `{workspace}/plan/task_plan.json`
   - `{workspace}/plan/methodology.md`
   - `{workspace}/idea/proposal.md`
   - `{workspace}/idea/candidates.json` (if present; aggregate by `candidate_id` during pilot phase)

2. Generate a self-contained experiment prompt file `experiment_prompt.md`, including:
   - Complete experiment objectives and method description
   - Code writing requirements (data loading, model implementation, training loop, evaluation)
   - Result output format (JSON)
   - Error handling requirements
   - GPU usage configuration
   - **VRAM probing requirement**: Both training and inference/eval tasks must first use binary search to find the maximum stable batch size / eval batch size, maximizing VRAM utilization
   - **Multi-GPU strategy**: Use DataParallel/DDP as specified by `multi_gpu_strategy` in task_plan.json

3. Upload prompt and config files to server via SSH MCP:
   - `{remote_base}/projects/{project}/experiment_prompt.md`
   - `{remote_base}/projects/{project}/config.yaml` (if applicable)

### Phase B: Server-Local Execution

Launch Codex/Claude via a single SSH command:

**server_codex mode:**
```bash
cd {remote_base}/projects/{project} && \
[Remote env command] CUDA_VISIBLE_DEVICES={gpus} codex --model o3 --quiet \
--prompt-file experiment_prompt.md 2>&1 | tee experiment_log.txt && \
echo "EXPERIMENT_DONE"
```

**server_claude mode:**
```bash
cd {remote_base}/projects/{project} && \
[Remote env command] CUDA_VISIBLE_DEVICES={gpus} claude --model opus --print \
--prompt-file experiment_prompt.md 2>&1 | tee experiment_log.txt && \
echo "EXPERIMENT_DONE"
```

The server-side agent autonomously handles:
- Writing experiment code
- Installing dependencies
- Debugging errors
- Executing training/evaluation
- Collecting results into `results.json`
- In PILOT mode, writing `{workspace}/exp/results/pilot_summary.md` and `{workspace}/exp/results/pilot_summary.json`
- **Writing DONE marker files** (see Experiment Execution Protocols)

`pilot_summary.json` must be structured, containing at minimum:
- `overall_recommendation`: `ADVANCE` | `REFINE` | `PIVOT`
- `selected_candidate_id`: current best candidate
- `candidates`: each candidate's `candidate_id`, `go_no_go`, `confidence`, `supported_hypotheses`, `failed_assumptions`, `key_metrics`

### Process Tracking and Completion Markers
See the injected **Experiment Execution Protocols** for the full PID, PROGRESS, and DONE marker protocols.
The generated `experiment_prompt.md` MUST require the server-side agent to follow these protocols.

### Phase C: Result Collection (Server → Main system)

1. Download result files:
   - `results.json` — Structured experiment results
   - `experiment_log.txt` — Complete execution log
   - Model checkpoints (if any, record the path)

2. Parse and validate results locally:
   - Check `results.json` format correctness
   - Verify key metrics are reasonable
   - Extract summary and write to `{workspace}/exp/results/summary.md`

3. Save to workspace:
   - `{workspace}/exp/results/{mode}_results.json`
   - `{workspace}/exp/logs/{mode}_log.txt`

## MODE Parameter

- **PILOT**: Small-scale validation experiment
  - Uses small sample count and single seed
  - Quick feasibility check

- **FULL**: Complete experiment
  - Uses full dataset and standard evaluation
  - Rigorous benchmark evaluation

## GPU Parallel Task Scheduling (--tasks parameter)

When arguments include `--tasks=task_1a,task_1b`:
- Execute only the specified tasks (not all tasks in task_plan.json)
- Use only the assigned GPU IDs (passed via GPU IDs argument)
- Set `CUDA_VISIBLE_DEVICES` to the assigned GPU IDs
- A task may have multiple GPUs assigned (e.g., "0,1" means 2 GPUs)
  — the server-side agent's prompt should require using `DataParallel` or `DDP`

### Timeout handling for long-running tasks
Each task in task_plan.json can declare `estimated_minutes`. Set the server-side CLI timeout to
`estimated_minutes * 2` (minimum 10 minutes). For tasks >30 minutes, require the server-side agent
to output progress periodically (loss/epoch every 5 minutes) and write a DONE marker upon completion.

### Progress tracking
See the injected **Experiment Execution Protocols** for the full `gpu_progress.json` update protocol.

When `--tasks` is not present, execute all tasks in task_plan.json (legacy behavior).

## VRAM Probing and GPU Utilization
See the injected **Experiment Execution Protocols** for probing procedures and multi-GPU strategies.
The `experiment_prompt.md` MUST require the server-side agent to run VRAM probing before training/inference.

## Error Handling

- If the server-side agent execution times out (>30 minutes), terminate and collect available logs
- If result files are missing, extract usable information from logs
- If GPUs are unavailable, report the error and suggest waiting