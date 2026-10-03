# 10. Style the web client with Tailwind CSS

Date: 2026-10-03

## Status

Accepted

Supersedes [3. Frontend framework and UI styling](0003-frontend-framework-and-ui-styling.md)

## Context

The React single-page application ([9. Build the web client as a React single-page application](0009-build-the-web-client-as-a-react-single-page-application.md)) needs a styling approach. The forces:

- **Mobile first.** Most runners will open the app on a phone before or after a run, and every screen must also work on desktop (NFR-6.1, NFR-6.2).
- **First paint.** First contentful paint must stay under 3 s on 5G (NFR-1.4), and the map bundle already costs a lot, so the styling approach should add little CSS and no runtime cost.
- **Custom screens.** The key screens (run cards with a run-suitability indicator, a full-screen map with overlays, ranked runner cards) are specific to this product rather than standard admin forms.
- **Thai and English UI** (NFR-6.3): labels change length between languages, so layouts must not depend on fixed text widths.
- **Several developers** working on screens at once need consistent spacing, colors, and breakpoints without a long style guide.

Options considered:

- **Tailwind CSS (utility classes).** Responsive variants (`sm:`, `md:`) make mobile-first layout the default; unused classes are removed at build time, so the CSS shipped stays small; design tokens in one config keep spacing and colors consistent. Class lists in markup get long and noisy, and the team must learn the utility vocabulary.
- **A component library (for example MUI or Mantine).** Ready-made, accessible buttons, dialogs, and form controls save time on standard widgets. It ships more CSS and JavaScript than we need, its look is recognizable and needs theming to fit custom screens, and some libraries style at runtime, which costs first paint.
- **CSS Modules or plain CSS.** No new tool and no runtime cost; styles are scoped per component. Consistency depends on discipline: every developer picks their own spacing and breakpoints unless the team writes and maintains shared variables.
- **CSS-in-JS (for example styled-components).** Styles live next to components and can use props. Styles are generated at runtime in the browser, which adds JavaScript and slows first paint on mobile, against NFR-1.4.

## Decision

We will style the web client with Tailwind CSS, with the colors, spacing, and breakpoints defined once in the Tailwind config, and repeated patterns extracted into shared React components rather than repeated class lists.

## Consequences

- Responsive, mobile-first layouts are quick to build, and the shipped CSS stays small, which helps NFR-1.4.
- Long class lists make markup harder to read; extracting shared components (button, card, form field) is required, not optional.
- Accessible widgets such as dialogs, menus, and date pickers are not included. The team must build them or add a headless component library, a follow-on decision when the first such widget is needed.
- Visual consistency depends on using the config tokens rather than arbitrary values; code review should reject one-off values.
- Moving away later means restyling every component, so this is costly to reverse once many screens exist.

Source: Deliverable 1, "Frontend Framework and UI Styling," split per review feedback; this ADR adds the styling options that the original Context did not weigh.
