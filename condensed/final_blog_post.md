# Beyond the Prompt: A Framework for Scaling Generative Media in Commerce

Generative AI models can produce stunning product media — 360-degree spins, virtual try-on images, lifestyle scenes. But the gap between a single impressive demo and reliable production across thousands of SKUs is vast. This post describes a framework for bridging that gap.

---

## The Problem

E-commerce is fundamentally a visual-first medium. Shoppers who engage with immersive product media — such as 360-degree spins, virtual try-on experiences, and contextual lifestyle imagery — convert at dramatically higher rates, often seeing a [144% increase in add-to-cart events](https://www.shopify.com/research/2025-product-media). Moreover, high-quality visualization bridges the gap between expectation and reality, directly addressing the [42% of returns caused by products looking "different than expected"](https://www.shopify.com/research/2025-product-media).

Traditional production — studio shoots, motorized turntables, model bookings, post-production stitching — costs $100–500+ per SKU. For a retailer with 100,000 items, catalog-wide coverage is a financial and logistical impossibility.

Generative AI models like [Veo](https://cloud.google.com/vertex-ai/generative-ai/docs/models/veo), [Gemini](https://cloud.google.com/vertex-ai/generative-ai/docs/models/gemini), and [Imagen](https://cloud.google.com/vertex-ai/generative-ai/docs/models/imagen) can synthesize photorealistic media from a handful of static product photographs. But raw model outputs are unpredictable: a spinning video of a backpack might reverse direction mid-spin or grow an extra pocket; a generated shoe might hallucinate extra eyelets; a product's brand-defining color might drift in a way that damages trust. Manual review of every generated asset across thousands of SKUs introduces a bottleneck that defeats the entire purpose of automation.

---

## The Core Insight

**The generation call is the easy part.** The hard part is everything around it: preparing inputs so models succeed on the first try, and automatically evaluating outputs so no human needs to review them.

This leads to a three-pillar framework — **Input Optimization, Guided Generation, and Automated Evaluation** — that applies across all generative media use cases. By surrounding the core generation step with rigorous pre-processing and automated quality gates, the framework transforms stochastic model calls into a predictable production engine.

---

## Pillar 1: Input Optimization

Generative models amplify whatever they receive. Noisy inputs produce noisy outputs. The first pillar is a toolkit of techniques — composed differently per use case — that reduce the decision space for the generation model:

- **Extraction** isolates the product from cluttered backgrounds using segmentation
- **Enhancement** upscales low-resolution catalog images to give the model more visual detail
- **Classification** categorizes inputs (viewpoint, category, usability) and routes them — proceed, discard, or abort early
- **Selection** picks the best subset of images when multiple are available, maximizing quality and viewpoint coverage

For example, in a footwear spinning pipeline, a fine-tuned viewpoint classifier automatically categorizes every input image:

```json
{
  "image_id": "img_88291a",
  "classification": "front_right",
  "confidence": 0.98,
  "action": "route_to_generation_queue"
}
```

If the input set lacks the necessary cardinal views (front, back, left, right), the pipeline aborts early — saving expensive generation compute through fail-fast feasibility checks.

The key principle: every pixel of background clutter, every compression artifact, every irrelevant object is an opportunity for the model to hallucinate. Clean inputs mean fewer retries, lower costs, and higher yield.

---

## Pillar 2: Guided Generation

This is where many teams over-invest in prompt engineering. In this framework, generation is deliberately kept **simple and constrained**.

**Reference images lead.** The model should reproduce what it sees in the images, not what it interprets from text. A short description ("a blue running shoe") anchors the model. A detailed description ("blue mesh upper with reflective heel tab and crosshatch pattern") risks conflicting with the visual reference — and when text conflicts with images, models follow the text, introducing errors.

A prompt for a 360-degree spin describes the *scene and camera motion* — never the product itself:

```text
[Subject]: A red ceramic mug standing still in a completely white studio void.

[Action]: The camera performs one continuous, seamless 360-degree orbit around the
stationary product. The camera movement is perfectly smooth and steady, maintaining
a constant distance. The product does not move or rotate; only the camera moves.

[Scene]: A completely white studio void. The only visible element is the product.
```

The counterintuitive corollary: **detailed descriptions are valuable — but for evaluation, not generation.** Rich product descriptions give evaluation models the vocabulary to assess fidelity ("is the crosshatch pattern present on the heel counter?"). The same descriptions would hurt generation by overriding visual references.

---

## Pillar 3: Automated Evaluation

Evaluation is the hardest pillar and the one that makes the framework production-grade. It combines two categories of methods:

**Deterministic deep learning models** — fast, free, reproducible. Face embedding similarity for identity preservation, optical flow for rotation direction, structural similarity for pixel-level comparison. These form the first line of defense.

**LLM-as-judge with [Gemini](https://cloud.google.com/vertex-ai/generative-ai/docs/models/gemini)** — semantic understanding that catches what deterministic models can't. Visual artifact detection (hallucinated features, morphing, discontinuities), product accuracy scoring, and multi-view consistency checks where all reference and generated views are evaluated together in a single call.

```json
{
  "evaluation_type": "multi_view_consistency",
  "criteria": {
    "strict_must_pass": [
      "Feature placement (buckles, straps)",
      "Structural integrity",
      "No hallucinated features"
    ],
    "lenient_can_vary": [
      "Text legibility",
      "Logo fine detail",
      "Generative artifacts (slight texture variation)"
    ]
  },
  "result": {
    "is_valid": false,
    "explanation": "Failed strict criteria: The crosshatch pattern on the heel counter is missing in the generated back-left view."
  }
}
```

### Evaluation Strategies

These methods combine into composable strategies:

- **Pass/fail with retry** — generate, evaluate, regenerate if it fails (up to a budget)
- **Score-and-rank** — generate N variations in parallel, evaluate all, pick the best
- **Multi-step self-correction** — evaluate, detect the specific deficiency, generate a targeted correction, pick the better result

These strategies compose: parallel variations can each receive correction passes, then be ranked. The right combination depends on the use case and cost profile.

### Cost-Aware Sequencing

Evaluations are sequenced from cheapest to most expensive: free deterministic checks first, then Gemini evaluation, then regeneration only as a last resort. This ensures expensive compute is spent only when cheap checks can't resolve the quality question.

### Calibration

Every evaluation must distinguish real defects from acceptable imperfections. Text and logos should be treated as "visual blobs" — correct position and approximate color is sufficient. Generative models fundamentally struggle with text rendering, and holding them to a legibility standard would reject otherwise excellent outputs. The strict/lenient distinction in the evaluation schema above reflects this principle: structural features must be exact, but text and fine logo detail are expected to vary.

---

## In Practice: Product 360 Spinning

The framework's power shows in concrete applications. Product spinning — generating 360-degree rotation videos from static photographs — demonstrates the full pipeline.

### Generic Products

Generic products (mugs, backpacks, electronics) use the framework at its simplest: extract and upscale inputs, generate with [Veo](https://cloud.google.com/vertex-ai/generative-ai/docs/models/veo) using minimal prompts and reference images, then validate with two layers — a free optical flow check for rotation direction, followed by Gemini artifact detection for hallucinated features.

### Footwear: Pushing the Framework Further

Shoes are complex because they have highly structured geometries with distinct, named views (right, front, left, back, sole) that customers expect to see. They also present unique challenges: pair ambiguity (photos often show two shoes), category-specific failure modes (velcro straps moving unpredictably during generated spins), and the need for multi-view consistency across all angles.

The footwear pipeline deepens each pillar:

- **Deep Classification Routing:** A fine-tuned viewpoint classifier automatically detects and splits pairs, filters out problematic inputs like velcro, and verifies that the input images cover all four cardinal views before any expensive processing begins.
- **Semantic Spin Validation:** Instead of generic motion tracking, we validate the spin by classifying every sampled frame against a strict rotation graph (e.g., `right → front_right → front → front_left → left → ...`). If the video goes counter-clockwise, it is reversed. If it skips views, it fails.
- **Multi-View Consistency Validation:** Generated frames are matched to reference images by their classified viewpoint, and Gemini evaluates consistency across all views in a single call — catching hallucinations and color drift with high precision.

The architecture doesn't change between these use cases. The techniques within each pillar do.

---

## The Framework

The three pillars — input optimization, guided generation, automated evaluation — compose into a loop that turns non-deterministic model calls into reliable production systems:

1. **Optimize inputs** to minimize the model's opportunity to improvise
2. **Guide generation** with images, not text — let the model see, not read
3. **Evaluate automatically** with layered checks — cheap first, semantic second
4. **Retry within budgets** — retries don't guarantee convergence, so cap them and decide: best-effort or exclude

The specific techniques change per product category and use case. The architecture stays the same.

As generative models continue to increase in capability, the engineering scaffolding around them will determine who can deploy them safely in production environments. The three pillars provide a reusable blueprint for that scaffolding — applicable wherever generative media needs to be reliable, brand-consistent, and fully automated.

---
