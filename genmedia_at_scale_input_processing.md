# Part 3: Deep Dive — Input Processing

> **[Back to Overview](genmedia_at_scale_main.md)** | **Previous: [Part 2 — The Architecture Philosophy](genmedia_at_scale_architecture.md)**

---

## Why Input Processing Matters

Generative models amplify whatever they receive. Clean, well-prepared inputs produce clean outputs. Noisy, cluttered inputs produce noisy outputs — and more retries, higher costs, and lower yield.

Input processing is not about making inputs "look nice." It's about **reducing the decision space** for the generation model. Every pixel of background clutter, every compression artifact, every irrelevant object in the frame is an opportunity for the model to get distracted, hallucinate, or produce something unexpected.

The techniques in this section form a toolkit. Not every pipeline uses every technique. The art is in choosing the right composition for each use case.

---

## Extraction and Segmentation

### The Problem

Product images from retailer catalogs rarely arrive as clean, isolated subjects. They come with studio backgrounds (varying shades of grey, gradients), lifestyle contexts (products on shelves, in rooms), watermarks, color swatches, and other visual noise. Generation models trained on diverse data will attempt to reproduce or react to all of this.

### The Approach

Extraction isolates the product from its surroundings using image segmentation, then places it on a controlled background.

**Primary path:** A cloud-based segmentation model (Vertex AI Image Segmentation) generates a pixel-level mask of the product. The mask is applied to the original image, and the product is composited onto a uniform background (typically white for product media, light grey for try-on).

**Key implementation details:**

- **EXIF orientation correction** is applied before any processing — mobile-captured images often have rotation metadata that, if ignored, produces a sideways segmentation
- **Large image handling:** If the input exceeds the segmentation model's resolution limit (2048px), it's resized for segmentation, then the resulting mask is scaled back to the original dimensions. This preserves full-resolution detail while staying within API constraints.
- **Mask dilation:** The raw segmentation mask is dilated by a small margin (typically 2–3px) to avoid clipping product edges. Tight masks cut into fine details like zippers, stitching, and thin straps.
- **Contour-based cropping:** After masking, the largest contour in the mask is found and the image is cropped to its bounding box with a small tolerance margin. This eliminates wasted canvas space and centers the product.

### Fallback Strategy

Cloud segmentation APIs can fail — rate limits, transient errors, unsupported image formats. The architecture includes a local fallback:

| Step | Primary | Fallback |
|------|---------|----------|
| Segmentation | Vertex AI Image Segmentation | rembg (local ML model) |

The fallback produces slightly lower-quality masks but keeps the pipeline moving. Quality differences in segmentation are caught downstream by the evaluation stage.

> **Key Principle:** Never abort a pipeline due to a preprocessing failure. Degrade gracefully and let evaluation catch quality issues downstream.

---

## Enhancement and Upscaling

### The Problem

Generation models produce better results with higher-resolution reference images. A 200x300px product photo from a mobile phone gives the model far less visual information than a 2000x3000px studio shot. But retailer catalogs contain both.

Additionally, the upscaling process itself can introduce artifacts — especially around product edges where the segmentation mask meets the background. These artifacts then get amplified by the generation model.

### The Approach

Enhancement is a two-phase process: upscale, then clean.

**Phase 1: Upscaling.** A dedicated upscaling model (Imagen 4.0 Upscale) increases resolution by up to 4x. The upscaling is adaptive:

| Original Size | Upscale Factor | Rationale |
|---------------|---------------|-----------|
| Small (< ~1000px) | 4x | Maximize detail recovery |
| Medium (~1000–2000px) | 3x | Stay within output pixel limits |
| Large (> 2000px) | 2x | Avoid exceeding 17M pixel output cap |

The factor is automatically reduced if the target resolution would exceed the model's output limit. This prevents API errors without requiring manual configuration per image.

**Phase 2: Post-upscale artifact cleanup.** Upscaling models, despite their sophistication, can introduce artifacts around the edges of segmented objects — slight halos, color bleeding, softened boundaries. The cleanup step:

1. Takes the original segmentation mask (from the extraction step)
2. Scales it to match the upscaled image dimensions
3. Reapplies the mask to the upscaled image, removing any artifacts the upscaler introduced around edges
4. Re-crops to the mask bounding box

This two-phase approach — upscale broadly, then clean precisely — produces higher-quality results than either step alone.

[DIAGRAM: A horizontal flow showing three stages of a shoe image. Stage 1 "Original": a small, slightly blurry shoe on a grey gradient background (labeled "400x600px"). Stage 2 "Extracted + Upscaled": a larger, sharp shoe on white background with a red circle highlighting a subtle halo artifact around the shoe's edge (labeled "1600x2400px, note edge artifact"). Stage 3 "Cleaned": the same large shoe but with the red circle area now showing a clean edge against white (labeled "1600x2400px, clean edges"). Arrows connect the stages, labeled "Extract + Upscale 4x" and "Reapply Mask".]

### Parallelism

When processing multiple input images (common in spinning pipelines that accept 1–4 reference images), extraction and upscaling run in parallel across all images using a thread pool. This reduces wall-clock time from `N * per_image_time` to approximately `per_image_time` for typical batch sizes.

---

## Classification as Routing

### The Problem

Not all inputs are equal, and not all inputs should be processed the same way. A product image showing a shoe from the right side serves a different purpose than one showing the sole. An image containing two shoes needs to be split before it can be used. An image of a person wearing the shoes (not the product itself) should be discarded entirely.

### The Approach

Classification uses a vision model (Gemini) to categorize each input image, then routes it through the appropriate processing path.

**Classification is a routing decision, not a label.** The output of classification determines what happens next:

| Classification | Action |
|----------------|--------|
| Valid viewpoint (e.g., `right`, `front`, `back`) | Proceed to extraction and upscaling |
| Multiple objects (e.g., `multiple`) | Route to splitting step, then re-classify individual results |
| Invalid (e.g., person wearing product, non-neutral background) | Discard from pipeline |
| Special property (e.g., velcro closure on footwear) | Abort pipeline (known bad outcome) |

**Classification happens early and cheap.** It uses a lightweight vision model call (temperature 0, minimal output tokens, no chain-of-thought) to make fast routing decisions before expensive processing steps like upscaling. This is a deliberate cost optimization: filtering out unusable images before spending API credits on upscaling them.

### Feasibility Checks

After classification, the pipeline can perform **feasibility checks** — verifying that the available inputs are sufficient for the target output. For example, a footwear spinning pipeline requires images covering four cardinal viewpoints (front, back, left, right). If the classified inputs don't cover all four, the pipeline can abort early rather than producing a spinning video that's missing views.

This is a key difference from a naive approach that would attempt generation with whatever inputs are available and hope for the best. Classification enables **fail-fast** behavior that saves compute and produces more predictable results.

---

## Multi-Input Curation and Selection

### The Problem

Many use cases accept multiple reference images. A product might have 2, 4, or even 10 images in its catalog listing. Using all of them isn't always better — duplicate viewpoints add noise, low-quality images drag down the result, and most generation models have limits on how many reference images they can effectively use.

### The Approach

Selection identifies the best subset of inputs for the generation task.

**Deduplication by viewpoint.** When multiple images share the same classification (e.g., two `right` side views), the selection logic picks the best one. "Best" is defined by a proxy metric — typically the count of non-background pixels, which correlates with image completeness and detail.

**Ordered selection.** For use cases where viewpoint coverage matters, images are selected in a specific order that matches the target output. For a clockwise spinning video, images are selected in clockwise order: right → front → left → back. This gives the generation model a natural progression to follow.

**Capacity limits.** Generation models have practical limits on reference image count. Beyond 3–4 reference images, additional inputs show diminishing returns or can even degrade quality. The selection step enforces these limits, choosing the most valuable inputs within the budget.

### Stacking and Canvas Creation

After selection, images are prepared for the generation model's input format. This often means creating standardized reference canvases:

- Each selected image is placed on a uniform background at a target resolution (e.g., 3840x2160 for 4K video generation)
- All images are scaled to the same height for visual consistency
- Margins ensure the product doesn't touch canvas edges
- When more images are selected than the model's reference limit, images are composited side-by-side into a single canvas

This standardization removes another source of variance from the generation input.

[DIAGRAM: A visual showing the selection and canvas creation process. Top row: 6 shoe images in various sizes and angles, with classifications labeled below each ("right", "front_right", "front", "left", "back", "right (duplicate)"). Middle row: arrows showing selection — the duplicate "right" is crossed out (red X), and the remaining 5 are filtered to the best 4 (one gets an orange "exceeds budget" label). Bottom row: 3 large white 4K canvases, each containing one or two shoes centered with generous margins. The first two canvases have single shoes ("right" and "left"), the third canvas has two shoes side by side ("front" and "back"). All canvases are the same size and the shoes are at matching heights.]

---

## The Composition Pattern

Input processing is never a fixed pipeline. It's a **composition of techniques** selected based on the use case:

| Use Case | Extract | Enhance | Classify | Select | Special |
|----------|---------|---------|----------|--------|---------|
| Product Spinning (Generic) | Background removal + white canvas | 4x upscale + mask cleanup | No | No (use all inputs) | — |
| Product Spinning (Footwear) | Background removal + white canvas | 4x upscale + mask cleanup | 12-class viewpoint | Best 4 views in clockwise order | Pair splitting, velcro detection |
| Virtual Try-On | Background removal + grey canvas | 4x upscale | No | No | Face crop + separate processing |

The specific composition for spinning pipelines is detailed in **[Part 5](genmedia_at_scale_spinning.md)** and **[Part 6](genmedia_at_scale_spinning_shoes.md)**.

> **Next: [Part 4 — Deep Dive: Evaluation](genmedia_at_scale_evaluation.md)**
