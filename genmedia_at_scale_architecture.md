# Part 2: The Architecture Framework

> **[Back to Overview](genmedia_at_scale_main.md)** | **Previous: [Part 1 — Why GenMedia is Hard at Scale](genmedia_at_scale_why.md)**

---

## The Three Pillars

Every production-grade generative media pipeline is built on the same three-pillar framework. The pillars are sequential: optimize inputs, guide generation, evaluate outputs. But the system is a loop, not a line: failed evaluations trigger retries with the same optimized inputs, and in some cases evaluation results inform the next generation attempt.

[DIAGRAM: A horizontal flow diagram showing three large blocks labeled "Input Optimization", "Guided Generation", and "Automated Evaluation", connected by forward arrows. From "Automated Evaluation", two paths exit: one arrow pointing right to "Accept" (green), and one arrow looping back to "Guided Generation" labeled "Retry" (orange). Inside each block, show 2-3 representative sub-steps as smaller boxes. Input Optimization contains "Extract", "Enhance", "Classify", "Select". Guided Generation contains "Reference Images", "Minimal Prompt", "Model Call". Automated Evaluation contains "Quality Check", "Score", "Accept/Reject". A dotted line from Evaluation back to Input Optimization is labeled "Self-Correction" for advanced flows.]

---

## Pillar 1: Input Optimization

Input optimization is not a fixed sequence of steps. It's a **toolkit** — a set of techniques that are composed differently depending on the use case and the quality of available inputs.

### The Toolkit

| Technique | What It Does | When It's Used |
|-----------|-------------|----------------|
| **Extraction** | Isolates the subject from its background | Removes noisy backgrounds that confuse generation models |
| **Enhancement** | Upscales resolution, cleans artifacts | When input images are low-resolution or have compression artifacts |
| **Classification** | Identifies properties of the input (viewpoint, category, pose) | When the pipeline needs to route inputs, validate coverage, or filter unusable images |
| **Selection** | Chooses the best subset of inputs from available options | When multiple input images are available and quality/coverage varies |

> **Key Principle:** Input optimization is about reducing variance in what the model sees. The cleaner and more consistent the input, the less room the model has to improvise — and improvisation is the primary source of errors.

---

## Pillar 2: Guided Generation

The generation call itself is where many teams focus their effort — prompt engineering, parameter tuning, model selection. In this framework, generation is deliberately kept **simple and constrained**. The complexity lives in the pillars on either side.

### Reference Images Lead

For most generative media use cases, the reference images are the primary input. The model should reproduce what it sees in the images, not what it interprets from text.

> **Key Principle: Descriptions for generation should be minimal.** Let the images lead. A short, generic description ("A red ceramic mug") is sufficient to anchor the model. Detailed descriptions ("A red ceramic mug with a glossy finish, a curved handle on the right side, and a small chip near the base") risk conflicting with the visual reference — and when text conflicts with images, models often follow the text, introducing errors.

### The Role of Descriptions: Generation vs. Evaluation

This is perhaps the most counterintuitive insight in the framework:

**For generation:** Descriptions should be *minimal*. Just enough context for the model to understand the scene, not enough to override the visual references.

**For evaluation:** Descriptions should be *rich and detailed*. A thorough description of the product — materials, hardware, logos, structural details — provides the vocabulary for evaluation models to assess fidelity. "Does the generated image show the distinctive crosshatch pattern on the heel counter?" can only be asked if the evaluation knows about the crosshatch pattern.

This asymmetry is deliberate. Generation models work best when visually guided. Evaluation models work best when given explicit criteria to check against.

[DIAGRAM: A split diagram showing the same product image (a shoe) at the top, with two branches below. Left branch labeled "For Generation" shows a small text bubble: "A blue running shoe" pointing to a generation model icon, then to a generated video frame. Right branch labeled "For Evaluation" shows a large detailed text block listing "blue mesh upper, white midsole, reflective heel tab, crosshatch pattern on heel counter, three ventilation holes on medial side, black rubber outsole with hexagonal grip pattern" pointing to an evaluation model icon comparing the generated output against the reference. The visual contrast between the minimal generation prompt and the detailed evaluation description should be stark.]

---

## Pillar 3: Automated Evaluation

Evaluation is the pillar that makes the framework production-grade. Without it, every output is a gamble. With it, the pipeline can achieve quality levels that approach or exceed human review.

The evaluation system is covered in depth in **[Part 4: Deep Dive on Evaluation](genmedia_at_scale_evaluation.md)**. Here, we outline the architectural role it plays.

### The Generate-Evaluate Loop

The core production loop is:

```
Generate → Evaluate → Accept or Retry
```

Each evaluation produces one of three outcomes:

| Outcome | Action |
|---------|--------|
| **Accept** | Output passes all quality gates — deliver to production |
| **Retry** | Output fails a quality gate — regenerate within the retry budget |
| **Discard** | Output fails hard quality floors — cannot be used regardless of score |

### Retry Budgets

Retries don't guarantee convergence — some inputs simply won't produce acceptable output. The framework enforces **retry budgets** that cap attempts and define what happens when the budget is exhausted: either use the best available result (best-effort) or exclude the product entirely (strict). The choice is a business decision that trades quality against cost and coverage.

### Evaluation Informs Generation

In advanced pipelines, evaluation doesn't just accept or reject — it **informs the next attempt**. If evaluation detects a specific deficiency in the output, it can trigger a targeted correction pass rather than a blind retry. This creates a multi-step self-correction loop:

```
Generate → Evaluate → Correct Specific Issue → Re-Evaluate → Pick Best Step
```

This pattern is more effective than retrying from scratch, because it builds on a partially-correct result and gives the model specific feedback about what went wrong.

---

## Putting It Together

The three pillars compose into a general-purpose framework that can be instantiated for any generative media use case:

[DIAGRAM: A detailed architecture diagram showing the full pipeline. Top row: "Product Catalog" (database icon) feeds into "Input Optimization" box containing four connected steps: "Extract" (scissors icon) → "Enhance" (upscale arrow icon) → "Classify" (tag icon) → "Select" (checkmark icon), with dotted borders around Classify and Select indicating they're optional. Middle row: "Guided Generation" box containing "Reference Images" + "Minimal Prompt" feeding into "Model Call" (generic model icon). Bottom row: "Automated Evaluation" box containing three parallel tracks: "Deep Learning Models" (gear icon), "LLM-as-Judge" (brain icon), "Ground Truth Comparison" (side-by-side icon), all feeding into a "Composite Score" diamond. From the diamond, two arrows: green arrow right to "Production Asset" (checkmark), orange arrow looping back up to "Guided Generation" labeled "Retry (budget: N)". The whole diagram should flow top-to-bottom with clear separation between the three pillar zones.]

What changes across use cases is the specific composition of techniques within each pillar — which is exactly what the case studies in Parts 5 and 6 demonstrate.

> **Next: [Part 3 — Deep Dive: Input Processing](genmedia_at_scale_input_processing.md)**
