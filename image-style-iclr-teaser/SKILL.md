---
id: image-style-iclr-teaser
name: ICLR Teaser (Figure 1)
tags: [image-style, iclr, teaser, ml]
updated_at: 2026-05-05T17:37:40.235005Z
---

---
name: "ICLR Teaser (Figure 1)"
tags: [image-style, iclr, teaser, ml]
metadata:
  skill_kind: image-style
  default_size: "1536x1024"
  default_model: "gpt-image-2"
  venue_tags: ["iclr", "neurips"]
  description: "Two-panel before/after teaser. Editorial restraint. Reads at single-column thumbnail width."
  exemplars:
    - prompt: "Clean two-panel teaser figure for an ICLR paper on diffusion-model alignment. Left panel: noisy latent with chaotic gradient flow shown as tangled colored arrows in muted coral and ochre. Right panel: aligned latent with smooth parallel flows in muted teal and indigo. Minimal labels 'Before' and 'After' in serif. Soft white background, generous negative space, subtle drop shadows. 3:2 aspect, editorial Scientific-American aesthetic."
      model: "gpt-image-2"
      thumb: 1
    - prompt: "Two-panel ICLR teaser comparing reward-model behavior. Left: jagged scoring landscape rendered as crumpled topographic surface in faded rust. Right: smooth Bayesian-ridged landscape in muted sage. Tiny serif labels at panel bottoms. Off-white background, hairline panel borders, no decorative elements. 3:2 aspect."
      model: "gpt-image-2"
      thumb: 1
    - prompt: "ICLR Figure 1 teaser, two-panel. Left: a sparse attention map shown as scattered dots over a faded transformer outline, dim grey. Right: same map after structured pruning — clean diagonal cross-attention pattern in deep indigo. Both panels share thin stone-colored frame. Labels 'before pruning' / 'after pruning' in 10pt serif. Cream background, minimalist. 3:2."
      model: "gpt-image-2"
      thumb: 1
---

# ICLR Teaser (Figure 1)

Two-panel before/after teaser. Editorial restraint. Reads at thumbnail.

## Visual rules
- Exactly **two panels**, side by side, equal size.
- Each panel has at most one focal element. No clutter.
- Hairline panel borders (1px), thin shared frame.
- Drop shadows under panel content are *subtle* (4–8px blur, 30% opacity max).

## Composition cues
- 3:2 aspect overall. Panels are 1:1 square or 3:4 portrait.
- Generous outer margin (≥ 8% of canvas).
- Labels in **serif**, small (≤ 12pt), centered below each panel.
- The "after" panel always reads as more ordered / cleaner / more saturated.

## Color palette
- Background: `#f8f7f4` cream, off-white.
- "Before" anchor: `#c97a64` muted rust OR `#a3a3a3` neutral grey.
- "After" anchor: `#5a8da8` cool teal OR `#6f8159` sage green OR `#3d4f7a` indigo.
- Never both panels in the same hue family.

## Don't do
- ❌ Rendered LaTeX equations, axis numerics, citation strings — they will look like garbage.
- ❌ More than 2 panels (use `comparison-grid` skill instead).
- ❌ Heavy borders, pure-black ink, primary colors.
- ❌ Photorealistic textures inside panels (use illustration / vector style).

## Example prompts
1. *Diffusion alignment* — see exemplar 1 above. Best with `gpt-image-2`.
2. *Reward model smoothing* — see exemplar 2.
3. *Attention pruning* — see exemplar 3.

## When to pick a different skill
- For **3+ method comparison**: use `comparison-grid`.
- For **system overview** (pipeline of stages): use `neurips-system-overview`.
- For **explaining a method intuitively**: use `icml-method-cartoon`.
