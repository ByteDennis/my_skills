---
id: 6a05656bb1abb
name: Card Cascade
tags: [link]
updated_at: 2026-05-14T06:08:15.157871Z
---

## Relationship Types

| Kind | Stored as | Surfaced in cascade as |
|------|-----------|------------------------|
| Parent / child (tree) | `metadata.parent_id` | The 3 columns L1 / L2 / L3 |
| Related (DAG, free-form) | `metadata.links: [...]` | ↔ related pill row at the bottom |
| Tag clusters (implicit) | shared tags | 🏷 by tag pill row |

On disk: written as `parent: <id>` and `links: [<id>, <id>]` YAML frontmatter in `card.md`.

## Building Hierarchy

| Action | Effect |
|--------|--------|
| Click ⌄ next to title | Open cascade popover |
| Click a card row | Open that card |
| Click › chevron on a row | Drill into children without leaving current card |
| Drag card onto another card | Makes dragged card a child of target |
| Drag onto L1 empty space | Unparent → becomes top-level |
| Drag onto L2/L3 empty space | Child of that column's currently-selected parent |
| `+ new` at column bottom | Create child of that column's selection |

Drag indicators: ghost at 40% opacity (source), green-sage highlight (target). Cycle detection blocks loops (parent dropped onto its own descendant).

## Building the DAG (↔ Related)

- In cascade popover, click `+` in the ↔ related row → reference-window picker adds the card to `metadata.links`. PUT-saved immediately.
- Drag a card onto the editor textarea → drops `[[Card Title]]` at cursor **and** auto-adds to `metadata.links`.

## Reading Relationships

| Indicator | Meaning |
|-----------|---------|
| Breadcrumb at top | Full path (Database › SQL › Modal verify); each crumb clickable |
| ★ | Current card |
| ● | On the path to current |
| ▸ | Unrelated sibling |
| Number badge on row | Count of children |
