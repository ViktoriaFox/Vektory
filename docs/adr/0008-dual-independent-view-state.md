---
layout: default
title: "ADR 0008: Dual Independent View State"
---

# ADR 0008: Dual independent view state (SVG vs Original)

## Context

The preview area can display either the converted SVG or the original PNG. If zoom and pan state were shared, toggling between views would reset the user's viewport — for example, zooming into a detail in the SVG view and switching to Original to compare would lose the zoom level, forcing the user to find the same spot again.

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

The two preview modes are independent viewports of different content, not two states of a single viewport. Keeping them separate makes the UX intuitive and predictable. The cost is that every zoom/pan method must be view-aware — a discipline enforced by the store's structure.

## Consequences

- **Positive:** Zoom and pan are preserved when toggling between SVG and Original. Compare-at-detail workflows are smooth.
- **Negative:** Every future zoom/pan operation must follow the same view-check pattern. Forgetting to do so causes state to update the wrong view.

**Code:** `src/renderer/store/usePreviewStore.ts`
