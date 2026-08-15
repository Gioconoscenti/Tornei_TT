# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Torneo Manager is a single-file web app for ASD Tennistavolo San Polo (Parma) to run table-tennis tournaments: round-robin groups ("gironi") followed by a single-elimination bracket ("tabellone"). It's built to run entirely client-side with zero install: open `Tornei_TT.html` in a browser, no server, no account, no build step. All state lives in the browser's `localStorage`.

The full user-facing workflow and business rules are documented in `Manuale_TorneoManager.pdf` (Italian) — read it (`pdftotext -layout Manuale_TorneoManager.pdf -` works in this environment) before changing tie-break logic, bracket seeding, or the import/export formats, since those are user-visible contracts described there.

## Repository layout

This repo has no package manager, no build tooling, and no test suite — it's two files:
- `Tornei_TT.html` — the entire application (CSS in `<style>`, markup, and all JS in one trailing `<script>` block).
- `Manuale_TorneoManager.pdf` — the Italian user manual.

## Commands

There is no build/lint/test pipeline. Useful commands during development:

- **Run the app**: open `Tornei_TT.html` directly in a browser (double-click, or `file://`). Note from the manual: on iPad/iPhone, `file://` from iCloud Drive/Files blocks some JS — serve it over local HTTP instead (e.g. drag into netlify.com/drop, or `npx serve`).
- **Syntax-check the JS after an edit** (no linter is configured, but this catches typos fast):
  ```bash
  awk '/<script>/{flag=1;next}/<\/script>/{flag=0}flag' Tornei_TT.html > /tmp/check.js && node --check /tmp/check.js
  ```
- **Manual/browser smoke testing**: there's no test suite, so verify behavior by driving the file in a real or headless browser. Playwright works well for this (`npm install playwright` + `node node_modules/playwright/cli.js install chromium` in a scratch dir — plain `npx playwright install` can fail if the user's Windows profile path contains `&`, use the direct `node .../cli.js` invocation instead). Because state persists via `localStorage`, call `localStorage.clear()` and reload between test scenarios. Inline `let`/`const` top-level bindings in the page's `<script>` (e.g. the `S` state object) are **not** attached to `window` — access them as bare identifiers in `page.evaluate()`, not `window.S`. Functions declared with `function` (e.g. `tabSetWin`) *are* attached to `window`.
- **Extract the manual's text** (for reference, since PDF page-rendering tools aren't installed here): `pdftotext -layout Manuale_TorneoManager.pdf out.txt`.

## Architecture

### Two independent persistence stores

- `S` (`localStorage['tts_torneo']`) — the current tournament: `{parts, numGironi, gironi, tabelloni:{A,B}, qualAssign}`. Reset per-tournament.
- `ANAG` (`localStorage['tts_anag']`) — the club's permanent player roster (name + category). Survives tournament resets and import/export of `S`; managed entirely separately in the "Anagrafica" tab.

Both are saved via small `save()`/`saveAnag()` wrappers called after every mutation — there's no reactive framework, so **every function that mutates `S` or `ANAG` must explicitly call the matching save + a `render*()` call**, or the UI/localStorage will silently drift out of sync.

### Identity model: participant `id`, not name

Every participant gets `id: Date.now()+Math.random()` when added (in `addPart`/`addAnag`/`addFromAnag`). This `id` is the key used everywhere data must stay unambiguous across two players who happen to share a name — critically in `S.qualAssign` (board A/B/excluded assignment, keyed by id) and in girone standings (`computeClass` keys its stats dict by `p.id`).
The **bracket itself** (`S.tabelloni[board].rounds[][].p1/p2/win`) is the one place that still stores the player's **name** as a plain string, not id — this was a deliberate, scoped trade-off (see git history) rather than an oversight: bracket slot propagation is purely positional (round/match index), so name collisions don't corrupt match results there, only cosmetic seed labels. Don't assume the rest of the codebase is name-keyed just because the bracket is.

### Section/tab navigation

Four top-level sections (`#sec-anagrafica`, `#sec-partecipanti`, `#sec-gironi`, `#sec-tabellone`) are toggled via `.section.active` (plain `display:none`/`block`, all stay in the DOM). The nav click handler re-renders the target section's data on every switch (`renderAnag()`, `renderGironi()`, `renderTab()+renderQualAssign()`, `renderParts()+updateStats()`) — so most render functions don't need to be called defensively elsewhere; switching tabs is the recovery path.

### Data flow: Anagrafica → Partecipanti → Gironi → Tabellone

1. **Anagrafica** (`ANAG`) is the durable club roster; players get copied by value into `S.parts` via "Da anagrafica" (`addFromAnag`) or manual entry (`addPart`) — after that point `S.parts` entries are independent of `ANAG`.
2. **`creaGironi()`** distributes `S.parts` into `S.numGironi` groups, balancing by category (round-robins each category's players across groups so same-level players land in different gironi), then generates each girone's round-robin schedule via `genRR()` (standard circle method, with a synthetic `'riposo'` bye player for odd counts).
3. **`computeClass(g)`** computes a girone's standings from `g.incontri`, applying an 8-level tie-break cascade (points → head-to-head set diff → head-to-head sets won/lost → overall set diff → overall sets won/lost → category) documented in the manual §3.4 and mirrored by the `tieNotes()` helper that renders the "criteri avulsi" explanation box.
4. **Qualification** (`S.qualAssign`, id-keyed): either `proponiQual()` auto-assigns top-N-per-girone to A/B, or the user clicks per-player A/B/– badges (`setBoard`, wired via a single delegated `document.addEventListener('click', ...)` matching `.board-badge[data-sbn]` — badges are re-rendered constantly, so handlers are delegated rather than bound per-element).
5. **`creaTabellone()`** builds bracket slots from `getQual(board)` (girone standings filtered to that board) using `SEED_MAP` (hardcoded standard seeding order for 4/8/16/32/64-slot brackets) so top seeds are spread apart; unfilled slots become the sentinel `'X'` (bye).
6. Bracket results propagate round-to-round in `tabSetWin()`/`tabInlineScore()`: setting a winner writes them into the next round's match, and un-setting (or reassigning) one calls `clearCascade()` to unwind any downstream rounds that had already been decided based on the old result. `autoAdvanceByes()` runs on every `renderTab()` to auto-resolve any match where one side is a bye.

### Rendering pattern

No framework — every `render*()` function does a full `innerHTML =` rebuild of its container from the current `S`/`ANAG` state (e.g. `renderParts`, `renderGironi`→`renderGironeContent` per girone tab, `renderQualAssign`, `buildBracket`). Static, mostly-idempotent controls use inline `onclick="fn(...)"` attributes; controls that get re-rendered very frequently (board badges, bracket score inputs/buttons, bracket winner-name clicks) use `data-*` attributes read by a handful of delegated listeners near the bottom of the script instead, to avoid rebinding handlers on every re-render.

### PDF export

`pdfGironi()`/`pdfTabellone()` don't use a PDF library — they build a standalone HTML document (own `PDF_CSS`) and open it in a new window via `printPage()`, then call `window.print()` so the user saves it as PDF through the browser's print dialog (see manual §6.2/§6.3 for the exact print-to-PDF steps on Mac/Windows).

### Escaping

Player-supplied text (names, categories) goes through `esc()` (escapes `&<>"`) whenever rendered as element content. A handful of spots that build `data-*` attribute values only escape `&`/`"` (sufficient there since the value is inside a quoted attribute, never as element content) — keep that distinction in mind rather than assuming `esc()` is used everywhere.
