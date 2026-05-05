---
id: image-style-visual-abstract
name: Visual Abstract Banner
tags: [image-style, visual-abstract, banner, narrative]
updated_at: 2026-05-05T08:55:32.991898Z
---

---
name: "Visual Abstract Banner"
tags: [image-style, visual-abstract, banner, narrative]
metadata:
  skill_kind: image-style
  default_size: "3840x2160"
  default_model: "gpt-image-2"
  venue_tags: ["nature", "neurips", "twitter", "landing-page"]
  description: "Single horizontal banner divided into 4 numbered stages with arrows. Editorial illustration, 16:9."
  exemplars:
    - prompt: "Visual abstract for a retrieval-augmented generation paper. Single horizontal banner divided into 4 numbered stages with thin tapered arrows between them: (1) 'Query' shown as cartoon question mark on a card, (2) 'Retrieve' shown as a small stack of documents being lifted, (3) 'Re-rank' shown as a vertical ordered list with #1 highlighted, (4) 'Generate' shown as flowing text emerging from a quill nib. Limited palette: deep indigo, warm rust, cream. Editorial illustration style with hand-drawn precision. Stage numbers in serif circles. 16:9."
      model: "gpt-image-2"
      thumb: 1
    - prompt: "Horizontal visual abstract for an active-learning paper, 4 panels left-to-right with arrows: (1) 'Unlabeled pool' as faint dots scattered in a circle, (2) 'Acquire' as a magnifying glass selecting a subset, (3) 'Label' as a friendly hand drawing tags onto selected dots, (4) 'Train' as the dots reshaping into a clean classifier boundary. Cream background, soft shadows, sage and rust accents. Numbered serif circles above each panel. 16:9."
      model: "gpt-image-2"
      thumb: 1
    - prompt: "Visual abstract for a clinical-NLP triage system, banner format, 5 stages: (1) 'Patient note' as a stack of paper, (2) 'NER + relation' as colored highlights overlaid, (3) 'Risk score' as a horizontal gauge, (4) 'Triage tier' as one of three sage/amber/coral lozenges, (5) 'Care path' as a forking road. Hand-drawn precision, hairline panel dividers, cream paper, warm-rust risk accent. Stage numbers in small serif circles. 21:9."
      model: "gpt-image-2"
      thumb: 1
---

# Visual Abstract Banner

Single horizontal banner divided into 3-5 numbered stages with arrows.

## Visual rules
- **One row** of panels, equal heights.
- Each stage = one icon / vignette + one short label (1-3 words).
- **Numbered serif circle** above each stage (1, 2, 3...).
- Thin tapered arrows between stages, never thick block arrows.

## Composition cues
- 16:9 standard; 21:9 for 5+ stages.
- Each stage is its own mini-figure; visually self-contained.
- Optional title strip at very top in serif, NOT inside any panel.
- Final stage often shows the "result" / "outcome" — make it the most saturated.

## Color palette
- Paper: `#f5f0e4` warm cream.
- Numbered circle ink: `#3a3530` warm near-black.
- Accent A (process / "good"): `#6f8159` sage.
- Accent B (data / "input"): `#c97a64` warm rust.
- Accent C (highlight / final stage): `#3d4f7a` deep indigo.
- Subtle drop shadows on icons (4px blur, 30% opacity).

## Don't do
- ❌ Real photos in the icons (use illustration / vignettes).
- ❌ Vertical stacking — this skill is horizontal-only.
- ❌ More than 5 stages (split into two rows = use a different skill).
- ❌ Colored arrows — keep them neutral grey.
- ❌ Long labels under icons. ≤ 3 words.

## Example prompts
1. *RAG pipeline* — exemplar 1.
2. *Active learning loop* — exemplar 2.
3. *Clinical NLP triage* — exemplar 3.

## When to pick a different skill
- For **architectural detail** (boxes + arrows for an engine): use `neurips-system-overview`.
- For **before/after**: use `iclr-teaser`.
- For **explaining a single intuition**: use `icml-method-cartoon`.
