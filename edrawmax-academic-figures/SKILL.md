---
name: edrawmax-academic-figures
description: Use Wondershare EdrawMax for creating and refining academic conference paper figures. Use when the user mentions EdrawMax, Edraw, Wondershare, or wants to post-process AI-generated figures, create vector diagrams, or produce publication-ready figures using EdrawMax. Covers EdrawMax AI features, templates, and integration with Gemini-generated figures.
version: 1.0.0
tags: [local]
---

# EdrawMax for Academic Paper Figures

Use Wondershare EdrawMax to create, edit, and refine publication-ready figures for ML/AI conference papers. EdrawMax serves both as a standalone diagram tool and as a post-processing editor for AI-generated figures (e.g., from Gemini).

## Two Workflows

### Workflow A: EdrawMax as Primary Tool (Native Diagramming)
Create figures from scratch using EdrawMax's built-in shapes, templates, and AI features.

### Workflow B: EdrawMax as Post-Processor (Gemini + EdrawMax)
1. Generate initial figure with Gemini (see `gemini-academic-figures` skill)
2. Import raster output into EdrawMax
3. Refine layout, fix labels, ensure consistency
4. Export as high-resolution vector or raster

## EdrawMax AI Features

### AI-Powered Diagram Generation
EdrawMax includes built-in AI capabilities for generating diagrams from text descriptions:

1. **Edraw AI Assistant**: Open via the AI button in the toolbar
2. **One-click generation**: Describe your diagram in natural language
3. **Supported types**: Flowcharts, mind maps, org charts, timelines, lists, tables
4. **Refinement**: Edit AI-generated diagrams with full EdrawMax tools

### AI Flowchart Generation
1. Go to **New > AI Generated** or click the **AI** icon
2. Select "Flowchart" as diagram type
3. Enter description: e.g., "Training pipeline for a Vision Transformer with data augmentation, encoder, classification head, and loss computation"
4. EdrawMax generates an editable vector flowchart
5. Customize colors, fonts, shapes to match paper style

### AI Mind Map / Architecture Diagram
1. Select "Mind Map" from AI generation
2. Describe the model architecture hierarchically
3. Convert mind map nodes into a custom architecture diagram layout

## Academic Figure Templates

EdrawMax provides templates relevant to academic papers:

### Useful Template Categories
- **Flowcharts**: Algorithm flowcharts, decision trees, process flows
- **Network Diagrams**: Neural network architectures, system architectures
- **Science & Education**: Scientific illustrations, experiment workflows
- **Block Diagrams**: System block diagrams, pipeline diagrams
- **UML Diagrams**: Sequence diagrams, class diagrams (for software-oriented papers)

### Accessing Templates
1. **File > New > Templates** or visit the template gallery
2. Search for keywords: "flowchart", "architecture", "pipeline", "neural network"
3. Select and customize a template as starting point

## Creating NeurIPS/ICML/ICLR-Style Figures

### Step 1: Set Up Canvas
- **Page Size**: Custom width for column-width figures
  - Single column: ~3.25 inches (8.25 cm)
  - Double column: ~6.875 inches (17.5 cm)
  - Full page width: ~7.1 inches (18 cm)
- **Background**: White
- **Grid**: Enable snap-to-grid for alignment

### Step 2: Define Style Guide
Create a consistent style across all figures:

**Font Settings**:
- Component labels: Arial or Helvetica, 8-10pt
- Sub-labels/descriptions: Arial, 7-8pt
- Figure title: Not in diagram (goes in LaTeX caption)
- Use the same font across ALL figures in the paper

**Color Palette** (save as custom theme):
- Primary: `#1e40af` (deep blue)
- Accent: `#f59e0b` (orange)
- Secondary: `#64748b` (gray)
- Success/positive: `#059669` (green)
- Background of components: `#eff6ff` (light blue fill) or `#f8fafc` (light gray)
- Text: `#1e293b` (near-black)

**Line Settings**:
- Stroke width: 1-2pt for boxes, 1.5pt for arrows
- Arrow style: Simple triangle head
- Solid lines for data flow, dashed for optional/auxiliary connections
- Rounded corners (2-4px radius) on rectangles for modern look

### Step 3: Build Components

**Standard shapes for ML diagrams**:
| Component | Shape | Example |
|---|---|---|
| Data/Input | Cylinder or parallelogram | Dataset, embeddings |
| Processing block | Rounded rectangle | Encoder, Decoder, MLP |
| Decision/gate | Diamond | Gating mechanism, condition |
| Output | Rounded rectangle (darker) | Prediction, loss |
| Grouping | Dashed rectangle | Transformer block, module |
| Skip connection | Curved arrow | Residual connection |

**Tips**:
- Use **Groups** (Ctrl+G) to group related shapes for easy repositioning
- Use **Align & Distribute** tools for precise layout
- Copy-paste styled shapes to ensure consistency

### Step 4: Add Connections
- Use **Connector** tool for arrows between components
- Right-click connectors to set: style (solid/dashed), color, arrowhead
- Add text labels on connectors for data descriptions
- Use elbow connectors for clean right-angle routing

### Step 5: Add Labels and Annotations
- Keep labels short: use abbreviations (MHA, FFN, BN, LN)
- Place dimension annotations (e.g., "B x T x D") along data flow arrows
- Use consistent label positioning (centered in boxes, above arrows)
- Add a color legend if using >3 colors

## Importing and Refining AI-Generated Figures

### From Gemini/PNG to EdrawMax
1. Generate figure with Gemini (PNG output)
2. **Insert > Image** in EdrawMax to place the raster image
3. Use it as a background reference layer
4. Trace over with native EdrawMax vector shapes
5. Delete the raster layer when done
6. Result: fully editable vector figure

### Quick Refinement (Keep Raster)
If the Gemini output is mostly correct:
1. Import PNG into EdrawMax
2. Use EdrawMax text tool to fix/replace any labels
3. Add missing arrows or annotations with EdrawMax tools
4. Crop or adjust layout
5. Export at high DPI

## Exporting for Papers

### Recommended Export Settings

**For LaTeX papers (best quality)**:
- **Format**: PDF or EPS (vector)
- **File > Export & Send > PDF**
- Ensures text remains sharp at any zoom level
- Include in LaTeX with `\includegraphics`

**For Word/submission systems**:
- **Format**: PNG at 300 DPI minimum (600 DPI preferred)
- **File > Export & Send > PNG**
- Set DPI in export dialog
- Use transparent background if needed

**For SVG (web/arxiv)**:
- **File > Export & Send > SVG**
- Editable in Inkscape or browsers
- Good for supplementary materials

### Export Checklist
- [ ] All text is legible at final print size
- [ ] Colors are distinguishable in grayscale (print preview in B&W)
- [ ] Consistent style with other figures in the paper
- [ ] No EdrawMax watermarks (requires licensed version)
- [ ] DPI >= 300 for raster exports
- [ ] Correct aspect ratio for paper column width

## Batch Consistency Across Figures

When creating multiple figures for one paper:

1. **Create a template file**: Set up one .eddx file with your color theme, fonts, and standard shapes
2. **Save custom shapes**: Right-click > Save as My Shape for reusable components
3. **Use pages**: Keep all figures in one EdrawMax file as separate pages for consistency
4. **Custom color palette**: Theme > Custom Colors > save your academic palette
5. **Style copy**: Use Format Painter (paintbrush icon) to copy styles between elements

## Keyboard Shortcuts (EdrawMax)

| Action | Shortcut |
|---|---|
| Group | Ctrl+G |
| Ungroup | Ctrl+Shift+G |
| Align left | (Arrange > Align > Left) |
| Distribute horizontally | (Arrange > Align > Distribute H) |
| Duplicate | Ctrl+D |
| Snap to grid toggle | File > Options > Snap |
| Zoom to fit | Ctrl+Shift+W |
| Export | Ctrl+Shift+E |
| Format Painter | (Home > Format Painter) |

## Integration with Gemini Workflow

### Recommended End-to-End Pipeline

```
1. Draft concept (hand sketch or text description)
        |
2. Generate with Gemini API (3-5 candidates, ~$0.15 total)
        |
3. Pick best candidate
        |
4. Import into EdrawMax
        |
5. Refine: fix labels, align components, match paper style
        |
6. Export as PDF/PNG at 300+ DPI
        |
7. Include in LaTeX: \includegraphics[width=\columnwidth]{figure.pdf}
```

### When to Use Each Tool

| Task | Use Gemini | Use EdrawMax |
|---|---|---|
| Initial concept visualization | Yes | - |
| Quick draft / exploration | Yes | - |
| Complex architecture with many components | Yes (generate) | Yes (refine) |
| Simple flowchart (< 10 nodes) | Optional | Yes (faster native) |
| Precise alignment and layout | - | Yes |
| Consistent multi-figure styling | - | Yes |
| Vector output for print | - | Yes |
| Photo-realistic or artistic figures | Yes | - |

## EdrawMax vs Other Tools

| Feature | EdrawMax | Inkscape | draw.io | PowerPoint |
|---|---|---|---|---|
| AI generation | Yes | No | No | No (Copilot limited) |
| Academic templates | Many | Few | Some | Few |
| Vector export (PDF/SVG) | Yes | Yes | Yes | Yes (limited) |
| Ease of use | High | Medium | High | High |
| Precision alignment | Excellent | Good | Good | Fair |
| Cost | Subscription | Free | Free | Subscription |
| Cross-platform | Win/Mac/Linux/Web | Win/Mac/Linux | Web/Desktop | Win/Mac/Web |

## References

- [EdrawMax Official](https://www.edrawmax.com/)
- [EdrawMax AI Features](https://www.edrawmax.com/ai-features/)
- [EdrawMax Templates](https://www.edrawmax.com/templates/)
- [EdrawMax Flowchart Guide](https://www.edrawmax.com/flowchart/)
- [EdrawMax Export Guide](https://www.edrawmax.com/article/export-edrawmax-files.html)