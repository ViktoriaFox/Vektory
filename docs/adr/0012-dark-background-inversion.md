---
layout: default
title: "ADR 0012: Dark Background Inversion"
---

# ADR 0012: Dark background inversion preprocessing

> **Decision:** Manual inversion toggle as default, optional auto-detect as opt-in. Pipeline order: `flatten → negate({ alpha: false }) → greyscale → threshold`. **Why:** Each step prevents a specific failure mode; reordering any of them reintroduces a known bug.

## Context

Potrace traces dark pixels on a light background. Source images that are light-on-dark, white icons on a black canvas, white text on a dark photo, transparent PNGs with dark foregrounds, produce empty or incorrectly-traced SVGs by default. Potrace either finds nothing to trace, or traces the background as a filled rectangle.

I had been hitting this bug for weeks while using Vektory for my own design work; the release deadline simply forced the fix from "later" to "now".

## Options considered

- **Always invert.** Simplest code path, wrong for 95% of inputs (normal logos, dark art on white). Rejected.
- **Auto-detection only.** Analyse the source image and invert when needed. Good UX if the heuristic is right. Risky: silent auto-adjustment of user input is a UX antipattern when it misfires, and at release I had no large enough test corpus to calibrate confidence.
- **Manual toggle only.** A checkbox the user flips when input is dark-background. Explicit, predictable, no surprise. Requires the user to know their input.
- **Manual toggle + optional auto-detection flag.** Manual is the default behaviour; auto-detection is available as an opt-in for users who want it. Explicit path covers the common case; heuristic is there for batch workflows where per-file toggling would be tedious.

## Decision

**Manual toggle as the primary UX; optional `autoDetectInversion` flag available as opt-in.**

When either is set, run an inversion preprocessing step in `ConversionService` before tracing:

```
ensureAlpha() → flatten({ background: white }) → negate({ alpha: false }) → greyscale() → threshold(N)
```

## Rationale, the UX choice

I chose manual toggle as the default because as the primary user of the product, I wanted explicit control. I know which of my inputs are white-on-dark, I created them. Auto-detection would have added cognitive load *and* a failure mode: if it silently mis-inverted a high-key photo, the output would be wrong with no visible cause. Manual toggle fails predictably: if the output is empty, flip the toggle.

The optional auto-detection flag exists for batch flows where the user may not have pre-sorted their files. That path is heuristic (luminance analysis) and documented as such.

This is a product decision, not just an architectural one. Shipping the simpler, more predictable UX first I evaluated as the smallest correct move for v1. If usage patterns show that most users hit the same toggle repeatedly, auto-detection becomes the next iteration, and the infrastructure for it is already in place.

## Rationale, the pipeline order

Each step sits where it sits because of a specific failure mode I hit while debugging:

1. **`flatten()` before `negate()`**: transparent PNGs must be composited onto a white background first. If negated while transparent, the alpha channel gets treated as colour information and the background inverts to black. Potrace then traces the entire canvas as a filled rectangle.
2. **`negate({ alpha: false })`**: invert RGB channels only, not alpha. Inverting alpha turns transparent regions opaque and vice versa, producing nonsense output.
3. **`greyscale()` before `threshold()`**: Sharp's `.threshold()` on a multi-channel RGB image applies the cutoff per-channel independently, producing mixed pixels like `(0, 0, 255)` with luminance values that Potrace evaluates incorrectly. Converting to greyscale first ensures a single-channel input where threshold behaves as expected.

The Potrace threshold is passed as an integer in the 0–255 range (`Math.round(options.threshold * 255)`), not as a float, another failure mode found by tracing unexpected output and working backwards.

These three root causes were discovered sequentially, each from a different visible symptom: empty SVG, fully-filled rectangle, traces of noise instead of the subject. The final sequence is the minimal correct order that avoids all three failures. This ADR exists in large part so that future-me (or an LLM helping me) does not reorder the steps "for clarity" and reintroduce one of the bugs.

## Consequences

**Positive**
- White-on-dark images (icons, chalk art, inverted designs) convert correctly without external pre-processing, which was the dogfooding driver in the first place.
- The pipeline is documented well enough that the *reasoning* survives, not just the code. Future changes to any step can be evaluated against the failure mode that step prevents.

**Negative**
- Inversion is destructive. Colour information in the original image is lost (though monochrome tracing already discards colour, so this is a constrained loss).
- Auto-detection is heuristic and can misfire on intentionally high-key images (very bright photography, overexposed shots). Documented as opt-in for this reason.
- Manual toggle requires the user to know their input. For a first-time user with a mixed batch, this is friction.

## Follow-up

- Revisit auto-detection as default once there's a meaningful sample of real user files to calibrate against. Not before, silent auto-behaviour without calibration is worse than explicit UX.
- If a future version supports multi-colour tracing (ADR 0013 deferred this), inversion semantics will need rethinking: *"invert"* is a monochrome concept. The feature may either become obsolete or split into per-layer controls.

**Code:** `src/main/services/ConversionService.ts`, `runSharpPreprocessPipeline()`, `detectDarkBackgroundBuffer()`
