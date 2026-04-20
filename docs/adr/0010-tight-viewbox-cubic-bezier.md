---
layout: default
title: "ADR 0010: Tight SVG ViewBox via Cubic Bezier Extrema"
---

# ADR 0010: Tight SVG viewBox via cubic Bezier extrema

## Context

Potrace outputs SVG paths in pixel coordinates relative to the source image. Two problems arise with a naive viewBox:

1. **Over-estimation:** Using control-point bounding boxes over-estimates the true path bounds, because cubic Bezier control points can lie outside the curve itself. This leaves excess whitespace around the exported SVG.
2. **Editor compatibility:** Figma, Illustrator, and Sketch have known rendering issues with non-zero viewBox origins (`viewBox="131 96 400 300"` instead of `viewBox="0 0 400 300"`). These editors sometimes render path coordinates at face value rather than applying the viewport offset, misplacing the artwork.

## Decision

Compute a **tight bounding box** by solving for extrema of each cubic Bezier segment — the `t` values where the derivative equals zero. Set the viewBox to `"0 0 W H"` (zero origin always) and offset the path content with `<g transform="translate(-x, -y)">`.

```typescript
// ConversionService.ts
function cubicExtrema(p0: number, p1: number, p2: number, p3: number): number[] {
  const a = -p0 + 3 * p1 - 3 * p2 + p3;
  const b = 2 * (p0 - 2 * p1 + p2);
  const c = p1 - p0;
  // Solve at² + bt + c = 0 for t ∈ (0, 1) using the quadratic formula
}

// Result applied in pathBounds():
const newTag = `<svg viewBox="0 0 ${w} ${h}" width="${w}" height="${h}">`;
const newContent = `<g transform="translate(${-x}, ${-y})">${innerContent}</g>`;
```

## Rationale

Control-point bounding boxes are `O(n)` to compute but produce imprecise results. Solving for extrema is slightly more complex but gives pixel-accurate bounds. The zero-origin viewBox + translate pattern is the only approach that renders correctly across all major SVG editors tested (Figma, Illustrator, Sketch, Inkscape).

## Consequences

- **Positive:** Pixel-accurate bounding box with no excess whitespace. Compatible with all major SVG editors.
- **Negative:** The `translate` wrapper must not be stripped by SVG sanitizers. DOMPurify is configured to allow `transform` attributes on `<g>` elements. The algorithm adds ~40 lines of geometry code that must be maintained alongside any future path format changes.

**Code:** `src/main/services/ConversionService.ts` — `pathBounds()`, `cubicExtrema()`
