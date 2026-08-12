# National Park Webcams

A single-file, dependency-free dashboard for viewing U.S. National Park Service
webcams. Browse live static webcam images across 14 national parks, grouped by
area, with global favorites, adjustable auto-refresh, and full-resolution image
saving.

## Usage

Open [index.html](index.html) in any modern browser. That's it — no build step,
no dependencies, no server required.

## Features

- **14 parks**, cameras grouped by area, in an alphabetical park switcher.
- **Grid + detail views** with a minimalist dark design.
- **Auto-refresh** from 1s to 60s, with a live countdown.
- **Global favorites** (pin cameras from any park together), saved in
  `localStorage`.
- **Save image** — download the exact frame you're viewing (native Save As
  dialog where supported).
- **Resilient loading** — preload-then-swap plus retry-with-backoff so a slow
  or failed fetch never wipes the current frame.

## Adding cameras

Edit the `PARKS` array in [index.html](index.html). Each camera is
`{ id, name, url }`. See [AGENTS.md](AGENTS.md) for details and conventions.

## Documentation

- [AGENTS.md](AGENTS.md) — architecture and conventions for contributors/agents.
- [docs/DEVELOPMENT_HISTORY.md](docs/DEVELOPMENT_HISTORY.md) — how the project
  was built.

## Note on sources

Camera images come from official NPS pages. Some feeds are hosted by NPS
partners (e.g. `glacier.org`, `pixelcaster.com`, `volcanoes.usgs.gov`) and are
included only because the official NPS park page links directly to them. This
project is not affiliated with the National Park Service.
