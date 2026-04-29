---
layout: default
title: "ADR 0001: Electron and TypeScript"
---

# ADR 0001: Electron and TypeScript for the desktop app

> **Decision:** Use Electron + TypeScript end-to-end. **Why:** The Node ecosystem (Sharp, Potrace) makes Electron the only practical fit; TypeScript turns the IPC boundary into a compile-time contract.

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

**Rewriting the pipeline wasn't in scope for v1.**
A meaningful part of the conversion pipeline was already written in TypeScript when I evaluated Tauri. Switching would have meant a full Rust rewrite of that layer. The performance and bundle-size gains are real, but the scope of that rewrite wasn't justified against a v1 portfolio timeline.

**Cross-platform with one codebase.**
Electron ships one build target per OS from one source tree. For a solo portfolio project, that's not optional, I can't maintain three native codebases. The bundle-size / memory cost of Electron is real, but for a tool I use myself on a workstation, I accepted it as a tradeoff.

### TypeScript

**The Electron main ↔ renderer boundary is the sharpest reason.**
In Electron, renderer code runs in a sandbox and talks to main via IPC. Shapes pass across this boundary as serialized JSON, there's no runtime type enforcement. Untyped code fails silently: a shape mismatch doesn't throw, it delivers wrong data and the UI renders incorrectly. This is exactly the scenario TypeScript catches at compile time.

I centralized the boundary contract in a single `types.ts`, `ConversionOptions`, `ConversionResult`, file metadata, batch results. Main and renderer both import from it. When I change the shape of `ConversionOptions`, the compiler shows me every place the contract is violated, in both processes. This is the architectural win, not "TypeScript is better than JavaScript" in general, but *"TypeScript makes the Electron IPC boundary a compile-time contract, and that's where the most silent bugs live"*.

**Prior fluency, and reviewability under LLM assistance.**
I optimized for *ship in weeks* over *ideal runtime*. TypeScript was a language I already worked in professionally, so the learning curve went into the architecture — process boundary, state, pipeline — not into picking up a new language.

The less obvious reason is **reviewability under LLM assistance**. Much of the implementation was drafted with LLM help. That workflow only holds up if I can actually critique what comes back: IPC contracts, type narrowings, subtle async bugs. In TypeScript I catch those; in Rust today I couldn't. An unfamiliar language collapses the review loop — LLM drafts, but I can't verify or redirect with confidence. That trade-off wasn't worth it.

## Consequences

**Positive**
- Single cross-platform codebase.
- Full Node ecosystem available in the main process → Sharp and Potrace drop in.
- IPC boundary is a compile-time contract via `types.ts`.
- Fast iteration: React + Vite in the renderer, `tsc` for main (see [ADR 0004](0004-react-and-vite-for-renderer)).

**Negative**
- Bundle size is large (~200 MB) and memory footprint is heavier than a native or Tauri app.
- Startup time is noticeable vs native.
- Dependency on Chromium/Node release cycle for security updates.

I mitigate these by keeping the main process narrow (file + conversion only, no UI state there) and the renderer on a lean modern stack (Vite, React, Radix primitives only).

**Explicit trade-off I chose to accept**
- I could have picked a more performant runtime (Tauri) and invested the time into learning Rust and rebuilding the conversion pipeline. That would have been a different project, a performance exercise rather than an architecture exercise. For v1 portfolio goals, I chose the architecture exercise.

