---
id: 6a05656bb1abb
name: Markdown Editor Reference
tags: [reference, markdown, editor]
updated_at: 2026-06-07T03:25:56.954940Z
---

---
type: reference
audience: user
last_reviewed: 2026-06-01
---

Every markdown feature this editor supports, with live examples — the rendered preview *is* the documentation. For a terse one-line index aimed at the AI agent, see [[MD Cheatsheet]].

# Inline formatting

| Syntax | Renders | Notes |
|--------|---------|-------|
| `**bold**` or `Ctrl+B` | **bold** | wraps the current selection |
| `*italic*` or `Ctrl+I` | *italic* | select then press `*` to wrap |
| `~~strike~~` | ~~strike~~ | GFM |
| `==highlight==` | ==highlight== | a non-space char must touch the markers |
| `<Ctrl+Shift+P>` | <Ctrl+Shift+P> | any `<…>` that isn't real HTML / a URL → a key cap (`<kbd>`) |
| `` `code` `` | `code` | matched-tick spans |
| `[text](url)` or `Ctrl+K` | [text](https://example.com) | clipboard URL fills the slot |
| `[id]` + `[id]: url` | reference link | definition can sit anywhere |
| `<https://x.com>` | <https://example.com> | autolink |
| `[[Card Title]]` | wiki-link → matched card | click to open, hover to preview |
| `[[Card\|alias]]` | wiki-link with display text | optional pipe sets the label |
| `text[^1]` | footnote ref → bottom of card | hover for a tooltip preview |
| `$E=mc^2$` | inline math (KaTeX) | block form: `$$ … $$` |

> [!TIP]
> `==highlight==` only fires when a non-space character touches the markers: `==x==` ✓ but `== x ==` ✗.

# Block elements

## Headings (collapsible)

`#` through `######`. **H1 and H2 render as collapsible sections** — click the chevron to fold an entire section; each H1 gets a left rail when expanded.

## Lists

Nested bullets step through square → ring → dot. Ordered lists use bold accent-coloured numbers, switching to `a, b, c` then `i, ii, iii` as they nest.

- top-level item
  - second level (hollow ring)
    - third level (filled dot)
- another top

1. first step
2. second step
   1. nested a
   2. nested b
3. third step

Tasks:

- [ ] unfinished
- [x] done

> [!NOTE]
> Press **Enter** inside any list/quote marker and the next line repeats it automatically (numbered lists increment; checkboxes reset to `[ ]`). Press Enter on an empty marker to escape the list.

## Blockquote

> Plain blockquote uses `> `.
> Multiple lines stay grouped.

## Card excerpt / summary

Wrap a **whole line** in `>` … `<` to render a TL;DR summary box (`<ept>`). It is block-level only, so inline comparisons like `a > b < c` are never affected.

```md
>This editor turns a whole `>…<` line into a TL;DR summary box.<
```

>This editor turns a whole `>…<` line into a TL;DR summary box.<

## Tables

```md
| Left | Center | Right |
|:-----|:------:|------:|
| a    |   b    |     c |
```

| Left | Center | Right |
|:-----|:------:|------:|
| a    |   b    |     c |

The separator-row colons control text-align per column; the dash **count** sets relative width (`---` vs `-----` → ~40 % / 60 %).

## Horizontal rule

`---`, `***`, or `___` alone on a line.

---

# Math (KaTeX)

```md
Inline: $\alpha + \beta$
Display: $$\sum_{n=1}^{\infty} \frac{1}{n^2} = \frac{\pi^2}{6}$$
```

Inline: $\alpha + \beta$

$$\sum_{n=1}^{\infty} \frac{1}{n^2} = \frac{\pi^2}{6}$$

# Code blocks

````md
```python
def hello():
    return "world"
```
````

```python
def hello():
    return "world"
```

## Collapsible code blocks

Append `+` (open) or `-` (closed) to the language; anything after it becomes the summary title.

````md
```python+ How to install
pip install foo
```

```bash- Optional: build from source
./configure && make
```
````

```python+ How to install
pip install foo
```

```bash- Optional: build from source
./configure && make
```

`c++` stays the C++ language — only a *single* trailing `+`/`-` is treated as a marker.

# Custom block syntax

## Callouts

Five flavors — `NOTE` `TIP` `IMPORTANT` `WARNING` `CAUTION`. Suffix the type with `+`/`-` to make it collapsible; add `"Title"` to override the label.

```md
> [!NOTE]
> Static informational callout.

> [!TIP+] "Heads up"
> Collapsible — expanded by default.

> [!WARNING-] "Don't run on prod"
> Collapsible — collapsed by default.
```

> [!NOTE]
> Static informational callout.

> [!TIP+] "Heads up"
> Collapsible — expanded by default. Click the title strip to fold.

> [!WARNING-] "Don't run on prod"
> Collapsible — collapsed by default. Click to reveal.

Fenced code **works inside a callout** — prefix every line with `> `:

> [!IMPORTANT]
> ```bash+ chmod the installer first
> chmod +x install.sh
> ```

## Collapse block (`???` / `???+`)

`???` is collapsed by default; `???+` is open. Indent the body 4 spaces.

```md
???+ "Click to fold"
    Body is full markdown — **bold**, `code`, nested lists.
```

???+ "Click to fold"
    Body is full markdown — **bold**, `code`, even nested lists.

    - point a
    - point b

??? "Hidden details"
    Initially collapsed. Math renders here too: $\sqrt{a^2 + b^2}$.

## Content tabs (`===`)

Consecutive `=== "Title"` blocks merge into one tab strip. Each body is full markdown (indent 4 spaces).

```md
=== "Python"
    print("hi")
=== "JavaScript"
    console.log("hi")
```

=== "Python"
    ```python
    def greet(name: str) -> str:
        return f"Hello, {name}!"
    ```

=== "JavaScript"
    ```js
    const greet = (name) => `Hello, ${name}!`;
    ```

=== "Rust"
    ```rust
    fn greet(name: &str) -> String { format!("Hello, {}!", name) }
    ```

## Timeline (`:::timeline`)

Each indented `- ` bullet is one entry; text before the first colon is the date chip, the rest is inline markdown.

```md
:::timeline
    - 2024-01-15: Started the project
    - 2024-06-10: **Beta** with [docs](https://example.com)
```

:::timeline
    - 2024-01-15: Started the project
    - 2024-03-22: First commit — `alpha` tag
    - 2024-06-10: **Beta** release with [docs](https://example.com)
    - 2024-09-01: GA — v1.0

## Footnotes

An inline ref renders as a hoverable superscript chip[^1] with a floating tooltip. Refs are auto-numbered in order of appearance[^note], and a Footnotes section is appended at the bottom.

```md
First claim.[^1]
Second claim.[^note]

[^1]: Definition rendered at the bottom.
[^note]: Multi-line defs work if continuation lines indent 2+ spaces.
```

[^1]: Definition rendered at the bottom — supports **markdown** and `code`.
[^note]: Multi-line definitions work if continuation lines are indented by 2+ spaces.

## Image sizing

```md
![|50%](image.png)       # Obsidian — size in alt text
![|300x200](image.png)   # width × height
![](image.png =300)      # Typora — after a space
![](image.png =300x200)
```

Absolute local paths (whitelisted roots `~` `/home` `/data` `/tmp`) are auto-routed through `/file?path=…`:

```html
<img src="/file?path=/home/user/diagram.png" width="300">
```

## Frontmatter

A YAML block before the first heading renders as a compact key/value table above the body.

```yaml
---
author: Alice
status: draft
priority: high
---
```

# Cross-referencing cards

## Wiki links — `[[Card Title]]`

```md
See [[MD Cheatsheet]] for the syntax index.
See [[MD Cheatsheet|the index]]   ← optional pipe sets display text
```

Live: [[MD Cheatsheet]] · [[MD Cheatsheet|the index]] · [[Nonexistent Card]] (intentionally broken)

- Click a chip to jump to that card; hover for a title + tags preview.
- Drag a card from the **cascade popover** (⌄ by the title) onto the editor — drops a `[[Card Title]]` token and records the link in `metadata.links`.

> [!TIP-]
> Three ways to relate cards, by intent:
> - **Hierarchy** (parent / children) — set in the cascade popover
> - **DAG** (`metadata.links`, shown as ↔ related) — drag-drop or `+ link`
> - **Implicit grouping** (shared tags, shown as 🏷 by tag) — just tag them

# Editor shortcuts & features

## Keyboard

| Shortcut | Action |
|----------|--------|
| `Ctrl/Cmd+S` | Save the card |
| `Ctrl/Cmd+Z` / `Ctrl/Cmd+Shift+Z` | Undo / redo |
| `Ctrl/Cmd+F` | Find / replace widget |
| `Ctrl/Cmd+B` / `Ctrl/Cmd+I` / `Ctrl/Cmd+K` | Wrap selection: bold / italic / link |
| `Tab` / `Shift+Tab` | Indent / outdent (multi-line aware) |
| `Enter` on a list marker | Continue the list (empty marker → escape) |
| Select + `*` `_` `` ` `` `(` `[` `{` `"` `'` | Wrap the selection |
| `/` at line start | Slash menu — insertable blocks |

## Slash command menu

Type `/` at the start of a line to open a filterable menu: every heading level, lists, checklists, blockquotes, code (regular + collapsible), `???` collapsibles, `===` tabs, `:::timeline`, a 3×3 table, horizontal rule, math block, all five callouts, wiki links, and footnotes. Filter by typing, ↑/↓ to navigate, Enter/click to insert, Esc to cancel.

## Images

- **Paste** from the clipboard → uploaded under `/omi/images/<card-id>/` and inserted as `![](url)`.
- **Drag-and-drop** image files onto the textarea → same upload + insertion (multiple files per drop supported).
- Both need the card saved first (an ID namespaces the upload).

## View tools

- **Outline** — floating H1–H4 list; click to scroll (auto-targets the focus pane when no modal is open).
- **Reference cards** — pin any other card as a floating, resizable window.
- **Cascade** — parent / siblings / children dropdown for the current card.
- **Edit/View toggle** (focus pane) — swap rendered preview ↔ textarea without disturbing the AI chat.
- **Edit ↔ preview scroll sync** — proportional; both panes show the same fraction of the doc.

# Rules of thumb

- **Portability** (GitHub + oh-my-skill): stick to **GFM + `$math$`**; everything under *Custom block syntax* is oh-my-skill-only.
- **Indented bodies** (tabs / `???` / `:::timeline` / collapsible callouts): use **4 spaces** or 1 tab.
- **Image sizing**: `![|50%](path)` is the most portable form.
- **Highlight**: a non-space char must touch the `==` markers.
- **Long cards**: split into `##` sections (auto-collapsible) or break into child cards via the cascade — there is no lazy-segment mode.

# Quick reference

```
**bold** *italic* ~~strike~~ ==hl== `code` $math$  <key>  >excerpt line<
[link](url)  [[Wiki Link]]  text[^1]

# H1 (collapsible)    ## H2 (collapsible)    ### H3
- list   1. ordered   - [ ] task   > quote   ---

| col | col |     ```lang+ Title   > [!NOTE]
|-----|-----|     code              > body
| a   | b   |     ```               > ```bash …
                                    > ```

??? "title"       === "Tab A"      :::timeline
    body              content          - 2026-01-01: event

$$ block math $$  ![](img.png =50%)  ---
```
