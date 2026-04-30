# Portal of Collective Imagination — WebGL MVP

WebGL implementation of the Display Mac compositor for the Portal of Collective Imagination arch sculpture. Sibling project to [`tunnel-mvp`](../tunnel-mvp) (the Unity implementation). Validates that browsers can drive the same 8K-through-Fresco-to-16-screens architecture, allowing browser-based interactive art to feed the production hardware.

## What this is

The Display Mac produces a single 8K (7680×4320) HDMI signal that the Fresco 8K splitter divides into 4× 4K outputs, each further split by a Portta into 4× 1080p HDMI signals — feeding 16 physical screens arranged on the arch.

This project demonstrates that Chrome (and other modern browsers) can target the full 8K resolution that the Fresco presents to the host as a single virtual monitor, rendering the same multi-pass strip-and-rotate compositor architecture that Unity uses on the production side.

A single HTML file with embedded JavaScript using three.js. No build step. Three.js is vendored locally for fully airgapped operation.

## Visualization architecture

Three render passes, in order:

```
┌─────────────────────────────────┐
│ Unrolled canvas (8640×3840)     │   ← content layers composite here
│  ┌──┬──┬──┬──┬──┬──┬──┬──┐     │
│  │  │  │  │  │  │  │  │  │     │
│  ├──┼──┼──┼──┼──┼──┼──┼──┤     │   8 cells × 2 rows = 16 cells
│  │  │  │  │  │  │  │  │  │     │
│  └──┴──┴──┴──┴──┴──┴──┴──┘     │
└─────────────────────────────────┘
            │
            ▼  Strip-and-rotate transform
            │  Left half → CCW 90° → left of plate
            │  Right half → CW 90° → right of plate
            ▼
┌──────────────────┐
│ Plate (7680×4320)│   ← what feeds the Fresco
│  ┌──┬──┬──┬──┐   │
│  ├──┼──┼──┼──┤   │   4 cells × 4 rows
│  ├──┼──┼──┼──┤   │   Q1: top-left, Q3: top-right
│  ├──┼──┼──┼──┤   │   Q2: bot-left, Q4: bot-right
│  └──┴──┴──┴──┘   │
└──────────────────┘
            │
            ▼  Display to screen (one of three views)
            │
   ┌────────┴────────┬─────────────┐
   ▼                 ▼             ▼
[unrolled view]  [flat plate]  [3D arch preview]
```

The unrolled canvas is the authoring metaphor: imagine standing under the arch and looking up at the unrolled-flat ceiling. Content placed at the canvas's horizontal center maps to the apex of the physical arch. Content at the canvas's left/right edges maps to the floor edges of the two walls.

## Content layers

Multiple content layers can be composited into the unrolled canvas, each independently toggleable:

- **B** — Background image (cosmic panoramas; cycle through with N)
- **Q** — Test cards: 16 cells, each one quadrant of one of the four 4K test cards. Used to verify the strip-and-rotate transform produces correct output on each physical screen.
- **3** — Orbiting 3D shapes: spheres orbiting the canvas's horizontal centerline. Through the strip-and-rotate, they appear to circulate around the apex of the arch.
- **P** — Particle network: drifting points with lines connecting nearby pairs. The "constellation of thoughts" aesthetic primitive.
- **M** — Markers: colored corner blocks (TL=red, TR=green, BL=blue, BR=yellow), apex line (cyan, horizontal center), tunnel midline (magenta, vertical center), and cell grid lines. Useful for authoring and debugging.

Layers composite back-to-front in the order: B → Q → 3 → P → M (markers always on top).

## Setup

### One-time

Clone the repo:

```
git clone https://github.com/partsdept/tunnel-webgl-mvp.git
cd tunnel-webgl-mvp
```

The project requires no build step or package manager. Three.js is vendored under `vendor/three/`. Test cards and background images live under `assets/`.

### Running locally

The browser security model prevents loading the test-card and background image assets when opening `index.html` directly via `file://`. A local web server is required. Python's built-in HTTP server works fine:

```
cd ~/tunnel-webgl-mvp
python3 -m http.server 8000
```

Then in Chrome (or any modern browser), open:

```
http://localhost:8000
```

Stop the server with Ctrl-C in the terminal.

### Folder structure

```
tunnel-webgl-mvp/
├── index.html                  ← single-file WebGL app
├── vendor/three/               ← three.js library (vendored, airgapped)
│   ├── three.module.js
│   └── addons/controls/OrbitControls.js
├── assets/
│   ├── test-cards/             ← 4× 3840×2160 PNGs (TestCard_Q1..Q4.png)
│   ├── backgrounds/            ← high-res panoramas (PCI_SPACE_IMAGES_*.png)
│   └── equirect/               ← reserved for equirect mode (planned)
└── README.md
```

## Hotkeys

```
Visualization (TAB cycles):
  unrolled       Unrolled canvas (the authoring view)
  flat           Letterboxed 8K plate (what feeds the Fresco)
  arch           3D preview on actual arch geometry

Content layers (independent toggles):
  B              Background image on/off
  N              Cycle to next background (turns B on if off)
  Q              Test cards on/off
  3              Orbiting 3D shapes on/off
  P              Particle network on/off
  M              Markers (corner blocks, apex line, midline, cell grid)

Other:
  F              Fullscreen toggle
  R              Reset arch preview camera
  Mouse drag     Orbit (in arch view)
  Mouse wheel    Zoom (in arch view)
```

## Adding more backgrounds

Drop additional `.png` or `.jpg` files into `assets/backgrounds/` and add their paths to the `backgroundPaths` array near the top of `index.html`:

```javascript
const backgroundPaths = [
  './assets/backgrounds/PCI_SPACE_IMAGES_01.png',
  './assets/backgrounds/PCI_SPACE_IMAGES_02.png',
  './assets/backgrounds/PCI_SPACE_IMAGES_03.png',
  './assets/backgrounds/your_new_image.png',  // add here
];
```

For best fidelity at 8K output, use images with at least 4096×2048 resolution; ideal is 8640×3840 to match the unrolled canvas dimensions exactly.

## Production deployment

For driving the actual installation hardware:

1. Connect the Display Mac to the Fresco 8K splitter via HDMI 2.1
2. Configure the Mac's display output to 8K30 (System Settings → Displays)
3. Drag the Chrome window onto the Fresco-presented "8K monitor"
4. Press F (fullscreen API) or use Chrome's command-line `--start-fullscreen` flag

The Fresco abstracts the four downstream 4K screens as a single virtual 8K monitor via EDID. Chrome targets the full 7680×4320 resolution; the Fresco internally splits the signal to the four 4K HDMI outputs. Each Portta further splits to four 1080p screens. Pixel-perfect output through the chain.

### Launching fullscreen automatically

For unattended deployment (e.g., login items or launchd integration), Chrome can be launched fullscreen on a specific display via shell script:

```bash
open -na "Google Chrome" --args \
  --new-window \
  --start-fullscreen \
  --window-position=X,Y \
  "http://localhost:8000"
```

`X,Y` are the desktop coordinates of the Fresco-presented display's top-left corner. On a Mac with the Fresco as a secondary display to the right of the laptop's built-in display, this might be `--window-position=2560,0`.

## Performance findings

Tested on M4 Pro MacBook Pro:

- **Browser window (windowed)**: 120 fps in unrolled mode with all layers active
- **Fullscreen on 8K display through Fresco**: 30 fps (capped by 8K30 display sync)

The 30 fps cap matches what Unity sees on the same hardware path. The 8K30 output is the system constraint, not the rendering work — the GPU has headroom to render faster, but the display sync limits delivery.

Performance characterization on the production M4 Mac Mini base is pending.

## Architectural validation

This MVP demonstrates that:

1. **Browsers can target the full 8K plate resolution** when the Fresco presents itself as a single virtual monitor (which it does via EDID)
2. **The strip-and-rotate compositor architecture is implementable in WebGL** with comparable performance to Unity
3. **Content authored on the unrolled canvas wraps correctly onto the arch geometry** through the same mathematical transform
4. **WebGL/WebGPU-based interactive content has a clear path to the production hardware** — the team's existing graph-of-thoughts visualization (or any browser-based 3D content) could feed the Display Mac compositor identically to how Unity does

The architectural patterns (composable content layers, per-layer hotkey toggles, 3D preview on actual geometry) mirror the Unity implementation, making it straightforward for the team to develop on either substrate.

## What's not in this MVP

Functionality present in the Unity sibling but not yet ported here:

- OSC integration for external control
- Watched folder for dynamic image loading
- Slideshow mode with crossfades
- Equirect "cheat mode" projection
- Heartbeat sweep diagnostic overlay
- Multi-language drifting text
- Multiple sources for the slideshow

These are tractable additions if the team chooses WebGL as the production substrate. The architecture supports them; the work is implementation.

## License

Private project. All code authored by Anthropic Claude in collaboration with the project team for the 2026 Burning Man installation.
