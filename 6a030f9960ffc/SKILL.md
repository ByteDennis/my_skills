---
id: 6a030f9960ffc
name: Markdown Dialects (cheatsheet)
tags: [markdown, obsidian, gfm, typora, cheatsheet]
updated_at: 2026-05-12T11:31:37.086018Z
---

Practical comparison of markdown extensions. Focus: what works where, and which syntax oh-my-skill renders.

## Feature matrix

| Feature | CommonMark | GFM | Obsidian | Typora | oh-my-skill |
|---|:-:|:-:|:-:|:-:|:-:|
| Tables \| col \| | ✗ | ✓ | ✓ | ✓ | ✓ |
| Strikethrough `~~x~~` | ✗ | ✓ | ✓ | ✓ | ✓ |
| Task list `- [ ]` | ✗ | ✓ | ✓ | ✓ | ✓ |
| Highlight `==x==` | ✗ | ✗ | ✓ | ✓ | ✓ |
| Footnote `[^1]` | ✗ | ✓ | ✓ | ✓ | partial |
| Alert `> [!NOTE]` | ✗ | ✓ | ✓ | ✗ | ✓ |
| Math `$x$` / `$$x$$` | ✗ | ✓ | ✓ | ✓ | ✓ KaTeX |
| Image sizing | ✗ | ✗ | `![\|50%]()` | `![]( =50%)` | both ✓ |
| Wiki link `[[Page]]` | ✗ | ✗ | ✓ | ✗ | ✗ |
| Embed `![[file]]` | ✗ | ✗ | ✓ | ✗ | ✗ |
| Hashtag `#tag` | ✗ | ✗ | ✓ | ✗ | via card tags |
| Frontmatter `---` | ✗ | ✓ | ✓ | ✓ | ✓ table |

## Obsidian-specific

### Wiki links

```
[[Note title]]               # bare link
[[Note|display text]]        # aliased
[[Note#Heading]]             # link to a heading
[[Note#^block-id]]           # link to a block
```

### Embeds (transclusion)

```
![[Note]]                    # embed another note
![[image.png]]               # embed image at natural size
![[image.png|300]]           # 300 px wide
![[image.png|300x200]]       # both dims
![[video.mp4]]               # embed video
```

### Image sizing via standard syntax (also rendered by oh-my-skill)

```
![|50%](image.png)           # alt-pipe — size goes in the alt text
![|300](image.png)
![|300x200](image.png)
```

### Callouts (GFM-compatible + Obsidian extras)

```
> [!NOTE]                    # core 5: NOTE, TIP, IMPORTANT, WARNING, CAUTION
> [!INFO]+                   # +/- suffix → expanded / collapsed by default
> body line 1
> body line 2
```

### Inline tags (Obsidian only — oh-my-skill ignores these)

```
#productivity
#research/llm                # nested with /
```

## Typora-specific

```
![](image.png =50%)          # image sizing — space then =
![](image.png =300)
![](image.png =300x200)
:smile: :rocket:             # emoji shortcodes
```

## GFM-only (vs CommonMark)

```
- [ ] todo
- [x] done                   # task lists

~~struck out~~               # strikethrough

> [!NOTE]                    # alert blocks (5 types, GitHub-rendered)
> body

http://auto-link.com         # bare URL → autolink
```

## What oh-my-skill renders

- ✓ All GFM: tables, task lists, strikethrough, alerts, autolinks
- ✓ `==highlight==` (boundary chars must be non-space — `==x==` ✓, `== x ==` ✗)
- ✓ KaTeX math: `$inline$` and `$$display$$`
- ✓ Image sizing in **both** Typora (`![](path =50%)`) and Obsidian (`![|50%](path)`) syntax
- ✓ Absolute host paths in `<img>` resolve via `/file?path=…` (whitelisted to `~`, `/data`, `/home`, `/tmp`)
- ✓ YAML frontmatter (`---…---`) rendered as a compact key/value table above the body
- ✗ `[[wiki-links]]` — no internal-link concept (link cards via shared tags instead)
- ✗ `![[embeds]]` — use plain `![](path)` (Obsidian-compatible)
- ✗ `#inline-tags` — tags live in the card's tag field, not in the body

## Rules of thumb

- **Portability** (works in GitHub + Obsidian + oh-my-skill): stick to **GFM + `$math$`**
- **Image at custom size**: `![|50%](path)` is most portable (Obsidian + oh-my-skill both render it)
- **Cross-reference between cards**: use a shared tag, not `[[wiki-links]]`
- **Heavy Obsidian features** (`[[]]`, `![[]]`, `#inline-tags`): great in Obsidian, won't render here or on GitHub
- **Highlight `==x==`**: put a non-space char immediately inside both markers, or it won't trigger
