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
tooling) — deployed as-is via **Cloudflare Pages** pointed at this repo, live at
`https://happysoda.pages.dev/` (this is the URL shared with the community — see the Changelog
entry on Open Graph tags below for why it needs to be kept in sync with `index.html`'s
`og:url`/`og:image`).

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

**Step 0 — sync both repos before doing anything else.** Before reading, analysing, or
changing any code, run this in **both** `app-script-backend` and `web-frontend` (they're edited
from more than one machine/session, and they depend on each other):
```
git fetch origin
git status -sb          # must not say "behind"
git pull --rebase origin main
```
Do it again right before committing/deploying. Working from a stale checkout means analysing
outdated code, and `deploy.sh` deploys whatever is on local disk: on 2026-09-18 a stale checkout
was deployed over newer live fixes and had to be rolled back (see the changelog).

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
   This repo has no build/deploy step beyond the git push itself — Cloudflare Pages picks up
   `index.html` (and any other files, e.g. `og-image.jpg`) directly from the pushed branch.
5. If the paired backend change in `app-script-backend` also needs deploying (JSON shape
   changed, etc.), follow that repo's `clasp push` + `clasp deploy` steps too — see
   `../app-script-backend/CLAUDE.md` → Workflow.

## Changelog

_Most recent first. Add one entry per change (or logical group of changes), dated._

- **2026-09-19** — Mirrored from `../app-script-backend/ui.html` (see that repo's CLAUDE.md
  for the full reasoning behind each):
  - **Name chips**: the four places that rendered a list of players as one comma-joined
    paragraph — the Dashboard's New & Returning / New / Departed boxes, League Draft's
    "N players loaded" box, and each Internal Leagues team card — now share one
    `nameChips_()` helper and a `.name-grid` / `.name-chip` component: an auto-filling grid
    of equal-width 12.5px cells, with a muted position suffix on the IL cards. The 104px
    column minimum was picked by measurement (92px/84px truncate real names on the live
    roster; 104px truncates none at 1280/900/390px). Also gave `.il-team-card-main` the
    `flex: 1; min-width: 0` it never had — it shrink-wrapped, which inline text hid but a
    grid did not, collapsing the cards to a single column.
  - **Copy / Export buttons** no longer destroy their own markup: they set `btn.textContent`
    for the transient "Copied!"/"Generating…" label, which flattened the icon and the
    two-line `.btn-label` into one text node and left the button reading "CopyMembership
    list" in the wrong font afterwards. New `btnLabel_()`/`setBtnLabel_()` swap only the
    label text.
  - **Outstanding / Player Credits** `.balance-name`/`.balance-amount` 16px → 15px (14px
    under 520px). The type was never bigger than the Dashboard's `.card-name` (also 16px) —
    it read that way because a balance row has nothing smaller in it to size against.
  - **Internal Leagues game forms** gained a calendar popover on the Date field and a
    30-minute-slot popover on Time (replacing the native `<input type="time">`), both
    `position: fixed` because `.il-table-wrap`'s `overflow-x: auto` would otherwise clip
    them, plus `isValidHM_` validation now that Time is a text field.
  Verified in headless Chrome against live data alongside `ui.html`, with identical results
  in both files: 0 truncated chips and no horizontal overflow at 1280/900/390px, the delta
  chip click still filtering All Players to "David, Hery, Kevin", the Copy button restoring
  to "Copy"/"Membership list" at 14px/11.5px, and both pickers opening inside the viewport
  and outside the table wrapper's box. No page errors. Deployment is separate — this repo has
  its own deploy step.

- **2026-09-19** — Mirrored from `../app-script-backend/ui.html`: both month pickers (Dashboard
  "as of" date, Monthly Records) replaced the native `<select>` — which still opened the
  browser's own unstyleable dropdown to actually pick a month — with a custom calendar-style
  popover (one year at a time, `‹ year ›` navigation, current month accent-highlighted). Same
  trigger pill, same `'YYYY-MM'` values, only how a value gets picked changed. See that repo's
  changelog for the full detail, including a real bug caught before shipping (clicking the
  year-nav arrows was closing the popover instead of navigating, fixed before this went out).
- **2026-09-18** — Four requests: the mobile hero banner's padding/logo size trimmed (was
  taking up roughly a third of the screen at phone width); the browser-tab favicon replaced
  with the actual "HS" rail-logo artwork (`--hs-mask`, decoded, tinted the dark-mode accent
  green, and re-encoded as a 64×64 PNG data URI) — **this one is `web-frontend`-only**, since
  Apps Script's iframe wrapper means `ui.html` can't control its own favicon (see the
  2026-08-30 entry below); Internal League cards now show a start–finish date range derived
  from the games' own dates (`computeLeagueDateRange_`), next to the existing team/game
  counts; and the player photo upload flow now resizes/recompresses the image client-side via
  `<canvas>` (capped at 400px on the longer side, re-encoded as JPEG) before upload, rather
  than relying solely on the 2MB size-limit backstop. The "no permission to call
  DriveApp.Folder.createFile" upload error some uploads hit is fixed on the
  `app-script-backend` side only (an authorization/scope issue, not app code) — see that
  repo's changelog for the fix and the one-time re-authorization step it still needs.
- **2026-09-18** — Mirrored from `../app-script-backend/ui.html`: three follow-up fixes —
  New & Returning Players' hover/click area is now scoped to just its tinted names box (not
  the whole outer card), matching New/Departed; the player list's scrollbar thumb fades in
  only while actively scrolling instead of staying permanently visible; and `callLeagueApi_`
  (every admin write, including photo uploads) now shows "Could not reach the server — please
  try again." instead of a raw `Unexpected token '<' ... is not valid JSON` error when a write
  hits the same occasional Google-edge 404 that reads already retry around. See that changelog
  for full detail.
- **2026-09-18** — Mirrored from `../app-script-backend/ui.html`: the player profile popup
  gained an admin-only **Upload Photo** button (hidden unless `isAdmin()`), backed by a new
  `uploadPlayerPhoto` write action on the Apps Script side. 2MB max, enforced both client- and
  server-side. On success, updates `DATA.playerPhotos` and patches the popup's avatar plus any
  already-rendered Dashboard card for that player directly, rather than a full reload. See that
  changelog for full detail.
- **2026-09-18** — Mirrored from `../app-script-backend/ui.html`: New & Returning Players'
  chevron moved into its tinted names box (matching New/Departed, which already had it there
  instead of in the untinted title row), and `.balance-chevron` now always reserves its 18px
  even on a dead-link row, so Outstanding/Credits/Monthly Records' amount column no longer
  jumps out of alignment when a row isn't a link.
- **2026-09-18** — Mirrored from `../app-script-backend/ui.html`: `loadData()` and
  `fetchLeagues_()` now retry (up to 3 attempts, 900ms/1800ms backoff) via a shared
  `fetchJsonWithRetry_()` before showing the fatal-error screen, since `script.google.com`'s
  `/exec` URLs occasionally bounce a request with a plain Google 404 page before the Apps
  Script backend ever runs — reported by the user as an occasional error, reproduced directly
  against the live URL. GET-only; write requests aren't retried. See that changelog for detail.
- **2026-09-18** — Mirrored from `../app-script-backend/ui.html`: seven fixes/features —
  search-clear (×) button no longer stuck hidden after a delta-box link, the "Updated" status
  dot restyled so it reads as an indicator instead of stray misalignment, Monthly Records
  rebuilt as `.balance-row` cards (same All-Players link/dead-link rule as Outstanding/
  Credits) in place of a plain table, League Draft's Export/Copy buttons restyled to match
  Monthly Records, League Draft/Team Manager team cards given tone-colored borders/zebra rows/
  jersey badges, About Us's "Good People/Better Basketball/Stronger Community" tiles removed
  with real card depth added to the value and info boxes instead, and a new player profile
  popup (photo/position/stats, openable only from the Dashboard) backed by player photos now
  read from the same Drive folder as the yearly cash-record sheets. See that changelog for
  full detail.
- **2026-09-18** — Mirrored from `../app-script-backend/ui.html`: shared page headers (icon
  badge + title + subtitle + updated stamp) on every page, Outstanding/Player Credits
  rebuilt with a search + Export toolbar, chevron rows that link into All Players where the
  player exists, closing note banners, and Monthly Records' action buttons restyled.
- **2026-09-18** — Mirrored from `../app-script-backend/ui.html`: About Us rebuilt (full-bleed
  artwork, big headline + script tagline, three value columns, script tiles, info panel, CTA,
  brand footer) and Internal Leagues rebuilt (champions banner, jersey-badged team cards,
  standings with highlighted leader and coloured Diff, winner-tinted scores, league footer).
  See that changelog for detail, including the note that the mockup's three photos are stood
  in for by tinted panels.
- **2026-09-18** — Mirrored from `../app-script-backend/ui.html`: fixed the search clear
  button's icon being positioned outside its circle (the magnifier rule was catching it),
  gave the New & Returning box a soft blue tint distinct from the green "New" box, and
  left-aligned Monthly Records' month picker.
- **2026-09-18** — Mirrored from `../app-script-backend/ui.html`: tinted names box in the New
  & Returning card, people icon on "Member/Regular Changes", and Monthly Records' month
  picker restyled to match the Dashboard's pill.
- **2026-09-18** — Deleted the four files from the two superseded green-jersey artwork
  versions (`about-us-v2.jpg`/`og-image-v3.jpg` — the "INTERNAL LEAGUE 2025" draft — and
  `about-us-v3.jpg`/`og-image-v4.jpg` — the jersey line-up), neither of which was referenced
  any more. The maroon-jersey originals (`about-us.jpg`, `og-image-v2.jpg`) are still kept as
  the archive, as is `about-us-v4.jpg`/`og-image-v5.jpg` in use now.
- **2026-09-18** — Swapped the About Us / link-preview artwork again, to
  `images/about-us-v4.jpg` + `images/og-image-v5.jpg` — a different illustration (five
  players seated on court under an "INTERN LEAGUE 2026" banner, "More People Better
  Basketball" / "Makassar Hoops Community" taglines) chosen over the jersey line-up. New
  filenames again for cache-busting. Note the artwork's banner reads "INTERN LEAGUE", not
  "INTERNAL LEAGUE" — flagged to the user, who may replace the file later; nothing in the
  code depends on that text. Superseded files (`about-us.jpg`, `og-image-v2.jpg`,
  `about-us-v2.jpg`, `og-image-v3.jpg`, `about-us-v3.jpg`, `og-image-v4.jpg`) are all kept
  unreferenced as archive.
- **2026-09-18** — Replaced the About Us / link-preview artwork added in the entry below with
  `images/about-us-v3.jpg` + `images/og-image-v4.jpg`: that entry used a superseded draft of
  the same illustration ("INTERNAL LEAGUE 2025", no tagline), and the intended version reads
  "INTERNAL LEAGUE 2026" with "Good People Better Basketball". New filenames again rather
  than overwrites, so WhatsApp/Cloudflare don't serve the cached draft. The short-lived
  `about-us-v2.jpg`/`og-image-v3.jpg` pair is left in place unreferenced alongside the
  maroon-jersey originals — nothing points at either now.
- **2026-09-18** — New artwork for the About Us photo and the WhatsApp/social link preview:
  `images/about-us-v2.jpg` (1200px, ~277KB) and `images/og-image-v3.jpg` (1200x800, ~281KB),
  both from the green-jersey version of the team illustration. `og:image`, `twitter:image`
  and the `SportsOrganization` structured data's `logo`/`image` all point at the v3 file — a
  new filename rather than an overwrite, which is what busts WhatsApp's preview cache (same
  reason as the 2026-08-31 og-image-v2 rename). The old `about-us.jpg` and `og-image-v2.jpg`
  are deliberately kept in the repo, unreferenced, as an archive of the maroon-jersey
  artwork. `../app-script-backend/ui.html` references the about-us file cross-origin, so this
  repo must be pushed before that one is deployed.
- **2026-09-18** — Mirrored from `../app-script-backend/ui.html`: a clear (×) button plus
  Escape shortcut for the Dashboard search box, and the 12-month period line restyled as an
  accent-tinted chip beside the month picker. See that changelog for detail.
- **2026-09-18** — Mirrored from `../app-script-backend/ui.html` (see that changelog for
  detail): centered banner wordmark, `<select>` month pickers, rail starting below the banner
  on desktop and hidden behind the hamburger on phones, "equiv. weekly visits" wording, a
  visible tab strip, and New/Returning + New/Departed boxes that link into All Players
  pre-filtered to their own names.
- **2026-09-18** — Mirrored from `../app-script-backend/ui.html` (see that changelog for
  detail): added `images/header-bg.jpg` (the banner's photo, also loaded cross-origin by
  `ui.html`), restored the original logo artwork for "Happy Soda" and the supplied HS
  monogram as CSS masks, harmonized corner radii (10/7/5px), added a "More players" scroll
  cue, and made admin login check the passcode with the server first (`verifyAdmin`). The
  backend must be deployed before this is pushed, or even the correct passcode is rejected.
- **2026-09-18** — Added "Step 0" to the Workflow section: fetch/pull both repos before
  analysing or changing any code, and again before committing/deploying. See
  `../app-script-backend/CLAUDE.md` for the stale-checkout deploy that prompted it.
  Documentation only.
- **2026-09-18** — Mirrored the re-theme follow-ups from `../app-script-backend/ui.html`: trend
  badge now compares last month with the month before and colours green/red/yellow for
  up/down/no change, "equiv. visits" shortened wording, banner tagline "For The Love of The
  Game". See that repo's changelog.
- **2026-09-18** — Green re-theme (hero banner, icon side rail, viewport-contained Dashboard
  player list with pinned search/tabs/heading, avatar player rows, new chart header). Applied
  identically to `../app-script-backend/ui.html`; see that repo's CLAUDE.md changelog (same
  date) for the full write-up. Specific to this file: the hero is static markup ahead of
  `.wrap`, `buildDashboard()` calls `hydrateIcons_(content)` right after injecting its
  markup, and the rail/hamburger stay inside that built markup as before.
- **2026-09-11** — Branding/SEO pass, requested so people searching "happy soda basketball"
  can actually find `https://happysoda.pages.dev/` (this repo's live URL — see
  `[web-frontend is primary]` in project memory for why this file, not `ui.html`, is the one
  that matters for public discovery).
  - **`<title>`**: `Happy Soda - Home` → `Happy Soda Basketball Community – Makassar` — the
    old title had no keywords a search engine or a searcher would actually type.
  - **Added a plain `<meta name="description">`** (there was previously only `og:description`,
    which link-unfurlers use but Google's own search-result snippet does not always fall back
    to) mentioning "basketball community," "Makassar," and the weekly schedule.
  - **Added `<link rel="canonical">`**, `<html lang="en">`, Twitter Card meta tags (`summary_large_image`,
    reusing the existing `og:image`), and a `SportsOrganization` JSON-LD block (name, sport,
    Instagram `sameAs`, and a `location`/`PostalAddress` pinned to Makassar, South Sulawesi, ID)
    so search engines have structured confirmation this is a real basketball community based in
    Makassar, not just page text.
  - **Added `robots.txt`** (allow-all + `Sitemap:` pointer) and a minimal **`sitemap.xml`**
    (single URL, the homepage) — this repo previously had neither. Google Search Console
    verification was already done before this session (`google1e7f278898037151.html` was
    already present); submitting the new sitemap there is a manual follow-up outside what this
    session can do.
  - **Bigger logo + a persistent tagline**: `.logo` height 92px → 140px, and added a
    `.site-tagline` ("Basketball Community · Makassar") between the logo and the `<h1>` — the
    `<h1>` itself (`#pageTitle`) is dynamic per-page ("Loyalty Board", "Team Manager", etc.), so
    it was never a place to put static branding; the tagline is the static text instead, and
    happens to also put "Basketball Community" and "Makassar" as real visible/indexable page
    text rather than only inside `<head>` meta.
  - **About Us copy**: now explicitly says "based in Makassar, Indonesia" in the intro
    paragraph, and the location link text is now "SLK Basketball Court, Makassar" instead of
    just the venue name — both were previously implied only by an unlabeled Google Maps link,
    which contributes nothing to a page's indexable text.
  Mirrored the non-SEO-specific parts (bigger logo, `.site-tagline`, About Us Makassar wording)
  into `../app-script-backend/ui.html` and its `doGet()` `setTitle()` call, since those are
  visible UI/content, not metadata specific to this repo's publicly-shared URL. The `<head>`
  SEO tags (meta description, canonical, JSON-LD, Twitter Card, `robots.txt`/`sitemap.xml`)
  were **not** mirrored — `ui.html` is served from the Apps Script `/exec` URL inside a
  sandboxed iframe, isn't the link the community shares, and isn't what should show up for a
  "happy soda basketball" search, matching the existing precedent for `og:*` tags (see the
  2026-08-31 entry below). Not yet verified against a live Google re-crawl (that takes time
  regardless); verified the JSON-LD parses as valid JSON and the page still renders correctly.
- **2026-08-31** — Mirrored from `../app-script-backend/ui.html`: added a helper line under
  Team Manager's "X / 10 selected" counter ("Select exactly 10 players to assemble two
  balanced teams.") — worded around the real constraint (exactly 10, not a minimum; the
  Assemble Teams button only enables at exactly 10 selected) after the initial request asked
  for "minimum" phrasing that didn't match the actual behavior. New `.tm-selection-hint` CSS.
- **2026-08-31** — Switched the About Us photo (`images/about-us.jpg`) from the portrait crop
  to the landscape one — same artwork as `images/og-image-v2.jpg`, just copied over the file
  rather than referenced, so the two stay independently editable if their needs ever diverge
  (og-image tuned for link-preview size/cache-busting, about-us tuned for in-page display).
  Reasoning: `.about-photo` is `width: 100%` inside the About Us card, so the portrait version
  forced a lot of scroll before reaching the actual bio text, and its vertically-stacked
  characters read as more cropped/overlapping than the landscape composition. No HTML changed
  in either this repo or `../app-script-backend/ui.html` — both already reference this same
  filename (the latter cross-origin), so the swap took effect just by replacing the file.
- **2026-08-31** — Renamed `images/og-image.jpg` → `images/og-image-v2.jpg` (and repointed
  `og:image` at it) purely to bust WhatsApp's preview cache. Confirmed the previous entry's
  fix was live and correct — `curl`ing the site showed `og:image` already pointed at the
  1200×800 landscape file, and the file at that URL really was 1200×800 — but WhatsApp still
  showed the old portrait image even after changing the *page* URL with a `?v=` query string.
  That's because WhatsApp's preview proxy appears to cache the **image** by its own URL,
  independent of whatever page URL referenced it, so changing the page URL alone can never
  bust a stale image the proxy already fetched — only a new image URL does. This is invisible
  to end users: the page URL people actually share (`https://happysoda.pages.dev/`) is
  completely unchanged, only the internal `og:image` filename changed. Worth remembering for
  any future og-image swap: the filename needs to change (or gain a version suffix) every
  time, not just the file contents, or WhatsApp may keep serving whatever it cached under the
  old filename.
- **2026-08-31** — Replaced both artwork images with newly redrawn versions (jersey fixes)
  and moved them into a new `images/` folder as real files instead of base64: `images/og-image.jpg`
  (1200×800 landscape, for the WhatsApp/Open Graph preview — supersedes the pillarboxed
  version from the entry below, which looked bad in practice once actually tested in
  WhatsApp) and `images/about-us.jpg` (800×1200 portrait, for the About Us page's
  `.about-photo`). The previous pillarbox approach is now moot since the replacement artwork
  is natively landscape, no padding needed. Deleted the old root-level `og-image.jpg`.
  Replacing the About Us page's inline `data:image/jpeg;base64,...` `<img src>` with a plain
  `images/about-us.jpg` reference dropped `index.html` from ~321KB to ~179KB — a real file
  is also independently browser-cacheable, unlike a blob baked into the page's own markup.
  `og:image`/`og:image:width`/`og:image:height` updated to match (`images/og-image.jpg`,
  1200×800). Not mirrored into `../app-script-backend/ui.html` — that page still embeds the
  team photo as base64 since it's server-rendered by Apps Script with no static file hosting
  of its own; it *could* instead reference `https://happysoda.pages.dev/images/about-us.jpg`
  cross-origin (a plain `<img src>`, not a fetch, so the sandboxed iframe Apps Script renders
  it in shouldn't block it) but that's a separate change, not done here.
- **2026-08-31** — Fixed `og-image.jpg` rendering as a narrow WhatsApp preview card: the
  source photo is portrait (700×1050), and link-preview cards size themselves to the image's
  aspect ratio, so the whole card came out tall and narrow. Regenerated it at the standard
  1200×630 Open Graph landscape ratio by centering the untouched artwork on a solid-color
  canvas (pillarboxed left/right) rather than cropping — cropping would have cut off the
  jumping figure at top or the group's feet at bottom, since the illustration is a vertically
  stacked composition with no good landscape crop. The pillarbox color (`#5A1E37`) was sampled
  directly from the artwork's own maroon accents (the "Happy Soda" logo tag and the right-hand
  banner), so the padding blends in rather than looking like an obvious border. Also added
  `og:image:width`/`og:image:height` meta tags alongside the existing Open Graph tags so
  clients don't need to fetch the image just to learn its dimensions. Verified visually by
  reading the regenerated file back.
- **2026-08-31** — Added Open Graph tags (`og:title`/`og:description`/`og:image`/`og:url`/
  `og:type`) to `<head>` so pasting `https://happysoda.pages.dev/` into WhatsApp (or any other
  link-unfurling client) shows a preview card instead of a bare link — requested when the user
  started sharing the URL in the community group. `og:image` needs a plain fetchable URL, not
  a `data:` URI (unfurlers generally won't fetch/embed those), so the same team photo already
  embedded as a base64 `data:image/jpeg` in the About Us page (`.about-photo`) was decoded out
  to a real file, `og-image.jpg` (700×1050, ~105KB), committed alongside `index.html` — this is
  the first non-`index.html` file in this repo. Confirmed via this session that the live site
  is hosted on Cloudflare Pages at that URL (the file's hosting note above was previously an
  unconfirmed guess at GitHub Pages), so `og:image`/`og:url` are hardcoded to it the same way
  `APPS_SCRIPT_URL` is hardcoded elsewhere in this file — if the Cloudflare Pages domain ever
  changes, both need updating together. Not mirrored into `../app-script-backend/ui.html`:
  that page is rendered at the Apps Script `/exec` URL, which isn't the link the community
  actually shares (see `[web-frontend is primary]` in project memory), so there was nothing to
  preview there. Not yet verified against a real WhatsApp unfurl (these are commonly
  aggressively cached per-URL by the client once fetched — some clients don't easily refresh an
  already-shared link's cached preview) — worth checking by pasting the URL fresh, or using a
  cache-busting query string, before relying on it in the field.
- **2026-08-31** — Mirrored from `../app-script-backend/ui.html`: added a first-load progress
  bar. Unlike `ui.html` (which had no visible markup at all before its data fetch resolves, so
  it uses a full-screen overlay), this page's header/logo already render immediately as static
  HTML — only `#content` showed a plain "Loading the latest player data…" placeholder — so the
  bar replaces that placeholder inline (`.initial-load-progress` / `#initialProgressFill`)
  instead of covering the whole page. Simulated fill (eases to 90%, holds, snaps to 100% on
  arrival — no real per-response progress signal exists to track). Wired into `loadData()`'s
  existing success/error paths via a `firstLoadTicker` guard so only the very first call is
  affected; every other `loadData()` caller (month-picker changes, league actions, the 15-min
  auto-refresh) already shows `#loadingOverlay`'s spinner instead, since `#content` holds the
  real dashboard by then. Removed the now-dead `.loading` CSS rule. This was originally skipped
  as "not the reported problem" when added to `ui.html` — wrong call, since this file (not
  `ui.html`'s Apps Script URL) is the page actually used day to day; see
  `../app-script-backend/CLAUDE.md`'s changelog for the fuller story. Verified with a local Node
  HTTP server standing in for the Apps Script backend and a real headless-Chrome session.
- **2026-08-30** — Mirrored from `../app-script-backend/ui.html`: reverted the single-column
  League Draft export (tall images suffer most from WhatsApp's resize) — back to the
  2-column grid at 732px, with the `ld-cols-*` class and `ldCaptureWidth_()` removed. The
  larger export type is kept and raised: player names 23px, and the rest to match.
- **2026-08-30** — Mirrored from `../app-script-backend/ui.html`: the League Draft export
  image is now a single 420px column for up to 4 teams (`ld-cols-1` class + new
  `ldCaptureWidth_()`, replacing the flat 732px capture width) and 560px/2 columns for 5, with
  larger export type — so the player names hold up on a phone without zooming. Export-only;
  the on-screen page is unchanged. See that repo's Changelog for the reasoning.
- **2026-08-30** — Mirrored from `../app-script-backend/ui.html`: the Monthly Records export
  image now flows its sections into 3 balanced columns (`columns: 3` + `break-inside: avoid`)
  instead of a 2-column grid, drops empty weeks from the image (new `fillRecordsPanel()` +
  `data-empty`), and tightens the table sizing with wrapping instead of ellipsized names. All
  export-only; the on-screen page is unchanged. See that repo's Changelog for the rationale
  and verification.
- **2026-08-30** — Mirrored from `../app-script-backend/ui.html`: the Monthly Records and
  League Draft exports now get the same gutter + rounded-card frame as the finance one.
  Shared `.export-card` class, gutter applied to all three capture roots only under
  `body.exporting-records`, grid rules moved onto the inner card, and the capture-time widen
  went 700px → 732px to keep the card at its previous width.
- **2026-08-30** — Mirrored from `../app-script-backend/ui.html`: the Outstanding &
  Player Credits export image had its credits columns overflowing the card, so
  `financeExportCapture` now joins `recordsExportCapture`/`ldExportCapture` in
  `exportImageFromElement`'s `needsWiden` 700px capture-time widening, and the fee cells
  are `white-space: nowrap`.
- **2026-08-30** — Mirrored from `../app-script-backend/ui.html`: the Outstanding &
  Player Credits export image's border was invisible at the image edge (html2canvas crops
  to the capture root's border box), so `#financeExportCapture` is now an opaque
  `var(--bg)` gutter and the bordered/rounded card is a new inner `.finance-export-card` —
  markup emitted by `buildDashboard()`.
- **2026-08-30** — Mirrored from `../app-script-backend/ui.html`: `#financeExportCapture`
  (the Outstanding & Player Credits export image) now gets the same rounded-card treatment
  `#recordsExportCapture` and `#ldExportCapture` already had — panel background, border,
  `var(--radius)` corners, 14px padding.
- **2026-08-30** — Mirrored from `../app-script-backend/ui.html`: Outstanding Payments now
  renders its fees (and total) in red and Player Credits in green, via a new `amountClass`
  argument to `renderBalanceList` plus a `--red` palette var (`#e0715c` dark / `#c0392b`
  light) that `.finance-table td.outstanding-cell` now also uses; and the Dashboard's tier
  tags (`renderRegulars`' Hybrid/Loyal, `renderPerks`' Member/Regular) moved to before the
  player name. See that repo's Changelog for the full rationale.
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
