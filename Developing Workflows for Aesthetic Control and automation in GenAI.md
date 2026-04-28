# Workflows for Aesthetic Control in Generative Pipelines

*A technical companion to* Finishing Generative Material. *Implementation notes for ComfyUI and aggregator platforms.*

---

This document assumes you've read the conceptual framing in the companion essay. The four core problems — language ambiguity, foundation-model generalisation, bit-depth ceilings, closed-ecosystem opacity — are stated there. What follows is the working pipeline: how I actually build for aesthetic consistency in production, what the choices cost, and how to verify each piece is doing what it claims.

A note on platform terminology: Weavy was acquired by Figma and is now branded Figma Weave. Where I refer to aggregator workflows, the reasoning applies equally to Figma Weave, Higgsfield, Freepik Spaces, and similar closed platforms hosting third-party models behind a unified interface.

---

## Structured Prompting: What It Actually Does

JSON-formatted prompts are widely described as "letting the model parse rules." This is wrong. Current models tokenise JSON the same way they tokenise any other text — there is no structured-parsing mode. The benefit is real but lives elsewhere: structured formatting enforces consistent vocabulary, ordering, and scope boundaries on the human author of the prompt, generation after generation. Variance drops because the prompt surface stops drifting, not because the model processes tokens differently.

The ordering matters more than people give it credit for. Front-loading constraints reduces structural drift; back-loading them lets earlier creative direction overwrite them. The order I keep returning to:

1. **`task`** — the core action (style_transfer, generate, edit). Primes generative intent.
2. **`preservation_rules`** — what must not change. Declared before any creative direction.
3. **`style_targets`** — film stock, lighting, lens character. Goes after structural boundaries.
4. **`negative_constraints`** — final guardrail before generation.

A working example for style transfer:

```json
{
  "task": "style_transfer",
  "preservation_rules": {
    "composition": "locked",
    "subject_identity": "locked"
  },
  "style_targets": {
    "camera_emulation": "ARRI LogC4",
    "lens_emulation": {
      "lens_type": "Zeiss Super Speed",
      "focal_length": "50mm"
    }
  },
  "negative_constraints": ["modern digital sharpness", "changed composition"]
}
```

Keep the JSON syntactically valid. Sibling objects without array wrapping, trailing commas, unescaped quotes — these degrade reliability on models that do attempt any structural interpretation, and they produce silent parsing errors when the prompt is constructed programmatically by an upstream LLM. If the pipeline auto-generates these prompts, validate the JSON before it reaches the image model. The validation step costs nothing and prevents the kind of drift that's hardest to diagnose later.

---

## LoRA vs. Style Reference vs. Lexicon Pipeline: Choosing

The companion essay covers what each technique fights. The production question is which one to reach for first.

**LoRA training** remains the highest-fidelity approach for any property the base model doesn't already know. Fifty to a hundred curated images per property, one training run per property, version-managed like any other production asset. For a project with four or five recurring aesthetic properties — camera look, lens character, film grain, lighting mood — that means four or five separate LoRAs. Datasets must be narrow and internally consistent. A LoRA trained on mixed properties learns muddled weights and reproduces none of them cleanly.

LoRAs only work in ecosystems that expose weight injection — primarily ComfyUI and other local tools running open-weight base models. On closed platforms, they're not an option.

**Style-reference image-edit models** — Nano Banana Pro, Recraft, Seedream, Qwen Image Edit — have largely replaced the older IP-Adapter workflow for new work. IP-Adapter itself entered maintenance-only mode in April 2025; it still works for SD 1.5 and SDXL pipelines but isn't being updated for newer base models. Treat it as a stable tool for legacy work, not a forward-looking choice. Recraft is built around style reference as a primary capability and accepts one to five reference images.

For video, **Kling 3.0 Elements** (available in Kling, Figma Weave, and Higgsfield) addresses subject consistency rather than general aesthetic style — the same character or product appearing recognisably across multiple shots. For character-driven work, this is often the more important problem to solve than aesthetic consistency, and prompting alone reliably cannot.

The decision sequence I follow: before committing to LoRA training for a given property, test whether a style-reference image-edit model gets you to acceptable results using the same reference dataset. Same inputs, much faster iteration. If the image-edit model scores within an acceptable range of the LoRA on the metrics defined below, the time saved on training is real production budget. If it doesn't, LoRA remains the correct investment for that property — but only that property.

**The VLM-to-LLM lexicon pipeline** is what I reach for when the work is locked to a closed platform. The essay describes the principle. The production implementation:

1. **Reference curation** — fifty to a hundred images precisely demonstrating the target aesthetic. Use the same dataset a LoRA would use, so the two approaches can be compared directly.
2. **VLM extraction with constrained vocabulary** — process the dataset through a vision-language model. Don't ask it to "describe the style" (you'll get web-caption clichés). Ask for "the properties a cinematographer would note on a camera report — stock, ISO, push/pull, diffusion filters, lens coating era, lighting ratio." This pushes the VLM toward a disciplined lexicon.
3. **Lexicon building** — identify the phrases the VLM repeats consistently across the dataset. Repetition across multiple unrelated images is the signal that a phrase is capturing something real rather than hallucinated.
4. **LLM gatekeeper** — an LLM takes simple user input ("woman walking down a beach at sunset") and rewrites it, appending the lexicon terms in the structured JSON above before passing to the image generator. The user writes naturally; the structured vocabulary is injected silently.

The caveat I'd want anyone using this pipeline to internalise: the VLM's training data was also scraped from 8-bit sRGB web imagery, captioned by humans describing that same imagery. The Style Lexicon you extract is therefore a second-order artefact of the same compressed visual culture that produced the base model's limitations. The constrained-vocabulary technique above mitigates this; it doesn't eliminate it. The pipeline works well for aesthetic direction (warm, filmic, slightly soft) and noticeably less well for optical specificity ("Zeiss Super Speed 50mm wide open at T1.3 with the specific swirly out-of-focus character"), because the closed model has no distinct internal representation of the latter to activate.

---

## ComfyUI: Production Graph

The graph below composes the techniques above into a single working pipeline. Every node has a reason to be there.

**LoRA chain.** From the primary Checkpoint Loader, chain Load LoRA nodes — one per stylistic property — adjusting multiplier strengths individually before the K-Sampler. Isolating properties into separate LoRAs lets strength be balanced per-element at generation time. A single LoRA at 0.8 strength rarely produces what four LoRAs at 0.4–0.7 each will, because the latter lets you tune each property independently against the others.

**Structural anchoring.** Process a layout reference image through a depth or Canny edge preprocessor. Feed the result into an Apply ControlNet node alongside the text prompts to lock scene geometry during diffusion. This addresses the subject and composition consistency that LoRAs alone don't handle — a LoRA biases style; ControlNet biases structure. They're complementary, not competing.

**Style reference (alternative or supplement).** Where a LoRA isn't yet trained, or where per-generation style control is needed, use an image-edit model with style reference input — Nano Banana Pro and Recraft are the current strong options — or feed a reference image into IP-Adapter for SDXL-based legacy workflows.

**High-fidelity decode and export.** This is the part most pipelines get wrong. After the K-Sampler, bypass the default 8-bit output path entirely. Use an HDR-aware VAE decode node (vae-decode-hdr) to preserve highlights above 1.0. Route into a 32-bit EXR save node from the ComfyUI-HQ-Image-Save family. Launch ComfyUI with `--fp32-vae` for more accurate decoding. The exported EXR will contain genuine float32 data, usable directly in Nuke, Fusion, or DaVinci Resolve without further conversion.

The companion essay covers why this matters and what it can't do. The shorthand: this is a quality floor improvement, not a path to scene-linear behavior. The model's training data still bounds the latent space.

---

## Aggregator Platforms: Production Graph

Closed platforms — Figma Weave, Higgsfield, Freepik Spaces — host multiple third-party models under a unified interface. Convenient, but reliance on external proprietary models introduces variables outside direct control. Figma Weave excels at chaining distinct AI services and is the natural environment for the VLM-to-LLM pipeline.

**LLM director node.** Initiate the workflow with an LLM text node. Provision its system prompt with the extracted Style Lexicon. Instruct it to format output according to the JSON structure above, and to validate the JSON before passing it downstream.

**Prompt concatenation.** Feed the LLM output into a string manipulation node. Programmatically append any platform-specific syntax, mandatory model tags, or trigger words required by the chosen generation model. This is where a prompt that was clean on the LLM side becomes the actual ugly thing the generator needs.

**Subject consistency via Kling Elements.** For video work, use Kling 3.0's Elements feature to lock character, product, or object appearance across shots. Upload the subject reference once, reference it across every generation in the sequence. This solves the continuity problem that used to require careful prompting and luck.

**Clean reference routing.** Where image-reference features are available, supply references that are clean and compositionally simple. Complex objects in a reference image are routinely misread by the model as structural requirements and bleed into generated content unexpectedly. If the reference contains a chair, expect chairs to appear in generations where no chair was prompted. Crop aggressively or composite reference images on neutral backgrounds.

**Compression mitigation.** For models locked to 8-bit sRGB output, run a post-processing pass in Topaz Video Enhance AI — Nyx for compression artefacts, Gaia HQ for upscaling, currently. This is mitigation, not recovery; it cannot generate information the model did not produce. But on locked-output platforms, it's the difference between material a colorist can grade and material that fights every adjustment. The lumakey example in the companion essay is exactly this — a Topaz pass on aggregator output to clean compression artefacts so a downstream lumakey would pull cleanly.

**App Mode wrapping.** Once the workflow is stable, wrap it in Figma Weave's App Mode to expose only a style selector and a natural language prompt field to end users. All structured prompting, lexicon injection, and platform tags happen invisibly. This enables clients and non-technical staff to generate on-aesthetic content without knowing the platform's vocabulary, while preserving the consistency guarantees built into the workflow. The abstraction is also the test: if a non-technical reviewer can get on-aesthetic results without prompt-engineering knowledge, the pipeline is doing its job.

---

## Verification

The companion essay argues that workflows are only as good as the way you measure them, and proposes consistency, fidelity, and artefact load as the three measurements. The implementation question is how to verify each one in practice.

**LoRA vs. Style Lexicon comparison.** For each style property, gather the same fifty-to-a-hundred-image dataset. Train a LoRA on it. Build a Style Lexicon from it via the VLM pipeline. Generate matched images in ComfyUI (LoRA path) and Figma Weave (lexicon path) using identical subject matter across multiple seeds. Score against LPIPS or DreamSim for consistency, blind panel ranking for fidelity, fixed test stimuli for artefact load. Expect the LoRA to win on optical specificity. Expect the Style Lexicon to hold up surprisingly well on general aesthetic direction. The gap between them at general direction is the gap that determines whether LoRA training is worth its cost on this project.

**Image-edit model vs. LoRA comparison.** Before LoRA training, test whether a style-reference image-edit model — Nano Banana Pro, Recraft — gets to acceptable results using the same reference dataset. If the image-edit model scores within tolerance of the LoRA on the same metrics, the time saved is real production budget. If it doesn't, LoRA remains correct for that property. Run this test before committing training compute, not after.

**Bitdepth verification.** Two specific tests, both diagnostic.

For tonal preservation: generate the same seed twice, save once as 8-bit PNG and once through the HDR EXR path. Plot histograms. If the EXR shows continuous values between the PNG's 8-bit steps, real information is being preserved. If every EXR value snaps to an 8-bit-aligned step, something in the pipeline is quantising before save and the EXR is lying to you.

For highlight preservation: generate a scene with bright specular highlights or direct light sources. In the EXR, those pixels should have values above 1.0 if vae-decode-hdr is working correctly. If everything is clamped to 1.0, the default VAE decode is still in the graph somewhere — find it and replace it.

Both tests take about ten minutes. Skip them and you'll be debugging compositing artefacts six weeks later that traced back to a save node nobody checked.

**Usability verification.** Hand the Figma Weave workflow to a non-technical reviewer behind the App Mode interface. Their ability to get on-aesthetic results without prompt-engineering knowledge is the test. If the answer is yes, the abstraction is working. If results drift the moment they phrase something unexpectedly, the LLM Gatekeeper's system prompt needs tightening. This is also the cheapest test — a single twenty-minute session usually surfaces every failure mode the prompt structure has.

---

## Working with Aggregator Drift

A few practical notes for working in closed ecosystems, where the platform itself is a moving target.

**Standardise prompt construction.** Manual prompting drifts between sessions and between people. The LLM Gatekeeper should be the single point of prompt construction for any workflow where consistency matters across time. Human operators write intent; the Gatekeeper constructs the actual prompt. This isn't about removing humans from the process — it's about removing the part of the process that's most variable from being where the variance lives.

**Track seeds.** When generation seeds are accessible, log them alongside every successful output. Seeds give you a reproducible baseline for iterative refinement and a diagnostic when output quality shifts unexpectedly after a platform update. Without them, you have no way to distinguish "the model changed" from "the prompt drifted."

**Document platform state.** Aggregators update silently. A workflow producing consistent results today may behave differently after an upstream model change with no visible notice. Where the platform supports version-pinning, use it. Where it doesn't, note the date of last successful calibration alongside the workflow and re-validate periodically. The validation cost is small. The cost of discovering mid-production that the model under your pipeline has been quietly replaced is not.

**Account for credit cost.** Aggregator pricing is denominated in credits, and cost per generation varies significantly by model. Premium tiers — Kling 3.0 Professional with Elements and audio, for instance — cost substantially more than standard tiers, and the quality difference on short clips is often small. Build the generation step with explicit model selection rather than defaulting to the highest-tier option. Reserve the premium tiers for generations that specifically require their unique capabilities.

---

*Current as of April 2026. Specific tools, models, and platform features in this document change quickly; the underlying workflow architecture changes more slowly. The companion essay,* Finishing Generative Material, *covers the conceptual frame this document implements.*
