---
name: gemini-academic-figures
tags: [local]
---

# Gemini Academic Figure Generation

Generate publication-ready figures for ML/AI conference papers using Google Gemini's image generation capabilities.

## 3-Stage Workflow

### Stage 1: ASCII Layout Preview
Given a research concept description, generate an ASCII diagram showing the layout and component relationships. Use boxes, arrows, and abbreviated labels. List full names of abbreviations below the diagram. The user reviews and edits the ASCII layout before any image generation.

### Stage 2: Low-Resolution Draft
Apply a style template to the approved ASCII layout and generate a small, low-resolution image for quick visual validation. Use `gemini-2.5-flash-image` at default resolution (~1024px). Iterate cheaply (~$0.04/image) until the user is satisfied with composition and content.

### Stage 3: High-Resolution Final
Once the draft is approved, generate the final publication-quality image using `gemini-3-pro-image-preview` at 2K or 4K resolution.

## How to Build the Prompt

```
[STYLE TEMPLATE (from templates/ folder)]

---

[CONTENT DESCRIPTION]

Title: "..."

ASCII Layout:
(paste the approved ASCII diagram from Stage 1)

Component Details:
- [Abbreviation]: [Full name] — [brief role description]
- ...
```

The style template defines the visual language. The content section defines what to draw. They are composed together at generation time.

## Models

| Model ID | Max Res | Speed | Cost | Use In |
|---|---|---|---|---|
| `gemini-2.5-flash-image` | 1024px | ~4s | ~$0.04 | Stage 2 (drafts) |
| `gemini-3-pro-image-preview` | 4096px | ~8-12s | ~$0.13-0.24 | Stage 3 (final) |

## Python SDK

```bash
pip install -U google-genai Pillow
```

```python
from google import genai
from google.genai import types
from PIL import Image
from io import BytesIO

client = genai.Client(api_key="YOUR_API_KEY")

def generate_figure(prompt, draft=True):
    model = "gemini-2.5-flash-image" if draft else "gemini-3-pro-image-preview"
    config_kwargs = {"aspect_ratio": "16:9"}
    if not draft:
        config_kwargs["image_size"] = "4K"

    response = client.models.generate_content(
        model=model,
        contents=prompt,
        config=types.GenerateContentConfig(
            response_modalities=["TEXT", "IMAGE"],
            image_config=types.ImageConfig(**config_kwargs),
        ),
    )
    for part in response.candidates[0].content.parts:
        if part.inline_data is not None:
            return Image.open(BytesIO(part.inline_data.data))
    return None
```

### Supported Aspect Ratios
`1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`

## Style Templates

Style templates live in `skills/gemini-academic-figures/templates/`. Each template defines a reusable visual style that gets prepended to the content prompt.

Available templates:

| Template | Style | Best For |
|---|---|---|
| **whiteboard-marker** | Hand-sketched dry-erase markers on whiteboard | Concept overviews, methodology, talks |
| **flat-vector-clean** | Crisp pastel flat shapes, grid-aligned | Camera-ready NeurIPS/ICML/ICLR figures |
| **deepmind-3d-blocks** | Isometric 3D blocks with depth shading | Teaser figures, transformer stacks, visual impact |
| **blueprint-technical** | Light-on-dark navy schematic | Systems ML, MLSys/OSDI, engineering-heavy papers |
| **minimal-grayscale** | Black/white/gray + one accent color | Theory papers, JMLR/PAMI, grayscale-safe printing |
| **gradient-modern** | Soft gradients with drop shadows | Industry papers (OpenAI/Meta style), blog-to-paper |

See `templates/` folder for full style prompt text for each.

## Key Techniques

1. **ASCII-first**: Always start with an ASCII layout so the user can verify structure before spending on image generation.
2. **Draft cheap, finalize once**: Iterate at ~$0.04/draft. Only generate 4K when layout is locked.
3. **Schema prompts**: Structure content as Component List + Connections, not narrative paragraphs.
4. **Color = meaning**: Assign each color to a semantic role consistently across the diagram.
5. **Single panel**: Generate multi-panel figures as separate images for better control.
6. **Reference image input**: Upload a sketch or previous figure to guide layout (Gemini 3 Pro supports up to 14 reference images).

## Useful Tools

- **gemimg** (`pip install gemimg`): Lightweight Gemini wrapper by Max Woolf. [GitHub](https://github.com/minimaxir/gemimg)
- **PaperBanana** (`pip install paperbanana`): Multi-agent academic figure framework. [GitHub](https://github.com/llmsresearch/paperbanana)

## Post-Processing

1. Import into EdrawMax / Inkscape / Illustrator for fine-tuning
2. Ensure consistent styling across all paper figures
3. Export at 300+ DPI for print

## References

- [Gemini API Image Generation Docs](https://ai.google.dev/gemini-api/docs/image-generation)
- [Google Prompting Guide](https://developers.googleblog.com/en/how-to-prompt-gemini-2-5-flash-image-generation-for-the-best-results/)
- [Nano Banana Prompt Engineering (Max Woolf)](https://minimaxir.com/2025/11/nano-banana-prompts/)
- [Methodology Diagram Guide](https://help.apiyi.com/en/nano-banana-pro-methodology-diagram-guide-en.html)
- [Figures for Papers (ML Community)](https://chengshuaizhao0.github.io/blog/2025/figures-for-papers/)