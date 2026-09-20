# 3. Frontend framework and UI styling

Date: 2026-09-20

## Status

Accepted

## Context

The product is web first and needs a responsive interface on desktop and mobile (NFR-6.1, NFR-6.2), with first contentful paint under 2 seconds on 4G (NFR-1.4). Its maps, meeting-point pinning, and discovery filters are interactive. The team considered React, Vue, Angular, and Next.js. React matches team experience; authenticated screens gain little from search engine optimization, while Next.js adds a server tier.

## Decision

Build a React and TypeScript single-page application with Vite and TailwindCSS. Use a generated typed API client. Put Leaflet and OpenStreetMap-based tiles behind a thin map wrapper so the tile provider can change.

## Consequences

- The team can use its existing React experience, and generated types help catch API integration errors.
- TailwindCSS supports responsive styling; shared components can contain repetitive class lists.
- Client rendering and map assets put the first-paint target at risk. Split routes and lazy-load the map bundle, then measure NFR-1.4.
- Server rendering remains a possible later migration if public discoverability or performance requires it.

Source: Deliverable 1, “ADR-002: Frontend Framework and UI Styling.”
