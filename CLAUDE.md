# CLAUDE.md — web-frontend

This file is the reference doc for Claude Code (and future contributors) working in this
repository. Keep it up to date: every code change made in this repo should get a dated entry
in the **Changelog** section at the bottom.

## What this is

A single **static HTML page** (`index.html`) that is a standalone client for the "Happy Soda"
basketball community dashboard. It is a **near-duplicate** of `../app-script-backend/ui.html`'s
client-side UI, adapted to run outside Apps Script:

- `ui.html` (in `app-script-backend`) is server-rendered by Apps Script — the data (`DATA`)
  is embedded directly into the page at render time.
- `index.html` (this repo) is plain static HTML with **no build step and no server code**. On
  load it calls `fetch(APPS_SCRIPT_URL + '?format=json')` against the Apps Script backend's
  deployed web app, gets the same JSON `app-script-backend`'s `doGet()` would embed, and then
  renders the dashboard client-side into `#content` via `buildDashboard()`.

There is only one file, `index.html`, no other repo content (no `package.json`, no build
tooling) — presumed deployed as-is via a static host (e.g. GitHub Pages) pointed at this repo.
**Verify/update this note** once the actual hosting is confirmed.

## Files

- **`index.html`** — everything: markup skeleton, CSS, and all client JS in one file.

## Data source

```js
const APPS_SCRIPT_URL = 'https://script.google.com/macros/s/AKfycbzkY97F_zyPptOZvJrTsWlslDvE8mWA4iiSL0IRj_ajZo6Tez13iDY7lla3gTPIHv2c/exec';
```

This is the Apps Script backend's **pinned live deployment URL** (see
`../app-script-backend/CLAUDE.md` → Deployment identifiers). `loadData()` fetches
`APPS_SCRIPT_URL + '&format=json'` (plus `month=` / `recordsMonth=` params from the date
pickers) and stores the result in the global `DATA`. If this URL ever changes (e.g. the
backend deployment is recreated instead of updated in place), it must be updated here by hand.

## Architecture / request flow

1. Static shell loads (`<body class="light-mode">` + a loading overlay).
2. `loadData()` fetches JSON from `APPS_SCRIPT_URL`, sets `DATA`, then calls `buildDashboard()`.
3. `buildDashboard()` injects the full page markup (side menu, all pages/panels) into
   `#content` via string concatenation — unlike `ui.html`, where this markup is static HTML
   already in the file.
4. From there on, rendering functions (`renderAll`, `renderMembers`, `renderRegulars`,
   `renderRecordsPage`, `renderTeamManagerLists`, `renderLeagueDraftLists`,
   `renderAttendanceChart`, etc.) work the same way as in `ui.html`, reading from `DATA`.
5. Changing the "As of" or "Monthly Records" month pickers calls `loadData(month, recordsMonth)`
   again (a fresh `fetch`), **not** a full page reload/URL navigation like `ui.html`'s
   `reloadWithParams()` — there is no `SELF_URL`/`selfUrl` concept here since this page isn't
   served by Apps Script.

## Domain rules

All loyalty-tier/points logic, sheet-parsing, and business rules live in
`../app-script-backend/Main.js` — this repo has **no business logic of its own** for tiers,
points, or eligibility; it only renders whatever JSON the backend computes. See
`../app-script-backend/CLAUDE.md` for the full rules (tier thresholds, New Comers, Comeback/
Departed, Outstanding/Credits, etc.).

Client-side logic that lives only in this file, NOT shared with `ui.html`: rendering/
formatting helpers, search/filtering, CSV-less table building, image export
(`exportImageFromElement`, via html2canvas), and WhatsApp-list copy helpers
(`copyMembershipList`, `copyPerksList`, `copyTeamList`).

Team auto-balance and League Draft splitting (`assembleTeams`, `snakeDraftAssign`,
`computeTeamAvgRating`, `evaluateAllSplits`, `assembleLeagueTeams`, `buildSnakeOrder`,
`computeTeamSizes`, `findMaxValidTeams`, `parseAttendanceNames`, `matchAttendanceToProfiles`,
and their supporting functions) **are** shared, mechanically — see
[Relationship to `../app-script-backend`](#relationship-to-app-script-backend) below. Don't
edit these functions in `index.html` directly; they live between
`// SHARED-LOGIC:<name>:START`/`:END` marker comments and get overwritten by
`../app-script-backend/scripts/sync-shared.js`.

## UI pages

Same set as `ui.html`: Dashboard, Outstanding Payments, Player Credits, Monthly Records, Team
Manager, League Draft, About Us — built by `buildDashboard()` instead of being static markup.

## Relationship to `../app-script-backend`

This repo's `index.html` and the sibling repo's `ui.html` implement **the same dashboard
twice**. See `../app-script-backend/CLAUDE.md` → "Relationship to `../web-frontend`" for the
authoritative statement of this, but in short:

- **Team-balancing/draft-algorithm changes** — edit `../app-script-backend/shared/draft-logic.js`
  (the canonical source), then run `node scripts/sync-shared.js` from `../app-script-backend`.
  It rewrites the `SHARED-LOGIC:*`-marked block in both this file and `ui.html` to match —
  never edit those blocks in `index.html` directly, the sync script will overwrite it. See
  `../app-script-backend/CLAUDE.md` → "Shared draft/team-balancing logic".
- **Other UI/rendering/client-logic changes** (styling, new panels, formatting not covered by
  the mechanism above, etc.) — still make the change here **and** in
  `../app-script-backend/ui.html` in the same session, unless the change is deliberately
  frontend-only (e.g. something about how this static page loads/fetches data, which has no
  equivalent in `ui.html`). There is no shared source for these, they're copy-kept-in-sync.
- **Business-logic/JSON-shape changes** — happen in `../app-script-backend/Main.js`. If a field
  used by this file's `DATA.*` accesses is renamed or removed there, this file needs a matching
  update, or the page will break silently (or via `showFatalError`).

## Workflow for making changes

1. Edit `index.html` as needed.
   - If the change is to team-balancing/draft-algorithm logic (anything inside a
     `SHARED-LOGIC:*` marked block), edit `../app-script-backend/shared/draft-logic.js`
     instead and run `node scripts/sync-shared.js` from that repo — see
     [Relationship to `../app-script-backend`](#relationship-to-app-script-backend) above.
2. If the change is some OTHER UI/rendering change not covered by the shared-logic mechanism,
   make the matching change in `../app-script-backend/ui.html` in the same session (see
   [Relationship to `../app-script-backend`](#relationship-to-app-script-backend) above and
   that repo's `CLAUDE.md`).
3. Add a dated entry to the [Changelog](#changelog) below in **this** file describing what
   changed and why.
4. Commit and push to GitHub:
   ```
   git add -A
   git commit -m "..."
   git push origin main
   ```
   This repo has no build/deploy step beyond the git push itself — whatever host serves this
   repo (e.g. GitHub Pages) picks up `index.html` directly from the pushed branch.
5. If the paired backend change in `app-script-backend` also needs deploying (JSON shape
   changed, etc.), follow that repo's `clasp push` + `clasp deploy` steps too — see
   `../app-script-backend/CLAUDE.md` → Workflow.

## Changelog

_Most recent first. Add one entry per change (or logical group of changes), dated._

- **2026-08-29** — Team-balancing/draft-algorithm code (attendance parsing, `evaluateAllSplits`/
  `assembleTeams`, `computeMultiTeamScore`/`assembleLeagueTeams`/etc.) is no longer hand-copied
  from `../app-script-backend/ui.html` — it's now generated from
  `../app-script-backend/shared/draft-logic.js` via that repo's `scripts/sync-shared.js`, into
  `SHARED-LOGIC:*`-marked blocks in this file. The first sync run fixed two small pre-existing
  drifts here (a missing comment on `choose5`, a missing inline comment in
  `evaluateAllSplits`). Everything else in this file (rendering, `buildDashboard()`, formatting
  helpers, etc.) is unaffected and still hand-kept-in-sync as before. See
  `../app-script-backend/CLAUDE.md`'s Changelog for the full rationale.
- **2026-08-29** — Added this `CLAUDE.md` (documentation only; no functional code changes).
  Confirmed this page has no build step and is a hand-kept-in-sync duplicate of
  `../app-script-backend/ui.html`'s client code, fetching JSON from the backend's pinned
  deployment URL rather than receiving data server-embedded.
