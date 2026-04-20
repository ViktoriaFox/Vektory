---
layout: default
title: "ADR 0003: React and Vite for the Renderer"
---

# ADR 0003: React and Vite for the renderer UI

## Context

The renderer needs a UI that can show file lists, live preview, parameter controls, and an SVG code editor. I want fast development iteration, hot reload during dev, and a maintainable component structure.

## Decision

- **React** for the renderer UI: components, hooks, and a small amount of local state plus **Zustand** for app-wide state (e.g. conversion options, theme).
- **Vite** for bundling and dev server: fast HMR, modern ESM, and simple config. The main process is built with `tsc` separately; Electron loads the Vite dev server in development and the Vite-built output in production.

## Consequences

- **Positive:** Fast feedback loop (Vite HMR for renderer); React component model fits preview, controls, and panels; Zustand keeps state predictable without heavy boilerplate. Radix UI used for accessible controls (sliders, tabs, tooltips).
- **Negative:** Build setup has two parts (main via tsc, renderer via Vite); dev script must start both and then Electron. I use `concurrently` / `wait-on` for dev orchestration.
- **Follow-up:** Main process changes still require a full restart of the Electron process; only renderer code benefits from HMR.
