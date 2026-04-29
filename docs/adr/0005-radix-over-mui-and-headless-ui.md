---
layout: default
title: "ADR 0005: Radix UI over MUI and Headless UI"
---

# ADR 0005: Radix UI over MUI and Headless UI

> **Decision:** Replace MUI with Radix UI unstyled primitives. **Why:** MUI's theme system couldn't be overridden cleanly enough to produce a distinctive visual identity; Radix gives full style ownership with accessibility and portal handling included.

## Context

Vektory started on MUI. It got me moving fast, but as the design took shape I hit a wall: I couldn't make the app look like *Vektory*. Every override fought the theme system, and the result still read as "a MUI app in a custom skin", recognisable at a glance. I wanted the opposite: a distinctive dark-purple-and-gold surface where nothing shouted its origin library, where the visual identity was mine.

That meant switching to an unstyled primitives library. The choice came down to **Radix UI** versus **Headless UI**.

## Options considered

**MUI** (incumbent)
Ships with a complete design system and styling engine. Great for shipping fast on a conventional visual. Overrides are possible but adversarial, I found myself fighting specificity wars and theme tokens I didn't need. The app kept looking like MUI no matter what I did.

**Radix UI**
Unstyled, accessible primitives. Each primitive ships as a separate npm package, installed independently. Good portal/focus/ARIA handling out of the box. Visual layer is 100% my responsibility.

**Headless UI**
Similar philosophy, unstyled primitives, accessibility built in. Distributed as a single monolithic package. Smaller primitive surface than Radix.

## Decision

**Radix UI.** The per-primitive package boundary and the richer primitive set made it the better fit for Vektory.

## Rationale

- **Primitive coverage.** Radix had every primitive I needed (Slider, Tabs, Tooltip, Select, Checkbox, Label, Separator) plus nuanced things like `Tooltip.Provider` and `Select.Portal`. Headless UI covered less of my surface, which would have meant hand-rolling components alongside library ones, an inconsistent foundation.
- **Portal control matters in Electron.** Tooltips, selects, and popovers that escape overflow-hidden parents are non-negotiable in a multi-panel desktop UI. Radix's portal primitives gave me precise control over mount targets; in an Electron renderer with custom layouts and scroll containers, this saves a lot of debugging.
- **Bundle discipline via package boundary.** I install only the primitives I use, `@radix-ui/react-slider`, `@radix-ui/react-tabs`, etc., and nothing else lands in the bundle. Headless UI is one package: import one component, pull the whole module graph.
- **Full control over the visual language.** Because every style is mine, there is no default look to fight. The dark-mode design tokens in `styles.css` are the single source of truth, no second theme system underneath to reason about.

## Consequences

**Positive**
- The UI looks like Vektory I dreamed about. Not like Vektory-on-MUI.
- Design tokens are authoritative. `--accent-primary`, `--brand-gold`, `--glass-border`, one file, one hierarchy, no library shadow.
- Bundle size is predictable: only the primitives I import ship.
- Accessibility baseline is strong out of the box, I don't have to re-implement focus management or ARIA patterns per component.

**Negative, the discipline tax**
- With MUI, consistency is a side effect of the library. With Radix, consistency is my responsibility. Every new primitive I introduce needs to follow the same styling conventions or the UI drifts. I had to write down the conventions (spacing scale, token usage, interaction states) and actually follow them.
- No "out of the box" wins. If I want a styled button, I style it.

- LLM-assisted component work requires `UI_CONSTRAINTS.md` in context — otherwise the model invents styles that don't match the system.

## Follow-up

- `UI_CONSTRAINTS.md` is the source of truth for the design system, it needs to keep pace with new components and any token changes.
- Future primitives I may add (combobox, dialog, toast) should be evaluated against Radix first; if a primitive doesn't exist there, build it hand-rolled rather than pulling a second library.
- If I rewrite for a different runtime (e.g. Tauri) the Radix choice should survive, the reasoning here is about the styling boundary.
