---
layout: default
title: "ADR 0011: Dual Independent View State"
---

# ADR 0011: Dual independent view state (SVG vs Original)

## Context

The preview area can display either the converted SVG or the original PNG. The pain came up during implementation testing, not from a spec. The first version of the preview shared a single zoom/pan state between the SVG and Original views. The moment I tried to compare a detail, zoom into a curve in the SVG, then flip to Original to check whether the source PNG had a sharp edge in the same place, the Original view jumped to the same zoom level, but the position was misaligned with the detail I had been looking at. Pan to fix it, flip back, the SVG view had moved too. The two views were behaving as one viewport with two contents, when conceptually they are two viewports of two different things. Compare-at-detail workflows were unusable while the state was shared.

## Options considered

- **Shared single viewport (the broken initial state).** What produced the pain. One `ViewState`, both views read and write it. Conceptually clean if the two views were the same content at different rendering, but they aren't.
- **Independent viewports per view (chosen).** Two `ViewState` objects, every action checks `showingOriginal` and writes to the relevant one.
- **Synced zoom level, independent pan.** Zoom matches across views; pan is per-view. Rejected: the typical compare-at-detail workflow involves zooming to the *same detail* in both views, and same-zoom-without-same-position doesn't match that intent, the user still pans independently to find the detail. Added complexity for marginal gain.
- **Per-file × per-view state.** Keyed by `(filePath, view)`. Preserves zoom even on file switch. Rejected as scope creep, the pain that drove this ADR was view-toggle within one file, not file-switch. Held as future enhancement (see Follow-up).

## Decision

Maintain two independent `ViewState` objects in `usePreviewStore`: `svgViewState` and `originalViewState`. Every zoom and pan action checks the `showingOriginal` flag and updates only the relevant state object.

```typescript
// usePreviewStore.ts
svgViewState: ViewState;      // zoom, pan, fitZoom, hasUserZoomed for SVG view
originalViewState: ViewState; // same fields for Original PNG view

setCurrentZoom: (zoom) => set((state) => {
  if (state.showingOriginal) {
    return { originalViewState: { ...state.originalViewState, zoom } };
  }
  return { svgViewState: { ...state.svgViewState, zoom } };
}),
```

## Rationale

The two preview modes are independent viewports of different content, not two states of a single viewport. Keeping them separate makes the UX intuitive and predictable. The cost is that every zoom/pan method must be view-aware, a discipline enforced by the store's structure.

## What this design does not preserve

The `ViewState` is keyed by view, not by `(filePath, view)`. Switching between files keeps the same SVG-view and Original-view zoom/pan across all files. If the user has zoomed into a detail in `logo.png`'s SVG view, switches to `icon.png`, then back to `logo.png`, the zoom level is still where it was, but it was the position relative to `logo.png` that they were inspecting, and `icon.png` may have inherited a viewport that lands on empty space.

This is deliberate scope-narrowing. The pain that drove this ADR was view-toggle within a single file, not file-switch. Solving the actual pain did not require per-file granularity, and per-file × per-view state has its own UX questions (does zoom reset on file switch, or persist across files? Is "fit-to-frame" per-file or global?) that I wasn't ready to commit to in v1. Held in Follow-up.

## Consequences

**Positive**
- Zoom and pan are preserved when toggling between SVG and Original. Compare-at-detail workflows are smooth.

**Negative**
- **The view-aware-setter discipline is a maintenance point.** Every zoom/pan action must check `showingOriginal` and dispatch to the right object. Forgetting causes a silent failure: state lands in the wrong view, with no error and no easy debug breadcrumb. **Mitigation:** route every setter through a single helper (`updateActiveViewState(state, partial)`) that reads the flag once and dispatches. Instead of repeating the `if/else` across five setters, the discipline becomes *"every action goes through the helper"*, which is enforceable by making the bare per-view setters non-exported, the type system then catches violations at compile time.
- File-switch does not reset or remap viewport state. See *"What this design does not preserve"*, accepted scope trade-off.

## Follow-up

- **Per-file × per-view state** is the natural next iteration. It would store `(filePath, view) → ViewState`, with explicit answers to the secondary questions (zoom-on-switch policy, fit-to-frame scope). Deferred until file-switch viewport behaviour becomes a real complaint, not a theoretical one.
- The `updateActiveViewState` helper, if introduced, should be the only place that reads `showingOriginal` for routing. Any other call-site that reaches into `svgViewState` or `originalViewState` directly is a signal to refactor.

**Code:** `src/renderer/store/usePreviewStore.ts`
