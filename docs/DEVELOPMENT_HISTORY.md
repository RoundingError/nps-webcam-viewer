# Development History

A narrative record of how this project was built, captured from the original
AI-assisted coding session. Intended as context for future work and for AI
agents. Newest features build on earlier ones.

## 1. Origin: a single auto-refreshing webcam
The project began as a viewer for one NPS webcam page (Sunrise Mountain, Mount
Rainier: `media/webcam/view.htm?id=81B4638D-...`). The NPS page auto-refreshes
an `#webcamRefreshImage` every 60 seconds using a cache-busting `?<timestamp>`
query parameter. The first build was a standalone `index.html` with an editable
image URL, a refresh interval, start/stop, and a countdown.

## 2. Finding the real image URL
The image `src` is set by JavaScript, so it isn't in the page's static HTML.
- First guess `sunrise.jpg` returned **403 (S3 "Access Denied", XML body)** →
  triggered the browser's ORB (Opaque Response Blocking) because an image
  request received `application/xml`.
- The real object turned out to be `SunriseMtn.jpg`. Lesson: resolve the true
  URL from the live DOM's `#webcamRefreshImage`, since filenames don't match
  display names.

## 3. Second camera + dashboard redesign
Added `SunriseLot.jpg`, then redesigned into a minimalist dark dashboard:
- **Grid preview** of all cameras → click a card for a **detail view** with a
  large image and a sidebar of the other cameras.
- Modern dark theme, custom scrollbars.

## 4. Reliability fixes
- **"Unavailable" overlay bug**: the `.imgError` overlay used `display: flex`,
  which overrode the HTML `hidden` attribute, so it was always painted over a
  perfectly good image. Fixed with `.imgError[hidden] { display: none; }`.
- **Transient failures**: switched to **preload-then-swap** — load each new
  frame in an off-screen `Image` and only swap it in on success, plus retry
  with capped backoff. A hiccup never wipes the current frame now.

## 5. Faster camera switching
Swapping cameras took ~500ms because it waited on a network fetch. Added a
`lastGoodSrc` cache so a swap shows the cached frame instantly (~20ms) and
refreshes in the background.

## 6. Refresh control
Changed the refresh input to a dropdown. Later reduced the minimum to 1s
(kept 2s, 5s, … 60s).

## 7. Grouping by area
Introduced an area-grouped data model (`AREAS`) with section headings in both
the grid and the detail sidebar. Added Mount Rainier's Paradise-area cameras
(`mountain.jpg`, `east.jpg`, `west.jpg`, `gh.jpg`, `tatoosh.jpg`, and the ARD
`mora.jpg`). Corrected labels per user: `gh` = Guide House, ARD `mora` = Air
Quality, `mountain` = Paradise Mountain.

## 8. Save full-res image
Added a **Save image** button. Iterations:
1. Fetch freshest frame → download blob.
2. Changed to save the **exact frame currently displayed** (uses
   `stageImage.src`, `force-cache`) so what you see is what you save, and
   swapped the button order (Refresh, then Save).
3. Added native **Save As** dialog via `showSaveFilePicker()`, with fallbacks.

## 9. Favorites
- Star icon on each thumbnail + a Favorite toggle on the detail page.
- Persisted in `localStorage` (`nps-webcam-favorites`), keyed by camera ID.
- A **★ Favorites** section pins picks to the top of the grid and sidebar.
- Later made **global across parks**: favorites from any park show together,
  each labeled `Park · Area`; clicking one switches park context automatically.

## 10. Multiple parks
Refactored the data model to `PARKS[] → areas[] → cameras[]` with a park
switcher in the header. Added Yellowstone (9 cams), then researched and added
many more by resolving each `#webcamRefreshImage` URL from the live NPS pages.

## 11. Source policy iterations
- First restricted to **only `nps.gov`** feeds (removed third-party-hosted cams
  and parks that only had them).
- Then relaxed to **allow external hosts, but only when the official NPS park
  page links directly to them** (e.g. `glacier.org`, `pixelcaster.com`,
  `volcanoes.usgs.gov`, `barharborcam.com`).

## 12. Header + ordering polish
- Park dropdown sorted **alphabetically** (option values still map to original
  `PARKS` indices).
- Made the **park name prominent** in the header (large H1 with a small
  "National Park Webcams" eyebrow).

## Current parks
Acadia, Crater Lake, Denali, Glacier, Grand Canyon, Grand Teton, Great Smoky
Mountains, Hawaiʻi Volcanoes, Mount Rainier, Olympic, Rocky Mountain,
Yellowstone, Yosemite, Zion.

## Known caveats
- Air-quality (ARD) cams (Grand Teton, Olympic, Denali, Great Smoky) update
  roughly every ~15 min, not every 60s.
- Some parks' live *video* streams (Old Faithful, Teton Range) and inactive
  seasonal cams (Yosemite Tuolumne Meadows) were intentionally excluded — they
  aren't static refreshing JPGs.
- `Save image` may fall back to opening a new tab for cross-origin feeds.
