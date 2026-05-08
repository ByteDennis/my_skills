---
id: image-style-comparison-grid
name: Comparison Grid (n×m)
tags: [image-style, results, grid, comparison]
updated_at: 2026-05-05T17:37:40.199875Z
---

---
name: "Comparison Grid (n×m)"
tags: [image-style, results, grid, comparison]
metadata:
  skill_kind: image-style
  default_size: "1536x1024"
  default_model: "gpt-image-2"
  venue_tags: ["neurips", "iclr", "cvpr"]
  description: "n×m results montage. Row labels on left, column labels on top, optional column dividers. Restrained borders."
  exemplars:
    - prompt: "2×3 grid comparison of segmentation results. Top row labelled 'Baseline U-Net', bottom row labelled 'Ours'. Three columns of medical scan images (chest X-rays) with overlaid segmentation masks. Each cell has a thin sage border. Subtle pastel mask colors: top-row masks rough/incomplete in faint coral, bottom-row masks precise in soft teal. Row labels in 11pt monospace at left edge, column labels in italic serif at top. Cream background, hairline grid, NeurIPS figure style. 16:9."
      model: "gpt-image-2"
      thumb: 1
    - prompt: "3×4 grid showing ablation: rows labelled 'No reg', 'L2 only', 'L2 + Dropout', columns labelled 'epoch 10', 'epoch 50', 'epoch 200', 'epoch 500'. Each cell shows a small generated digit image in soft greyscale, with sharper digits in lower rows / later columns. Hairline cream borders between cells. Tiny IBM Plex Mono labels. Off-white background, no other decoration. 16:9."
      model: "gpt-image-2"
      thumb: 1
    - prompt: "2×2 grid of style transfer results. Rows labelled 'Source', 'Stylized', columns labelled 'Style A: oil painting', 'Style B: ink wash'. Cells contain small landscape thumbnails with appropriate textures. Sage 1px cell borders, italic serif column labels, monospace row labels at left. Cream background, thin outer frame. 1:1 aspect."
      model: "gpt-image-2"
      thumb: 1
---

# Comparison Grid (n×m)

n×m results montage. Row labels on left, column labels on top.

## Visual rules
- Rectangular grid, cells equal size.
- Row labels at LEFT edge, column labels ABOVE the grid.
- Cell borders: 1px hairline, sage or warm-grey, NOT black.
- Inside each cell: one image / chart / icon. No internal text.

## Composition cues
- Aspect: 16:9 for wide grids (more cols than rows), 1:1 for square, 3:4 for tall.
- The "winning" row/col should be visually cleaner / more saturated to support the claim.
- Optional: bold-border around the "Ours" row to draw the eye.
- Captions outside the grid (top or bottom strip), small italic serif.

## Color palette
- Background: `#f8f7f4` cream.
- Cell borders: `#94a59c` sage (preferred) or `#a89c87` warm grey.
- Cell content: limited palette per row to keep visual coherence.
- Highlighted row border: `#3d4f7a` indigo, 2px.

## Don't do
- ❌ More than 5 cols or 5 rows in a single figure (split into multiple figures).
- ❌ Different aspect ratios per cell (chaos).
- ❌ Real photos AND illustrations mixed in the same grid.
- ❌ Heavy black gridlines or shadow-boxed cells.

## Example prompts
1. *Segmentation baseline vs ours* — exemplar 1.
2. *Ablation across regularization × epoch* — exemplar 2.
3. *Style-transfer source vs stylized × style* — exemplar 3.

## When to pick a different skill
- For **just two side-by-side panels**: use `iclr-teaser`.
- For **a multi-stage flow**: use `visual-abstract` or `neurips-system-overview`.
