---
layout: default
title: "ADR 0003: Potrace and Sharp for the Conversion Pipeline"
---

# ADR 0003: Potrace and Sharp for the conversion pipeline

> **Decision:** Potrace for tracing, Sharp for preprocessing. **Why:** Potrace is purpose-built for monochrome path extraction; Sharp is fast enough (~50 ms) for a real-time preview loop. Both are npm-native, no cross-platform binary packaging required.

## Context

Two libraries sit at the heart of this app: one does raster-to-vector tracing (PNG → SVG paths), one does image preprocessing (decode, composite, greyscale, threshold, the inputs the tracer consumes). Picking them correctly is the most consequential technical decision in the project, everything else is UI and state around that pipeline.

Two constraints shaped the choice:

1. **Monochrome scope.** v1 is committed to monochrome tracing (ADR 0013). Single fill colour, threshold-based conversion. Colour, if the user wants it, is a one-attribute edit on the output SVG, the file is XML, the fill is a string. This narrows the tracer field dramatically: I do not need a general-purpose vectoriser.
2. **Real-time preview latency budget.** The preview updates as the user drags the threshold slider, not after they release it. Every frame of that live update runs the full preprocessing pipeline over the source PNG. That puts a hard cap on how long preprocessing can take per frame, roughly tens of milliseconds. A sluggish preprocess pass destroys the slider feel.

This ADR also closes a forward-reference from ADR 0001: the Electron runtime was chosen over Tauri in part because the Node ecosystem gives me Sharp and Potrace as first-class, npm-installable dependencies. Picking those libraries now is not an independent decision, it is the decision that the runtime choice anticipated.

## Options considered

**For tracing:**

- **Potrace.** Academic algorithm (Peter Selinger, 2001) designed specifically for black-and-white path extraction. Available as an npm package that runs in Node, accepts a buffer, and returns an SVG string.
- **ImageTracer.js.** General-purpose JS tracer, supports colour and monochrome. On monochrome inputs, trace quality is visibly worse than Potrace. Project effectively abandoned since 2020.
- **AutoTrace.** Capable, high-quality, but distributed as a native CLI binary. Using it from Electron means compiling the utility for Windows and macOS and packaging the binaries inside the installer for each platform.
- **vtracer.** Rust-based, full colour support. Much more capable than Potrace, and much more than monochrome v1 needs.

**For image preprocessing:**

- **Sharp.** Native bindings around libvips. Buffer-in, buffer-out, composable pipeline. Fastest widely-used image library in the Node ecosystem.
- **Jimp.** Pure-JS image manipulation. Portable, zero native deps, slow.
- **Shell-out to ImageMagick / GraphicsMagick.** Spawn a subprocess per operation. Overhead of process startup per preview frame is prohibitive for a real-time slider.
- **node-canvas.** Native, viable, but slower than Sharp and a worse API fit for the operations this pipeline needs (flatten, greyscale, threshold).

## Decision

- **Potrace** for raster-to-vector tracing.
- **Sharp** for all image preprocessing.

## Rationale

### Why Potrace

**Built for the problem.** Potrace is not a generalist tracer. It is an algorithm designed from the outset for monochrome path extraction, with a published paper, a well-understood set of parameters (corner policy, curve optimisation, turd size), and decades of use in real-world tools (Inkscape's Trace Bitmap, for example). At the scope ADR 0013 commits to, it is the most purpose-fit option.

**Integration surface is minimal.** `npm install` pulls a pure-ish JS port. The API is a buffer in, an SVG string out, running inline in the Node main process. No child processes, no platform-specific binaries, no build-time compilation step. That simplicity is worth real architectural points, every dependency that packs clean reduces the surface area of things that can go wrong at build and release time.

**Rejected alternatives had concrete costs against them:**

- **ImageTracer.** Worse trace quality on the exact inputs I care about. Taking on a core dependency that was last maintained in 2020 is a bet against the project's future, if a security or correctness bug surfaces, I inherit it.
- **AutoTrace.** Native CLI distribution forces a cross-platform build step, a binary-packaging step inside the Electron bundle, and per-platform testing of the shell-out layer. All of that work produces no capability advantage over Potrace at monochrome scope. Over-engineering for what is already solved.
- **vtracer.** Colour-capable, Rust-based, genuinely impressive. Wrong tool for v1, using it would mean paying for colour machinery I do not use, routing logic I do not need, and a heavier native dependency. Held as the natural next step if and when colour tracing enters scope (ADR 0013 follow-up).

### Why Sharp

**Real-time preview is a UX requirement.** When the user drags the threshold slider, they expect to see the vectorisation respond as they drag. Every intermediate value of the slider re-runs the preprocessing pipeline: flatten, optionally negate (ADR 0012), greyscale, threshold, encode. The budget for that pipeline to feel live is in the tens of milliseconds, well under 100 ms per frame.

Sharp, via libvips, does that pipeline in ~50 ms on typical inputs. Jimp, by comparison, measures in hundreds to low thousands of milliseconds, the slider would feel broken. Shell-out to ImageMagick or GraphicsMagick adds process-spawn overhead to every frame, which is catastrophic for a live interaction.

**API fit matches the pipeline.** Sharp's method-chain shape (`.flatten().negate().greyscale().threshold()`) maps directly onto what ADR 0012's inversion preprocessing requires.

## Consequences

**Positive**
- The conversion pipeline is fast enough for real-time preview.
- Both libraries are mature, widely deployed, and actively maintained.
- No cross-platform native toolchain or custom binary packaging, Sharp ships prebuilts for each supported platform, Potrace is portable JS.
- Integration is clean: buffer in, buffer out, all in the Node main process behind the IPC boundary (ADR 0002).

**Negative**
- Potrace is monochrome-only. Colour tracing is not a toggle away, it requires a second tracer (vtracer, likely), and the pipeline gains a routing decision per image.
- Sharp is a native dependency. Electron packaging has to include the correct prebuilt for each target; a misconfigured build environment produces packages that break at runtime with a confusing-looking error. This is a known sharp edge (no pun intended) of the Sharp + Electron combination.

## Follow-up

- If and when ADR 0013's monochrome-first decision is revisited to add colour support, vtracer is the natural next step. The preprocessing layer (Sharp) can remain shared across tracers.
- Sharp prebuilt distribution occasionally causes Electron-packaging friction across platforms. The practical workaround when a prebuilt mismatch surfaces is `electron-rebuild`, which rebuilds Sharp's native module against Electron's Node ABI and resolves most such cases.

**Code:** `src/main/services/ConversionService.ts`, `convertPngToSvg()`, `runSharpPreprocessPipeline()`
