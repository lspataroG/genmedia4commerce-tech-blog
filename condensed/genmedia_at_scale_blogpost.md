# Beyond the Prompt: A Framework for Scaling GenMedia in Commerce

Generative AI models can produce stunning product media — 360-degree spins, virtual try-on images, lifestyle scenes. But the gap between a single impressive demo and reliable production across thousands of SKUs is vast. This post describes a framework for bridging that gap.

---

## The Problem

E-commerce runs on visuals. Rich product media — spinning videos, try-on experiences, contextual imagery — drives 144% higher add-to-cart rates, while 42% of returns happen because the product "looked different than expected."* Traditional production (studio shoots, turntables, model bookings) costs $50–500+ per SKU, making catalog-wide coverage impossible.

Generative AI should be the answer. Models like [Veo](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/models/veo/3-0-generate), [Gemini](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/start/get-started-with-gemini-3), and [Imagen](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/models/imagen/4-0-upscale) can synthesize photorealistic media from product photographs. But raw model outputs are unpredictable: spinning videos reverse direction, try-on images change faces, product renderings hallucinate extra features. At scale, every output needs quality assurance — and human review defeats the purpose of automation.

*\*Source: Shopify 2025*

---

## The Core Insight

**The generation call is the easy part.** The hard part is everything around it: preparing inputs so models succeed on the first try, and automatically evaluating outputs so no human needs to review them.

This leads to a three-pillar framework that applies across all generative media use cases.

---

## Pillar 1: Input Optimization

Generative models amplify whatever they receive. Noisy inputs produce noisy outputs. The first pillar is a **toolkit** of techniques — composed differently per use case — that reduce the decision space for the generation model:

- **Extraction** isolates the product from cluttered backgrounds using segmentation
- **Enhancement** upscales low-resolution catalog images to give the model more visual detail
- **Classification** categorizes inputs (viewpoint, category, usability) and routes them — proceed, discard, or abort early
- **Selection** picks the best subset of images when multiple are available, maximizing quality and viewpoint coverage

The key principle: every pixel of background clutter, every compression artifact, every irrelevant object is an opportunity for the model to hallucinate. Clean inputs mean fewer retries, lower costs, and higher yield.

---

## Pillar 2: Guided Generation

This is where many teams over-invest in prompt engineering. In this framework, generation is deliberately kept **simple and constrained**.

**Reference images lead.** The model should reproduce what it sees in the images, not what it interprets from text. A short description ("a blue running shoe") anchors the model. A detailed description ("blue mesh upper with reflective heel tab and crosshatch pattern") risks conflicting with the visual reference — and when text conflicts with images, models follow the text, introducing errors.

The counterintuitive corollary: **detailed descriptions are valuable — but for evaluation, not generation.** Rich product descriptions give evaluation models the vocabulary to assess fidelity ("is the crosshatch pattern present on the heel counter?"). The same descriptions would hurt generation by overriding visual references.

---

## Pillar 3: Automated Evaluation

Evaluation is the hardest pillar and the one that makes the framework production-grade. It combines two categories of methods:

**Deterministic deep learning models** — fast, free, reproducible. Face embedding similarity for identity preservation, motion tracking for rotation direction, structural similarity for pixel-level comparison. These form the first line of defense.

**LLM-as-judge with [Gemini](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/start/get-started-with-gemini-3)** — semantic understanding that catches what deterministic models can't. Visual artifact detection (hallucinated features, morphing, discontinuities), product accuracy scoring, and multi-view consistency checks where all reference and generated views are evaluated together in a single call.

### Evaluation Strategies

These methods combine into strategies:

- **Pass/fail with retry** — generate, evaluate, regenerate if it fails (up to a budget)
- **Score-and-rank** — generate N variations in parallel, evaluate all, pick the best
- **Multi-step self-correction** — evaluate, detect the specific deficiency, generate a targeted correction, pick the better result

These strategies compose: parallel variations can each receive correction passes, then be ranked. The right combination depends on the use case and cost profile.

### Cost-Aware Sequencing

Evaluations are sequenced from cheapest to most expensive: free deterministic checks first, then Gemini evaluation, then regeneration only as a last resort. This ensures expensive compute is spent only when cheap checks can't resolve the quality question.

### Calibration

Every evaluation must distinguish real defects from acceptable imperfections. Text and logos should be treated as "visual blobs" — correct position and approximate color is sufficient. Generative models struggle with text rendering, and holding them to a legibility standard would reject otherwise excellent outputs.

---

## In Practice: Product 360 Spinning

The framework's power shows in concrete applications. Product spinning — generating 360-degree rotation videos from static photographs — demonstrates the full pipeline.

**Generic products** (mugs, backpacks, electronics) use the framework at its simplest: extract and upscale inputs, generate with [Veo](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/models/veo/3-0-generate) using minimal prompts and reference images, then validate with two layers — a free motion tracking check for rotation direction, followed by Gemini artifact detection for hallucinated features.

**Footwear** pushes the framework further. Shoes have named viewpoints (right, left, front, back) that enable **semantic spin validation** — classifying every frame and checking that the sequence follows a valid rotation graph, a much stronger check than motion direction alone. The input optimization layer becomes classification-driven: detecting pairs, splitting them, verifying four-cardinal-view coverage before spending on generation, and ordering images to match the rotation path. A multi-view ground truth comparison with Gemini catches cases where the model hallucinated details or changed colors.

The architecture doesn't change between these use cases. The techniques within each pillar do.

---

## The Framework

The three pillars — input optimization, guided generation, automated evaluation — compose into a loop that turns non-deterministic model calls into reliable production systems:

1. **Optimize inputs** to minimize the model's opportunity to improvise
2. **Guide generation** with images, not text — let the model see, not read
3. **Evaluate automatically** with layered checks — cheap first, semantic second
4. **Retry within budgets** — retries don't guarantee convergence, so cap them and decide: best-effort or exclude

The specific techniques change per product category and use case. The architecture stays the same.
