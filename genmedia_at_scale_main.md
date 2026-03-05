# GenMedia at Scale: Production-Grade Generative Media for Commerce

## TL;DR

Generative AI models can produce stunning media — but getting them to work reliably at catalog scale is a different challenge entirely. This technical pattern describes the architecture and methodology for building **production-grade generative media pipelines** — the kind that produce 360-degree product spins, virtual try-on experiences, and contextualized product imagery across entire e-commerce catalogs with minimal human review.

The core insight: **the generation call is the easy part.** The hard part is everything around it — preparing inputs so models succeed on the first try, and automatically evaluating outputs so no human needs to review them.

---

## The Problem

Retailers need rich product media — videos, contextual images, try-on experiences — across catalogs of thousands or millions of SKUs. Manual production doesn't scale. Generative AI should be the answer, but raw model outputs are unpredictable: a spinning video might reverse direction mid-spin, a try-on image might change the model's face, a product rendering might hallucinate extra buttons on a jacket.

At scale, raw model outputs require human review to catch failures — and when you're generating media across thousands of SKUs, that review becomes a bottleneck that defeats the purpose of automation. The business case demands fully automated pipelines that can detect, retry, and resolve quality issues without human intervention.

---

## Technical Patterns

The architecture is built on three pillars that apply across all generative media use cases:

### 1. Input Optimization
> **[Part 2: The Architecture Philosophy](genmedia_at_scale_architecture.md)** | **[Part 3: Deep Dive on Input Processing](genmedia_at_scale_input_processing.md)**

A toolkit of techniques — extraction, enhancement, classification, selection — composed differently for each use case. The goal: give the model the best possible starting point so it produces correct output more often.

### 2. Guided Generation
> **[Part 2: The Architecture Philosophy](genmedia_at_scale_architecture.md)**

Let reference images lead. Keep text descriptions minimal for generation — just enough to set the scene, not enough to conflict with visual references. The model should look at the images, not try to reconstruct details from text.

### 3. Automated Evaluation
> **[Part 4: Deep Dive on Evaluation](genmedia_at_scale_evaluation.md)**

The hardest and most important piece. Every generated asset is automatically scored using a combination of deterministic deep learning models, LLM-as-judge evaluations, and ground truth comparisons. Assets that fail are retried or discarded — no human in the loop.

---

## Document Structure

| Part | Title | Focus |
|------|-------|-------|
| **[Part 1](genmedia_at_scale_why.md)** | Why GenMedia is Hard at Scale | Business case, core challenges, why "just call the API" fails |
| **[Part 2](genmedia_at_scale_architecture.md)** | The Architecture Philosophy | Three pillars, input optimization as a toolkit, the generate-evaluate loop |
| **[Part 3](genmedia_at_scale_input_processing.md)** | Deep Dive: Input Processing | Extraction, enhancement, classification, multi-input curation |
| **[Part 4](genmedia_at_scale_evaluation.md)** | Deep Dive: Evaluation | Evaluation methods, strategies, composite scoring, retry budgets |
| **[Part 5](genmedia_at_scale_spinning.md)** | Case Study: Product 360 Spinning | Generic product spinning pipeline with two-level validation |
| **[Part 6](genmedia_at_scale_spinning_shoes.md)** | Case Study: Footwear Spinning | Specialized footwear pipeline with semantic validation |

---

## Key Technologies

| Component | Technology | Role |
|-----------|-----------|------|
| Video Generation | **Veo 3.1** | Reference-conditioned video synthesis |
| Image Generation | **Nano Banana** | Virtual try-on image synthesis |
| Vision Evaluation | **Gemini 3 Flash** | Glitch detection, garment accuracy, product consistency |
| Image Enhancement | **Imagen 4 Upscale** | Reference image upscaling |
| Face Similarity | **InsightFace ArcFace** | Deterministic face embedding comparison |
| Rotation Detection | **Optical Flow (Lucas-Kanade)** | Deterministic spin direction classification |
| Segmentation | **Vertex AI Image Segmentation** | Background removal and object extraction |

---

## Quick Impact

| Metric | Impact |
|--------|--------|
| **Conversion** | Shoppers are 144% more likely to add-to-cart after engaging with video or contextualized imagery* |
| **Returns** | 42% of returns happen because the product "looked different than expected"* |
| **Cost** | Media generation costs reduced to a fraction of manual production |
| **Scale** | Fully automated pipelines across entire catalogs — no human review required |

*\*Source: Shopify 2025*

---

> **Next: [Part 1 — Why GenMedia is Hard at Scale](genmedia_at_scale_why.md)**
