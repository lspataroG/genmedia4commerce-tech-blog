# Part 2: The Architecture Philosophy

> **[Back to Overview](genmedia_at_scale_main.md)** | **Previous: [Part 1 — Why GenMedia is Hard at Scale](genmedia_at_scale_why.md)**

---

## The Three Pillars

Every GenMedia pipeline — regardless of whether it produces spinning videos, try-on images, or lifestyle scenes — is built on the same three-pillar architecture. The pillars are sequential: optimize inputs, guide generation, evaluate outputs. But the system is a loop, not a line: failed evaluations trigger retries with the same optimized inputs, and in some cases evaluation results inform the next generation attempt.

[DIAGRAM: A horizontal flow diagram showing three large blocks labeled "Input Optimization", "Guided Generation", and "Automated Evaluation", connected by forward arrows. From "Automated Evaluation", two paths exit: one arrow pointing right to "Accept" (green), and one arrow looping back to "Guided Generation" labeled "Retry" (orange). Inside each block, show 2-3 representative sub-steps as smaller boxes. Input Optimization contains "Extract", "Enhance", "Classify", "Select". Guided Generation contains "Reference Images", "Minimal Prompt", "Model Call". Automated Evaluation contains "Quality Check", "Score", "Accept/Reject". A dotted line from Evaluation back to Input Optimization is labeled "Self-Correction" for advanced flows.]

---

## Pillar 1: Input Optimization

Input optimization is not a fixed sequence of steps. It's a **toolkit** — a set of techniques that are composed differently for each use case.

### The Toolkit

| Technique | What It Does | When It's Used |
|-----------|-------------|----------------|
| **Extraction** | Isolates the subject from its background | Most use cases — removes noisy backgrounds that confuse generation models |
| **Enhancement** | Upscales resolution, cleans artifacts | When input images are low-resolution or have compression artifacts |
| **Classification** | Identifies properties of the input (viewpoint, category, pose) | When the pipeline needs to route inputs or validate coverage |
| **Selection** | Chooses the best subset of inputs from available options | When multiple input images are available and quality/coverage varies |

### Composition by Use Case

The key insight is that these techniques are **composed per use case**, not applied uniformly. Different pipelines need different combinations:

| Use Case | Extract | Enhance | Classify | Select |
|----------|---------|---------|----------|--------|
| Product Spinning (Generic) | Yes | Yes | No | No |
| Product Spinning (Footwear) | Yes | Yes | Yes (viewpoint) | Yes (best 4 views) |
| Virtual Try-On | Yes | Yes | No | No |
| Background Transformation | Sometimes | Sometimes | No | No |

For generic product spinning, the pipeline extracts the product from its background and upscales it — classification isn't needed because the model doesn't need to know what angle it's looking at. For footwear spinning, classification becomes critical: the pipeline needs to identify each image's viewpoint (right side, front, back, etc.) to verify coverage and select the best images for generation.

> **Key Principle:** Input optimization is about reducing variance in what the model sees. The cleaner and more consistent the input, the less room the model has to improvise — and improvisation is the primary source of errors.

### Fallback Strategies

In production, individual optimization steps can fail. The architecture handles this with cascading fallbacks:

- **Segmentation:** If the primary segmentation model fails (API timeout, unsupported input), fall back to a local ML model
- **Upscaling:** If the target upscale factor would exceed output limits, automatically reduce the factor (4x to 3x to 2x)
- **Classification:** If classification is ambiguous, apply conservative routing (e.g., treat an uncertain classification as "needs splitting" rather than risking incorrect downstream processing)

The goal is never to abort the pipeline due to a preprocessing failure. Degrade gracefully, continue with the best available input, and let the evaluation stage catch any quality issues downstream.

---

## Pillar 2: Guided Generation

The generation call itself is where many teams focus their effort — prompt engineering, parameter tuning, model selection. In the GenMedia architecture, generation is deliberately kept **simple and constrained**. The complexity lives in the pillars on either side.

### Reference Images Lead

For most generative media use cases, the reference images are the primary input. The model should reproduce what it sees in the images, not what it interprets from text.

This creates a clear principle for prompt construction:

> **Key Principle: Descriptions for generation should be minimal.** Let the images lead. A short, generic description ("A red ceramic mug") is sufficient to anchor the model. Detailed descriptions ("A red ceramic mug with a glossy finish, a curved handle on the right side, and a small chip near the base") risk conflicting with the visual reference — and when text conflicts with images, models often follow the text, introducing errors.

In practice, generation prompts follow a pattern:

1. **Subject:** A brief, factual description — type and primary color only
2. **Action:** What should happen (e.g., "a continuous 360-degree orbit")
3. **Scene:** The environment (e.g., "a completely white studio void")

The subject description is generated by a vision model (Gemini) analyzing the input images with deterministic settings (temperature 0, no chain-of-thought). This keeps descriptions factual and minimal — the model identifies "a blue backpack" rather than inventing details.

### The Role of Descriptions: Generation vs. Evaluation

This is perhaps the most counterintuitive insight in the architecture:

**For generation:** Descriptions should be *minimal*. Just enough context for the model to understand the scene, not enough to override the visual references.

**For evaluation:** Descriptions should be *rich and detailed*. A thorough description of the product — materials, hardware, logos, structural details — provides the vocabulary for evaluation models to assess fidelity. "Does the generated image show the distinctive crosshatch pattern on the heel counter?" can only be asked if the evaluation knows about the crosshatch pattern.

This asymmetry is deliberate. Generation models work best when visually guided. Evaluation models work best when given explicit criteria to check against.

[DIAGRAM: A split diagram showing the same product image (a shoe) at the top, with two branches below. Left branch labeled "For Generation" shows a small text bubble: "A blue running shoe" pointing to a generation model icon, then to a generated video frame. Right branch labeled "For Evaluation" shows a large detailed text block listing "blue mesh upper, white midsole, reflective heel tab, crosshatch pattern on heel counter, three ventilation holes on medial side, black rubber outsole with hexagonal grip pattern" pointing to an evaluation model icon comparing the generated output against the reference. The visual contrast between the minimal generation prompt and the detailed evaluation description should be stark.]

### Generation Parameters

Across use cases, generation parameters follow consistent patterns:

| Parameter | Typical Value | Rationale |
|-----------|---------------|-----------|
| Temperature | 0 – 0.1 | Minimize creativity; maximize faithfulness |
| Reference type | Asset / Visual reference | Tell the model to reproduce, not interpret |
| Audio | Disabled (video) | Unnecessary for product media |
| Aspect ratio | Use-case specific | Match the target format (16:9 for video, 3:4 for try-on) |

---

## Pillar 3: Automated Evaluation

Evaluation is the pillar that makes the architecture production-grade. Without it, every output is a gamble. With it, the pipeline can guarantee quality levels that approach or exceed human review.

The evaluation system is covered in depth in **[Part 4: Deep Dive on Evaluation](genmedia_at_scale_evaluation.md)**. Here, we outline the architectural role it plays.

### The Generate-Evaluate Loop

The core production loop is:

```
Generate → Evaluate → Accept or Retry
```

Each evaluation produces one of three outcomes:

| Outcome | Action | Example |
|---------|--------|---------|
| **Accept** | Output passes all quality gates | Video rotates clockwise, no glitches detected |
| **Retry** | Output fails a quality gate; regenerate | Video rotates anticlockwise; try again |
| **Discard** | Output fails hard quality floors; cannot be used | Try-on image completely missing a garment |

### Retry Budgets

Unlimited retries would eventually produce perfect output but at unbounded cost. The architecture enforces **retry budgets** — maximum attempts per generation — that balance quality against economics.

| Strategy | Budget | Behavior on Exhaustion |
|----------|--------|----------------------|
| Conservative | 3–4 attempts | Use best available result (best-effort) |
| Strict | 5+ attempts | Exclude product from output |

The retry budget is a business decision, not a technical one. Higher budgets produce higher quality at higher cost. The architecture supports both approaches.

### Evaluation Informs Generation

In advanced pipelines, evaluation doesn't just accept or reject — it **informs the next attempt**. For virtual try-on, if the first generation changes the model's face, the evaluation detects this and triggers a correction pass that explicitly instructs the model to fix the face using the reference image. This creates a multi-step self-correction loop:

```
Generate → Evaluate Face → Correct Face → Re-Evaluate → Pick Best Step
```

This pattern — evaluate, then generate a targeted correction — is more effective than simply retrying from scratch, because it gives the model specific feedback about what went wrong.

---

## Putting It Together

The three pillars compose into a general-purpose architecture that can be instantiated for any generative media use case:

[DIAGRAM: A detailed architecture diagram showing the full pipeline. Top row: "Retailer Catalog" (database icon) feeds into "Input Optimization" box containing four connected steps: "Extract" (scissors icon) → "Enhance" (upscale arrow icon) → "Classify" (tag icon) → "Select" (checkmark icon), with dotted borders around Classify and Select indicating they're optional. Middle row: "Guided Generation" box containing "Reference Images" + "Minimal Prompt" feeding into "Model Call" (Veo/Gemini/Imagen icon). Bottom row: "Automated Evaluation" box containing three parallel tracks: "Deterministic ML" (gear icon), "LLM-as-Judge" (brain icon), "Reference Comparison" (side-by-side icon), all feeding into a "Composite Score" diamond. From the diamond, two arrows: green arrow right to "Production Asset" (checkmark), orange arrow looping back up to "Guided Generation" labeled "Retry (budget: N)". The whole diagram should flow top-to-bottom with clear separation between the three pillar zones.]

The architecture is the same whether producing spinning videos with Veo, try-on images with Gemini, or any other generative media format. What changes is the specific composition of techniques within each pillar — which is exactly what the case studies in Parts 5 and 6 demonstrate.

> **Next: [Part 3 — Deep Dive: Input Processing](genmedia_at_scale_input_processing.md)**
