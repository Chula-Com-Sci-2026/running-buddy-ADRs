# 25. Shared CSS and reusable components for UI styling

Date: 2026-10-17

## Status

Accepted for the local Deliverable 4 implementation with project-owner approval. This does not claim teammate review.

Supersedes [ADR-0010](0010-style-the-web-client-with-tailwind-css.md) for the primary styling approach in the local Deliverable 4 client only. Historical ADR-0003 remains superseded by the existing split decisions. The framework decision is recorded separately in [ADR-0024](0024-frontend-framework.md).

## Context

Profile forms, activity rows and run screens must remain consistent and usable across desktop and mobile layouts (NFR-6.1 and NFR-6.2). The current frontend already expresses its colours, spacing, breakpoints and component classes in a shared stylesheet, with reusable buttons, cards and status indicators.

A utility-first Tailwind approach could standardize spacing through utility classes, but making it the primary styling mechanism would require rewriting the existing component classes. A component library would provide ready-made widgets but introduce theme overrides and a stronger dependency on its visual conventions. Independently styling each page would reduce consistency and repeat fixes.

## Decision

Use shared CSS variables and semantic component classes in `src/styles.css`, applied through reusable UI components, as the primary styling approach.

Tailwind remains imported and configured in the existing build; this decision changes the primary authoring convention and does not claim that Tailwind has been removed.

## Consequences

- Shared tokens and UI components make visual corrections reusable across pages.
- Responsive rules and focus styles remain visible in one stylesheet.
- Global selectors and cascade order can cause regressions; page-specific rules need explicit scope and desktop/mobile visual checks.
- The shared stylesheet can grow unwieldy. Splitting it by component can be considered when maintenance needs justify it.

Implementation evidence: `cant-run-alone-frontend/src/styles.css` and `src/components/ui.tsx` in the sibling frontend repository.
