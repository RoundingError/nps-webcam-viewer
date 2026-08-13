# AGENTS.md

Context for AI agents working on this repository.

## Project

A single-file, dependency-free dashboard for viewing U.S. National Park Service
(and NPS-linked) webcams. Everything lives in [index.html](index.html) — HTML,
CSS, and vanilla JavaScript in one file. No build step, no framework, no
package manager. Open the file in a browser to run it.

Published via GitHub Pages at
<https://roundingerror.github.io/nps-webcam-viewer/> (served from `main`).

Covers ~35 National Parks (~200+ feeds): refreshing still images **and** live
video streams, each tagged Online / Live / Air Quality / Stale / Offline / Stream.
The camera
list was seeded from the NPS webcam catalog (see "Camera catalog & the NPS Solr
API" below).

## Architecture

All application code is inside the single IIFE `<script>` at the bottom of
[index.html](index.html).

### Data model
- `PARKS` — the source of truth. An array of `{ park, areas[] }`.
  - Each area is `{ area, cameras[] }`.
  - Each camera is `{ id, name, url?, off?, type?, embed?, live?, hero? }`:
    - `id` — stable unique slug (also the favorite key / cross-park lookup key).
    - `name` — display label.
    - `url` — base image URL, no query string (image cameras).
    - `off: true` — NPS lists the camera as *Inactive* → red **Offline** badge.
    - `type: 'stream'` + `embed` — a live video feed (YouTube / HDOnTap / …);
      has no `url`. See "Streams" below.
    - `live` — optional URL to an external live-video page for an image camera
      whose stream can't be embedded (token-gated Pixelcaster: Old Faithful,
      Yosemite Falls / Half Dome / El Capitan). Adds a **Watch live ↗** button
      on the detail page and upgrades its badge from **Online** to **Live**.
    - `hero: true` — optional; makes this camera the preferred preview image for
      its park card on the home view (else the first active image is used).
- `ALL_CAMERAS` / `camById(id)` — flat global lookups across every park.
- `currentCameras()` — cameras for the currently selected park only.
- `PARK_CODES` — park name → NPS park code; drives the header title link to
  `nps.gov/<code>/index.htm` (`updateBrandLink`).
- `CAM_PAGES` — camera `id` → its official NPS page (mostly
  `media/webcam/view.htm?id=…`, air-quality cams → `subjects/air/webcams.htm`).
  Built by matching each camera's `url` against the Solr catalog's `Webcam_URL`
  (see "Camera catalog"); drives the detail view's camera-name link (the
  `stageTitle` anchor) so users can open a camera's official page to check its
  status. ~130 of ~140 image cams are covered; unmatched ones show a plain
  (unlinked) name.
- `parkIndex` / `AREAS` — current park selection state.
- Parks may be appended in any order: the dropdown sorts alphabetically but
  keeps each option's value as its original `PARKS` index.

### Views
Three top-level views (`homeView` / `gridView` / `detailView`), toggled by
`setView(name)` which also sets `body[data-view]` (drives which top-bar controls
show). Navigation tiers: **home → park grid → camera detail**.
- **Home view** — the landing page (`openHome` / `buildHome`). A hero banner with
  live stats, a global **★ Favorites** section, and an "All parks" grid of park
  cards (`makeParkCard`): each shows a live preview thumbnail, name, and camera
  count, and opens that park's grid. Preview picking + failure fallback is in
  `homePreviewSrcs` / `loadParkPreview` (see "Home previews").
- **Grid view** — one park's cameras grouped by area as cards. A global
  **★ Favorites** section renders first when any favorites exist. A search box
  and filter chips (**Hide offline**, **Streams**) live in the top bar.
- **Detail view** — one large "stage" (image, or an embedded stream player)
  plus a sidebar of other cameras. Image cameras show Refresh + Save image;
  streams show a **Watch ↗** link instead. The image stage has an overlay
  **fullscreen** toggle (`stageFull`, image cams only) that fullscreens the whole
  `.stage` — image *and* the info/button bar — so the name/Save/Share stay usable
  and small frames scale up (`object-fit: contain`, no crop). Toggle via the
  overlay icon, clicking the image, or the `f` key; `Esc` exits.
  - The sidebar list (`buildSidebar`) groups cameras into **collapsible groups**
    (`.side-group`, header shows name + count + chevron). Area groups default
    expanded (in-session only); the **★ Favorites** group's collapsed state
    persists (`nps-webcam-fav-collapsed`). The group holding the active camera is
    always force-expanded.
- The **"National Park Webcams" eyebrow** (`homeLink`) returns to home from
  anywhere; the back button is contextual (detail → park grid, grid → home).
- Navigating pushes browser history; the URL hash is shareable/deep-linkable
  (`#home`, `#park`, `#park/cam`) and Back/Forward work (see "State, URLs &
  history").

### Key behaviors
- **Refresh** — a 1-second ticker counts down; on reaching zero every visible
  camera is refreshed. Interval is chosen from a dropdown (1s min → 60s). While
  the tab is hidden (`document.hidden`) `tick()` skips refreshing to save
  bandwidth/battery; a `visibilitychange` handler does one immediate refresh and
  restarts the countdown on return.
- **Cache busting** — `bust(url)` appends `?<timestamp>` to force fresh fetches.
- **Preload-then-swap** — `refreshCamera(id)` loads the new frame into an
  off-screen `Image` first and only swaps it into visible `<img>` elements on
  success, so a failed/slow fetch never wipes the last good frame or flashes
  the "Unavailable" overlay.
- **Retry with backoff** — failed loads retry with capped backoff; the
  "Unavailable" overlay only appears after several consecutive failures on a
  camera that has never loaded.
- **Favorites** — global across all parks, persisted in `localStorage` under
  `nps-webcam-favorites` (array of camera IDs). Favorite cards show their
  origin (`Park · Area`). Clicking a favorite from another park switches the
  park context automatically.
- **Save image** — downloads the exact frame currently displayed (not a fresh
  fetch). Prefers the native Save As dialog via `showSaveFilePicker()`, falling
  back to an anchor download, then to opening the image in a new tab.
- **Status badges** — every tile/stage carries a badge (`applyBadge` / `setBadge`):
  - **Loading** (grey, pulsing) — the initial state for every image camera until
    its first frame confirms the feed is up (`!lastGoodSrc[id]`), so we never
    flash "Online" on a camera that turns out to be dead. Streams skip this (we
    can't verify stream liveness) and show **Stream** immediately.
  - **Online** (green) — refreshing still-image cameras, once a frame has loaded.
  - **Live** (green) — image cameras that also expose an external live-video
    feed (`cam.live` is set); same green styling as Online, plus a
    **Watch live ↗** button on the detail page.
  - **Air Quality** (amber) — NPS Air Resources air-quality cams (URL matches
    `AIRQUALITY_RE` = `featurecontent/ard/webcams`). Their update cadence varies
    (some near-live, some daytime-only) and their host sends no CORS headers so
    we can't read a real age — hence a neutral label rather than a freshness
    claim. Some `name`s still end with `(Air Quality · daytime only)`.
  - **Stale · Nh/Nd** (amber) — measured frame age exceeds `STALE_MS` (1h). Age
    comes from a throttled `HEAD` read of `Last-Modified`, which is only
    readable cross-origin for hosts that send CORS headers — in practice
    `volcanoes.usgs.gov` and `cdn.pixelcaster.com` (`AGE_READABLE_RE`).
  - **Offline** (red) — `off: true` (NPS-inactive), or a feed that persistently
    fails to load. The latter are tracked at runtime in a `deadIds` set
    (`isOffline(cam)` = `cam.off || deadIds.has(id)`); a camera is added after it
    exhausts retries without ever loading, and removed if it later succeeds.
  - **Stream** (blue) — `type: 'stream'`.
- **"Hide offline" filter** — `passesFilters` uses `isOffline()`, so it hides
  both `off:true` and runtime-dead cameras. If a camera dies while the filter is
  active, the grid auto-rebuilds so it drops out.
- **Sidebar suggestions** — the detail "Other cameras" list hides offline
  cameras (`isOffline`), except the currently-viewed one. Each side-cam's status
  dot turns red on failure and shows an "Unavailable" overlay (like grid tiles).
- **Lazy loading** — an `IntersectionObserver` (`imgObserver`, `rootMargin`
  400px) tracks which registered `<img>`s are near the viewport. Each ref
  carries a `visible` flag; the observer kicks off the first load when a tile
  scrolls in, and `refreshAll()` only re-fetches cameras with a visible ref, so
  offscreen tiles in big parks don't load or refresh until needed. (Falls back
  to eager loading if `IntersectionObserver` is unavailable.)
- **Last-updated readout** — the detail stage shows `updated Nm ago`
  (`stageUpdated`, via `updateUpdatedLabel`) whenever a real frame age is known,
  i.e. only for the CORS-readable hosts in `AGE_READABLE_RE`. Grid cards in the
  current-park view show the same measured age in their bottom-right `.time`
  label (`ageLabel`); it's left blank when no age is measurable rather than
  claiming "live" for feeds we can't verify.
- **Keyboard nav** — a single `document` `keydown` handler: `/` focuses search;
  in detail view **←/→** (and ↑/↓) step through `navigableCameras()` (current
  park, skipping non-inline streams so no external tab opens), **f** toggles
  fullscreen (image cams), and **Esc** returns to the grid. Ignored while typing
  in an input/select.
- **Share** — the detail bar's **Share** button (icon + label) prefers the native
  share sheet via `navigator.share` (mobile) with the camera name + deep-link
  URL, then falls back to `navigator.clipboard`, then a hidden `textarea` +
  `execCommand('copy')`; shows a transient `Copied!` label on the copy paths.
- **Search & filters** — the search box matches camera + park name across
  **all** parks (results grouped by park); the **Hide offline** and **Streams**
  chips filter via `passesFilters()`. `refreshAll()` refreshes every visible
  `<img>` (tracked in `imgRefs`), so search results from other parks update too.
  Empty results render a friendly message via `gridEmpty()`.
- **Session persistence** — selected park + refresh interval are saved to
  `localStorage` (`nps-webcam-state`) and restored on load.

## Streams (live video)
- Grid tiles show a lightweight poster (YouTube thumbnail, or a placeholder)
  with a play button; clicking opens the **detail view** with the player
  embedded inline (`#stageStream` iframe). Helpers: `ytId`, `streamPoster`,
  `streamWatch`, `streamEmbedSrc`, `canEmbedInline`, `streamHost`,
  `opensExternally`.
- **Non-embeddable streams open in a new tab** rather than inline. `makeCard`
  shows an **Opens on <host> ↗** pill (and subtitle) on such tiles, and the
  detail button reads **Watch on <host> ↗** (`streamHost` names the host).
- **YouTube live embeds fail from `file://`** with "Error 153" (null origin), so
  `canEmbedInline()` is false for YouTube on `file:` — those open on youtube.com
  instead. Served over **http(s)** they embed inline (with an `&origin=` param).
- **HDOnTap** now sends a CSP `frame-ancestors` allowlist (nps.gov / hdontap.com
  only), so its streams can't embed from our origin at all — `canEmbedInline()`
  returns false for any `hdontap.com` embed and they open externally.
- **angelcam** feeds need a per-visit token (`X-Frame-Options: deny` without it);
  a static embed URL can't carry it, so they were removed.
- **Pixelcaster HLS** streams (e.g. Old Faithful) can't embed without an HLS
  player — instead use their refreshing snapshot JPG
  (`.../snapshots/<name>/latest_1920x.jpg`, found in
  `pixelcaster.com/live/customer/<cam>.json`) as a normal image camera.

## Home previews
- Each park card's thumbnail comes from `homePreviewSrcs(park)` — an ordered
  candidate list (`hero` cams → active image cams → inactive image cams →
  stream posters). `loadParkPreview` tries them in order, advancing on load
  failure (`onerror` or `naturalWidth < 2`) and leaving the dark placeholder if
  all fail. This hides cards backed by a dead lead feed.
- **Limitation:** a feed that returns a *valid but black* JPEG (night frame, or
  a powered-off cam still serving an image) can't be auto-detected — canvas
  pixel reads are CORS-blocked for cross-origin images. Use `hero: true` to pin
  a known-good camera for such parks.

## Responsive / mobile (≤820px, phone tweaks ≤560px)
- The top bar's grid-only controls (search, filter chips, Park/Refresh selects)
  are hidden on the **detail** view (`body[data-view]`) to keep the pinned
  header compact; a `--topbar-h` CSS var (tracked via `ResizeObserver`) measures
  the wrapped bar height.
- **Detail is a fixed-height flex column:** the `.stage` is fixed on top and the
  `.sidebar` scrolls internally (mirrors desktop side-by-side), so the current
  camera stays put while browsing others. The `.sidebar-toggle` ("Other cameras
  (N)") is `position: sticky` at the top of that scroller so the list can be
  collapsed/expanded from any scroll position.
- Home uses a denser 2-col park grid; the detail `.stage-img` is capped
  (`max-height`) so the list is always reachable.

## State, URLs & history
- `localStorage`: `nps-webcam-state` = `{ park: <slug>, interval }`;
  `nps-webcam-favorites` = array of IDs.
- URL hash: `#home` (landing), `#<parkSlug>` (grid), or `#<parkSlug>/<camId>`
  (detail) — shareable and deep-linkable. `initState()` parses it on load (a
  valid `camId` wins, then a park slug); with no/`#home` hash it opens the home
  view. Only the refresh interval persists in `localStorage`.
- Navigation uses `history.pushState`; a `popstate` handler (`applyHashState`)
  re-applies the view so browser **Back/Forward** work. Re-applies are
  suppressed from creating new entries (`suppressHistory`). The back button is
  contextual: from a camera → the park grid, from a grid → home. Browser
  Back/Forward still step through history entry-by-entry independently.

## Camera catalog & the NPS Solr API
The authoritative list of every NPS webcam is the Solr endpoint behind
`media/multimedia-search.htm`. Query it **same-origin** from any `www.nps.gov`
page (e.g. `page.evaluate(() => fetch(...))` via the browser tools):

```
https://www.nps.gov/solr/?q=*:*&fq=Type:"Webcam"&fl=*&rows=300&wt=json&defType=edismax
```

Useful fields per doc: `Title`, `Parks` / `Parks_Codes`, `Status`
(`Active`/`Inactive` → `off`), `Is_Streaming` (`true` has no `Webcam_URL` — the
embed lives on its `view.htm` page), `Webcam_URL` (direct image), `PageURL`,
`Latitude`/`Longitude`, `Abstract`. As of Aug 2026: 287 webcams across 78 park
codes (241 active / 46 inactive; 253 image / 34 video).

## Conventions & gotchas
- Keep everything in the single file; do not introduce a build system or
  dependencies unless explicitly requested.
- **PWA (inline / single-file):** a small `<script>` in `<head>` draws the app
  icon on a canvas → PNG data URI and builds a web-app manifest as a blob URL,
  wired up with `apple-touch-icon` + Apple `meta` tags. This makes it
  installable / add-to-home-screen (esp. iOS) without extra files. There's **no
  service worker**, so no Android/desktop install prompt and no offline caching
  (those need a real same-origin `sw.js`, which would break single-file).
- When toggling visibility via the `hidden` attribute, remember a CSS
  `display` rule on a class will override `[hidden]`. The `.imgError` overlay
  needed an explicit `.imgError[hidden] { display: none; }` rule for this
  reason.
- Image `id`s are used as favorite keys and for cross-park lookup — do not
  reuse or casually rename them, or saved favorites break.
- `Webcam_URL` from Solr is the direct image; some feeds are hosted off
  `nps.gov` (`glacier.org`, `pixelcaster.com`, `volcanoes.usgs.gov`,
  `hdontap.com`, …) and are included because the official NPS park page links
  them. (Before the Solr API was found, URLs were scraped from the JS-rendered
  `#webcamRefreshImage` element on each `media/webcam/view.htm?id=...` page.)
- A URL returning `200` + a valid JPEG can still be a **frozen/static photo**
  (e.g. a dated `wp-content/uploads/YYYY/MM/...` path). Check `Last-Modified`
  and/or eyeball day-vs-night before trusting a feed is live.
- `common/.../inactive_webcam.png` is the NPS "offline" placeholder — inactive.
- NPS park webcam pages mostly **don't** group cameras into areas, so new parks
  usually get a single `Cameras` area; the big legacy parks have curated areas.
- Some NPS-listed feeds are traffic/road cams (e.g. `az511.com`,
  `udottraffic.utah.gov`) rather than scenic views.
- The **VS Code Simple Browser** can't render YouTube iframes (shows black) —
  even a known-embeddable control video is black there. Don't judge
  embeddability from it; test image loads and headers instead.

## Adding / auditing cameras
1. Pull candidates from the Solr API (above), filtered to the parks you want.
   `Webcam_URL` is the image; `Status` → `off`; for `Is_Streaming` cams grab the
   embed from the camera's `view.htm` page instead.
2. Add entries to `PARKS` — the grid, sidebar, dropdown, refresh loop,
   favorites, search, and badges all derive from it.
3. **Verify each URL loads as an image** before trusting it: load it as
   `new Image()` from the local `file://` build (no CSP) and check
   `naturalWidth > 1`. `page.request.get` is unreliable — hotlink-protected
   hosts 403 without a browser referer.
4. For "is it live?", read `Last-Modified` (navigate to the URL in the browser
   and inspect the response headers).

This kind of work is driven by the browser tools: hit Solr from an `nps.gov`
page, scrape `view.htm` pages for stream embeds, and validate/preview against
the local `file://` build.
