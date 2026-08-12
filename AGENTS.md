# AGENTS.md

Context for AI agents working on this repository.

## Project

A single-file, dependency-free dashboard for viewing U.S. National Park Service
(and NPS-linked) webcams. Everything lives in [index.html](index.html) — HTML,
CSS, and vanilla JavaScript in one file. No build step, no framework, no
package manager. Open the file in a browser to run it.

Covers ~35 National Parks (~200+ feeds): refreshing still images **and** live
video streams, each tagged Online / Live / Daytime / Stale / Offline / Stream.
The camera
list was seeded from the NPS webcam catalog (see "Camera catalog & the NPS Solr
API" below).

## Architecture

All application code is inside the single IIFE `<script>` at the bottom of
[index.html](index.html).

### Data model
- `PARKS` — the source of truth. An array of `{ park, areas[] }`.
  - Each area is `{ area, cameras[] }`.
  - Each camera is `{ id, name, url?, off?, type?, embed?, live? }`:
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
- `ALL_CAMERAS` / `camById(id)` — flat global lookups across every park.
- `currentCameras()` — cameras for the currently selected park only.
- `parkIndex` / `AREAS` — current park selection state.
- Parks may be appended in any order: the dropdown sorts alphabetically but
  keeps each option's value as its original `PARKS` index.

### Views
- **Grid view** — landing page. Cameras grouped by area as cards. A global
  **★ Favorites** section renders first when any favorites exist. A search box
  and filter chips (**Hide offline**, **Streams**) live in the top bar.
- **Detail view** — one large "stage" (image, or an embedded stream player)
  plus a sidebar of other cameras (grouped by area, favorites first). Image
  cameras show Refresh + Save image; streams show a **Watch ↗** link instead.
- Navigating pushes browser history; the URL hash is shareable/deep-linkable
  and Back/Forward work (see "State, URLs & history").

### Key behaviors
- **Refresh** — a 1-second ticker counts down; on reaching zero every visible
  camera is refreshed. Interval is chosen from a dropdown (1s min → 60s).
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
  - **Online** (green) — default for refreshing still-image cameras.
  - **Live** (green) — image cameras that also expose an external live-video
    feed (`cam.live` is set); same green styling as Online, plus a
    **Watch live ↗** button on the detail page.
  - **Daytime** (amber) — NPS Air Resources air-quality cams (URL matches
    `AIRQUALITY_RE` = `featurecontent/ard/webcams`). They refresh only in
    daylight and freeze overnight, so their `name` ends with
    `(Air Quality · daytime only)`.
  - **Stale · Nh/Nd** (amber) — measured frame age exceeds `STALE_MS` (1h). Age
    comes from a throttled `HEAD` read of `Last-Modified`, which is only
    readable cross-origin for hosts that send CORS headers — in practice
    `volcanoes.usgs.gov` and `cdn.pixelcaster.com` (`AGE_READABLE_RE`).
  - **Offline** (red) — `off: true`, or an image that keeps failing to load.
  - **Stream** (blue) — `type: 'stream'`.
- **Search & filters** — the search box matches camera + park name across
  **all** parks (results grouped by park); the **Hide offline** and **Streams**
  chips filter via `passesFilters()`. `refreshAll()` refreshes every visible
  `<img>` (tracked in `imgRefs`), so search results from other parks update too.
- **Session persistence** — selected park + refresh interval are saved to
  `localStorage` (`nps-webcam-state`) and restored on load.

## Streams (live video)
- Grid tiles show a lightweight poster (YouTube thumbnail, or a placeholder)
  with a play button; clicking opens the **detail view** with the player
  embedded inline (`#stageStream` iframe). Helpers: `ytId`, `streamPoster`,
  `streamWatch`, `streamEmbedSrc`, `canEmbedInline`.
- **YouTube live embeds fail from `file://`** with "Error 153" (null origin), so
  `canEmbedInline()` is false for YouTube on `file:` — those open on youtube.com
  instead. Served over **http(s)** they embed inline (with an `&origin=` param).
  angelcam / HDOnTap embeds have no such restriction.
- **angelcam** feeds need a per-visit token (`X-Frame-Options: deny` without it);
  a static embed URL can't carry it, so they were removed.
- **Pixelcaster HLS** streams (e.g. Old Faithful) can't embed without an HLS
  player — instead use their refreshing snapshot JPG
  (`.../snapshots/<name>/latest_1920x.jpg`, found in
  `pixelcaster.com/live/customer/<cam>.json`) as a normal image camera.

## State, URLs & history
- `localStorage`: `nps-webcam-state` = `{ park: <slug>, interval }`;
  `nps-webcam-favorites` = array of IDs.
- URL hash: `#<parkSlug>` (grid) or `#<parkSlug>/<camId>` (detail) — shareable
  and deep-linkable. `initState()` parses it on load (a valid `camId` wins),
  else falls back to `localStorage`, else park index 0.
- Navigation uses `history.pushState`; a `popstate` handler (`applyHashState`)
  re-applies the view so browser **Back/Forward** work. Re-applies are
  suppressed from creating new entries (`suppressHistory`). The in-app
  "← All cameras" button uses `history.back()` when there's app history
  (`history.state.d > 0`), else `openGrid()`, so a deep-linked detail's Back
  drops to the park grid instead of leaving the page.

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
