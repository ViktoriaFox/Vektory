---
layout: default
title: Architecture Decision Records
nav_title: ADRs
---

# Architecture Decision Records

These records document the key design decisions made during development of Vektory. Each ADR captures the context, the decision, and its consequences, including trade-offs consciously accepted.

The numbering reflects the order in which the decisions were made. Reading them in order tracks the architectural narrative, runtime first, then security, then the libraries that justified the runtime, then UI, then state and algorithms, with scope-freeze and the last preprocessing fix at the end. ADR 0014 documents the testing strategy that runs through all of them.

| # | Title | Decision |
|---|-------|----------|
| [0001](0001-electron-and-typescript) | Electron and TypeScript | Single codebase (Win/macOS) using Electron; TypeScript throughout for shared types between main and renderer |
| [0002](0002-context-isolation-and-preload-api) | Context isolation and preload-only API | Renderer has zero direct Node or IPC access; all calls go through a typed `contextBridge` surface (`window.electronAPI`); four-layer defense-in-depth |
| [0003](0003-potrace-and-sharp) | Potrace and Sharp for the conversion pipeline | Potrace for monochrome tracing, Sharp for image preprocessing; closes the ecosystem-bet from ADR 0001 |
| [0004](0004-react-and-vite-for-renderer) | React and Vite for the renderer | React for the UI (forced by state-model scale, not aesthetics); Vite for the visual feedback loop; ecosystem compounding with Radix and Zustand |
| [0005](0005-radix-over-mui-and-headless-ui) | Radix over MUI and Headless UI | Unstyled primitives for full design control; per-primitive packages for bundle discipline; `UI_CONSTRAINTS.md` carries the design system into LLM prompts |
| [0006](0006-zustand-over-redux) | Zustand over Redux, sunk-cost-aware trade-off | Initial pick was unexamined; Redux migration attempted, reverted; staying on Zustand became the deliberate decision |
| [0007](0007-per-file-options-selective-persistence) | Per-file options with selective persistence | Dual-layer options (global + per-file overrides); `partialize` allowlist persists only `themeMode` and `showEditor` |
| [0008](0008-options-based-svg-cache) | Options-based SVG cache invalidation | Cache SVG + options per file; `JSON.stringify` equality avoids re-tracing on file switch |
| [0009](0009-batch-concurrency-control) | Batch concurrency control | Groups-of-5 via `Promise.all`; "survive the accident, not optimise it" for thousand-file edge case |
| [0010](0010-tight-viewbox-cubic-bezier) | Tight SVG viewBox via cubic Bezier extrema | Solve derivative = 0 (quadratic, closed-form) for tight bounds; zero-origin viewBox + translate for editor compatibility |
| [0011](0011-dual-independent-view-state) | Dual independent view state | Separate `ViewState` for SVG and Original views; zoom/pan preserved when toggling between them |
| [0012](0012-dark-background-inversion) | Dark background inversion | Preprocessing pipeline (flatten → negate → greyscale → threshold) for white-on-dark inputs; manual toggle as primary UX |
| [0013](0013-ship-monochrome-tracing-first) | Ship monochrome tracing first | Defer multi-colour / palette vectorisation; ship focused single-colour conversion with clear Known Limitations |
| [0014](0014-testing-strategy) | Risk-based testing strategy | Vitest for the deterministic core (image pipeline, security boundary, file safety); manual validation for visual UX; tests as regression contracts on architectural decisions |
