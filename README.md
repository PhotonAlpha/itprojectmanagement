# IT Project Management — Kanban Board

A single-page Kanban board for tracking IT project tasks. Everything — markup,
styles and behaviour — lives in one `index.html` file with no dependencies, no
build step and no server.

**Live demo → https://photonalpha.github.io/itprojectmanagement/**

> This is an internal demo / training tool, not a product. The "UOB IT PMO"
> heading is a plain text wordmark: the project uses no real UOB logos or
> trademarks and does not imitate an official UOB system.

## Running it

Download `index.html` and double-click it. That is the whole setup — it runs
straight from `file://`.

```bash
open index.html
```

A local server is only needed for browser automation, which cannot open
`file://` URLs:

```bash
python3 -m http.server 8765
```

## Features

- **Four-column board** — Backlog, In Progress, Blocked, Done, each with a live
  count that reflects the current filter.
- **Drag and drop** between columns, with a highlighted drop target.
- **Keyboard fallback** — every card carries a `Move ▸` menu, so the board is
  fully usable without a pointer.
- **Task fields** — title, description, project/workstream, category, assignee,
  priority, due date and status. Tasks get sequential `UOB-ITPM-####` ids.
- **Priority pills** — Critical, High, Medium, Low. Colour is never the only
  signal; each pill carries its own text.
- **Due-date awareness** — dates are compared in local time, so "overdue" agrees
  with the date the user actually sees in the picker.
- **Filtering** by project, assignee and priority. Column badges show the
  filtered counts while the header summary always reflects every task.
- **Inline validation and deletion** — errors render under the field they belong
  to, and deleting a card uses an inline "Delete? Yes / No" row. No `alert()` or
  `confirm()` anywhere.
- **Email notification** on task creation via FormSubmit's AJAX endpoint,
  fired so that a network failure can never break the board.

## Tech stack

Vanilla HTML, CSS and JavaScript. No framework, no bundler, no npm, no tests.
Icons are inline SVG or Unicode glyphs; the favicon is an inline data URI. The
only outbound request the page makes is to FormSubmit.

## Architecture

`state` is the single source of truth, holding `tasks`, `filters`, a transient
`ui` object and the id counter. Every mutation goes through `addTask()`,
`moveTask()` or `deleteTask()`, and each one ends by calling `renderBoard()` —
nothing mutates card DOM directly.

The four column shells are static markup; only the list contents and count
badges are rebuilt on render. Drop listeners therefore bind once to the static
columns, while `dragstart` and all card buttons are **delegated** on `#board` so
they survive a re-render. Because re-rendering destroys focus and open menus,
`state.ui` carries the open menu, pending delete confirmation and focus target
across the rebuild.

`renderCard()` builds HTML strings, so every interpolated value passes through
`escapeHtml()`.

## Constraints

These are project requirements rather than preferences. A change that breaks one
breaks the deliverable:

| Constraint | Why it matters |
| --- | --- |
| Vanilla only | No React/Vue/jQuery/Tailwind, no bundler, no npm, no build step |
| One file | Everything stays in `index.html`; no separate `.css` or `.js` |
| Runs from `file://` | Double-clicking the file must work |
| No persistence | No `localStorage`, cookies or IndexedDB — a refresh resetting the board to seed data is intended, and the filter bar says so |
| No external resources | No CDN scripts, web fonts or image files |
| No `alert()` / `confirm()` | Validation and deletion are inline |

Accessibility is part of the spec: `<label for>` on every input, `aria-label` on
icon-only buttons, `aria-live="polite"` on the toast region, and visible focus
rings.

`CLAUDE.md` holds the fuller guidance, including how to sanity-check a change.

## Configuration

`FORMSUBMIT_ENDPOINT` near the top of the script block is the single place the
notification address appears. It currently holds the placeholder
`YOUR_EMAIL@example.com`, so notifications fail into a non-blocking warning
toast until you swap it. FormSubmit also needs a one-time activation: the first
submission emails a confirmation link to that address, and nothing is delivered
until it is clicked.

## Deployment

Pushing to `main` triggers `.github/workflows/static.yml`, which publishes the
repository to GitHub Pages.
