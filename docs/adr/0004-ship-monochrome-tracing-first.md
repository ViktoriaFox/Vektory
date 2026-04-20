---
layout: default
title: "ADR 0004: Ship Monochrome Tracing First"
---

# ADR 0004: Ship monochrome tracing first before multi-color tracing

## Context

I want to support multi-color / palette-based vectorization eventually. The core algorithm (Potrace) and current UI are built around a single threshold and single fill color. Adding multi-color tracing would require palette detection, multiple passes or regions, and a different settings model.

## Decision

Ship and iterate on **monochrome tracing first**. The first release focuses on single fill color, threshold-based conversion with the existing parameters (threshold, corner policy, artifact area, smoothness, path optimization). Multi-color tracing is explicitly deferred to a later version.

## Consequences

- **Positive:** Faster to ship a focused, well-documented product. Monochrome covers logos, icons, silhouettes, and high-contrast art — the main use cases. Parameter guide and presets stay simple. I can validate the app and gather feedback before adding complexity.
- **Negative:** Users who need multi-color output must use other tools or wait for a future release. This is documented in Known Limitations and Roadmap.
- **Follow-up:** When I add multi-color tracing, I will need a new ADR for the approach (palette extraction, layer/region model, UI changes).
