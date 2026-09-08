# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page IT Project Management Kanban board for an internal "UOB IT PMO" demo/training tool. The
entire application is one file: `index.html` (~1,180 lines) holding the markup, a `<style>` block and a
`<script>` block. There is no package.json, no dependencies, no build step and no test suite.

It is a demo, not a product: the branding is a plain text wordmark, and it must not use real UOB logos,
trademarks, or imitate an official UOB system.

## Hard constraints

These are requirements of the project, not stylistic preferences. Breaking any of them breaks the
deliverable, so check a change against this list before making it:

- **Vanilla only.** No React/Vue/jQuery/Tailwind, no bundler, no npm, no build step.
- **One file.** Everything stays in `index.html`. Do not split out `.css` or `.js` files.
- **Runs from `file://`.** Double-clicking the file must work — no server required.
- **No persistence.** No `localStorage`, `sessionStorage`, IndexedDB or cookies. Board state lives only in
  the in-memory `state.tasks` array; a refresh resetting the board to seed data is intended behaviour, and
  the filter bar carries a visible note saying so.
- **No external resources.** No CDN scripts, Google Fonts or image files. System font stack and inline SVG
  for icons. The only outbound request is the FormSubmit endpoint. This is now enforced by the CSP `<meta>`
  in the head, not just by convention: anything you add from another origin is blocked by the browser with
  no visible error, so widen the CSP deliberately or not at all.
- **No `alert()` / `confirm()`.** Validation errors render inline under each field; card deletion uses an
  inline "Delete? Yes / No" row inside the card.

## Running and verifying

```bash
open index.html                       # normal usage: no server needed
python3 -m http.server 8765           # only for browser automation — the Chrome
                                      # extension cannot open file:// URLs
```

There are no tests. To sanity-check a change:

```bash
# JS syntax check: extract the <script> block and parse it
python3 -c "import re;print(re.search(r'<script>(.*)</script>',open('index.html').read(),re.S).group(1))" \
  > /tmp/app.mjs && node --check /tmp/app.mjs

# constraint audit
grep -niE "localstorage|sessionstorage|indexeddb|document\.cookie" index.html   # expect no hits
grep -oniE "https?://[^\"' )]+" index.html                                      # expect only formsubmit.co
grep -n "alert(\|confirm(" index.html                                           # expect only the comment
```

Beyond that, verify interactively in the browser — drag a card between columns, use the keyboard
`Move ▸` fallback, and confirm the column counts and header summary stay consistent with `state.tasks`.

## Architecture

**`state` is the single source of truth** (`index.html:632`). It holds `tasks`, `filters`, a transient
`ui` object and `nextSeq` (the counter behind `UOB-ITPM-####` ids). Every mutation goes through
`addTask()` / `moveTask()` / `deleteTask()`, and each ends by calling `renderBoard()`. Nothing mutates card
DOM directly — if you need a card to look different, change state and re-render.

**Two-layer DOM.** The four `.column` shells are static markup in the HTML; only the `.column-list`
contents and the count badges are rebuilt by `renderBoard()`. This split matters:

- Drag-and-drop *drop* listeners bind once to the static columns in `bindBoardInteractions()`
  (`index.html:904`); the `dragstart` listener and all card button clicks are **delegated** on `#board` so
  they survive re-render. Add new card interactions as delegated `data-action` handlers, not direct
  listeners inside `renderCard()`.
- Each column tracks a `depth` counter for `dragenter`/`dragleave`, because those events also fire for
  child elements and would otherwise flicker the `.drop-target` highlight.

**Re-render destroys focus and open menus**, so `state.ui` carries `menuFor`, `confirmFor` and
`focusSelector` across the rebuild; `restoreFocus()` re-focuses the element after render. Any new
per-card transient UI (an inline editor, say) needs the same treatment or it will close on the next render.

**Escaping is mandatory.** `renderCard()` builds HTML strings, so every interpolated value passes through
`escapeHtml()`. Keep that invariant if you add fields.

**`sanitizeText()` guards the other sink.** Escaping protects the DOM; it does nothing for the FormSubmit
payload, where the title lands in an email `_subject`. Free-text fields are run through `sanitizeText()` in
`validateForm()` before they are length-checked, so the value validated is the value stored and sent. Any
new free-text field needs the same treatment.

**Filtering is a pure read.** `applyFilters()` returns a filtered copy; column count badges reflect the
*filtered* view while the header summary strip always reflects all of `state.tasks`.

**Dates** are compared as `YYYY-MM-DD` strings against `todayISO()`, which builds a local-time (not UTC)
date so the `<input type="date">` min comparison matches what the user sees.

## FormSubmit integration

`FORMSUBMIT_ENDPOINT` (`index.html:624`) is the one place the notification email address appears — swap it
there. FormSubmit needs a **one-time activation**: the first submission sends a confirmation email to that
address and nothing is delivered until the link in it is clicked.

The AJAX JSON endpoint is used deliberately so the page never navigates away. The flow in `bindForm()` is
optimistic: the card is added to the board and the success toast fires *before* the network call, then the
submit button shows a disabled "Sending…" state while `notifyNewTask()` is in flight. A failure must never
break the board — it is caught and reported as a non-blocking warning toast. Note that under `file://` the
request has a `null` Origin and may be blocked by CORS; that path is expected to land in the same warning
toast.

Never send the user's email address anywhere except this endpoint.

## Conventions

CSS uses custom properties for the palette and a `--s-1`…`--s-6` spacing scale, with no `!important`.
Both the style and script blocks are divided into numbered comment sections; keep new code inside the
matching section. Accessibility is part of the spec, not optional polish: `<label for>` on every input,
`aria-label` on icon-only buttons, `aria-live="polite"` on the toast region, visible focus rings, and
colour is never the only signal (priority pills carry their text).
