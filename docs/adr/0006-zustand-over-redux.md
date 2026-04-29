---
layout: default
title: "ADR 0006: Zustand over Redux, a sunk-cost-aware trade-off"
---

# ADR 0006: Zustand over Redux, a sunk-cost-aware trade-off

## Context

Early in the project I needed app-wide state for conversion options, theme, file list, and preview view state. I picked **Zustand** without running a deep comparison against Redux, Jotai, or Context. I wanted momentum, the store needed to exist in an afternoon, not a week.

This ADR is not about initial choice. This ADR is about what happened later, when the state model had grown and I asked myself whether I had picked the right technology.

## The migration attempt

Once the store had accumulated real shape, persisted slices, per-file options, derived state, zoom/pan state, SVG cache, I started to wonder if Redux would be a better long-term fit. Stronger conventions, better devtools story, more structured reducers. I decided to try.

The prototype broke in several places at once:

- **Persist middleware.** Zustand's `persist` with `partialize` had a specific shape I relied on (`themeMode`, `showEditor` only, see ADR 0007). Porting to `redux-persist` required re-deriving which slices persist and which don't, with a different API.
- **TypeScript inference.** Zustand's `create<T>()(...)` gives me end-to-end inference essentially for free. Porting to Redux Toolkit meant re-typing every slice, action, and selector. A lot of the "invisible" type safety I relied on became explicit surface.
- **Store shape.** Zustand lets me co-locate state and actions in one function. Redux splits them, reducer, actions, selectors. Every one of my hooks had to change shape.

I was fixing three unrelated things at once in the migration PR, and whatever I fixed, something else would surface. I reverted the whole attempt.

## Decision

**Stay on Zustand.** Because the cost of migration, measured in days of broken state, regression risk, and re-learning, was visibly larger than the expected benefit.

## Rationale

*"I already have a tool, is it worth the rewrite."*

The two decisions have different decision rules:

- **Upfront choice:** weigh the options on their merits.
- **Migration decision:** weigh the *delta*, current pain vs rewrite cost + regression risk + re-learning curve.

When I reverted the Redux prototype,  I had been learning, by contrast, what Zustand was actually giving me that I hadn't previously articulated: minimal ceremony, one-file stores with state + actions co-located, excellent TS inference, and a `persist` middleware that already fit my needs. The migration attempt turned out to be the clearest audit of the incumbent I could have run.

## Consequences

**Positive**
- State model is stable. No half-migrated code, no mixed patterns.
- The store continues to scale with new features (see ADR 0007 for per-file options, ADR 0008 for cache, ADR 0011 for dual view state) without fighting the framework.

**Negative**
- The initial pick was unexamined. If I had picked Redux first and later tried to move to Zustand, I'd probably have stayed on Redux for the same reason.
- The devtools story is weaker than Redux DevTools. I have not felt the gap enough to pay the migration tax, but it is a real trade-off.


**Looks like architecture is not just the first decision, it's also the decisions you make about *not* re-deciding.**

Writing this ADR honestly mattered because the portfolio-genre temptation is to say *"I weighed Zustand, Redux, and Jotai, and chose Zustand for reasons X, Y, Z."* But that is not what happened. What happened is: picked fast, tried to migrate later, found the migration wasn't worth it, and made staying a considered call.


## Follow-up

- If Zustand ever becomes a bottleneck (state explosion, need for time-travel debugging, cross-process state sync in Electron), the migration decision should be re-opened. Current signal: no such pressure.
- Devtools: Zustand has middleware for Redux DevTools integration. If devtooling becomes a real pain point, this is the first step before considering a full migration.
