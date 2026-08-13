# National Park Webcams

A single-file, dependency-free dashboard for viewing U.S. National Park Service
(and NPS-linked) webcams — refreshing still images **and** live video streams
across ~35 national parks, with a home overview, global favorites, adjustable
auto-refresh, fullscreen viewing, and full-resolution image saving.

## 🔴 Live Page

**[roundingerror.github.io/nps-webcam-viewer](https://roundingerror.github.io/nps-webcam-viewer/)**

Served via GitHub Pages. 

## Usage

Open [index.html](index.html) in any modern browser. That's it — no build step,
no dependencies, no server required.

- **Deep links:** the URL hash is shareable — `#home`, `#<park>`, or
  `#<park>/<camera>` opens straight to that view.
- **Keyboard:** `/` focuses search; on a camera, **←/→** step between cameras,
  **f** toggles fullscreen, **Esc** goes back.
- **Running locally:** open the file directly, or serve it
  (`python -m http.server`) so YouTube live streams embed inline instead of
  opening on youtube.com.

## Features

- **Home overview** — hero with live stats, global favorites, and an "All parks"
  grid of cards with live preview thumbnails and camera counts.
- **~35 parks / 200+ feeds** — refreshing still images and live video streams,
  grouped by area.
- **Status badges** — Loading → Online / Live / Air Quality / Stale / Offline,
  plus Stream for video. Cameras that fail to load are detected at runtime and
  can be hidden via **Hide offline**.
- **Detail view** — large stage plus a collapsible "Other cameras" sidebar;
  fullscreen toggle, favorite, refresh, save, and share.
- **Auto-refresh** 1s–60s with a live countdown; pauses while the tab is hidden.
- **Global favorites** (pin cameras from any park together), saved in
  `localStorage`.
- **Save image** — download the exact frame you're viewing (native Save As
  dialog where supported).
- **Share** — native share sheet on mobile, clipboard copy elsewhere.
- **Per-camera / per-park NPS links** — jump to the official NPS page to check a
  camera's status.
- **Responsive** — a mobile layout that pins the current camera and scrolls the
  suggestions beneath it.
- **Resilient loading** — lazy-loading, preload-then-swap, and
  retry-with-backoff so a slow or failed fetch never wipes the current frame.

## Adding cameras

Edit the `PARKS` array in [index.html](index.html). Each camera is
`{ id, name, url?, off?, type?, embed?, live?, hero? }`. See
[AGENTS.md](AGENTS.md) for the data model, the NPS Solr catalog API, and
conventions.

## Documentation

- [AGENTS.md](AGENTS.md) — architecture and conventions for contributors/agents.
- [docs/DEVELOPMENT_HISTORY.md](docs/DEVELOPMENT_HISTORY.md) — how the project
  was built.

## Note on sources

Camera images come from official NPS pages. Some feeds are hosted by NPS
partners (e.g. `glacier.org`, `pixelcaster.com`, `volcanoes.usgs.gov`,
`hdontap.com`) and are included only because the official NPS park page links
directly to them. This project is not affiliated with the National Park Service.
