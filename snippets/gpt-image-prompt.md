# GPT Image Prompt Engineering — Print-on-Demand Artwork

## Production Prompt (Working, Locked)

```
From the concept list below, use ONLY Concept 1. Ignore Concept 2 and Concept 3 entirely.

Concept list: {previous_step_output}

Create ONE single wall art graphic design artwork based on Concept 1.

IMPORTANT REQUIREMENTS:
- ONE design only, centred in the image
- Design artwork ONLY — do NOT show a frame, room, mockup, or product photo
- Flat graphic design on a plain solid white background
- Bold and clear from a distance
- No watermarks
- Style: modern, clean, suitable for direct-to-surface printing
```

**Model:** gpt-image-1 | **Size:** 1024x1024 | **Quality:** hd | **Style:** natural

## Why Each Line Exists

### `use ONLY Concept 1. Ignore Concept 2 and Concept 3 entirely.`

The upstream AI step generates multiple product concepts per run. Without explicit concept isolation, the image model attempts to merge all concepts into a single design — producing incoherent mashups (e.g., a sunset landscape combined with geometric patterns and botanical illustrations).

**Before:** "Create a design based on these concepts: {multiple concepts}" → model mashes all together.

**After:** Explicit selection + explicit exclusion. Both parts are necessary — "use ONLY X" without "Ignore Y and Z" still produced blended outputs in testing.

### `Design artwork ONLY — do NOT show a frame, room, mockup, or product photo`

This was the most critical fix. The original prompt (inherited from a deprecated image model configuration) generated **photographs of framed prints** — realistic mockups showing artwork in a room setting. Sending this image to a print-on-demand service would produce a **print of a photo of a print** (product-on-a-product).

For print production, the image must be the **artwork only** — the isolated graphic that gets printed onto the surface. The fulfilment provider composites it onto the product.

### `Flat graphic design on a plain solid white background`

Print production requirements:
- White background = transparent when printed (the surface shows through)
- Flat design = no 3D effects, shadows, or photographic elements that break at print resolution
- "Plain solid" prevents gradient backgrounds or textured fills

### `Bold and clear from a distance`

Print legibility constraint. Wall art is viewed from several feet away. Fine detail, thin lines, and small text become illegible at typical viewing distance. This instruction biases the model toward high-contrast, clean designs.

### `No watermarks`

GPT Image occasionally adds decorative borders or watermark-like elements. Explicit exclusion prevents these from appearing in the print area.

## Output Format

The GPT Image step outputs a **Data URL** (base64-encoded image). This feeds directly into the Printify image upload endpoint's `contents` field — no file download or format conversion needed.

## Model Migration Notes

| Aspect | Previous Model (deprecated) | GPT Image (gpt-image-1) |
|---|---|---|
| API availability | Deprecated — removed from API | Current, available |
| Default behaviour | Generated product photos | Generates artwork (with proper prompting) |
| Concept handling | Less prone to mashing | Requires explicit concept isolation |
| Output quality | Lower resolution | Higher fidelity, better for print |
| Zapier integration | Via OpenAI action | Via OpenAI action (same step type) |

The migration was not a drop-in replacement. The prompt had to be rewritten from scratch due to the different default behaviours of the two models.

---

> *Implementation details adapted for portfolio purposes.*
