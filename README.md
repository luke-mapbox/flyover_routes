# Flyover Route Visualiser

A single-file, Relive-style **cinematic flyover** for GPX routes, built with
[Mapbox GL JS v3](https://docs.mapbox.com/mapbox-gl-js/) using the Mapbox
**Standard** / **Standard Satellite** styles and a free-camera path animation.

Pick a route from the dropdown and the camera flies along the track over 3D
terrain, drawing the trail as it goes — with a live stats bar (distance /
duration / elevation gain), an elevation profile, and a full panel of style and
camera controls.

![style reference](videoframe_25016.png)

---

## What it does

- **Cinematic flyover** — a chase camera follows the route point-by-point
  (`FreeCameraOptions` + `MercatorCoordinate`, per the
  [free camera path example](https://docs.mapbox.com/mapbox-gl-js/example/free-camera-path/)).
  It samples terrain height (`queryTerrainElevation`) each frame so it clears
  mountains instead of clipping through them.
- **Progressive trail** — the yellow route line draws in as the camera advances
  (`line-trim-offset`), over a faint full-route "ghost" preview, with start /
  finish pins and a pulsing head marker.
- **Optional 3D model avatar** — a glTF model (e.g. a **boat** or a **duck**) can
  ride the route head, sitting on the surface and turning to face the direction of
  travel. It's **off by default**; pick one from the **3D model** dropdown
  (native Mapbox `model` layer, animated with `setModels` each frame). Selecting
  it replaces the pulsing head marker; "None" restores it.
- **9 bundled routes** — a deliberate mix of terrain: Paris city rides/runs, a
  Rome stroll, Fontainebleau forest, a Dutch waterway, and alpine hikes
  (Mt Rainier ×2, Acadia, Melbourne's 1000 Steps).
- **Smart per-route styling** — mountain / outdoor / water routes default to
  **Standard Satellite** (satellite imagery + 3D terrain); city routes default
  to **vector Standard**. You can override the style for any route.
- **Live stats** — distance, duration, and elevation gain, with a **km ↔ mi**
  toggle, plus an interactive **elevation profile** that tracks flyover progress.
- **Playback** — play / pause / restart / scrub / loop.

---

## Quick start

### 1. Add your Mapbox token

The token lives in a local **`config.js`** that is **git-ignored** (see
`.gitignore`), so it never gets committed. Copy the template and paste your
**public** token (`pk.…`):

```bash
cp config.example.js config.js
# then edit config.js:
#   window.MAPBOX_TOKEN = 'pk.your_real_token';
```

Get a token from your [Mapbox account](https://account.mapbox.com/access-tokens/).
(Alternatively, pass one at runtime with `?access_token=pk.…` in the URL, or set
`localStorage.mapbox_token` — both override `config.js`.)

### 2. Run it

The bundled route dropdown loads GPX files with `fetch('./gpx/…')`, which needs
the page served over **http** (not opened directly as a `file://` path). From
this folder:

```bash
python3 -m http.server 8000
```

Then open **http://localhost:8000**.

> **No server?** Just double-click `index.html` and use the **"upload a .gpx
> file"** link (or drag-and-drop a `.gpx` onto the page). That path works from
> `file://` — only the pre-bundled dropdown needs the server.

---

## Using the app

| Control | What it does |
|---|---|
| **Route** | Pick one of the bundled GPX routes (or upload your own `.gpx`). |
| **Base map style** | Standard Satellite (satellite + terrain), vector Standard (3D), or classic Satellite Streets. Auto-defaults per route type; override any time. |
| **Light / time of day** | Dawn / day / dusk / night (Standard `lightPreset`). |
| **Route colour + width** | Trail colour (with quick swatches, incl. Relive yellow) and thickness. |
| **Terrain exaggeration** | Vertical relief multiplier. Auto-seeded higher for mountainous routes. |
| **Zoom level** | Camera-distance preset (Street-level → Bird's-eye) that sets height + follow distance together. Nudging the fine sliders switches it to "Custom". |
| **Camera height / follow distance** | Fine control of how high above ground and how far behind the moving point the camera sits (together these set the effective tilt). |
| **Flyover speed** | Playback speed multiplier (route length also scales the duration). |
| **Map labels / 3D objects** | Toggle **all** Standard labels (place, POI, road, transit, pedestrian) for a clean cinematic look, and 3D buildings/trees. |
| **3D model** | Choose an optional glTF avatar to ride the route head (**None / Boat / Duck**). It rests on the surface and turns to face the direction of travel. Defaults to **None** — pick Boat or Duck to enable it on any route. |
| **Loop** | Restart the flyover automatically when it finishes. |
| **Units** | Switch stats between km and mi. |
| **Transport bar** | Scrub, restart, and play/pause; the elevation profile cursor follows along. |
| **Reset** (panel header) | Restores every setting to its default in one click. Keeps your current route and base-map selection, and re-seeds the smart per-route terrain exaggeration (so an alpine route resets to its higher relief, not a flat default). |

### Advanced — cinematic (expandable)

| Control | What it does |
|---|---|
| **Pacing** | *Even speed* (constant) or *Match recorded GPS pace* — replays the route at the actual recorded speed using the GPX timestamps (fast on descents, slow on climbs). |
| **Camera smoothing** | LERP between frames to damp jitter from noisy GPS/terrain — higher = smoother, more floaty. Bypassed while scrubbing. |
| **Look-ahead** | Aims the camera a set distance *ahead* of the current point, so it leads into corners instead of staring straight down at the trail. |
| **Orbit drift** | Rotates the camera around the moving point across the flyover for a sweeping, drone-like reveal. |
| **Atmospheric haze** | `setFog`-based depth haze whose colour follows the light preset (incl. stars at night); intensity is adjustable. |
| **Ease in / out** | `easeInOutQuad` easing so the flyover accelerates smoothly from the start and decelerates into the finish (the scrubber still tracks linear time). On by default. |
| **Lock map while playing** | Disables drag/zoom/rotate gestures during playback so a stray gesture can't fight the scripted camera; re-enabled when paused. On by default. |

---

## How it works (architecture)

Everything lives in **`index.html`** — no build step. Mapbox GL JS and
[Turf.js](https://turfjs.org/) are loaded from CDNs.

1. **GPX parsing** (`parseGpx`) — `DOMParser` extracts `<trkpt>` coordinates,
   `<ele>` (metres) and `<time>` (ISO-8601) into arrays.
2. **Route model** (`buildRoute`) — builds a Turf `LineString`, computes length
   (`turf.length`), bounding box (`turf.bbox`), duration, min/max elevation, and
   **elevation gain**. Gain is computed from a **window-21 moving-average** of
   the elevation series, so raw GPS jitter on flat/marine tracks isn't
   accumulated as thousands of phantom metres while real climbs are preserved.
3. **Map + terrain** — Mapbox `standard` / `standard-satellite` styles on a
   globe projection; a `mapbox-dem` raster-DEM source drives `setTerrain` with
   the exaggeration slider. Standard config is applied via
   `setConfigProperty('basemap', …)` — `lightPreset`, `show3dObjects`, and one
   labels toggle that drives every label type (place, POI, road, transit,
   pedestrian) for a clean cinematic look. Atmospheric haze uses `setFog`.
4. **Route layers** — a GeoJSON source (`lineMetrics: true`) with three line
   layers: faint full-route ghost, dark casing, and the bright trail. The trail
   and casing use `line-trim-offset` to reveal only the travelled portion.
5. **Flyover** (`renderAt(phase)` + a `requestAnimationFrame` loop) — for each
   progress `phase ∈ [0,1]` (optionally shaped by an `easeInOutQuad` curve) it
   places the target and camera points along the route (`turf.along`), positions
   the free camera above the terrain, aims it at the target, and updates the trail
   trim, head marker, and elevation cursor. During playback `setInteractive(false)`
   locks map gestures so they can't fight the scripted camera.
6. **3D model avatar** (optional) — a native Mapbox `model` layer fed by a
   declarative `model` source. Every frame, `updateModel()` calls
   `source.setModels({ avatar: { uri, position, orientation } })` to move the
   model to the route head and set its heading (the
   [animate-a-model-along-a-route](https://docs.mapbox.com/mapbox-gl-js/example/add-3d-model-and-animate-along-route/)
   pattern). `orientation` is `[x,y,z]°`: a base tilt stands the model upright
   (glTF is Y-up, Mapbox is Z-up → `[90,0,0]`) and the travel heading plus a
   per-model `yaw` offset go on the z (up) axis so the nose points along travel.
   This needs **Mapbox GL JS ≥ 3.19** — `setModels` does not exist on older 3.x.
7. **UI** — the config panel and HUD are plain DOM; changes update `state.cfg`
   and re-apply live. Defaults live in one `DEFAULT_CFG` object; **Reset** restores
   from it and `syncUI()` pushes the whole config back into every control.

### Adding / changing routes

Drop new `.gpx` files into `gpx/` and add an entry to the `ROUTES` array in
`index.html`:

```js
{ file:'my_route.gpx', title:'My Route', type:'mountain' },
```

`type` (`mountain` | `outdoor` | `city` | `water`) chooses the default base
style and seeds the terrain exaggeration. Any `.gpx` can also be loaded
ad-hoc via the upload / drag-and-drop fallback without editing the manifest.

To give a route a default 3D model, add `model:'<key>'` referencing a
`MODELS` entry (see below):

```js
{ file:'my_waterway.gpx', title:'My Waterway', type:'water', model:'boat' },
```

### Adding / changing 3D models

Models are defined in the `MODELS` catalog in `index.html` and appear in the
**3D model** dropdown automatically:

```js
const MODELS = {
  none: { label:'None' },
  boat: { label:'Boat', uri:'boat_model/boat_centered.gltf', scale:6,  orientation:[90,0,0], yaw:90,  emissive:0.5 },
  duck: { label:'Duck', uri:'boat_model/Duck.glb',           scale:12, orientation:[0,0,0],  yaw:-90, emissive:0.2 },
};
```

- **`scale`** — uniform scale. Real-world-metre models look tiny from the
  flyover camera, so these are enlarged for visibility (boat ≈ 6, duck ≈ 12).
- **`orientation`** — `[x,y,z]°` base rotation to stand the model upright.
  Most glTF models are Y-up, so `[90,0,0]` (Mapbox is Z-up). A model exported
  already Z-up (e.g. the sample Duck) uses `[0,0,0]`.
- **`yaw`** — degrees added to the travel heading on the z axis so the model's
  **nose points along the direction of travel**. Calibrate by eye: if the nose
  points 90° clockwise of travel, use `yaw:-90`; anticlockwise, `yaw:+90`.
- **`emissive`** — base `model-emissive-strength`; auto-boosted at dusk/night.

> ⚠️ **Asset compatibility — important.** Mapbox GL JS's model loader **rejects
> glTF files authored by [glTF-Transform](https://gltf-transform.dev/)** with
> `Could not load model … offset is out of bounds`, even though the files are
> spec-valid. Re-encoding through glTF-Transform (or gltf-pipeline) does **not**
> fix it. Author/convert models with a **different** tool — e.g. Blender's glTF
> exporter, the Khronos sample models, or an online converter (the bundled
> `boat2.gltf` came from ImageToStl.com). `boat_centered.gltf` is `boat2.gltf`
> with a root-node `translation` added (a pure JSON edit — buffers untouched, so
> it stays compatible) to move the model's origin to its centroid so it sits
> **centered** on the route point instead of offset. The unused
> `boat_model/boat_model.gltf` (+ `buffer.bin`) is the original glTF-Transform
> export kept for reference — it will **not** load.

Model URIs are resolved to absolute URLs at runtime (a Mapbox `model` source
rejects relative paths), so a plain repo-relative path like above works when the
page is served over http.

---

## Requirements & notes

- A **public** Mapbox access token (`pk.…`). Keep secret tokens out of
  client-side code.
- A modern browser (Chrome / Safari / Firefox / Edge) with WebGL.
- Network access for the Mapbox tiles/styles and the two CDN scripts
  (`mapbox-gl` and `@turf/turf`).
- Bundled GPX routes live in `gpx/`. Reference video frames
  (`videoframe_*.png`) show the target Relive look.
