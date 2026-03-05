# Part 3: Deep Dive — Input Processing

> **[Back to Overview](genmedia_at_scale_main.md)** | **Previous: [Part 2 — The Architecture Framework](genmedia_at_scale_architecture.md)**

---

## Why Input Processing Matters

Generative models amplify whatever they receive. Clean, well-prepared inputs produce clean outputs. Noisy, cluttered inputs produce noisy outputs — and more retries, higher costs, and lower yield.

Input processing is not about making inputs "look nice." It's about **reducing the decision space** for the generation model. Every pixel of background clutter, every compression artifact, every irrelevant object in the frame is an opportunity for the model to get distracted, hallucinate, or produce something unexpected.

---

## Extraction and Segmentation

Product images from real catalogs rarely arrive as clean, isolated subjects. They come with studio backgrounds, lifestyle contexts, watermarks, color swatches, and other visual noise. Generation models will attempt to reproduce or react to all of this.

Extraction isolates the subject from its surroundings using image segmentation, then places it on a controlled, uniform background. The segmentation produces a pixel-level mask of the product, which is applied to crop and composite the subject onto a clean canvas.

---

## Enhancement and Upscaling

Generation models produce better results with higher-resolution reference images. Real catalogs contain everything from low-resolution mobile photos to high-quality studio shots. Upscaling models can recover detail and increase resolution, giving the generation model more visual information to work with and improving output quality.

---

## Classification as Routing

Not all inputs are equal, and not all inputs should be processed the same way. Real catalog images arrive unlabeled — you don't know the viewpoint, the product category, whether the image is cropped, whether it shows the product in isolation or in context, or whether it's even usable for the target pipeline.

Classification uses a vision model to categorize each input image, then **routes it** through the appropriate processing path. The key insight is that classification is not labeling for its own sake — it's a **routing decision** that determines what happens next: proceed to generation, split and re-process, discard, or abort early for known failure modes.

**Classification happens early and cheap.** A lightweight vision model call makes fast routing decisions before expensive steps like upscaling and generation. Filtering out unusable images early is one of the most impactful cost optimizations in the pipeline.

**Feasibility checks.** After classifying all inputs, the pipeline can verify whether the available images are sufficient for the target output. If critical inputs are missing, the pipeline can abort early rather than producing a generation that's incomplete. This fail-fast behavior saves compute and produces more predictable results.

---

## Multi-Input Curation and Selection

Many use cases accept multiple reference images. A product might have 2, 4, or even 10 images in its catalog listing. Using all of them isn't always better — duplicate viewpoints add noise, low-quality images drag down the result, and generation models have practical limits on how many reference images they can effectively use.

Selection identifies the **best subset** of inputs for the generation task:

- **Deduplication.** When multiple images cover the same viewpoint or angle, pick the one with the highest quality or completeness.
- **Coverage optimization.** Select images that maximize viewpoint diversity — covering as many distinct angles as possible within the model's reference image budget.

---

## Processing at Scale

When processing thousands of products, input optimization must be parallelized. Individual images within a product are processed concurrently, and products themselves can be processed independently. The key constraint is API rate limits on cloud services (segmentation, upscaling), which require backoff and retry logic at the infrastructure level — or provisioned throughput to guarantee capacity at scale.

The input processing stage is also where **the most cost can be saved.** By classifying and filtering inputs early, the pipeline avoids spending expensive generation and evaluation resources on products that were never going to produce acceptable output.

> **Next: [Part 4 — Deep Dive: Evaluation](genmedia_at_scale_evaluation.md)**
