---
layout: default
title: "ADR 0013: Ship Monochrome Tracing First"
---

# ADR 0013: Ship monochrome tracing first before multi-color tracing

## Context

I want to support multi-colour / palette-based vectorisation eventually. The core algorithm (Potrace) and the current UI are built around a single threshold and a single fill colour. Adding multi-colour tracing would require palette detection, multiple passes or regions, and a different settings model.

The question this ADR answers is not *"is monochrome enough forever?"*, it isn't. The question is *"what should v1 ship?"*, and that is a scope decision, made under a release deadline, informed by what I had actually been using the app for and what I could validate before publishing.

## Options considered

- **Ship monochrome only (chosen).** Single fill colour, threshold-based conversion. Potrace as the tracer (ADR 0003), no palette detection, no multi-pass routing, no per-layer UI. Smallest correct surface for v1.
- **Ship multi-colour from day 1 via vtracer.** Replace Potrace with vtracer up front so colour and monochrome are both supported from release. Rejected, vtracer's monochrome trace quality is visibly worse than Potrace, and pulling in colour machinery, routing logic, and a heavier Rust-native dependency for a feature I had not validated end-to-end (palette UX, per-layer controls, batch behaviour with mixed colour/mono inputs) was scope and risk I would not have closed by the release date. The full reasoning is in [ADR 0003](0003-potrace-and-sharp).
- **Monochrome + colour quantisation via multiple Potrace passes.** Reduce the source image to N quantised colours (e.g. via Sharp's palette mode or k-means), run Potrace once per palette colour against a binary mask, layer the resulting paths in a single SVG. Rejected, it is a real approach and I considered it seriously, but it has its own design problems: how does the user pick N? How are the layers ordered? What does the threshold control do per layer? Each of these is a UX decision that needed real product input, not a v1-deadline guess.
- **Monochrome + manual N-colour palette.** Variant of the above where the user explicitly picks colours instead of automatic quantisation. Same UX questions, more user-facing complexity, no clearer answer.

## Decision

Ship and iterate on **monochrome tracing first**. The first release focuses on single fill colour, threshold-based conversion with the existing parameters (threshold, corner policy, artifact area, smoothness, path optimisation). Multi-colour tracing is explicitly deferred to a later version, and the implementation path for it is the natural next step from [ADR 0003](0003-potrace-and-sharp): introduce vtracer alongside Potrace, route per-image based on a colour-mode flag, share the Sharp preprocessing layer.

## Rationale

**Why monochrome covers the use cases that mattered for v1.**
This is grounded in dogfooding, not market research. The work I had been doing with Vektory while building it, logos for my own project, icons for a friend's design work, the assets for this very docs site (the Vektory logo in the taskbar, see ADR 0010), was almost entirely monochrome: logos and icons are typically one or two colours, designed flat, with colour applied downstream as a CSS or SVG fill attribute. That is the real workflow vector tools serve: take a raster mark, get a clean path, colour it later. Full-colour illustration vectorisation is a different category of need, served by different tools, and not where I or the people I had been making the app for were working.


## Consequences

**Positive**
- Faster to ship a focused, well-documented product. The full surface is testable against real inputs by a single person before release.
- Parameter guide and presets stay simple, every parameter applies the same way, with no per-layer or per-colour fork.
- The validated v1 baseline gives me a reference point for evaluating colour features when they're added (does the colour path produce monochrome output that matches v1 quality?).

**Negative**
- Users who need multi-colour output must use other tools or wait for a future release. Documented in Known Limitations and Roadmap, not hidden.
- "Monochrome only" frames Vektory as a narrower tool than it could be on day 1. For a portfolio piece this is a real trade-off, some readers will see it as scope-limited rather than scope-disciplined. The ADR exists in part to make the difference legible.

## Follow-up

- When multi-colour tracing is added, **vtracer is the natural next step** ([ADR 0003](0003-potrace-and-sharp) anticipated this). The pipeline structure stays: Sharp preprocessing remains shared, Potrace and vtracer become two tracers selected per image by a colour-mode flag.
- The open UX questions that blocked colour from v1, palette size selection, layer ordering, threshold semantics per layer, batch behaviour with mixed inputs, should be designed against real user files, not specced upfront. Adding colour is a product decision, not a library swap.
- ADR 0012 (dark background inversion) explicitly notes that "invert" is a monochrome concept and may need to be rethought when colour enters scope. That cross-cuts this ADR and should be revisited together.
