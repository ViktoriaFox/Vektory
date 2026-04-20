---
layout: default
title: "ADR 0009: Batch Concurrency Control"
---

# ADR 0009: Batch conversion concurrency model

## Context

Batch export must process multiple files efficiently without overwhelming the system. Three approaches were considered:

- **Fully sequential** — simple, but 20 files means 20 × ~200 ms = 4 seconds of pure wait time with no parallelism benefit.
- **Fully parallel** — fast, but Sharp decoding + Potrace tracing are both CPU and memory intensive. On a 50-file batch with large images, spawning all conversions simultaneously risks memory exhaustion and system slowdown.
- **Bounded parallel groups (groups-of-N)** — process N files concurrently, wait for the group to finish, then start the next group.

## Decision

Use the **groups-of-N** pattern: sequential outer loop over chunks, `Promise.all` within each chunk. The current group size is 5.

```typescript
// ConversionService.ts
const batchSize = 5;
for (let i = 0; i < filePaths.length; i += batchSize) {
  const batch = filePaths.slice(i, i + batchSize);
  const batchResults = await Promise.all(
    batch.map((filePath, localIdx) => {
      const opts = Array.isArray(options) ? (options[i + localIdx] ?? {}) : options;
      return this.convertPngToSvg(filePath, opts as ConversionOptions);
    })
  );
  results.push(...batchResults);
}
```

The method accepts either a single `ConversionOptions` (uniform across files) or `ConversionOptions[]` (per-file settings), so the renderer can pass each file's individually stored options.

## Rationale

Groups-of-N is the right pattern here because the bottleneck is memory, not I/O latency — a queue-based worker pool would add complexity without changing the resource profile. The group size of 5 is a tuning constant derived empirically; it is named (`batchSize`), not inlined, so it can be adjusted without touching logic.

## Consequences

- **Positive:** Bounded memory usage regardless of batch size; meaningful parallelism within each group.
- **Negative:** Groups-of-N has a serialization cost between groups — if one file in a group is slow, the next group waits. A worker-pool model would avoid this but adds significant complexity for a tool targeting batches of tens, not thousands, of files.

**Code:** `src/main/services/ConversionService.ts` — `batchConvert()` · `src/main/main.ts` — IPC handler
