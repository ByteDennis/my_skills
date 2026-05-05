---
id: image-style-neurips-system-overview
name: NeurIPS System Overview
tags: [image-style, neurips, pipeline, architecture]
updated_at: 2026-05-05T08:55:32.950909Z
---

---
name: "NeurIPS System Overview"
tags: [image-style, neurips, pipeline, architecture]
metadata:
  skill_kind: image-style
  default_size: "1536x1024"
  default_model: "gpt-image-2"
  venue_tags: ["neurips", "icml", "iclr"]
  description: "Horizontal pipeline / data-flow diagram with rounded-box stages, soft shadows, monospace labels."
  exemplars:
    - prompt: "Horizontal pipeline diagram of a neural training system. Five rounded boxes left-to-right: Raw data → Tokenizer → Transformer encoder (with stacked-layer icon inside) → Cross-attention head → Output logits. Connecting arrows in muted slate-grey. Two side branches feeding the encoder labelled 'positional emb' and 'segment emb'. Cream background, soft shadow under each box, monospace labels ≤ 3 words. NeurIPS figure aesthetic, 16:9."
      model: "gpt-image-2"
      thumb: 1
    - prompt: "Six-stage horizontal data-flow diagram for a multimodal retrieval system. Boxes: Image batch → Vision encoder (camera icon) → Joint embedding space → Cross-modal attention → Text decoder → Generated caption. Each box has a thin sage border and faint drop shadow. Arrows are thin tapered triangles. Labels in IBM Plex Mono, 11pt. Cream paper background, hairline divider beneath title. 16:9."
      model: "gpt-image-2"
      thumb: 1
    - prompt: "Pipeline figure for a federated learning paper. Three client boxes on left labelled 'Hospital A', 'Hospital B', 'Hospital C', each with a small clipboard icon. Arrows converge into a central rounded box labelled 'Aggregator' (cog icon). One return arrow from aggregator labelled 'updated weights'. Soft cream background, monospace labels, indigo accent on aggregator. 3:2."
      model: "gpt-image-2"
      thumb: 1
---

# NeurIPS System Overview

Horizontal pipeline / data-flow diagram. Rounded boxes, soft shadows, monospace labels, sage or slate accents on cream.

## Visual rules
- **Rounded rectangles** for processing stages (8–12px radius).
- **Circles** only for endpoints / states.
- **Arrows**: thin tapered triangles or stick arrows, NOT chunky block arrows.
- Each box gets a **single icon** (≤ 28px) inside, optional.
- All boxes the same height; widths can vary by label length.

## Composition cues
- 16:9 horizontal canvas for ≥ 4 stages; 3:2 for fewer.
- Equal horizontal spacing between boxes.
- Side-branches enter perpendicular (90°), labelled in italic small-caps.
- Optional title strip across the top in serif (no border).
- Optional caption strip at bottom in 10pt monospace.

## Color palette
- Background: `#f8f7f4` cream.
- Box fill: `#ffffff` (or `#fafafa`).
- Box border: `#94a59c` muted sage OR `#7a86a3` cool slate (1px).
- Drop shadow: `rgba(80, 70, 50, 0.08)`, 4px y-offset, 8px blur.
- Arrow ink: `#5b6660` warm grey.
- Accent box (aggregator / model under study): `#3d4f7a` indigo or `#6f8159` sage.

## Don't do
- ❌ Rendered LaTeX inside boxes — labels must be plain text.
- ❌ Gradients on box fills (looks like PowerPoint 2007).
- ❌ Colored arrows (stay neutral grey unless one specific arrow is highlighted).
- ❌ More than 7 stages on one row — break into two rows or use `visual-abstract`.

## Example prompts
1. *Transformer training* — exemplar 1.
2. *Multimodal retrieval* — exemplar 2.
3. *Federated learning* — exemplar 3.

## When to pick a different skill
- For **before/after**: use `iclr-teaser`.
- For **explaining intuition**: use `icml-method-cartoon`.
- For **multi-row results comparison**: use `comparison-grid`.
