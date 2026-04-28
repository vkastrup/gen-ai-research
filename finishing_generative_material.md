# Finishing Generative Material

*Notes on the seam between what the model produces and what the rest of the pipeline needs.*

---

A frame from a generative model is not a frame. It is a candidate for a frame. It looks right, often surprisingly so, and it behaves wrong — sometimes immediately, more often three steps later, when someone tries to grade it, key it, comp it into something else and the material refuses.

Most of my recent work has been the part that happens after the model and before the deliverable. The frames arrive looking like photographs and don't survive being treated like photographs. Someone has to know what's wrong with them, recognize it before the colorist or the compositor finds out the hard way, and have a pipeline ready to fix it. Twenty years of post craft turn out to be unexpectedly relevant to a problem that didn't exist when I started.

These are observations from doing that work. Not a guide. The specifics will be stale within a year. The shape of the problem won't be.

---

## What's Actually Hard

Aesthetic inconsistency in generative workflows isn't a single problem. It's four problems stacked on top of each other, each demanding a different class of solution. Conflating them is how teams end up with workflows that address symptoms instead of causes.

**The first is language itself.** Words like *cinematic*, *moody*, *filmic* are semantically broad. The model picks an interpretation from whatever region of its training distribution those words most strongly activate, and that activation is not deterministic. The same prompt at different seeds produces meaningfully different images. The same seed after a silent model update produces meaningfully different images. Natural language is not a stable instrument for specifying optical behavior.

**The second is what the foundation models actually know.** Stable Diffusion, FLUX, and their derivatives are trained for broad capability rather than optical precision. They cannot reliably simulate specific physical camera and lens characteristics from text alone, because those characteristics were rarely labelled explicitly in training data. A prompt referencing *Arri Alexa 65* or *Zeiss Super Speed* activates a general cluster of cinema-looking imagery, not a learned optical model of that specific hardware. The label is in the training set; the physics is not.

**The third is bit depth, and it goes deeper than the file format.** Most generative outputs are 8-bit sRGB. That caps dynamic range, introduces banding on smooth gradients, and bakes compression artefacts into anything downstream. The decode-stage and save-stage parts of this are addressable in open ecosystems — you can preserve the float tensor through save, you can use HDR-aware decode nodes, you can write 32-bit EXR instead of PNG. What you cannot fix at decode is what wasn't there at training. Most generative image models are trained on 8-bit sRGB datasets scraped from the public web. The latent space itself encodes a compressed, gamma-shaped representation of the visual world. A wider container preserves the tonal information the model actually generates. It cannot recover information that was never encoded in the first place. The difference between *our container is wider* and *our model knows more* is the difference between a useful 16-bit image and a 16-bit wrapper around 8-bit data.

**The fourth is the closed ecosystems.** Proprietary platforms — both aggregators and first-party services — don't expose the customisation surfaces available in open workflows. Custom weight injection, structural conditioning, sampler configuration, CFG scale — abstracted away. Users interact with the model through text prompts and a small set of platform-exposed controls. Consistency becomes a matter of reverse-engineering which specific phrases reliably activate the desired latent regions in a model whose training vocabulary is undocumented and whose behavior can change overnight without notice.

These are the four things any serious workflow has to negotiate with. None of them are going away soon.

---

## What the Workarounds Are For

Three families of technique address these problems, and they are not interchangeable. Each one fights a different fight.

**Structured prompting** addresses language ambiguity. The popular framing is that the model "parses the rules." It doesn't. Current models tokenise JSON the same way they tokenise any other text. The actual benefit is that structured formatting enforces consistent vocabulary, ordering, and scope boundaries on *the human author* of the prompt, generation after generation. Variance drops not because the model is processing tokens differently but because the prompt surface no longer drifts. Defining a vocabulary and sticking to it is most of the work.

**LoRA training** is the only technique on this list that can teach a model something new. A small set of trained weights, injected into the attention layers, structurally biases output toward properties that text alone cannot reliably activate. A LoRA trained on fifty to a hundred curated images of a specific lens character or film stock will reproduce that character far more reliably than any prompt. It's also the most expensive option — each distinct property needs its own dataset, its own training run, its own version management — and it's only available in ecosystems that expose weight injection. On closed platforms, LoRA isn't an option at all.

**Style-reference systems** sit between these. Image-edit models like Nano Banana Pro, Recraft, and Seedream accept reference images as a first-class input and generate new content in that visual language without training new weights. They've largely replaced earlier IP-Adapter workflows for new work in the past year. For subject consistency specifically — keeping a character recognisable across multiple shots — features like Kling Elements solve a problem that prompting alone reliably cannot. None of these techniques teach the model anything; they're all variations on *push the prompt toward a region the model already knows.* But they often get close enough to acceptable that the cost of LoRA training isn't justified, and recognising that distinction in advance saves real production time.

The fourth technique is one I find myself reaching for most often when working in closed ecosystems: a VLM-to-LLM pipeline that extracts a stable vocabulary from a reference dataset and injects it automatically into every prompt. A vision-language model processes the dataset and describes only optical and photographic properties — *constrained vocabulary*, not generic web-caption clichés. The phrases that recur across multiple references become a "Style Lexicon" — the model's own descriptive vocabulary for that aesthetic. An LLM gatekeeper takes simple user input and rewrites it, appending the lexicon terms in structured JSON before passing it to the image generator. The user writes naturally; the structured vocabulary is injected silently.

This is not a substitute for LoRA. It cannot teach the model anything new. It can only push prompts more reliably toward regions the model already recognises. But on platforms where weight injection isn't available, it's the closest functional equivalent — and it has the side benefit of removing prompt-engineering knowledge as a prerequisite for getting consistent output, which makes it usable by collaborators who shouldn't have to learn the platform's vocabulary to get work done.

---

## Bit Depth, Specifically

This deserves its own section because it's the part of the work that most directly involves my craft, and the part most often glossed over by people writing about generative workflows.

The default output path in most generative tools is: latent → VAE decode → clamp to 0–1 float → quantise to 8-bit sRGB → save as PNG. Each step after the VAE decode discards information. The final save is where banding gets introduced, because continuous float values are being compressed into 256 levels per channel. Smooth gradients posterise. Highlight detail clips. Shadows lose subtlety. None of this is recoverable by the colorist downstream — the information is gone before they ever open the file.

In an open ecosystem, two improvements are available. First, you can preserve the float tensor through save: ComfyUI carries images internally as float32, and dedicated save nodes (the HQ-Image-Save family) write that tensor directly as 32-bit EXR. This isn't a wider container around an 8-bit image; it's the actual float data the VAE produced. The verification is straightforward — generate the same seed twice, save once as PNG and once as EXR, compare histograms. If the EXR shows continuous values between the PNG's 8-bit-aligned steps, real information is being preserved. If every EXR value snaps to an 8-bit step, something in the pipeline is quantising before save and the EXR is lying.

Second, you can preserve dynamic range through decode. The standard VAE decode clamps highlights to 1.0. Custom decode nodes bypass this normalisation and let highlight values above 1.0 pass through into the EXR. This produces genuine HDR output, suitable for compositing and grading rather than an SDR image saved in an HDR container. Verification: generate a scene with bright specular highlights or direct light sources. In the EXR, those pixels should have values above 1.0. If everything is clamped, the default decode is still in the graph somewhere.

Both of these address decode-stage and save-stage compression. Neither addresses what happens at training. If the base model was trained on 8-bit sRGB imagery, its latent representation is bounded by what that training data encoded. A high-bitdepth decode can preserve information the model produces; it cannot generate information the model didn't. Treat these techniques as a quality floor — smoother gradients, less banding, preserved highlights where the model did predict them — not as a path to scene-linear behavior. For genuine scene-linear output, comparable to a raw camera negative or a path-traced render, the underlying model would need to be fine-tuned on HDR linear data, which is beyond what LoRAs are designed to do and beyond what's currently practical for most productions.

On closed aggregator platforms, none of this is available. Output is 8-bit sRGB, full stop. The only mitigation is post-processing — Topaz's compression-fix and upscaling models will recover the appearance of dynamic range and clean up the worst of the artefacts, enough that the colorist can pull a usable lumakey from material that would otherwise refuse one. This is genuinely useful, but it's mitigation, not recovery. The work is making the material *behave* as if it had dynamic range it never had. That's a different job than working with camera-original footage, and pretending otherwise produces the kind of pipeline that fails quietly two steps from where the problem started.

---

## What Counts as Working

A workflow is only as good as the way you measure it. Without a definition of success articulated in advance, comparison between approaches collapses into taste, and the loudest opinion in the room wins.

Three measurements have served me well. **Consistency** can be quantified with perceptual similarity scores — LPIPS, DreamSim — across generations of matched prompts at different seeds. Lower scores mean more consistent output. **Fidelity** is harder to automate and probably shouldn't be: a small panel of three to five people ranking blind pairs against reference imagery is enough for most production decisions, and the qualitative disagreement is itself useful information. **Artefact load** is observable directly on fixed test stimuli — smooth gradient, high-contrast edge, low-light detail, fine-texture region. Track banding, posterisation, edge artefacts, noise character.

Define these before testing. The temptation is to compare approaches by generating ten images and squinting at them. That tells you almost nothing. Define what you're measuring, generate matched material across approaches, score against the definitions. The discipline of articulating success in advance is most of what separates a workflow that improves over time from one that drifts.

---

## Honest Limits

The techniques in this document are the best available practice within the current generation of image and video models. They are not solutions to the underlying problems. They are disciplined workarounds.

Some of what this work fights is genuinely unsolvable without changes upstream of any production pipeline. Until foundation models are trained on HDR linear data with proper colour-science labels rather than 8-bit sRGB web imagery, the phrase *Arri LogC4* will continue to activate a cluster of cinematic-looking images rather than the physical transfer function the name refers to. Until subject consistency is solved at the model architecture level rather than bolted on through Elements-style reference systems, character continuity across shots will remain a workflow problem rather than a generation capability. Until model providers commit to versioning and disclosure, building stable production pipelines on closed platforms will remain an exercise in negotiating with weather.

This ceiling is not a reason to avoid the techniques above. It is a reason to set accurate expectations with collaborators, clients, and non-technical users.

A workflow that delivers reliable aesthetic consistency while being honest about what it cannot do is more useful in production than one that oversells its capabilities and produces unpredictable results in edge cases. The honest frame: these techniques close most of the gap between user intent and model output. The gap that remains is the current state of the art, not a failure of the workflow.

The work I do, increasingly, is the work of holding that gap visible. Telling the colorist what the source material won't do before they try. Telling the producer what the pipeline can't promise before they sell it to the client. Telling the director where the model's failure modes live so they can decide whether to route around them or commit to them. Generative tools have made it easier than ever to produce frames that look right. Making frames that *behave* right — frames that survive the rest of the pipeline, that arrive at the audience having been finished rather than just generated — is increasingly where the craft lives.

That's the part I'm interested in. The part that's still about pixels, and the discipline of looking at them honestly.

---

*Current as of April 2026. Specific tools and techniques change quickly; the underlying problems change slowly. A companion piece with implementation details for ComfyUI and Figma Weave workflows is available separately.*
