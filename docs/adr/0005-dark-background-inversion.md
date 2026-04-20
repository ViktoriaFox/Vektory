---
layout: default
title: "ADR 0005: Dark Background Inversion"
---

# ADR 0005: Dark background inversion preprocessing

## Context

Potrace traces dark pixels on a light background. Source images that are light-on-dark (white icons on a black canvas, white text on a dark photo) produce empty or inverted output by default — Potrace finds nothing to trace because the foreground pixels are lighter than the background.

I needed a way to handle these images without requiring users to pre-process them externally.

## Decision

Add an optional preprocessing step in `ConversionService` that inverts the image before tracing, using a specific Sharp pipeline order:

```
ensureAlpha() → flatten({ background: white }) → negate({ alpha: false }) → greyscale() → threshold(N)
```

The order is not arbitrary — three root causes required this exact sequence:

1. **`flatten()` before `negate()`** — transparent PNGs must be composited onto white first. If negated while transparent, the alpha channel inverts the background to black, which Potrace then traces as a filled rectangle.
2. **`negate({ alpha: false })`** — invert RGB channels only, not the alpha channel.
3. **`greyscale()` before `threshold()`** — Sharp's `.threshold()` on a multi-channel RGB image applies the cutoff per-channel independently, producing mixed pixels (e.g. `0, 0, 255`) with unexpected luminance values that Potrace evaluates incorrectly. Converting to greyscale first ensures a single-channel input.

The Potrace threshold is passed as an integer in the 0–255 range (`Math.round(options.threshold * 255)`), not as a float.

An `autoDetectInversion` flag is also provided, which analyses average luminance of the source image to detect likely dark-background inputs automatically.

## Rationale

Each constraint in the pipeline was discovered through debugging failures: empty SVG output, fully-filled rectangles, and traces of noise instead of the intended subject. The final sequence is the minimal correct order that avoids all three failure modes.

## Consequences

- **Positive:** White-on-dark images (icons, chalk art, inverted designs) convert correctly without external pre-processing.
- **Negative:** Inversion is a destructive step — colour information in the original image is not preserved (though monochrome tracing already discards colour). Auto-detection is heuristic and can misfire on intentionally high-key images.

**Code:** `src/main/services/ConversionService.ts` — `runSharpPreprocessPipeline()`, `detectDarkBackgroundBuffer()`
