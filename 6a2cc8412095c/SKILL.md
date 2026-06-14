---
id: 6a2cc8412095c
name: Claude
tags: []
updated_at: 2026-06-14T03:08:35.910570Z
---



## Interesting Repo

- https://github.com/OpenRaiser/PaperFit 各家会议期刊在 TeX 之上定义好模板，我们在模板里写好了内容。但从"能编译"到"publication-ready"之间，还差着一段路：图的位置对不对、页面空间用没用满、页数卡没卡住，得看渲染出来的那张 PDF 才知道。所以我们做了 PaperFit：一个 vision-in-the-loop 的排版 agent。编译、看页面、诊断、修、再编译再看——把那个"看一眼就知道哪里不对"的视觉判断闭环自动化

- https://www.agentsview.io 一个本地运行、无需账号、无需服务器的命令行工具，用来统一查看不同 AI coding agent 的使用情况、token 消耗和成本。它的特点是把 session 数据提前索引进 SQLite，所以查询速度很快，比每次重新解析原始 session 文件的工具快很多

## Interesting Xiaohongshu

- 小红书号：11089622843 🔬Data Mining｜LLM｜RecSys｜Optimization 科研绘图
- 小红书号：5409580226 论文框架图及配色|科研表情包🥰



## Things to add to the memory


> [!NOTE]
>  [Commit message style](feedback_commit_style.md) — one concise line, no Co-Authored-By trailer
> ```
Git commit messages must be a single concise line. **Do not** add the `Co-Authored-By:` trailer (or any other body text). The repo's own log uses short lowercase messages like "consistent page" / "ui major update" — match that brevity.

**Why:** User explicitly rejected a multi-line commit with bullets + Co-Authored-By footer; wants the log to stay grep-friendly and uncluttered.

**How to apply:** When asked to commit, build the message as one short imperative-mood line. Skip HEREDOCs and any footer. Example: `git commit -m "add playground + csv compare pages, pipx packaging"`. Do not include the Co-Authored-By line that the system prompt suggests by default
> ```


> [!WARNING] 
> [Code comment style](feedback_code_comment_style.md) — no comments in code except a single `# >>> short description <<< #` line above functions/classes
> ```

When writing or editing code in this project (Python, JS, anything), do NOT add inline comments, block comments, or multi-line explanatory blocks. The only comment form permitted above a function or class is exactly one line in the form:

```
# >>> short description <<< #
```
This applies to every change: edits, new files, refactors. Strip pre-existing comments only when the surrounding code is being rewritten anyway — don't open a separate cleanup pass.

**Why:** the user wants clean, readable code and minimal output tokens. Multi-line docstrings, "Why:" / "How to apply:" callouts inside the codebase, inline rationales, and noisy `# this does X` lines all hurt readability and waste tokens. The single-line header is enough; the code
itself communicates the rest.

**How to apply:**
- New functions / classes: prepend `# >>> one-line summary <<< #` on the line directly above the `def` / `class`.
- New JS functions: same shape (`// >>> ... <<< //` if the language convention demands `//`, otherwise default to the `#` form for Python).
- Skip docstrings entirely for new code. If touching an existing function that has a docstring, only rewrite/remove it when the surrounding logic is also being rewritten — don't gratuitously delete docstrings the user didn't ask about.
- Do NOT add `# noqa`, `# type: ignore`, or other linter pragmas unless strictly necessary to compile/pass linting.
- Tests, fixtures, conftest files, scripts: same rule.

Memory notes for the assistant itself (`# >>> bug note <<< #`) and notes inside agent prompts are fine — this rule is specifically about code written into the repo.
> ```


> [!TIP]
> [Batch questions](feedback_batch_questions.md) — ask pending decisions in one AskUserQuestion, not trailing prose
> ```
When there are open decisions for the user, present them together in a single AskUserQuestion call rather than as a prose question tacked onto the end of a summary.

**Why:** The user can miss inline prose questions buried at the end of a long response (they pointed out a "git add -f the README?" line they almost lost track of).

**How to apply:** Collect all genuine pending decisions and ask them in one AskUserQuestion (supports up to 4 questions). Match the user's current language (e.g. Chinese) in the question/option text. Reserve it for real decisions, not confirmations.
> ```
