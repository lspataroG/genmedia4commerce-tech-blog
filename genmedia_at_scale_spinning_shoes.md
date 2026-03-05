# Part 6: Case Study — Footwear Spinning

> **[Back to Overview](genmedia_at_scale_main.md)** | **Previous: [Part 5 — Case Study: Product 360 Spinning](genmedia_at_scale_spinning.md)**

---

## Why Shoes Are Harder

Footwear is the most challenging product category for AI-generated spinning videos. Where a generic product (a mug, a backpack) has few well-defined viewpoints, a shoe has a highly structured geometry with **distinct, named views** — right side, left side, front, back, sole, top — that customers know and expect to see.

This creates problems that don't exist for generic products:

**1. Pair ambiguity.** Product photos frequently show two shoes together. The generation model needs a single shoe to spin, which means the pipeline must detect pairs and split them into individual shoe images — a segmentation and classification problem that generic products don't face.

**2. Viewpoint coverage requirements.** A spinning video must show the shoe from all angles. If the input images only cover the right side and front, the model must hallucinate what the back and left side look like — and it will often get it wrong (mirroring logos, inventing stitching patterns). The pipeline needs to verify that input images cover all cardinal viewpoints before attempting generation.

**3. Rotation verifiability.** Because shoes have named viewpoints, the pipeline can do something impossible with generic products: verify that the generated video actually passes through the correct sequence of views. A clockwise rotation of a right shoe should show: right → front → left → back → right. This enables **semantic spin validation** — a much stronger quality check than optical flow alone.

**4. Category-specific failure modes.** Some shoe types produce consistently poor spinning results. Shoes with velcro/hook-and-loop closures generate artifacts because the straps move unpredictably during the AI-generated rotation. The pipeline must detect and filter these before wasting generation budget.

The result is a pipeline with **14 steps** compared to the generic pipeline's 5 — but one that produces fully automated, human-review-free output for the most demanding product category.

---

## Pipeline Overview

[DIAGRAM: A vertical flow diagram showing the footwear spinning pipeline, significantly more complex than the generic version. The flow proceeds top-to-bottom with branching paths. Starting with "Input Images (1+)", the steps are: "Classify Viewpoints (Gemini)" → diamond "Any 'multiple'?" with Yes branch going to "Split Pairs (Vertex AI Segmentation)" then back to classification, and No branch continuing → "Re-classify + Feasibility Check" → diamond "4 cardinal views covered?" with No path going to "Exclude Product" (red, exit), Yes continuing → "Velcro Detection (Gemini)" → diamond "Has velcro?" with Yes going to "Exclude Product" (red, exit), No continuing → "Extract & Upscale (parallel)" → "Select Best 4 Views" → "Canvas Creation (4K)" → "Generate Description (Gemini)" → "Generate Video (Veo 3.1)" → "Spin Path Validation (frame-by-frame classification)" → diamond "Valid path?" with Fail looping back to Generate Video → "Product Consistency (Gemini multi-view)" → diamond "Consistent?" with Fail looping back to Generate Video → "Frame Post-Processing (reorder, crop, resize)" → "Output: Video + 50 Frames". Retry counters show "max 5 attempts" on both validation loops. Annotate technologies on the right side of each step.]

---

## Step 1: Shoe Classification

Every input image is classified into one of **12 categories** describing the shoe's viewpoint and orientation:

| Category | Description | Pipeline Action |
|----------|-------------|-----------------|
| `right` | Toe on the right, heel on the left | Use for generation |
| `left` | Toe on the left, heel on the right | Use for generation |
| `front` | Toe facing camera directly | Use for generation |
| `front_right` | Front and right side visible | Use for generation (counts as front + right) |
| `front_left` | Front and left side visible | Use for generation (counts as front + left) |
| `back` | Heel facing camera directly | Use for generation |
| `back_right` | Back and right side visible | Use for generation (counts as back + right) |
| `back_left` | Back and left side visible | Use for generation (counts as back + left) |
| `top_front` | Camera above, looking down at front | Use for generation (does NOT count as front for feasibility) |
| `sole` | Upside down, sole dominant | Discard |
| `multiple` | Two or more shoes in the image | Route to pair splitting |
| `invalid` | No footwear, person wearing shoes, non-neutral background | Discard |

Classification uses Gemini with a detailed system prompt, temperature 0, and no chain-of-thought. All images are classified in parallel.

> **Key Design Decision:** `top_front` does NOT count as `front` for the feasibility check. A top-down view doesn't provide the same reference information as a straight-on front view, and treating it as equivalent leads to worse generation quality.

---

## Step 2: Pair Splitting

Images classified as `multiple` (containing two shoes) are split into individual shoe images using automated segmentation.

**The process:**

1. **Segment both shoes** using Vertex AI Image Segmentation with the prompt "Segment both footwears in the image"
2. **Filter masks** to the top 30% by confidence score (minimum 3 masks retained)
3. **Sort by horizontal position** — a scoring function sums `(pixel_x - midpoint)` for all foreground pixels in each mask. The rightmost mask becomes the "right shoe," the leftmost becomes the "left shoe"
4. **Subtract overlapping regions** — each mask is subtracted from the other to produce clean, non-overlapping masks
5. **Validate** — each final mask must have at least 10 foreground pixels and at most 1 external contour
6. **Extract** — each mask is applied to the original image, cropped to its bounding box, and placed on a white background

The resulting individual shoe images are sent back through classification (Step 1) to determine their viewpoints.

---

## Step 3: Feasibility Check

After classification (and potential splitting + re-classification), the pipeline checks whether video generation is even worth attempting.

**The rule is simple:** the input set must cover all **four cardinal views** — front, back, left, and right. Compound classifications count toward their components (e.g., `front_right` covers both `front` and `right`).

If four-view coverage is not met, the product is **excluded** — the pipeline stops before any expensive processing (upscaling, video generation) occurs.

This is a key application of the **fail-fast** principle from the architecture. Rather than generating a video and hoping the model fills in missing views correctly, the pipeline refuses to attempt generation with insufficient input. The cost of exclusion (no output for this product) is lower than the cost of producing a low-quality video that misrepresents the product.

---

## Step 4: Velcro Detection

Before committing to expensive upscaling and generation, the pipeline checks for a known failure mode: **velcro/hook-and-loop closures**.

Shoes with velcro produce consistently poor spinning results because the straps move unpredictably during AI-generated rotation, creating visual artifacts. Rather than wasting generation budget and evaluation cycles, the pipeline detects velcro early and aborts.

**Implementation:**
- Up to 4 images are selected from relevant viewpoints (front-facing and side views where velcro is visible)
- All selected images are sent to Gemini in a single call
- The model checks for velcro straps, hook-and-loop closures, and fuzzy fastener strips
- Side-release buckles and pin-buckle fastenings are explicitly excluded (these work fine in spinning videos)
- Response: `{has_velcro: bool, explanation: str}`

If velcro is detected, the pipeline aborts. This is another application of classification-as-routing: a cheap classification call prevents an expensive failed generation.

---

## Step 5: Image Selection and Ordering

From all classified images, the pipeline selects the **best 4 images** in a specific order that matches the target clockwise rotation.

### Deduplication

When multiple images share the same classification (e.g., two `right` side views from the catalog), the one with the **most non-white pixels** is selected. This proxy metric correlates with image completeness — a shoe filling more of the frame provides more visual detail for the generation model.

### Ordered Selection

Images are selected to match the clockwise rotation path: `right → front_right → front → front_left → left → back_left → back → back_right`.

The top 4 views are chosen with fallback priorities:

| Slot | Primary | Fallback 1 | Fallback 2 |
|------|---------|------------|------------|
| 1st (right side) | `right` | `front_right` | `back_right` |
| 2nd (left side) | `left` | `front_left` | `back_left` |
| 3rd (front) | `front` | `front_left` | `front_right` |
| 4th (back) | `back` | `back_left` | `back_right` |

No image is used twice across slots. This ensures maximum viewpoint diversity within the 4-image budget.

---

## Semantic Spin Validation

This is the **core differentiator** of the footwear pipeline. Instead of relying only on optical flow (which measures motion but not content), the shoes pipeline validates the spin by **classifying every frame** and checking that the sequence follows a valid rotation path.

### The Rotation Graph

The valid clockwise rotation is defined as a directed graph:

```
right → front_right → front → front_left → left → back_left → back → back_right → right
```

[DIAGRAM: A circular graph with 8 nodes arranged in an octagon, connected by clockwise arrows. The nodes are labeled (starting from the right, going clockwise): "right", "front_right", "front", "front_left", "left", "back_left", "back", "back_right". Each node has a small shoe silhouette showing the corresponding viewpoint. The arrows go clockwise: right → front_right → front → front_left → left → back_left → back → back_right → right. Self-loops on each node (a node can stay on itself for consecutive frames). The center of the octagon says "Valid Clockwise Path". Below the graph, show a sample frame sequence with shoe images and their classifications, with green checkmarks for valid transitions and a red X for an invalid one.]

### How It Works

1. **Frame extraction:** All frames are extracted from the video (24fps → ~192 frames for 8 seconds)
2. **Frame sampling:** Sampled at 12fps → ~96 frames
3. **Frame classification:** Each sampled frame is classified using the shoe classifier in "validation" mode — a simplified prompt without `top_front`, `sole`, and `multiple` categories (which shouldn't appear in a proper spinning video)
4. **All-classes check:** Verifies that all 8 positions appear in the classified frames. If any position is missing, the video is invalid.
5. **Path validation:** The sequence of classifications is checked against the rotation graph

### Tolerance for Noise

The classifier is applied per-frame with a vision model, which means individual frames may be misclassified. The path validator tolerates:

| Tolerance | Limit | Rationale |
|-----------|-------|-----------|
| Random misclassifications | Unlimited (if not consecutive) | Individual frame noise is expected |
| Oscillations between adjacent classes | Max 3 consecutive | Transition boundaries are ambiguous |
| Skip-ahead (jumping intermediate nodes) | Unlimited | Classifier may skip a view that only lasts 1–2 frames |
| Consecutive violations | Max 5 | Extended violations indicate real problems |

### Direction Detection

If the clockwise path check fails, the **reversed sequence** is checked. If the video is valid anticlockwise, the frames are reversed to produce clockwise output — the video is saved, not discarded.

### Completeness Check

After path validation, the pipeline checks whether the video represents a **full 360-degree rotation**:

- If the starting viewpoint class reappears later (50+ frames apart), the video went past 360° and is **trimmed** at the frame most similar (by SSIM) to frame 0
- If no second occurrence is found, the pipeline checks that the first and last frames share sufficient overlap in the same class (25+ combined frames at the boundary)

---

## Product Consistency Validation

After spin validation passes, a second evaluation checks whether the **generated product matches the reference images** — detecting cases where the model hallucinated details, changed colors, or altered the shoe's structure.

### The Multi-View Approach

Rather than checking individual frames against individual references, the pipeline evaluates consistency across all views simultaneously:

1. **Label matching:** Generated frames are mapped to reference images by their classified viewpoint
2. **SSIM frame selection:** For each reference, the generated frame with the highest structural similarity is selected, along with neighboring frames (±4 positions) for context
3. **Single Gemini evaluation:** All references and selected frames are sent to Gemini in a single API call

### Evaluation Criteria

The consistency evaluator has calibrated strict and lenient criteria:

| Strict (Must Match) | Lenient (Can Vary) |
|---------------------|-------------------|
| Feature placement (buckles, straps, eyelets) | Text legibility and spelling |
| Structural integrity (sole shape, heel height) | Logo fine detail |
| Color consistency across views | Minor angle differences |
| Hallucinated features (extra eyelets, missing straps) | Generative artifacts (slight texture variation) |

> **Key Principle:** Text and logos are treated as "visual blobs." Correct position and approximate color is sufficient — exact text rendering is not required. This prevents rejecting otherwise excellent generations due to the well-known weakness of generative models with text.

If product consistency fails, the entire video generation is retried (back to video generation, within the retry budget of 5 attempts).

---

## Frame Post-Processing

Once a video passes both spin validation and product consistency checks, the frames are post-processed for final output.

### Reordering

The validated frames are reordered so the sequence starts from the `right` viewpoint. If reference images include a `right` view, the frame most similar (by SSIM) to that reference is chosen as the starting frame. This ensures consistent framing across different products.

### Cropping and Resizing

To produce clean output frames:

1. **Product bounds detection:** 30 evenly-sampled frames are analyzed using Canny edge detection to find the bounding box that encompasses the product across all frames (with 5% margin)
2. **Background color detection:** The median color of border pixels is detected (handles cases where the background isn't perfectly white)
3. **Uniform crop and resize:** All frames are cropped to the product region, scaled to fit within 1000x1000px, and padded with the detected background color

**Final output:** The pipeline produces both the full MP4 video and 50 uniformly-sampled JPEG frames at 1000x1000px — suitable for interactive 360 viewers on product pages.

---

## Key Differences from Generic Spinning

| Feature | Generic Pipeline | Footwear Pipeline |
|---------|-----------------|-------------------|
| **Classification** | None | 12-class viewpoint classifier |
| **Pair handling** | Not supported | Automatic detection and splitting |
| **Feasibility check** | None (always attempt) | 4-view coverage required |
| **Failure mode filtering** | None | Velcro detection and abort |
| **Image selection** | All inputs used | Best 4 views in clockwise order |
| **Spin validation** | Optical flow (motion-based) | Frame-by-frame classification (semantic) |
| **Glitch detection** | Gemini vision | Replaced by semantic validation |
| **Product consistency** | Not included | Multi-view Gemini evaluation |
| **Frame post-processing** | Not included | Reorder + crop + resize to 1000x1000 |
| **Retry budget** | 4 attempts, best-effort | 5 attempts, strict |
| **Output format** | Video only | Video + 50 sampled frames |

---

## Fully Automated, No Human Review

The footwear pipeline is the most complex instantiation of the GenMedia architecture, but it achieves the same goal as every other pipeline: **fully automated, production-grade output with no human review.**

The layered approach makes this possible:

1. **Classification** filters out unusable inputs before expensive processing
2. **Feasibility checks** prevent doomed generation attempts
3. **Failure mode detection** catches category-specific problems early
4. **Semantic spin validation** verifies the output tells the right visual story
5. **Product consistency validation** catches hallucinations and drift
6. **Retry budgets** bound costs while maximizing quality

Each layer catches a different class of problem. Together, they produce a pipeline that can process thousands of footwear SKUs autonomously, with quality levels that match or exceed manual curation.

---

> **[Back to Overview](genmedia_at_scale_main.md)**
