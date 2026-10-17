# 9. Build the web client as a React single-page application

Date: 2026-10-03

## Status

Accepted

Supersedes [3. Frontend framework and UI styling](0003-frontend-framework-and-ui-styling.md)

Reaffirmed for the local Deliverable 4 client by [ADR-0024](0024-frontend-framework.md); the framework choice is unchanged.

## Context

The product is web first; native mobile apps are out of scope, so the browser client must work well on a phone and on desktop (NFR-6.1, NFR-6.2), with first contentful paint under 3 s on 5G (NFR-1.4). The interface is map-heavy and interactive: pinning a meeting point, a nearby-runs map, and discovery filters that update as the runner changes them. Almost every screen is behind login. The team's existing frontend experience is in React.

This ADR covers only the framework. Styling is [10. Style the web client with Tailwind CSS](0010-style-the-web-client-with-tailwind-css.md) and the map library is [11. Render maps with Leaflet and OpenStreetMap tiles](0011-render-maps-with-leaflet-and-openstreetmap-tiles.md).

Options considered:

- **React + TypeScript single-page application built with Vite.** Matches team experience; the map and filter interactions are client-side by nature; the build output is static files with no server tier to run. A client-rendered app ships a larger initial bundle, which puts NFR-1.4 at risk.
- **Next.js (React with server-side rendering).** Same React skills, plus server rendering and file-based routing that help SEO and first paint. Our screens are behind login, where SEO does not apply, and it adds a Node server tier to deploy and operate next to the backend services.
- **Vue.** Capable and lighter to learn than Angular, but nobody on the team has shipped with it, so learning it would cost project time.
- **Angular.** A complete framework with strong conventions, but heavier than this project needs and new to the team.

## Decision

We will build the web client as a React and TypeScript single-page application, built with Vite and served as static files. It talks only to the API gateway.

## Consequences

- The team is productive immediately; no framework learning curve consumes project time.
- TypeScript types generated from the gateway's OpenAPI contract catch integration mistakes at compile time and support the contract tests (NFR-7.2).
- Meeting NFR-1.4 requires route-level code splitting and lazy-loading the map bundle; first paint must be measured, not assumed.
- Authenticated screens get no SEO. A small static landing page can be added separately if public discoverability is ever needed.
- If server rendering becomes necessary, moving a React SPA to Next.js is a well-trodden path, so this decision is reversible at moderate cost.
- Follow-on decision: where the client stores tokens (see [13. Authenticate with short-lived JWTs and rotating refresh tokens](0013-authenticate-with-short-lived-jwts-and-rotating-refresh-tokens.md)).

Source: Deliverable 1 (revised), "ADR-002: Frontend Framework and UI Styling," split per review feedback so the framework and styling decisions each have their own options.
