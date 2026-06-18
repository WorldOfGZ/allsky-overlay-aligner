# AllSky Constellation Overlay Aligner

A standalone browser tool that helps [AllSky](https://github.com/AllskyTeam/allsky) all-sky camera owners align the VirtualSky constellation overlay — without the tedious trial-and-error cycle of editing a config, rebooting the Pi, recapturing, and comparing.

## The problem it solves

AllSky displays a VirtualSky star map overlay on top of the all-sky camera image. Getting it to line up with the actual stars requires setting six parameters in `configuration.json`:

- `az` — rotation of the sky map
- `overlayWidth` / `overlayHeight` — size of the overlay disk
- `overlayOffsetLeft` / `overlayOffsetTop` — position of the disk on the image
- `projection` — lens projection type

The official workflow is: guess values → reboot Pi → wait → compare → repeat. This tool replaces that loop with a live visual editor.

## How to use it

### Step 1 — Import your existing config (recommended)

Paste the contents of your AllSky `configuration.json` and click **Import & pre-fill all fields**. This fills in your latitude, longitude, image size, current az, offsets, projection, and shows your lens model to help choose the right projection.

### Step 2 — Enter the date and time of the photo

Latitude and longitude are filled automatically from your imported config. You only need to set the **date**, **local time**, and **UTC offset** to match when the photo was taken. The tool shows the resulting UTC time so you can verify it.

> This is critical. VirtualSky renders the sky at the exact time you enter. A wrong time produces a completely different star pattern that cannot be aligned no matter what you do.

### Step 3 — Load your photo

Drag and drop (or click to browse) a raw all-sky image captured at that date and time. The overlay appears immediately.

### Step 4 — Choose your lens projection

AllSky default is **Equisolid (fisheye)**. After importing your config, the tool shows your lens model to help you choose. If in doubt, start with fisheye. Wrong projection = stars drift at the edges even when the center aligns.

| Button | AllSky key | Typical lenses |
|--------|-----------|----------------|
| Equisolid | `"fisheye"` | **AllSky default — start here.** RPi HQ + M12 2.7 mm / 3.6 mm fisheye · IMX462 / IMX477 / IMX585 + M12 fisheye · ArduCam fisheye · Sigma 8 mm f/3.5 circular · Samyang / Rokinon 8 mm fisheye |
| Equidistant | `"polar"` | Some wide-angle M12 lenses · GoPro-style action cameras · lenses marketed as "wide-angle" rather than "fisheye" |
| Orthographic | `"ortho"` | Rare; horizon ring is heavily compressed — try this only if both fisheye and polar don't align at the edges |
| Stereographic | `"stereo"` | Dome projectors · some 360° CCTV fisheye lenses |

> **Not sure?** The difference between equisolid and equidistant is subtle near the center and only visible at the edges. Use the **🔭 Predicted Big Dipper** button: if the green circles land on actual stars with fisheye selected, that's your projection.

### Step 5 — Set AllSky camera settings

Match the **Rotation** and **Flip** values from your AllSky "Both Day & Night Settings". For a standard camera (North physically at the top, no horizontal flip) leave both at **None**.

### Step 6 — Align the overlay

**Lock the position first** (`🔓 Lock position`) to prevent accidental moves, then use:

- **az slider** — rotates the star map. `az = 180` = North at top, East on left (standard looking-up view). If your camera points exactly North, `az = 180`. Small deviations (e.g. `az = 173`) mean the camera top is a few degrees off true North — that is normal and the az value compensates for it automatically.
- **Diameter slider** — resizes the overlay disk to match the sky circle in your photo.
- **±1 / ±10 step buttons** — nudge the disk position precisely in AllSky pixel coordinates.
- **Auto-detect circle** — radial pixel scan to snap the disk to the sky circle boundary automatically.
- **Re-center** — moves the disk center to the image center.
- **Drag & scroll** — available when unlocked; use for rough positioning then re-lock.

#### Polaris anchor (gold crosshair)

When an image is loaded, a **gold crosshair** appears on the photo at the predicted position of Polaris (α UMi). Polaris is time-independent — always at altitude ≈ your latitude, due North. A matching **gold ring** in the overlay marks where VirtualSky places the North Celestial Pole.

Workflow: set `az = 180`, drag the disk until the gold crosshair lands on actual Polaris in your photo, then fine-tune az until the gold ring meets the crosshair.

#### Predicted Big Dipper diagnostic

Click **🔭 Show predicted Big Dipper on photo** to draw **green circles** at the mathematically computed positions of the 7 main Ursa Major stars + Polaris + Kochab, directly on your photo — completely bypassing VirtualSky.

- If the green circles land on actual stars → your time, disk position, and projection are correct. At `az = 180` the VirtualSky overlay should align automatically.
- If they don't match → check your date/time/UTC offset, or reposition the disk first.

#### 3-Star Alignment (automatic solver)

Click **✦ 3-Star Alignment** to let the tool compute `az`, disk center, and radius automatically from three stars you identify:

1. Click a star in your photo.
2. Identify it by name from the list of stars visible at your time/location.
3. Repeat for two more stars.

The solver finds the unique combination of az, center, and radius that fits all three clicks.

### Step 7 — Copy the output values

Each output value has its own **copy** button. Click it to copy just the value (e.g. `173`, `1197`, `"fisheye"`) and paste it into the matching key in your `configuration.json`.

## Features at a glance

| Feature | Description |
|---------|-------------|
| Live VirtualSky rendering | Same `virtualsky.js` engine as AllSky — pixel-accurate |
| Config import | Pre-fills all controls from your `configuration.json`, shows lens and projection |
| Lock position | Prevents accidental drag/scroll from ruining alignment |
| Auto-detect circle | Radial scan finds the sky disk boundary automatically |
| Polaris anchor | Gold crosshair on photo + gold ring in overlay — time-independent alignment guide |
| Predicted Big Dipper | Green circles show computed star positions — verifies time and disk without VirtualSky |
| 3-Star Alignment | Auto-solver: click 3 stars, get az + position + size computed automatically |
| Per-key copy buttons | Copy individual values, not the whole block |
| AllSky Rotation/Flip | Dropdown matching AllSky's "Both Day & Night Settings" |
| Single HTML file | No install, no server. Open in any modern browser |

## Output

The tool produces the values to update in your AllSky `configuration.json`:

```json
"az": 173,
"overlayWidth": 1197,
"overlayHeight": 1197,
"overlayOffsetLeft": -143,
"overlayOffsetTop": -260,
"projection": "fisheye"
```

## Technical notes

- The `az` value maps directly to AllSky's `az` config key — no conversion needed.
- `overlayWidth` and `overlayHeight` are the pixel dimensions at full `imageWidth` resolution. AllSky rescales them automatically for responsive views.
- The tool does not modify any files on your Pi — it only helps you compute the correct values.
- The `az` value compensates for physical camera rotation. `az = 180` is perfect North-up; small deviations like `az = 173` mean the camera top is ~7° off true North, which is normal.

---

## Credits

This tool would not exist without the [AllSky project](https://github.com/AllskyTeam/allsky) by the [AllSky Team](https://github.com/AllskyTeam). The overlay engine ([VirtualSky](https://github.com/slowe/VirtualSky) by Stuart Lowe), the star data, and the constellation line data are all part of AllSky and are embedded here unmodified solely to allow the aligner to work as a standalone offline file.

**If you use AllSky, please support the project:**
👉 [https://github.com/AllskyTeam/allsky](https://github.com/AllskyTeam/allsky)

---

## Disclaimer

> **This tool has only been tested on a single configuration** — an RPi HQ all-sky camera located in north-east France (Lorraine). Results on other hardware, lens types, or AllSky versions may vary. Use the output values as a starting point and verify by comparing the overlay with your actual sky photo.
>
> **This project was coded entirely with [Claude Code](https://claude.ai/code)** (Anthropic's AI coding assistant), with the author providing direction, domain knowledge, and testing.

Feedback and bug reports welcome via [GitHub Issues](../../issues).
