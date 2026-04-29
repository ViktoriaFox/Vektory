---
layout: default
title: "ADR 0010: Tight SVG ViewBox via Cubic Bezier Extrema"
---

# ADR 0010: Tight SVG viewBox via cubic Bezier extrema

## Context

I used Vektory to vectorise my own Vektory logo for the app icon and this website. The recursion was satisfying; the result was not. In the Windows taskbar and macOS Dock, my icon looked visibly smaller than the surrounding icons, as if the system had given me a smaller frame. The icons were all rendering into the same fixed frame; mine just had extra empty space *inside* its SVG viewBox, which the OS dutifully scaled along with the artwork.

I had actually seen the same symptom earlier, while helping a friend with some design work, and filed it under "fix when it matters." It mattered now, the logo of a vector-graphics app shipping at the wrong size in the taskbar is the kind of detail that undermines the entire pitch.

Once I understood what I was looking at, two problems underlay the same symptom:

1. **Control-point bounding boxes over-estimate the true path bounds.** Potrace outputs cubic Bezier paths. A naive bounding box over the control points is easy (min/max of all `(x, y)` pairs) but imprecise, control points can sit *outside* the curve itself. The resulting viewBox contains whitespace the artwork never occupies.
2. **Non-zero viewBox origins break popular SVG editors.** A tight viewBox like `viewBox="131 96 400 300"` is mathematically correct, but Figma, Illustrator, and Sketch have well-known edge cases where they render path coordinates at face value and ignore the viewport offset, misplacing or clipping the artwork. Ship a file with a non-zero origin and a designer opening it in Figma sees a broken SVG.

## What the LLM got wrong

This fix is worth calling out because it was one of the cases where LLM assistance stalled early. The model could not diagnose the symptom from my description. I had to investigate the geometry myself to understand that the problem was excess whitespace inside a too-loose viewBox.

Once I had framed the problem, the LLM's proposed fixes each created side effects, either simplistic bounding boxes that still over-counted, or viewBox manipulations that broke in SVG editors. The working solution came from independent research into cubic Bezier geometry: the analytical way to find the true extrema of a cubic segment is to solve for the points where the derivative equals zero, which reduces to a quadratic equation over the parameter `t ∈ (0, 1)`.

Naming this is the point. LLMs are fast collaborators on code I can specify precisely; they are not a substitute for understanding the problem. When the symptom is visual and the cause is mathematical, the architect still has to do the diagnosis.

## Decision

Compute a **tight bounding box** by solving for cubic Bezier extrema, the `t` values in `(0, 1)` where each segment's derivative equals zero, and evaluating the curve at those `t` values plus the endpoints. Set the viewBox to `"0 0 W H"` (zero origin, always) and offset the path content with `<g transform="translate(-x, -y)">`.

```typescript
// ConversionService.ts
function cubicExtrema(p0: number, p1: number, p2: number, p3: number): number[] {
  const a = -p0 + 3 * p1 - 3 * p2 + p3;
  const b = 2 * (p0 - 2 * p1 + p2);
  const c = p1 - p0;
  // Solve at² + bt + c = 0 for t ∈ (0, 1) using the quadratic formula
}

const newTag = `<svg viewBox="0 0 ${w} ${h}" width="${w}" height="${h}">`;
const newContent = `<g transform="translate(${-x}, ${-y})">${innerContent}</g>`;
```

## Rationale

**The naive bounding box, and the irony of an already-installed library.** The trivial approach is to take every point of every segment (including the two control points of each cubic) and compute min/max in each axis. `O(n)`, two lines of code. That is also exactly what `svg-path-bounds`, which was *already in `package.json`* as a transitive dependency, does in its core. Its entire algorithm is a loop that tracks min/max across all points. The result is the convex hull of the control polygon, which is always greater than or equal to the real curve bounds, never smaller. Cubic Bezier control points almost never lie on the curve; a segment is guaranteed to sit *inside* the hull of its four defining points but usually nowhere near the two inner ones. So convex hull leaves real whitespace, exactly what a tight viewBox is supposed to eliminate.

The library's sub-dependencies (`parse-svg-path`, `abs-svg-path`, `normalize-svg-path`) are useful on their own for parsing Potrace output into a normalised segment list, and they *are* imported directly. The top-level `svg-path-bounds` is not, because its core is the wrong algorithm for this problem. Reusing the preprocessing while replacing the core is the pragmatic move: no second parser, no duplicated normalisation code, just a correct algorithm on top of an existing pipeline.

**Analytical, not numerical sampling.** The obvious replacement for convex hull is to sample `B(t)` at many values of `t` per segment (say 100) and track min/max. It works, and loses on every axis:

|  | Sampling (N=100) | Analytical (discriminant) |
|---|---|---|
| Accuracy | misses extrema between samples, up to ~1% of width | exact to float precision |
| Cost per segment | 100 curve evaluations | one discriminant check + ≤2 evaluations |
| Code | loop + tunable N, with edge cases on tight curvature | ~15 lines, school-book discriminant |

So... *The derivative of a cubic polynomial is quadratic.* A quadratic has a closed-form root formula. No Newton iteration, no bisection, no sampling grid, just the discriminant. If SVG paths ever included quintic (degree-5) Beziers, closed form would disappear and numerical methods would be the only option. SVG uses cubics and quadratics; both have closed forms.

**Zero-origin viewBox + `<g transform="translate(...)">`.** This half of the decision is for compatibility. Non-zero viewBox origins (`viewBox="131 96 400 300"`) are spec-correct, but Figma, Illustrator, and Sketch all have well-known rendering bugs where they ignore the viewport offset and draw path coordinates at face value, misplacing or clipping the artwork. Every editor I tested handles `viewBox="0 0 W H"` with a translated content group correctly.

## Consequences

**Positive**
- The exported SVG sits tight against its content. When the file is used as an app icon, a favicon, or embedded in a layout, it fills the allocated frame instead of floating inside extra whitespace.
- Compatibility with every mainstream SVG editor holds because the viewBox always starts at `0 0`.

**Negative**
- Around 40 lines of geometry code must be maintained alongside any future change to path output format. If Potrace ever emits non-cubic segments (quadratic, arc), the extrema solver needs corresponding branches.

## Follow-up

- If the path format ever gains elliptical arcs or quadratic Beziers, extend `cubicExtrema` with analogous solvers rather than falling back to control-point bounds.
- The DOMPurify allowlist for `transform` is an implicit dependency of this decision (ADR 0002). Any sanitiser-config change should cross-check this ADR.

**Code:** `src/main/services/ConversionService.ts`, `pathBounds()`, `cubicExtrema()`

## Appendix, cubic Bezier extrema via the discriminant

For readers who want the derivation behind `cubicExtrema`'s coefficients.

A cubic Bezier segment on one coordinate is:

```
B(t) = (1−t)³·p₀ + 3t(1−t)²·p₁ + 3t²(1−t)·p₂ + t³·p₃,   t ∈ [0,1]
```

Expanding and grouping by powers of `t` gives a cubic polynomial. Its derivative `B'(t)` is a quadratic polynomial, always, because differentiation drops the degree by one. Setting `B'(t) = 0` and factoring out the shared 3 yields:

```
a·t² + b·t + c = 0
  a = −p₀ + 3p₁ − 3p₂ + p₃
  b = 2·(p₀ − 2p₁ + p₂)
  c = p₁ − p₀
```

These are exactly the coefficients in `cubicExtrema`. Roots come from the standard discriminant formula. `a ≈ 0` is the degenerate case (the derivative becomes linear, one root from `b·t + c = 0`).

Roots are filtered to `t ∈ (0, 1)` with strict inequalities, the endpoints `t = 0` and `t = 1` correspond to the segment's start and end points, which are added to the bounding box separately in the main sweep over `M`/`C` segments. The extrema check is for interior maxima and minima only; roots outside `(0, 1)` would be extrema of the curve's mathematical extension beyond the segment, not part of the SVG path.
