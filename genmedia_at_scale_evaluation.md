# Part 4: Deep Dive — Evaluation

> **[Back to Overview](genmedia_at_scale_main.md)** | **Previous: [Part 3 — Deep Dive: Input Processing](genmedia_at_scale_input_processing.md)**

---

## Why Evaluation is the Hardest Part

Input optimization and guided generation are engineering challenges with known techniques. Evaluation is fundamentally harder because it requires the system to **judge quality** — a task that traditionally requires human perception.

The evaluation system must answer questions like:
- Does this spinning video actually rotate smoothly in the correct direction?
- Does this try-on image show the right jacket on the right person?
- Is this product rendering faithful to the original, or has the model hallucinated details?

These questions span a spectrum from purely objective (rotation direction is measurable) to deeply subjective (does this "look right"?). The evaluation architecture must handle the full spectrum.

The stakes are high. A false positive — accepting a bad generation — means a defective asset reaches the customer-facing catalog. A false negative — rejecting a good generation — means wasted compute and potentially failing to produce media for a product. The evaluation system needs to be both sensitive enough to catch real problems and robust enough to avoid rejecting acceptable output.

---

## Evaluation Method Types

The architecture employs three categories of evaluation methods, each suited to different aspects of quality assessment.

### 1. Deterministic ML Models

**What:** Traditional machine learning and computer vision models that produce repeatable, objective measurements.

**When to use:** When the quality dimension can be quantified numerically — face identity preservation, rotation direction, structural similarity.

**Examples in production:**

| Method | Model | What It Measures | Output |
|--------|-------|------------------|--------|
| Face embedding similarity | Face embedding model | Whether a generated face matches the reference person | Cosine similarity (0–100%) |
| Rotation direction | Optical flow (Lucas-Kanade) | Whether a video rotates clockwise, anticlockwise, or erratically | Classification + confidence |
| Frame similarity | SSIM (Structural Similarity Index) | Pixel-level similarity between two frames or images | Similarity score (0–1) |

**Face embedding similarity** is a critical evaluation in virtual try-on. The pipeline extracts face embeddings from both the reference face and the generated image's face, then computes cosine similarity. This is rotation-invariant, lighting-invariant, and completely deterministic — the same pair of faces always produces the same score.

**Optical flow classification** uses sparse Lucas-Kanade tracking to follow feature points across video frames, computing average horizontal displacement per frame. The displacement pattern is analyzed through a rule-based classifier that detects clockwise rotation, anticlockwise rotation, direction changes, and motion spikes.

> **Key Advantage:** Deterministic models are fast, cheap (no API call), and reproducible. The same input always produces the same evaluation. They are the foundation of the evaluation stack.

### 2. LLM-as-Judge with Gemini

**What:** Large vision-language models that analyze generated media and produce structured quality assessments.

**When to use:** When the quality dimension requires semantic understanding — detecting visual glitches that break physical plausibility, assessing whether a garment is accurately reproduced, identifying hallucinated features.

**Examples in production:**

| Method | Model | What It Measures | Output |
|--------|-------|------------------|--------|
| Glitch detection | Gemini 3 Flash | Visual artifacts in spinning videos (direction reversals, teleportation, unnatural transformations) | `{is_valid: bool, explanation: str}` |
| Garment accuracy | Gemini 3 Flash | Whether each reference garment is faithfully reproduced in a try-on image | Score 0–3 per garment |
| Product consistency | Gemini 3 Flash | Whether a generated video matches the reference product across multiple viewpoints | `{is_valid: bool, explanation: str}` |

**Glitch detection** sends a video to Gemini at reduced frame rate (2fps) and asks it to identify specific categories of visual problems: sustained direction reversals, text/logo mirroring, features appearing or disappearing, sudden jumps or teleportation. The prompt is carefully calibrated to distinguish between actual glitches and acceptable artifacts (minor wobbles, slight lighting variations, natural reflections).

**Garment accuracy** evaluates each reference garment individually against the generated try-on image. The scoring scale is designed around a critical decision boundary: score 0 (garment completely missing) triggers automatic discard, while scores 1–3 represent varying degrees of reproduction quality that are all usable.

> **Key Principle: Descriptions serve evaluation, not generation.** This is where rich, detailed product descriptions become valuable. While generation prompts should be minimal to avoid conflicts with reference images, evaluation prompts benefit from exhaustive descriptions. "Check whether the distinctive crosshatch pattern is present on the heel counter" requires knowing about the crosshatch pattern — and that knowledge comes from a detailed product description, not from the generation prompt.

[DIAGRAM: A two-column layout. Left column titled "Generation Prompt" shows a minimal text: "A blue running shoe standing still in a white studio void" with a small icon. Right column titled "Evaluation Prompt" shows a much longer, detailed text (use a text block with visible but unreadable small text to convey density, with a few legible highlighted phrases like "crosshatch pattern on heel counter", "three ventilation holes on medial side", "reflective heel tab with embossed logo"). Below both columns, a generated shoe video frame is shown. An arrow from the left column points to the video frame labeled "Creates". An arrow from the right column points to the video frame labeled "Judges". The visual contrast in text density between the two columns should be dramatic.]

### 3. Ground Truth Comparison with Gemini

**What:** Direct comparison between generated output and reference inputs, using either pixel-level metrics or multi-view LLM evaluation.

**When to use:** When the question is "does the output match the input?" rather than "is the output good?"

**Examples in production:**

| Method | Approach | What It Measures |
|--------|----------|------------------|
| SSIM comparison | Structural similarity between reference image and closest generated frame | How closely a specific view in the generated video matches the reference photo |
| Multi-view LLM comparison | Gemini evaluates multiple generated frames against multiple reference images simultaneously | Whether the product maintains consistent structure, color, and features across all views |
| Embedding similarity | Cosine distance between face embeddings | Whether the person's identity is preserved |

**Multi-view comparison** is particularly powerful for product consistency validation. Rather than checking individual frames in isolation, it sends all reference images and a set of matched generated frames to Gemini in a single call, asking it to evaluate consistency across the entire set. This catches problems that single-frame evaluation misses — like a product that looks correct from the front but wrong from the side.

The frame selection for multi-view comparison uses a label-matching + SSIM approach: generated frames are classified by viewpoint, matched to reference images by label, and then the frame with highest SSIM to each reference is selected along with neighboring frames for context.

---

## Evaluation Strategies

The three method types above are combined into **strategies** — patterns for how evaluation results drive pipeline behavior.

### Strategy 1: Pass/Fail with Retry

**Pattern:** Generate → Evaluate → If fail, regenerate (up to N attempts)

**Best for:** Use cases where a single correct output is needed and quality is binary (acceptable or not).

**Example: Product spinning.** A spinning video either rotates correctly or it doesn't. If the optical flow classifier detects anticlockwise rotation, or the glitch detector finds a direction reversal, the video is rejected and regenerated.

```
Attempt 1: Generate → Rotation check FAIL (anticlockwise) → Retry
Attempt 2: Generate → Rotation check PASS → Glitch check FAIL (direction reversal at frame 120) → Retry
Attempt 3: Generate → Rotation check PASS → Glitch check PASS → Accept
```

The checks are **sequenced by cost:** the optical flow check (free, local computation) runs before the glitch detection check (paid, Gemini API call). This saves money on obviously bad generations.

### Strategy 2: Score-and-Rank with Parallel Variations

**Pattern:** Generate N variations in parallel → Evaluate all → Select the best

**Best for:** Use cases where quality is a spectrum rather than binary, and the best result among multiple attempts is desired.

**Example: Virtual try-on.** The pipeline generates N try-on variations simultaneously. Each variation receives a composite score (face similarity + garment accuracy). Variations below a hard quality floor (any garment scoring 0) are discarded. Among the remaining variations, the highest-scoring one is presented first, but all passing variations are made available.

```
Variation 1: face=72%, garments=89% → final=80.5% → Ready (rank 1)
Variation 2: face=65%, garments=0%  → Discarded (missing garment)
Variation 3: face=58%, garments=95% → final=76.5% → Ready (rank 2)
```

This strategy trades compute cost for quality — generating 3 variations costs 3x but dramatically increases the probability of producing at least one excellent result.

### Strategy 3: Multi-Step Self-Correction

**Pattern:** Generate → Evaluate → Generate targeted correction → Re-evaluate → Pick best step

**Best for:** Use cases where specific, identifiable aspects of the generation can be corrected without starting from scratch.

**Example: Virtual try-on face correction.** The initial generation often changes the model's face. Rather than discarding and regenerating entirely, the pipeline:

1. Generates the try-on image (Step 1)
2. Evaluates face similarity
3. Generates a face correction pass using the Step 1 result + reference face (Step 2)
4. Evaluates face similarity on Step 2
5. Selects whichever step has higher face similarity

This is more efficient than pure retry because it builds on a partially-correct result. The garments are usually correct in Step 1 — only the face needs fixing.

[DIAGRAM: Three horizontal lanes showing the three evaluation strategies. Lane 1 "Pass/Fail with Retry": a linear flow of Generate → Evaluate → diamond decision (Pass/Fail) → Fail loops back to Generate (labeled "retry 1, 2, ... N"), Pass goes to Accept. Lane 2 "Score-and-Rank": three parallel Generate→Evaluate flows producing scores (80.5, discarded, 76.5), converging into a "Rank & Select" step that outputs the best. Lane 3 "Multi-Step Self-Correction": Generate → Evaluate Face → "Correct Face" → Re-Evaluate Face → diamond "Compare Scores" → Pick Better. Each lane should be color-coded differently and labeled with its primary use case (Spinning, Try-On Selection, Try-On Face).]

---

## Composite Scoring

When multiple evaluation dimensions are assessed, they must be combined into a single actionable score. The architecture uses **weighted composite scoring** with **hard discard gates**.

### Weighted Combination

For virtual try-on, the final score combines face preservation and garment accuracy:

```
final_score = face_similarity * 0.5 + garments_score * 0.5
```

The weights reflect business priorities. Face preservation and garment accuracy are equally important — a perfect garment on the wrong face is no better than the right face wearing the wrong garment.

### Hard Discard Gates

Composite scores can mask critical failures. A high face similarity (90%) combined with zero garment accuracy (garment completely missing) would produce a composite score of 45% — which might appear marginally acceptable. But an image of the right person wearing the wrong clothes is completely unusable.

**Discard gates** are hard floors that override the composite score:

| Gate | Condition | Action |
|------|-----------|--------|
| Missing garment | Any garment scores 0 | Discard regardless of other scores |
| No face detected | Face detection returns null | Discard — can't evaluate identity |

These gates ensure that critically flawed outputs are never accepted, regardless of how well they score on other dimensions.

---

## Retry Budgets and Cost/Quality Tradeoff

Every retry costs money — API credits, compute time, and latency. The architecture makes the cost/quality tradeoff explicit through **retry budgets**.

### The Economics

| Component | Approximate Cost Per Call |
|-----------|--------------------------|
| Video generation (Veo 3.1) | High |
| Image generation (Gemini) | Medium |
| LLM evaluation (Gemini Flash) | Low |
| Deterministic evaluation (embeddings, optical flow) | Negligible |

This cost hierarchy influences evaluation sequencing. Cheap evaluations run first to catch obvious failures before expensive regeneration. A spinning video that fails the free optical flow check is rejected without spending on a Gemini glitch detection call.

### Budget Strategies

| Use Case | Budget | Exhaustion Behavior | Rationale |
|----------|--------|---------------------|-----------|
| Generic spinning | 4 attempts | Best-effort (use last result) | Most videos are acceptable; rare failures are tolerable |
| Footwear spinning | 5 attempts | Exclude product | Higher quality bar; better to skip than ship bad |
| Virtual try-on | N parallel + correction | Best of N (no retry) | Parallel generation replaces sequential retry |

> **Key Insight:** Parallel generation (Strategy 2) fundamentally changes the retry economics. Instead of sequential retry (generate, evaluate, retry if bad), parallel generation (generate N simultaneously, pick best) achieves similar quality improvement in 1x wall-clock time instead of Nx. The tradeoff is higher concurrent cost but dramatically lower latency.

---

## Evaluation Calibration

Evaluation models — especially LLM-as-judge — require careful calibration to distinguish between real problems and acceptable artifacts.

### What to Catch vs. What to Accept

For glitch detection in spinning videos:

| Catch (INVALID) | Accept (VALID) |
|-----------------|----------------|
| Sustained rotation direction reversal | Minor wobbles or hesitations |
| Text/logo mirroring | Slight lighting variations |
| Features appearing/disappearing | Natural reflections and shadows |
| Sudden jumps or teleportation | Minor surface imperfections |

For garment accuracy in virtual try-on:

| Strict (Must Match) | Lenient (Can Vary) |
|---------------------|-------------------|
| Feature placement and structure | Text legibility and spelling |
| Color consistency | Logo fine detail |
| Structural integrity | Minor angle variations |
| Hallucinated features | Generative artifacts |

The distinction between strict and lenient criteria is critical. Being too strict produces excessive false rejections (wasted compute). Being too lenient lets defective assets through. The calibration is informed by production experience — analyzing which kinds of imperfections customers actually notice and complain about.

> **Key Principle:** Text and logos in generated images should be treated as "visual blobs." Correct position and color is sufficient; exact legibility is not required. This reflects the reality that generative models struggle with text rendering, and holding them to a legibility standard would reject otherwise excellent outputs.

---

> **Next: [Part 5 — Case Study: Product 360 Spinning](genmedia_at_scale_spinning.md)**
