# Erebus Map Editor

A standalone, static web app for building RoboCup Junior Rescue Simulation
maps and exporting them as Webots `.wbt` world files for the
[Erebus simulator](https://github.com/wilsoncheng-sgcs/erebus) (this repo
targets the `entry-level-floor-victims` branch, which adds the
`FloorVictim.proto` the Entry Level tier below depends on).

**Live app:** https://wilsoncheng-sgcs.github.io/erebus-map-editor-RCJA/

**Design rationale:** the plan behind this repo and the companion Erebus
`entry-level-floor-victims` branch is documented in
[`docs/entry-level-plan.md`](https://github.com/wilsoncheng-sgcs/erebus/blob/entry-level-floor-victims/docs/entry-level-plan.md)
in that branch.

## Origin and attribution

This project is extracted from the `sim_editor` module of
[robocup-junior/rcj-rescue-cms](https://github.com/robocup-junior/rcj-rescue-cms)
(specifically the `develop/2026-sim` branch: `sim_editor.2026.js`,
`sim_editor_2026.pug`, the `sim_editor_modal.2026.html` /
`custom_room_4_modal.2026.html` tile-editor templates, and `maze_edit.css`).

`rcj-rescue-cms` is MIT licensed, copyright (c) 2016 Fredrik Lofgren. See
[`LICENSE`](./LICENSE) for the full license text, which this repository
retains and extends per the terms of the MIT license (substantial code from
the original project has been copied here, largely verbatim).

The original `sim_editor` is an AngularJS 1.x single-page app embedded in a
Node.js/Express/MongoDB CMS. Its map data model and `.wbt` export logic are
100% client-side JavaScript (JSON load/save via the browser File API, `.wbt`
generation via plain string templating) with no backend/MongoDB coupling, so
it was possible to lift it out into this standalone repo essentially as-is.

## What changed in the extraction

- `sim_editor.2026.js` -> [`js/sim_editor.js`](./js/sim_editor.js), copied
  verbatim except for: removing the `angular-translate` module dependency and
  hardcoding its ~30 translation keys to their English strings (see the
  `en.json` locale file from the source repo for the original key/value
  pairs); removing the `bootstrap-fileinput` widget initialization (the JSON
  import button is now a plain `<input type="file" id="select">`, wired to
  the exact same native `change` event listener that already existed in the
  file); and adding the ruleset tier selector / Entry Level branch described
  below.
- `sim_editor_2026.pug` -> [`index.html`](./index.html): dropped the CMS
  layout (`extends ../includes/layout`, navbar/breadcrumb/footer), kept the
  editor grid/controls markup, and added explicit `<script>`/`<link>` tags
  (via CDN) for Angular 1.x, Angular UI Bootstrap, Bootstrap 4 CSS/JS
  (matching the version the markup's `input-group-prepend`/`input-group-text`/
  `mb-3`/`btn-outline-*` classes require), jQuery, SweetAlert2, html2canvas,
  and OpenCV.js (needed by the Room 4 custom-shape image-processing feature).
- `sim_editor_modal.2026.html` and `custom_room_4_modal.2026.html` (the two
  Angular UI Bootstrap modal templates the controller opens for per-tile
  editing and the Room 4 custom-shape drawing tool) -> `templates/`, with the
  same translation-key hardcoding and image-path adjustments, plus the Entry
  Level UI changes described below.
- `maze_edit.css` -> [`css/maze_edit.css`](./css/maze_edit.css), copied as-is
  (identical on `master` and `develop/2026-sim`).
- `LICENSE` copied from `rcj-rescue-cms` and extended with a second copyright
  line for the extraction/Entry Level work.

## Running it

This is a fully static app — no build step, no server-side code.

- Simplest: open `index.html` directly in a browser (`file://` works fine,
  Angular and the File API don't need a server).
- Or serve it with any static file server, e.g.:
  ```
  python3 -m http.server 8000
  ```
  then visit `http://localhost:8000/index.html`.
- Or use the hosted copy above — no setup at all.

### Deploying to GitHub Pages

This repo is Pages-ready as-is: all local asset references (`css/`, `js/`,
`images/`, `templates/`) are relative paths, which is what's needed since a
GitHub Pages project site is served from a subpath
(`https://<user>.github.io/<repo>/`), not the domain root. To (re)deploy:

1. In the repo's GitHub **Settings → Pages**, set "Build and deployment"
   source to "Deploy from a branch", branch `main`, folder `/ (root)`.
2. A `.nojekyll` file is committed at the repo root — required because
   GitHub Pages runs Jekyll by default, which ignores directories/files
   starting with `_` and could otherwise interfere with the plain static
   files here.
3. Pages serves whatever is on `main` — no build step, no GitHub Actions
   workflow needed. Push to `main` and the live site updates automatically
   (usually within a minute or two).

## Ruleset tier selector

The top control panel has a new **Ruleset tier** selector: **Original**,
**Intermediate**, **Entry Level** (defaults to Original).

- **Original** / **Intermediate**: export identically to the original
  `sim_editor` — no behavior change.
- **Entry Level**: `createWorld()` in `js/sim_editor.js` (see the `tag entry
  level floor victims` comments) takes a different branch:
  - Wall-mounted victim tokens, cognitive-target (hazmat) signs, fake
    victims, and half-wall victims are skipped entirely — zero such nodes
    are emitted.
  - Every tile that has a victim assigned on **any** wall side (top, right,
    bottom, or left — Entry Level has no wall-side/orientation concept, so
    "assigned anywhere" just means "this tile has a floor victim") instead
    emits a single `FloorVictim` node at the tile's center, reusing the
    exact tile-center coordinate (`x * 0.3 * tileScale[0] + startX`, `z * 0.3
    * tileScale[2] + startZ`) and tile-side scale factor (`0.3 *
    tileScale[0]`) the rest of `createWorld()` already computes for
    wall/floor placement — no new coordinate math was introduced.
  - These `FloorVictim` nodes are grouped under a new `DEF FLOORVICTIMGROUP
    Group { children [...] }` node, alongside the existing
    `HUMANGROUP`/`FAKES`/`TARGETGROUP` groups (which are emitted empty for
    Entry Level, since nothing populates them in that branch).
  - The generated file's `EXTERNPROTO` header swaps `Victim.proto`,
    `CognitiveTarget.proto`, and `Fake.proto` for `FloorVictim.proto` (all
    the other EXTERNPROTOs — walls, tiles, obstacles, Area 4, etc. — are
    unchanged).

### Colour legend (Entry Level floor markers)

| Colour | Type       |
|--------|------------|
| Red    | `harmed`   |
| Green  | `unharmed` |
| Yellow | `stable`   |

### Tile editor UI for Entry Level

When Entry Level is selected, the per-tile editor modal
(`templates/sim_editor_modal.html`) hides the full per-wall-side victim
table, cognitive-target code inputs, victim-rotation/fake-victim controls,
and the half-wall-victim grid, and shows instead a single row of three
colour buttons (red/green/yellow, plus "none") for the tile — reusing the
same `'H'`/`'U'`/`'S'` letters (`HUMAN_H`/`HUMAN_U`/`HUMAN_S`) the original
per-wall-side controls already used, just written unconditionally to
`cell.tile.victims.top` (see `$scope.setEntryLevelVictim` /
`$scope.getEntryLevelVictim` in `js/sim_editor.js`).

### Advisory floating-wall check

At export time, if `ruleTier === "entryLevel"`, `exportW()` runs
`$scope.checkFloatingWalls()`, a best-effort, **non-blocking** scan of the
wall-cell data structure for wall segments that aren't adjacent to any tile
on either side. If it finds any, it shows a plain `alert()` listing them —
export proceeds regardless. This is intentionally a simple heuristic (checks
only "does this wall cell have a tile-typed cell as either immediate
neighbour"), not a full reproduction of `createWorld()`'s wall-resolution
logic (which also deals with half-walls, curved walls, and corner-merging).
It will not catch every way the data model can technically represent an
inconsistent wall; see "Known gaps" below.

## Loading Entry Level `.wbt` files in Webots

Entry Level exports reference `EXTERNPROTO "../protos/FloorVictim.proto"`,
matching the relative-path convention the rest of the exporter already uses
for every other proto. `FloorVictim.proto` is expected to live in the Erebus
simulator repo's `game/protos/` directory. Concretely, this means either:

- Cloning this repo as a sibling directory next to a clone of the Erebus
  simulator repo, and generating/loading `.wbt` files from a location one
  level up from that simulator's `protos/` directory (mirroring where the
  original CMS's other exported worlds already expect to sit relative to
  `protos/`), or
- Adjusting the `../protos/...` paths in a generated `.wbt` file to point at
  wherever your local `game/protos/FloorVictim.proto` actually lives.

`FloorVictim.proto` fields used by the generated nodes: `translation`
(`SFVec3f`), `name` (`SFString`), `type` (`SFString`, one of
`"harmed"`/`"unharmed"`/`"stable"`), `scoreWorth` (`SFInt32`), `size`
(`SFFloat`, side length in meters). `rotation` and `found` are left at their
proto defaults since floor markers have no meaningful orientation.

## Known gaps

- **Floating-wall advisory check is a heuristic**, not a full topology
  validator (see above) — it may miss some invalid wall configurations and,
  in principle, could flag a wall segment that is actually fine under some
  half-wall/curved-wall configuration not modeled by the simple "does either
  neighbour have `isTile`" check. Since it's advisory-only and never blocks
  export, this was judged an acceptable trade-off rather than reimplementing
  `createWorld()`'s full wall-resolution pass a second time just for
  validation.
- **`makeImage`/`makeImageDl`** (an optional "download a PNG of the map"
  feature in the original CMS) reference a `#outputImageArea` DOM element
  and, in one code path, a CMS backend endpoint (`/api/maps/line/image/...`)
  that no longer exists. Neither function is wired to any button in
  `index.html` (they weren't in the original `sim_editor_2026.pug` either),
  so they are dead code left in place for fidelity with the source file
  rather than something that can actually be triggered from the UI.
- `mapId`/`competitionId` globals (previously injected by the CMS's
  server-rendered Pug template) are now just inert local placeholders, since
  nothing in the map-editing/export path actually depends on them.
