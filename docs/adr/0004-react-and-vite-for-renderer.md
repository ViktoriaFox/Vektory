---
layout: default
title: "ADR 0004: React and Vite for the Renderer"
---

# ADR 0004: React and Vite for the renderer UI

> **Decision:** React + Vite for the renderer. **Why:** Vite's HMR makes visual iteration sub-second; React's declarative model was the only practical escape from the imperative state glue that was breaking the Vanilla JS prototype.

## Context

The renderer needs a reactive UI: file list, live conversion preview, parameter controls (sliders, toggles), and an SVG code editor. Two distinct pain points shaped the final stack.

**Pain point 1, the feedback loop.** A lot of the work on this app is visual tuning: gradients, spacing, the exact angle of a glass-border effect, how a golden accent reads next to purple. The inner loop for that work is "tweak → see result." If that loop requires a full page reload on every change, it stops being a loop and starts being a tax. Early on, without HMR, it genuinely hurt, I would change one CSS value, reload, add image, find the value was wrong, change again, reload again. Minutes per micro-adjustment on work that should be sub-second.

**Pain point 2, state.** The app started as **Vanilla JS**. That was enough to prototype the Electron boundary and the conversion flow. What it was not enough for was the real UI: per-file options, persisted settings, zoom/pan state across two views, an SVG cache keyed on options. Handling that amount of coupled state imperatively, writing updates for each DOM node, tracking which subscribers needed to re-read which slice, became the bottleneck. It was clear I needed a declarative component model and a proper state layer.

## Options considered

**Build tooling:** Webpack vs Vite. Webpack would have solved the feedback-loop problem, HMR has existed there for years. The cost is a heavier config, slower dev server startup, and more machinery to reason about. For a single-window desktop app bundler, that complexity is unearned.

**UI framework:** stay on Vanilla JS vs rewrite in React. Staying would have meant building more of the coupling glue by hand, forever. React's component model, hooks, and reactive re-rendering map directly to the kind of UI this app is, many interdependent controls driving a single preview.

Radix UI, which was the primitives library I already wanted for accessibility and dark-theme control (ADR 0005), is React-only. That narrowed the choice.

## Decision

- **Vite** for the renderer build and dev server. HMR in development, standard ESM build in production. Minimal config.
- **React** for the UI. Rewritten from the Vanilla-JS prototype once the state model outgrew imperative handling.
- **Main process stays on `tsc`** (ADR 0001). No bundling there, it's Node code that Electron loads directly.

State management (Zustand) is covered in its own ADR (0006). The initial pick was fast and unexamined; ADR 0006 documents the deliberate decision to stay after aborting a Redux migration attempt.

## Rationale

**Vite for UI work, specifically.** The UX of this app is tuned visually, not just logically. Gradients, opacity, border-radius, font weight at specific sizes, none of these are specced in advance, they are found by iteration. Vite's HMR keeps my hands on the tweak surface and my eyes on the result, same second. That is the dev-loop equivalent of the product decision in ADR 0012 (manual toggle with auto-detect opt-in): keep the explicit, fast path for the thing you do constantly, and don't let tooling friction accumulate. Webpack would have worked; the config overhead is real and would have slowed me down on every project-level change.

**React over Vanilla JS, forced by scale.** I prefer small stacks. I would happily have stayed on Vanilla if the app's state had stayed simple. It did not. The rewrite was a response to pain I felt in the Vanilla version, hand-rolled subscription code, subtle update ordering bugs, long imperative chains to sync the UI with store changes. React replaces all of that with one model: state changes, components re-render, diffing handles the rest. At the scale of this app, that model is a net simplification, not a ceremony.

## Consequences

**Positive**
- The visual feedback loop is instant during development. UI polish actually happens instead of being deferred.
- The state model is declarative.
- The ecosystem fit (React → Radix → Zustand → hooks) keeps the renderer consistent.
**Negative**
- Two build pipelines (main via `tsc`, renderer via Vite) mean dev orchestration: `concurrently` starts both watchers, `wait-on` holds Electron until the renderer is ready. Documented in the dev script, but it is a moving part.
- The Vanilla-JS rewrite cost a meaningful chunk of time.

## Follow-up

- Main-process HMR via vite-plugin-electron or similar: viable if main-process iteration ever becomes frequent. Currently it does not.
- The two-pipeline structure should survive a future runtime change (e.g. Tauri), the build boundary is orthogonal to the Electron decision.

**Code:** `vite.config.ts` · `tsconfig.json` · `package.json` (dev scripts)
