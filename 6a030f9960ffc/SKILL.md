---
id: 6a030f9960ffc
name: MD Cheatsheet
tags: [markdown, obsidian, gfm, typora, cheatsheet]
updated_at: 2026-05-22T20:09:35.718184Z
---

---
lazy: false
---

## Feature matrix

| Feature | CommonMark | GFM | Typora | oh-my-skill |
|--|:-:|:-:|:-:|:-:|
| Tables `\| col \|` | ✗ | ✓ | ✓ | ✓ |
| Strikethrough `~~x~~` | ✗ | ✓ | ✓ | ✓ |
| Task list `- [ ]` | ✗ | ✓ | ✓ | ✓ |
| Highlight `==x==` | ✗ | ✗ | ✓ | ✓ |
| Footnote `[^1]` | ✗ | ✓ | ✓ | ✓ tooltip |
| Alert `> [!NOTE]` | ✗ | ✓ | ✗ | ✓ + collapse + title |
| Collapse `??? "…"` | ✗ | ✗ | ✗ | ✓ |
| Tabs `=== "…"` | ✗ | ✗ | ✗ | ✓ |
| Timeline `:::timeline` | ✗ | ✗ | ✗ | ✓ |
| Lazy segments | ✗ | ✗ | ✗ | ✓ for long meeting logs |
| Math `$x$` / `$$x$$` | ✗ | ✓ | ✓ | ✓ KaTeX |
| Image sizing | ✗ | ✗ | `=50%` | both ✓ |
| Frontmatter `---` | ✗ | ✓ | ✓ | ✓ table |
| Link `[txt](url)` | ✓ | ✓ | ✓ | ✓ |
| Wiki-link `[[card]]` | ✗ | ✗ | ✗ | ✓ render + drag-drop |

## Text Formatting

| Syntax | Rendered |
|---|---|
| `**bold**` | **bold** |
| `*italic*` | *italic* |
| `~~strikethrough~~` | ~~strikethrough~~ |
| `==highlight==` | ==highlight== |
| `` `inline code` `` | `inline code` |

> `==highlight==` — non-space char must touch markers: `==x==` ✓  `== x ==` ✗

## Tables (with alignment)

The separator-row colons control text-align per column; dash counts control width.

```md
| Left | Center | Right |
|:-----|:------:|------:|
| a    |   b    |     c |
```

| Left | Center | Right |
|:-----|:------:|------:|
| a    |   b    |     c |

## Task Lists

```md
- [ ] unfinished
- [x] done
```

- [ ] unfinished
- [x] done

## Lists (stylish markers)

Nested bullets step through square → ring → dot. Ordered lists use bold accent-coloured numbers, switching to `a, b, c` then `i, ii, iii` on nesting.

- top-level item
  - second level (hollow ring)
    - third level (filled dot)
- another top
  - and child

1. first step
2. second step
   1. nested a
   2. nested b
      1. deeper i
      2. deeper ii
3. third step

## Alerts

5 types: `NOTE` `TIP` `IMPORTANT` `WARNING` `CAUTION`.

- Add `+` (open) or `-` (collapsed) after the type for a collapsible variant.
- Add `"Custom title"` after the bracket to override the default label.

```md
> [!NOTE]
> Static informational callout.

> [!TIP+] "Heads up"
> Collapsible with a custom title, expanded by default.

> [!WARNING-] "Don't run on prod"
> Collapsible with a custom title, collapsed by default.
```

> [!NOTE]
> Static informational callout.

> [!TIP+] "Heads up"
> Collapsible — expanded by default. Click the title strip to fold.

> [!WARNING-] "Don't run on prod"
> Collapsible — collapsed by default. Click to reveal hidden content.

> [!IMPORTANT] "Read first"
> Custom title overrides the default label.

## Collapse Block (`???` / `???+`)

Indent the body by 4 spaces. `???` is collapsed by default; `???+` is open.

```md
???+ "Click to fold"
    Body is full markdown — **bold**, `code`, even nested lists work.

??? "Hidden details"
    Initially collapsed.
```

???+ "Click to fold"
    Body is full markdown — **bold**, `code`, even nested lists work.

    - point a
    - point b

??? "Hidden details"
    Initially collapsed.

    Math also renders inside: $\sqrt{a^2 + b^2}$.

## Tabs (`===` blocks)

Consecutive `=== "Title"` blocks become one tab strip. Each tab body is full markdown — including code, math, admonitions.

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
    fn greet(name: &str) -> String {
        format!("Hello, {}!", name)
    }
    ```

## Timeline (`:::timeline`)

Material-style fenced block — matches the tabs / collapse family.
Each indented `- ` bullet becomes one entry. Body before the first colon is
the date chip; the rest renders as inline markdown.

```md
:::timeline
    - 2024-01-15: Started the project
    - 2024-03-22: First commit — `alpha` tag
    - 2024-06-10: **Beta** release with [docs](https://example.com)
    - 2024-09-01: GA — v1.0
```

:::timeline
    - 2024-01-15: Started the project
    - 2024-03-22: First commit — `alpha` tag
    - 2024-06-10: **Beta** release with [docs](https://example.com)
    - 2024-09-01: GA — v1.0

## Footnotes

Inline ref renders as a hoverable superscript chip[^1] with a floating tooltip. Multiple refs are auto-numbered in order of appearance[^note] and a Footnotes section is appended at the bottom of the card.

```md
First claim.[^1]
Second claim.[^note]

[^1]: Definition rendered at the bottom.
[^note]: Multi-line defs work if continuation lines
    are indented by 2+ spaces.
```

[^1]: Definition rendered at the bottom — supports **markdown** and `code`.
[^note]: Multi-line definitions are supported if continuation lines
    are indented by 2+ spaces.

## Math (KaTeX)

```md
Inline: $E = mc^2$

Display:
$$\sum_{n=1}^{\infty} \frac{1}{n^2} = \frac{\pi^2}{6}$$
```

Inline: $E = mc^2$

$$\sum_{n=1}^{\infty} \frac{1}{n^2} = \frac{\pi^2}{6}$$

## Image Sizing

Two syntaxes both work:

```md
![|50%](image.png)       # Obsidian-style — size in alt text
![|300](image.png)       # fixed px width
![|300x200](image.png)   # width × height
![](image.png =300)      # Typora-style — after a space
![](image.png =300x200)
```

Local absolute paths via `<img>` (whitelisted roots: `~`, `/data`, `/home`, `/tmp`):

```html
<img src="/file?path=/home/user/diagram.png" width="300">
```

## Frontmatter

YAML block before the first heading — rendered as a compact key/value table above the body:

```yaml
---
author: Alice
status: draft
priority: high
---
```

## Links

```md
[inline link](https://example.com)
[link with title](https://example.com "hover")
[reference link][id]   ← prose body
[id]: https://example.com  ← collected anywhere

<https://autolink.com>
<email@host.com>

[Anchor to a heading](#feature-matrix)   ← lowercase, spaces → -
```

[inline link](https://example.com) · [reference][ex] · <https://autolink.com>

[ex]: https://example.com

### Wiki-style card links — `[[Card Title]]` (oh-my-skill only)

```md
See [[Card Cascade]] for syntax.
See [[Card Cascade|cascade docs]]   ← optional pipe sets display text
```

Live examples: [[Card Cascade]] · [[Git Skills]] · [[Card Cascade|cascade docs]] · [[Nonexistent Card]] (intentionally broken)

- Click any chip to jump to that card.
- Hover for a quick preview (title + tags).
- Drag any card from the **cascade popover** (⌄ next to the title) onto the
  editor textarea — drops a `[[Card Title]]` token at the cursor and adds
  the source card to this card's `metadata.links` (DAG cross-reference).
- Linked cards appear in the cascade popover's **↔ related** row.

> [!TIP-]
> Cross-reference cards three ways depending on intent:
> - **Hierarchy** (parent / children) — set in the cascade popover
> - **DAG** (`metadata.links`, surfaced as `↔ related`) — drag-drop or `+ link`
> - **Implicit grouping** (shared tags, surfaced as `🏷 by tag`) — just tag

## Rules of Thumb

- **Portability** (GitHub + oh-my-skill): stick to **GFM + `$math$`**
- **Image sizing**: `![|50%](path)` is most portable
- **Cross-reference cards**: hierarchy (cascade) for tree, `[[wiki-links]]` (drag-drop) for DAG, shared tags for implicit clusters
- **Highlight**: non-space char must immediately touch `==` markers
- **Tabs / collapse / admonition-collapse / timeline**: indented body must use **4 spaces** (or 1 tab) — MkDocs-Material convention
- **Meeting-log cards**: write each meeting as a `## YYYY-MM-DD …` H2 heading. When you cross 8 sections, oh-my-skill auto-switches to lazy mode — newest 5 render eagerly, the rest become stubs hydrated via a **Show 5 older** button. Opt out with frontmatter `lazy: false`; tune with `segments: 10`, `segmentMin: 20`, `reverse: false`. Alternative: use the cascade hierarchy (parent + monthly child cards) instead of one growing card.
