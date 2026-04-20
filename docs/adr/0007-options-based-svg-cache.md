---
layout: default
title: "ADR 0007: Options-Based SVG Cache Invalidation"
---

# ADR 0007: Options-based SVG cache invalidation

## Context

Converting a PNG with Sharp + Potrace is CPU-bound and takes 100–500 ms per file depending on size and complexity. When a user switches between files in a batch, re-running the full conversion even when nothing changed was wasteful and made the UI feel slow.

## Decision

Cache each file's SVG alongside the exact `ConversionOptions` used to generate it. On file switch, compare the cached options against the current options using `JSON.stringify` equality. Only trigger re-conversion when the cache is empty (`null`) or the options have changed.

```typescript
// useAppStore.ts
svgCache: Record<string, { options: ConversionOptions; svg: string }>;

// on file switch:
const cached = svgCache[file.path];
const optsMatch = cached && JSON.stringify(cached.options) === JSON.stringify(fileOptions);
const currentSvg = optsMatch ? cached.svg : null;

// useApp.ts — only convert if currentSvg is null:
if (currentFile && currentSvg === null) triggerConversion();
```

## Rationale

`JSON.stringify` comparison is simple and deterministic for flat option objects constructed from the same shape. More sophisticated strategies (structural diffing, hashing) would add complexity for no practical benefit at this scale. The cache is intentionally not persisted — it is rebuilt as needed within a session.

## Consequences

- **Positive:** Switching between previously converted files is instant with no Potrace call.
- **Negative:** Object key order must be consistent for `JSON.stringify` equality to hold. Options objects are always constructed from the same shape, making this safe in practice. If options shape changes, cache entries silently become stale and trigger a re-conversion (correct behaviour).

**Code:** `src/renderer/store/useAppStore.ts` · `src/renderer/hooks/useApp.ts`
