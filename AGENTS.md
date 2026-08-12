# AGENTS.md

Context for AI agents working on this repository.

## Project

A single-file, dependency-free dashboard for viewing U.S. National Park Service
(and NPS-linked) webcams. Everything lives in [index.html](index.html) — HTML,
CSS, and vanilla JavaScript in one file. No build step, no framework, no
package manager. Open the file in a browser to run it.

## Architecture

All application code is inside the single IIFE `<script>` at the bottom of
[index.html](index.html).

### Data model
- `PARKS` — the source of truth. An array of `{ park, areas[] }`.
  - Each area is `{ area, cameras[] }`.
  - Each camera is `{ id, name, url }` where `id` is a stable unique slug,
    `name` is the display label, and `url` is the base image URL (no query
    string).
- `ALL_CAMERAS` / `camById(id)` — flat global lookups across every park.
- `currentCameras()` — cameras for the currently selected park only.
- `parkIndex` / `AREAS` — current park selection state.

### Views
- **Grid view** — landing page. Cameras grouped by area as cards. A global
  **★ Favorites** section renders first when any favorites exist.
- **Detail view** — one large "stage" image plus a sidebar of other cameras
  (grouped by area, favorites first). Includes Refresh, Save image, and
  Favorite buttons.

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

## Conventions & gotchas
- Keep everything in the single file; do not introduce a build system or
  dependencies unless explicitly requested.
- When toggling visibility via the `hidden` attribute, remember a CSS
  `display` rule on a class will override `[hidden]`. The `.imgError` overlay
  needed an explicit `.imgError[hidden] { display: none; }` rule for this
  reason.
- Image `id`s are used as favorite keys and for cross-park lookup — do not
  reuse or casually rename them, or saved favorites break.
- Camera image URLs are resolved from the JS-rendered `#webcamRefreshImage`
  element on each NPS `media/webcam/view.htm?id=...` page (the filenames rarely
  match the display names). Some feeds are hosted off `nps.gov` (e.g.
  `glacier.org`, `pixelcaster.com`, `volcanoes.usgs.gov`); these are included
  only because the official NPS park page links directly to them.

## Adding a park or camera
Add an entry to `PARKS`. That's it — the grid, sidebar, park dropdown
(alphabetically sorted), refresh loop, and favorites all derive from it.
Verify each new `url` actually returns an image (some NPS paths 403 for
hotlinking or return an S3 XML error for wrong filenames).
