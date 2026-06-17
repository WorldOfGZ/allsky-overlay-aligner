# AllSky Constellation Overlay Aligner — Project Context for Claude Code

## What this tool does

A single standalone HTML file that lets AllSky all-sky camera owners align the VirtualSky constellation overlay without the trial-and-error cycle documented in the AllSky wiki ("edit config → reboot Pi → recapture → compare").

The user:
1. Loads their raw all-sky photo
2. Enters location + time of the photo
3. Drags / rotates / resizes the overlay until stars align
4. Copies the 6 output values directly into AllSky's `configuration.json`

---

## Verified facts (gathered from AllSky source code)

### VirtualSky polar projection math (`virtualsky.js` line 316–342)
```js
azel2xy: function(az, el, w, h) {
    var radius = h/2;
    var r = radius * ((Math.PI/2) - el) / (Math.PI/2);  // equidistant
    return { x: (w/2 - r*Math.sin(az)), y: (radius - r*Math.cos(az)), el: el };
}
```
Called as `azel2xy(star_az - az_off * d2r, ...)`.

`az_off` is derived from the config `az` key: `az_off = (config_az % 360) - 180`.

**Orientation at config `az=180` (az_off=0):**
- North (star_az=0) → x=w/2, y=0 → **TOP** ✓
- East  (star_az=π/2) → x = w/2 - r → **LEFT** (standard astronomical "looking up" convention)
- West  (star_az=3π/2) → x = w/2 + r → **RIGHT**

### Correct az formula (key bug fix in Phase 1)

**Old (wrong) formula:** `az = (180 - rot) % 360`
- This is correct only when `flip=true` (mirrored camera) + AllSky camera flip enabled
- Produced az=351 instead of az=180 for a North-up camera at rot=0

**Correct formulas:**
```js
// flip=false  (East left, standard looking-up, NO AllSky camera flip)
az = (rot + 180) % 360;

// flip=true   (East right, mirrored camera, requires AllSky "flip camera horizontal")
az = ((180 - rot) % 360 + 360) % 360;
```

Derivation: in the tool, the star at compass bearing `rot` moves to the top of the disk as `rot` increases. In VirtualSky, increasing `az_off` by 1° rotates the sky CCW (so the point at az_off° is at the top). Matching both: `az = (rot + 180) % 360` for flip=false.

### Offset and size formulas (were already correct)

The `#starmap` element is positioned with CSS `margin-left` and `margin-top` in AllSky. VirtualSky's canvas is exactly `overlayWidth × overlayHeight` pixels, with the sky disk filling it exactly (radius = overlayWidth/2 = overlayHeight/2 when they're equal).

```js
const sc   = imageWidth / imageNaturalWidth;   // scale factor
const size = Math.round(2 * diskRadius * sc);  // overlayWidth = overlayHeight
const offL = Math.round((diskCenterX - diskRadius) * sc);  // overlayOffsetLeft
const offT = Math.round((diskCenterY - diskRadius) * sc);  // overlayOffsetTop
```

### AllSky controller.js behavior (key findings)
- `virtualSkyData.width  = c.overlayWidth`  → VirtualSky canvas width
- `virtualSkyData.height = c.overlayHeight` → VirtualSky canvas height
- When `icWidth < imageWidth` (responsive/mobile): offsets are scaled by `icWidth / imageWidth`
- AllSky does **not** pass a clock to VirtualSky → it runs in `live` mode (current sky)
- VirtualSky has **no flip parameter**; mirroring is done by flipping the camera image

### Test case for acceptance
- Lat 48.644°N, Lon 5.512°E, 2026-06-17 00:06 UTC+2, projection: polar
- Camera: **North at top**, East at left (standard)
- Expected output: `az ≈ 180`, rot ≈ 0°
- Failed values (pre-fix): `az:351, overlayWidth:1197, offL:-143, offT:-260`

---

## Phase 1 — DONE (current state of `allsky-overlay-aligner.html`)

- [x] Full English translation of the UI
- [x] **Fixed az formula** — `az = (rot + 180) % 360` for flip=false
- [x] Correct flip=true formula with warning about AllSky camera flip
- [x] `Auto-detect circle` button — radial pixel scan to find the sky circle boundary
- [x] `Pre-fill from configuration.json` — paste existing config, controls auto-populate
- [x] Reverse import: az→rot, overlayWidth→radius, offsets→center, legacy `overlaySize` support
- [x] Initial radius defaults to `min(iw, ih)/2` (better for 4:3 cameras)
- [x] English cardinal points N/E/S/W (was N/E/S/O in French)
- [x] Reset uses `min(iw, ih)/2` as default radius

---

## Phase 2 — DONE (2026-06-17): VirtualSky engine integrated

**Goal achieved:** Replaced the custom renderer with AllSky's actual `virtualsky.js`. Controls are the AllSky config keys directly — no az conversion, no offset math.

### ### Files embedded (all from AllSky GitHub fork)

| File | Size | Notes |
|------|------|-------|
| `stuquery.js` | 17 KB | jQuery replacement used by VirtualSky |
| `virtualsky.js` | 149 KB | Main engine (unmodified) |
| `lines_latin.json` | 10 KB | Constellation line data (inline as `VS_LINES`) |

Total embedded: ~200 KB. `stars.json` (55 KB) is NOT embedded — VirtualSky has built-in bright stars (mag ≤ 4) which are sufficient for alignment. Loading more stars would require a `stars.json` load which we disable.

### What was NOT embedded and why
- `stars.json` — VirtualSky has 238+ built-in bright stars. Extra stars only load when `magnitude > 4` AND `changeMagnitude()` is called. We bypass `changeMagnitude` by setting `sky.magnitude` directly, so `stars.json` never loads. Magnitude capped at 5.0 in the slider.
- `galaxy.json` — `showgalaxy: false` in init options.
- Language files — English is built-in default; AJAX patch makes all `loadJSON` calls no-ops.

### Key implementation decisions

1. **loadJSON patch:** `VirtualSky.prototype.loadJSON = function() { return this; }` placed AFTER virtualsky.js embeds. This is a no-op — silently disables all AJAX. Works because we pre-load all needed data.

2. **Lines pre-loaded:** Pass `lines: VS_LINES.lines` in init options → sets `this.lines` directly → `checkLoaded()` skips the network load.

3. **VirtualSky canvas ID:** `id: 'vs-container'` → VirtualSky creates `<canvas id="vs-container_inner">` inside the container div.

4. **Overlay positioning:** `#vs-wrap` (position:absolute) wraps `#vs-container`. We position `#vs-wrap` in CSS pixels, VirtualSky sets `#vs-container` to `position:relative`. The rendered canvas is inside.

5. **Re-init on size change:** `initVS()` is called when the display size changes by >1.5%. This keeps VirtualSky rendering at full CSS resolution (no blurriness from CSS scaling).

6. **Projection update:** `sky.selectProjection(proj)` is called on every `updateVS()` (idempotent, cheap).

7. **No `changeMagnitude`:** Update `sky.magnitude` directly and call `sky.draw()`. This never triggers `stars.json` loading.

### VirtualSky initialization (derived from `controller.js` line 102–110)
```js
// stuQuery replaces jQuery's $()
stuQuery('#starmap_inner').virtualsky({
    width:     overlayWidth,     // = overlayWidth config value
    height:    overlayHeight,    // = overlayHeight config value
    latitude:  lat,
    longitude: lon,
    clock:     new Date('2026-06-17T22:06:00Z'),  // UTC datetime of photo
    live:      false,             // freeze at photo time, not current sky
    projection: 'polar',          // direct AllSky config key
    az:        180,               // direct AllSky config key — NO conversion needed!
    showgalaxy: false,
    gridlines_az: false,
    gridlines_eq: false,
    showdate: false,
    showposition: false,
    // transparency: the VirtualSky canvas must be positioned as an overlay
    // over the user's photo (CSS: position:absolute, background:transparent)
});
```

### Key property mappings
- `d.az` → `az_off = (d.az % 360) - 180` (VirtualSky internal)
- `d.width` → `this.wide`
- `d.height` → `this.tall`
- `d.clock` accepts a JS Date or date string
- `d.live = false` → no real-time updates (use `s.draw()` to refresh)

### Architecture (Phase 2 actual implementation)

```
#canvas-host  (position:relative)
├── <canvas id="cv">            ← photo canvas, also handles drag/scroll events
├── <div id="vs-host">          ← transparent overlay (position:absolute, inset:0, pointer-events:none)
│   └── <div id="vs-wrap">      ← positioned by positionVS() in CSS pixels
│       └── <div id="vs-container">   ← VirtualSky sets position:relative + size
│           └── <canvas id="vs-container_inner">  ← VirtualSky renders here (transparent bg)
└── <div id="drop">             ← file drop zone (hidden after image load)
```

Controls become direct config keys:
- `az` slider (0–360°) — replaces the `rot` + conversion chain
- `projection` buttons — same as now
- `overlayWidth` / `overlayHeight` — controls the VirtualSky canvas size (= disk diameter)
- `overlayOffsetLeft` / `overlayOffsetTop` — controlled by dragging the canvas

Output becomes a trivial read-off of the current values — no formula needed.

### VirtualSky background transparency
VirtualSky by default fills its canvas with a dark background. To make it transparent:
```js
// After init, override the background colour
sky.background = 'rgba(0,0,0,0)';
sky.draw();
```
Or set `sky.col.background = 'rgba(0,0,0,0)'`.

Check virtualsky.js line ~515: `ctx.fillStyle = (this.hasGradient()) ? ... : this.col.black` — override `this.col.black` or `this.background` before drawing.

---

## Phase 3 — Future ideas (not urgent)

- **Radial lens calibration** — for lenses that don't fit any standard projection (VirtualSky issue #56), allow the user to define a custom radial mapping table (zenith angle → normalised radius). Useful for lenses with strong fisheye distortion beyond 150°.
- **Lens profile library** — pre-baked profiles for common all-sky lenses (2.7mm M12 fisheye, etc.)
- **Direct AllSky JSON update** — if hosted locally, patch `configuration.json` via fetch
- **Night-sky photo scoring** — after alignment, measure how well the overlay matches the actual star positions (correlation metric)

---

## Phase 2 UX changes vs Phase 1

- **Section 1** (new): Import full `configuration.json` → pre-fills lat, lon, imageWidth, opacity, projection, az, overlayWidth, offsets. Pending config stored and applied when image loads.
- **Section 3 (projection)**: Better documentation, removed M12-specific mention, explains all 4 types and when to use.
- **Section 4 (alignment)**: `az` slider (0-360°) replaces `rot`. Fine step buttons (±1, ±10 AllSky px) for both X and Y. No flip toggle (VirtualSky has no flip option).
- **Section 5 (output)**: Individual `copy` button per key. Copying `"az": 351` updates just that one key in the user's config. No "copy block" that would erase other settings.

## Repo structure

```
allsky-overlay-aligner/
├── allsky-overlay-aligner.html   ← main tool (single file, no deps, ~200KB)
├── CLAUDE.md                     ← this file (project context for Claude Code)
└── example.json                  ← example AllSky configuration.json from user
```

---

## AllSky source references (GitHub: AllskyTeam/allsky)

| File | Path |
|------|------|
| controller.js | `html/allsky/js/controller.js` |
| virtualsky.js | `html/allsky/virtualsky/virtualsky.js` |
| stuquery.js   | `html/allsky/virtualsky/stuquery.js` |
| allsky.css    | `html/allsky/css/allsky.css` |
| stars.json    | `html/allsky/virtualsky/stars.json` |

Key discussions:
- #1399 — VirtualSky default orientation (South up, West right at az=0)
- #3462 — a similar prototype by another user
- #3729 — responsive overlay fix (offsets rescaled on resize)
- VirtualSky issue #56 — lenses beyond 150° don't match any projection
