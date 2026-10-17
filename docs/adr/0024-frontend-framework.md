# 24. React for the browser application

Date: 2026-10-17

## Status

Accepted for the local Deliverable 4 implementation with project-owner approval. This does not claim teammate review.

Reaffirms [ADR-0009](0009-build-the-web-client-as-a-react-single-page-application.md) for the implemented Deliverable 4 client. The React framework choice is unchanged; this does not supersede ADR-0009 or reopen historical ADR-0003. Styling is addressed separately by [ADR-0025](0025-ui-styling.md); API-client generation and map choices are outside this decision.

## Context

Discovery filters, meeting-point selection and run participation require an interactive browser interface. The team needs a maintainable frontend that supports desktop and mobile screens (NFR-6.1 and NFR-6.2). The existing working prototype uses React, TypeScript and Vite, matching the team's experience.

Vue offers a smaller component API but requires migration and additional team learning. Angular supplies stronger framework conventions at a higher setup cost. Next.js adds rendering and server capabilities, but the demonstrated authenticated application has no established requirement for server rendering or search-engine indexing.

## Decision

Use a client-rendered React single-page application, implemented in TypeScript and built with Vite.

## Consequences

- Existing components and team experience can be retained rather than rebuilding the prototype.
- Client rendering requires explicit loading, error and session-recovery states.
- Route and map code splitting can limit initial downloads, but the first-contentful-paint target in NFR-1.4 still requires measurement; framework selection does not prove it.
- Public indexing or measured rendering problems would justify revisiting this choice.

Implementation evidence: `cant-run-alone-frontend/package.json`, `src/main.tsx` and `src/App.tsx` in the sibling frontend repository.
