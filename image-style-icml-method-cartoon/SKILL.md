---
id: image-style-icml-method-cartoon
name: ICML Method Cartoon
tags: [image-style, icml, intuition, cartoon]
updated_at: 2026-05-05T08:55:32.911001Z
---

---
name: "ICML Method Cartoon"
tags: [image-style, icml, intuition, cartoon]
metadata:
  skill_kind: image-style
  default_size: "1536x1024"
  default_model: "gpt-image-2"
  venue_tags: ["icml", "neurips", "workshop"]
  description: "Whimsical hand-drawn cartoon explaining one method intuition. Cream paper, soft sage + warm rust accents."
  exemplars:
    - prompt: "Whimsical but precise cartoon explaining knowledge distillation: a large cartoon brain labelled 'Teacher' on the left, friendly face, whispering to a small cartoon brain labelled 'Student' on the right via a speech bubble containing soft probability bars. Background: faded textbook-page texture with hairline horizontal lines. Hand-drawn line style with cream paper, soft sage and warm rust accents. ICML workshop poster aesthetic, 3:2 aspect."
      model: "gpt-image-2"
      thumb: 1
    - prompt: "Hand-drawn cartoon illustrating contrastive learning: two cartoon dogs of the same breed pulled together by a friendly green arrow labelled 'positive pair', while a third cartoon cat is pushed away by a dashed coral arrow labelled 'negative'. Cream paper background with subtle grid. Sketchy ink lines, soft watercolor washes in sage and rust. Caption at bottom in handwritten serif: 'pull alike, push different'. 3:2."
      model: "gpt-image-2"
      thumb: 1
    - prompt: "Cartoon explaining mixture-of-experts: a friendly cartoon router (small robot at center) reading an envelope labelled 'input token' and pointing to one of three expert houses on a hill, each with a different roof color (sage, rust, indigo). Two darker houses are unlit. Hand-drawn ink + watercolor, cream paper, hairline dashed roads. ICML workshop aesthetic. 3:2."
      model: "gpt-image-2"
      thumb: 1
---

# ICML Method Cartoon

Whimsical hand-drawn cartoon explaining one method intuition. Cream paper, soft sage + warm rust accents, friendly characters.

## Visual rules
- **One central concept**, dramatized through cartoon characters or anthropomorphized objects.
- Hand-drawn ink line, slightly wobbly. NOT clean vector.
- Soft watercolor or pastel fills inside lines.
- Speech bubbles, arrows, and labels all hand-drawn-feeling.

## Composition cues
- 3:2 aspect, slight horizontal bias (story reads left-to-right).
- 1–3 main characters max; supporting elements can be small icons.
- Background = faded textbook-page or graph-paper texture.
- Caption at bottom in handwritten serif (e.g., Caveat, Architects Daughter feel).

## Color palette
- Paper: `#f5f0e4` warm cream.
- Ink: `#3a3530` near-black with slight brown tint, NOT pure black.
- Accent A: `#6f8159` sage green (the "good" / "positive" / "model" color).
- Accent B: `#c97a64` warm rust (the "bad" / "negative" / "data" color).
- Optional indigo `#3d4f7a` for "router" / "neutral" elements.
- Subtle wash backgrounds (10–15% opacity).

## Don't do
- ❌ Photorealism. This skill is for illustrations only.
- ❌ Pure-white backgrounds — always add a faint paper texture or warm tint.
- ❌ Sans-serif geometric labels — use handwritten serif throughout.
- ❌ More than 3 characters or more than 1 main concept per figure.

## Example prompts
1. *Knowledge distillation* — exemplar 1.
2. *Contrastive learning* — exemplar 2.
3. *Mixture-of-experts routing* — exemplar 3.

## When to pick a different skill
- For **rigorous architecture diagrams**: use `neurips-system-overview`.
- For **before/after**: use `iclr-teaser`.
- For **showing results across many cells**: use `comparison-grid`.
