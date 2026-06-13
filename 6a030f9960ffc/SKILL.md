---
id: 6a030f9960ffc
name: MD Cheatsheet
tags: [markdown, syntax, cheatsheet, md-guide]
updated_at: 2026-06-07T03:25:56.928131Z
---

---
type: agent-guide
audience: agent
---

One-line index of every markdown syntax the oh-my-skill editor supports. Use this to know the toolbox; for rendered examples and editor UX, see [[Markdown Editor Reference]].

## Inline

- `**bold**` — bold (or `Ctrl+B` on a selection)
- `*italic*` — italic (or `Ctrl+I`)
- `~~strike~~` — strikethrough (GFM)
- `==highlight==` — highlight; a non-space char must touch the markers
- `<key>` — keyboard key cap → `<kbd>` (e.g. `<Ctrl+Shift+P>`, `<Enter>`); any `<…>` that isn't a real HTML tag / URL becomes a key
- `` `code` `` — inline code
- `[text](url)` — link; `<url>` autolinks; `[id]` + `[id]: url` reference links
- `[[Card Title]]` / `[[Card|alias]]` — wiki-link to another card (click / hover / drag-drop)
- `text[^1]` — footnote; auto-numbered, hover tooltip, section appended
- `$x$` / `$$x$$` — inline / block math (KaTeX)

## Blocks

- `#` … `######` — headings; **H1 and H2 render as collapsible sections**
- `- ` / `* ` / `+ ` — bullet list (nested markers: square → ring → dot)
- `1. ` — ordered list (nesting numerals: 1 → a → i)
- `- [ ]` / `- [x]` — task list
- `> ` — blockquote
- `---` / `***` / `___` — horizontal rule
- `| a | b |` + separator `|:--|--:|` — table; colons set alignment, dash-count sets relative column width

## Code

- ` ```lang ` — fenced code block (syntax highlight + copy button)
- ` ```lang+ Title ` / ` ```lang- Title ` — **collapsible** code, open / closed; text after the language becomes the summary title (`c++` stays C++ — only a single trailing `+`/`-` is a marker)
- fenced code works **inside a callout** — prefix each line with `> `

## Custom blocks (oh-my-skill)

- `> [!NOTE|TIP|IMPORTANT|WARNING|CAUTION]` — alert callout (5 types)
- `> [!TIP+]` / `> [!TIP-]` — collapsible callout (open / closed)
- `> [!TIP] "Title"` — custom callout title
- `??? "Title"` / `???+ "Title"` — collapse block (closed / open); body indented 4 spaces
- `=== "Tab"` — content tabs; consecutive blocks merge into one strip; body indented 4 spaces
- `:::timeline` — timeline; each `- date: text` bullet is one entry; body indented 4 spaces
- `![|50%](img)` · `![](img =300x200)` — image sizing (Obsidian / Typora syntaxes)
- `>summary line<` (a **whole line**) — card excerpt / TL;DR box → `<ept>`; whole-line only, so inline `a > b` is safe
- `---` … `---` YAML at the very top — frontmatter → key/value table

## Gotchas

- Tabs / `???` / `:::timeline` / collapsible callouts: indent the nested body **4 spaces** (or 1 tab) — MkDocs-Material convention.
- `==highlight==`: a non-space char must immediately touch the `==` markers.
- Portability: for GitHub + oh-my-skill stick to **GFM + `$math$`**; everything under "Custom blocks" is oh-my-skill-only.
- Long cards: split into `##` sections (auto-collapsible) or break into child cards via the cascade hierarchy. There is **no** lazy-segment mode.
- Cross-reference cards three ways: **hierarchy** (cascade parent/children), **DAG** (`[[wiki-links]]` / `metadata.links`, shown as ↔ related), **implicit** (shared tags, shown as 🏷 by tag).
