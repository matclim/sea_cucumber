<!--
SPDX-FileCopyrightText: CERN for the benefit of the SHiP Collaboration
SPDX-License-Identifier: LGPL-3.0-or-later
-->

# sea_cucumber manual

The SHiP event display. It reads a GeoModel geometry database and the official
SHiP RNTuple data model, and renders geometry and events two ways: the built-in
ROOT REve viewer, and a standalone web frontend (three.js) that you fully
control.

This manual covers everything: building, the command-line tools, the view
configuration, and the web interface.

---

## 1. Concepts and architecture

sea_cucumber follows the ALICE O2 event-display pattern (arXiv:2503.00088): the
C++ side is the single source of truth — it loads the `.db`, reads the data
model, resolves colours, and tessellates geometry — and it *produces display
files*; a separate viewer *renders* them. Nothing about the physics or geometry
is re-interpreted at view time.

There are three ways to view:

- **REve** (`sea_cucumber`): the ROOT web viewer, launched from C++.
- **Web frontend** (`web/`): a static three.js page consuming produced JSON.
- **Inspection** (`--inspect`): a text dump of the geometry, no rendering.

The reusable core (`src/`) is split behind two interfaces so the pieces are
independently testable:

- `IEventSource` — yields an event's hits, MC particles, etc.
  (`RNTupleEventSource` reads the `events` RNTuple).
- `IGeometrySource` — emits geometry shapes (`GeoModelGeometrySource` walks the
  GeoModel `.db`; `CachedGeometrySource` replays a pre-built cache).

---

## 2. Building and running

sea_cucumber is a pixi workspace. All workflows go through pixi tasks:

| Task | What it does |
|------|--------------|
| `pixi run build` | Configure + compile (CMake + Ninja). |
| `pixi run test` | Build, then run the CTest suite. |
| `pixi run lint` | Run the pre-commit hooks (formatting, cpplint, REUSE…). |
| `pixi run sea_cucumber …` (aliases `sc`, `run_event_display`, `ED`) | Launch the REve display. |
| `pixi run web-data …` | Produce the web display files into `web/data`. |
| `pixi run web` | Serve `web/` at http://localhost:8080. |
| `pixi run geo-cache …` | Build a geometry cache to speed up REve start-up. |
| `pixi run clean` | Remove the build directory. |

### The REve display

```
pixi run sc --geometry files/ship_geometry.db --data files/llp_display.root \
            --view views/default.toml --event 16
```

Key flags: `--geometry <db>`, `--data <root>`, `--view <toml>`, `--event <i>`,
`--ntuple <name>`, `--scale <f>`, `--logo <dir>`, `--geo-cache <prefix>`.

### The web display

Two steps — produce the data, then serve it:

```
pixi run web-data --geometry files/ship_geometry.db --data files/llp_display.root \
                  --view views/default.toml
pixi run web        # open http://localhost:8080
```

`web-data` flags: `--geometry`, `--data`, `--view`, `--out <dir>` (default
`web/data`), `--events all|<i>`, `--depth <n>` (walk depth, default 4),
`--max-shapes <n>` (cap, default 20000).

---

## 3. Inspecting the geometry

To find where subsystems sit in z (so you can set region windows), dump the
geometry without any data file:

```
pixi run sc --geometry files/ship_geometry.db --inspect 4
pixi run sc --geometry files/ship_geometry.db --inspect 6 --inspect-match "*snd*"
```

`--inspect [depth]` prints each volume name aggregated with its count and z
span. `--inspect-match <pat>` filters by name (regex or glob, e.g. `*snd*`).
`--inspect-all` lists every instance instead of aggregating. You can also read
names straight from the SQLite `.db`:

```
sqlite3 files/ship_geometry.db "SELECT name FROM LogVols ORDER BY name;"
```

---

## 4. The view configuration (`views/*.toml`)

The view config drives the REve display and seeds the web producer. Globals:

- `hit_scale` — mm → scene-unit scale.
- `ntuple` — RNTuple name (default `events`).
- `scan_depth` / `region_depth` — how deep the name-scan and region walks go.
- `region_exclude` — volume patterns dropped from region walks.

### `[ui]` — web interface defaults

Web-only settings, all optional (the client also lets you change them live):

- `color_scheme` — the default colour scheme name (see §5). e.g. `"ship_original"`.
- `font_scale` — global text-size multiplier (default `1.0`).
- `sidebar_width` — menu width in CSS pixels.
- `[ui.fonts]` — a sub-table of per-category base sizes (px): `window_title`,
  `menu`, `heading`, `dialog`, `brand`.

```toml
[ui]
color_scheme = "ship_original"
font_scale = 1.0
sidebar_width = 232

[ui.fonts]
window_title = 13
menu = 13
heading = 13
dialog = 13
brand = 15
```

### `[geometry]`

`db_file`, `include`, `exclude` (regex or glob), `max_depth`, `stop_at_match`,
`default_color` (hex), `default_transparency`. Volumes matching a `[[style]]`
get that colour; unmatched volumes get `default_color`.

### `[hits]` and `[decay]`

`[hits]`: `marker_style`, `marker_size`, `color_by_energy`, `color_low`,
`color_high`. Energy is encoded as marker size — `marker_size` is the lowest
bin, the highest renders at 2.8×.

`[decay]`: `draw`, `color`, `marker_style`, `marker_size` — the truth decay
vertex (the first MC particle with a mother).

### `[[region]]` — zoom views

Each region is a windowed sub-view. Selection is **per axis** and independent of
the camera:

- `xmin`/`xmax`, `ymin`/`ymax`, `zmin`/`zmax` — any subset; a volume/hit is kept
  only if inside every axis window that is set.
- `match` — derive a z window by volume name instead of hardcoding it.
- `camera` — `"side"`/`"xz"` (looking along y), `"front"`/`"xy"` (along z,
  beam's-eye), `"top"`/`"xy0"`, `"3d"`. Raw ROOT enum names also accepted.
- `exclude`/`include` — drop/restrict volumes in THIS view only.
- `max_shapes`, `max_transparency`, `margin_frac`.
- `recenter` (+ `offset_x/y/z`) — centre the region on its window so the camera
  frames it (Eve7 auto-fits about the origin).
- `hit_marker_size`, `draw_decay`, `decay_clip`, `decay_marker_size` — per-view
  event styling.
- `panel_x`, `panel_y`, `panel_w`, `panel_h` — the region's panel position and
  size in the **web** view, as **viewport percentages** (0–100), so a layout is
  resolution-independent. Omit to let the client cascade the panel. This makes
  the TOML the single source of the web layout: edit these to arrange the
  panels.

Camera and window are independent: a side view is normally a z slab, a front
view an x or y slab. A mismatch is a warning, not an error — whatever windows
you supplied are used, z preferred.

---

## 5. The web frontend

Layout: a control **sidebar** (left), the **main** 3D view (centre), and the
**region views** — floating panels defined by the `[[region]]` blocks in the
view TOML (placed via their `panel_x/y/w/h`, in viewport %), plus any you create
interactively.

### Sidebar controls

The Show toggles (geometry / hits / decay vertex) and the hit-size slider act on
the **selected view**, or the main view if none is selected. Each view keeps its
own settings, and a new view inherits them from the view it was created from.
Camera buttons (3D / Side / Front / Top) reorient the selected view (else main).
Clicking the main view deselects.

### Creating views

- **+ New view** — a floating panel showing the full detector, inheriting the
  current view's display options. Resize by the corner, move by the title bar,
  rename by double-clicking the title, close with ×.
- **Select view location** — click it, then drag a rectangle on any view (the
  "mother"). A new child view is created showing that boxed region. Orient the
  mother first (Side/Front/Top): the two on-screen axes become the window, the
  axis into the screen stays unconstrained. Select the mother first so you know
  which panel you are drawing on; dragging on the main view uses the full
  detector.

### View context menu (right-click a view)

- **Resize numerically…** — set an exact pixel width/height (current shown).
- **Stack left / right / top / bottom** — move the view to that edge, keeping
  its other coordinate, stopping on collision with another view. Chain them
  (e.g. right then bottom) to tile into a corner.
- **Lock window** — strip the chrome for a clean display: thin seamless border,
  no shadow, frozen position, hidden close button (title stays). Right-click a
  locked view for **Unlock**.
- **Set boundary colours…** — a colour picker with a hex field; applied live,
  persists across lock/unlock, with **Restore default**.

### Layout persistence

The default layout is the `[[region]]` blocks in the view TOML — their windows,
cameras, and `panel_x/y/w/h` (viewport %). Loaded from the manifest on start,
this is the authoritative, version-controlled layout; edit the TOML and
regenerate (`pixi run web-data`) to change it. There is no browser localStorage.

For capturing or sharing an interactive arrangement:

- **Save setup** writes the current arrangement (names, positions, sizes,
  windows, cameras, options, lock state, colours — never the event). Served by
  the writable dev server it saves to the default config; on a read-only server
  it downloads `sea_cucumber_setup.json` instead.
- **Load setup** reads such a JSON back and rebuilds those views.

### Colour schemes

A **Colour scheme** selector sits at the bottom of the sidebar; switching it
re-themes everything live — background, text, accents, borders, the 3D clear
colour, the hit/vertex markers, and the detector palette. Schemes are defined in
`web/js/schemes.js`; the default is set by `[ui] color_scheme`.

Built-in schemes: `ship_original` (the classic look — brown background, the
producer's baked detector colours, pink hits), `ship_db` (dark-blue background,
lighter-blue/gold detectors), `ship_ht` (pinks throughout, bright-pink hits),
and the terminal-style `cobalt2`, `apprentice`, `ayu_dark`, `dracula`, `noctis`,
`shades_of_purple`. In every scheme except `ship_original`, the detector is
coloured **per subsystem** (one colour per subsystem, from that scheme's
palette); `ship_original` uses the producer's baked colours as-is.

### Fonts and text styling

Text is grouped into categories (`window_title`, `menu`, `heading`, `dialog`,
`brand`), each driven by a size variable with defaults from `[ui.fonts]`.

- **Right-click any label** (including menu buttons) → a **Text style** dialog:
  set the size and colour of *that element*, or of its *whole category* in one
  click.
- `+` / `-` scale all text; the global factor also comes from `[ui] font_scale`.
- `Ctrl` `+` / `-` is left to the browser's native zoom.

Colour tweaks are live/session; the persistent defaults live in the config.

### Sidebar

Drag the sidebar's right edge to resize the menu (any width). The default width
is `[ui] sidebar_width`.

### Keyboard shortcuts

Press `?` for an in-app list. Keys are ignored while typing in a field.

| Key | Action |
|-----|--------|
| `←` / `→` | previous / next event |
| `n` | new view |
| `3` / `s` / `f` / `t` | camera 3D / side / front / top (selected view, else main) |
| `g` / `h` / `v` | toggle geometry / hits / vertex |
| `+` / `-` | all text larger / smaller |
| `Ctrl` `+`/`-` | browser zoom (native) |
| `?` | shortcut help |

### Assets

- `web/logo/sc.png` — square logo (sidebar + tab icon); auto-copied from the
  repo `logo/` by the producer.
- `web/fonts/mononoki-Regular.woff2` — the wordmark font (SIL OFL, from
  github.com/madmalik/mononoki). Falls back to system monospace if absent.

---

## 6. The geometry cache

Reading a ~1M-volume `.db` takes seconds on every launch. The cache converts it
once to a ROOT file the display loads quickly:

```
pixi run geo-cache --geometry files/ship_geometry.db --view views/default.toml
pixi run sc --geo-cache geocache --geometry files/ship_geometry.db \
            --data files/llp_display.root --view views/default.toml
```

`geo-cache` writes `geocache.main.root` (envelopes) and `geocache.region.root`
(deep walk). With `--geo-cache`, the display skips the GeoModel read entirely.

---

## 7. Versioning

The version lives in the `VERSION` text file at the repo root, mirrored in
`pixi.toml` (`[workspace] version`, per the ShipSoft pixi convention, readable
with `pixi workspace version get`) and in `CITATION.cff`. The web producer copies
`VERSION` next to the served page, and the frontend shows it under the wordmark.
To release, bump all three in step (and the conda recipe in `ship-conda-recipes`
when publishing).

---

## 8. Troubleshooting

- **Web page shows chrome but no 3D** — check the browser console (F12). A
  blocked three.js CDN import (locked-down network) is the usual cause; vendor
  three.js under `web/js/vendor/` and point the import map at it.
- **A region view is empty** — its window may not intersect this geometry;
  verify with `--inspect` and adjust `zmin/zmax`.
- **Hits float away from the geometry** — the data and the loaded `.db` may be
  from different detector layouts; check their z ranges match.
- **REve right-click menu empty** — the navigator dictionary must be compiled
  into the executable, not the static library (see `CMakeLists.txt`).
