---
layout: default
title: "ADR 0001: Electron and TypeScript"
---

# ADR 0001: Electron and TypeScript for the desktop app

## Context

I needed Vektory to be a **desktop application**, cross-platform (**hard requirement**), offline-first, with direct file system access, and capable of running CPU-bound image processing (vectorization, raster operations) without round-tripping through a server.
The MVP target was Windows and macOS; Linux was a nice-to-have. A real-time preview loop, batch export, and a responsive UI were non-negotiable.

## Options considered

**Electron**
Chromium + Node in one bundle. JavaScript/TypeScript end-to-end. Cross-platform out of the box. Full Node ecosystem available in the main process, critical for what I needed (see Rationale).

**Tauri**
Lighter, more performant, native webview on each OS + Rust backend. Cross-platform. But: the image processing pipeline I needed (Potrace for vectorization, Sharp for raster preprocessing) lives in the **Node ecosystem**. In Tauri, the frontend can stay JS/TS, but the CPU-bound conversion layer would have to be rewritten in Rust crates, I'd be maintaining a Rust conversion pipeline I didn't want to spend v1 becoming expert enough in Rust to build well.

**PWA**
Ruled out for workflow reasons. The browser sandbox restricts file system access, especially for batch processing across directories with output paths chosen by the user. Research showed that Vektory's core flow, *"drop a folder, tune settings, export SVGs to disk"*, doesn't work well inside PWA constraints. Offline-first + local-file-heavy + designer-controlled output paths wants a native shell.

## Decision

**Electron** for the desktop shell. **TypeScript** end-to-end.

## Rationale

### Electron

**The ecosystem decided it, not the runtime itself.**
I evaluated Electron as part of a stack, not in isolation. The stack I needed was: Electron + Node + Sharp + Potrace. Sharp is a time-proven Node wrapper around libvips for raster preprocessing; Potrace has been the reference vectorization algorithm for two decades. Both are Node-native. Picking Tauri would have meant reimplementing the conversion layer in Rust, not because Tauri is worse in general, but because in my stack it breaks the critical dependency. *(The choice of Sharp and Potrace specifically gets its own ADR, ADR 0003, because that is a separate axis of decision from the runtime choice.)*

**Sunk cost was real.**
A meaningful part of the conversion pipeline was already written in TypeScript calling Sharp and Potrace when I was evaluating whether to switch. Switching to Tauri meant porting that to Rust, a full rewrite of the conversion pipeline. Against a v1 portfolio timeline, the cost of that rewrite wasn't justified by the gains (smaller bundle, better performance).

**Cross-platform with one codebase.**
Electron ships one build target per OS from one source tree. For a solo portfolio project, that's not optional, I can't maintain three native codebases. The bundle-size / memory cost of Electron is real, but for a tool I use myself on a workstation, I accepted it as a tradeoff.

### TypeScript

**The Electron main ↔ renderer boundary is the sharpest reason.**
In Electron, renderer code runs in a sandbox and talks to main via IPC. Shapes pass across this boundary as serialized JSON, there's no runtime type enforcement. Untyped code fails silently: a shape mismatch doesn't throw, it delivers wrong data and the UI renders incorrectly. This is exactly the scenario TypeScript catches at compile time.

I centralized the boundary contract in a single `types.ts`, `ConversionOptions`, `ConversionResult`, file metadata, batch results. Main and renderer both import from it. When I change the shape of `ConversionOptions`, the compiler shows me every place the contract is violated, in both processes. This is the architectural win, not "TypeScript is better than JavaScript" in general, but *"TypeScript makes the Electron IPC boundary a compile-time contract, and that's where the most silent bugs live"*.

**Prior fluency, and reviewability under LLM assistance.**
Velocity mattered for a v1 portfolio project. I optimized for *ship in weeks* over *ideal runtime*. TypeScript was a language I already worked in professionally, so the learning curve went into the architecture (process boundary, state, pipeline) rather than into picking up a new language.

The deeper reason, which matters more in 2026 than it would have in 2022, is **reviewability under LLM assistance**. Much of the implementation was drafted with LLM help. That workflow only works as well as my ability to critique what comes back, shape contracts at the IPC boundary, edge cases, type narrowings, subtle async bugs. In TypeScript, which I'm closely fluent in, I catch those. In Rust today, I couldn't, not well enough. Familiar languages keep the human (me) in the review loop. LLM drafts, I verify, I accept or reject. That whole loop collapses if I can't actually read what was drafted.

This is the same principle I hit from the other side in [ADR 0005 on Radix UI](0005-radix-over-mui-and-headless-ui): unopinionated tools move the discipline onto the human. Choosing an unfamiliar language with LLM help shifts an even heavier discipline, *critical review of generated code*, onto a human who isn't equipped to do it. I deliberately avoided that trade-off, at least for this project, for this particular mission.

## Consequences

**Positive**
- Single cross-platform codebase.
- Full Node ecosystem available in the main process → Sharp and Potrace drop in.
- IPC boundary is a compile-time contract via `types.ts`.
- LLM-assisted development stays safe because I genuinely review everything that lands and guide it in the right direction.
- Fast iteration: React + Vite in the renderer, `tsc` for main (see [ADR 0004](0004-react-and-vite-for-renderer)).

**Negative**
- Bundle size is large (~200 MB) and memory footprint is heavier than a native or Tauri app.
- Startup time is noticeable vs native.
- Dependency on Chromium/Node release cycle for security updates.

I mitigate these by keeping the main process narrow (file + conversion only, no UI state there) and the renderer on a lean modern stack (Vite, React, Radix primitives only).

**Explicit trade-off I chose to accept**
- I could have picked a more performant runtime (Tauri) and invested the time into learning Rust and rebuilding the conversion pipeline. That would have been a different project, a performance exercise rather than an architecture exercise. For v1 portfolio goals, I chose the architecture exercise.

## Follow-up

- The renderer-to-main communication surface needs its own security model, see [ADR 0002: Context isolation and preload-only API](0002-context-isolation-and-preload-api).
- Image pipeline library choice (Sharp + Potrace, alternatives evaluated), see [ADR 0003: Potrace and Sharp for the Conversion Pipeline](0003-potrace-and-sharp).
- Renderer build stack is a separate decision, see [ADR 0004: React and Vite for the renderer](0004-react-and-vite-for-renderer).
- If I ever rewrite for a different runtime (Tauri, or something that doesn't yet exist), this ADR should be revisited against the same criteria: cross-platform, ecosystem fit, LLM-assisted reviewability, velocity.
