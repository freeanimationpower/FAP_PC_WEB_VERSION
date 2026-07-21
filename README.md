# FAP Desktop — 2D Animation Studio for PC

**FAP** (Free Animation Power) is a zero-backend, zero-dependency 2D frame-by-frame animation web application — now optimized for **desktop workstations with drawing tablet support**.

---

## Desktop vs Mobile — Key Differences

| Feature | FAP Mobile | FAP Desktop |
|---|---|---|
| **Target** | Phone / tablet (touch) | PC / workstation (mouse + drawing tablet) |
| **Canvas** | 9:16 portrait, auto-sized to phone viewport | **16:9 landscape, fixed 1920×1080 px** internal resolution |
| **Input** | Touch + mouse + basic pointer events | **Pointer Events API** (unified mouse + stylus + tablet) |
| **Pressure** | Not used in drawing | **Live pressure modulation** — brush size responds to stylus pressure in real time |
| **Tilt** | Not supported | **Stylus tilt** drives calligraphy brush angle |
| **Coalesced events** | No | **Yes** — `getCoalescedEvents()` for smooth high-rate strokes |
| **Viewport** | Phone simulator (390×844 px, rounded corners) | **Full browser window**, responsive to any monitor size |
| **Layout** | Tools below canvas (thumb zone) | **Unified toolbar on top** — single compact row |
| **Pan arrows** | 4 arrow buttons | **Space+drag**, hand tool, mouse wheel zoom to cursor |
| **Colors** | 32 | **64** (including ultra fluor/neon palette) |
| **Scrollbars** | Hidden (mobile UX) | **Visible** on all tool bars for small screens |
| **Shortcuts** | Ctrl+Z, arrows, Space | **Full keyboard map** (B/E/G/H tools, 1-9 brushes, `[`/`]` size, Ctrl+S/O, mouse wheel zoom, etc.) |

---

## Credits

**Design & Development:** Eduardo Fierro Duque

- Website: [www.fierroduque.com](https://www.fierroduque.com)
- LinkedIn: [eduardofierroduque](https://www.linkedin.com/in/eduardofierroduque/)
- GitHub: [eduardofierroduque-sudo](https://github.com/eduardofierroduque-sudo)

**Repositories:**

| Version | Repo |
|---|---|
| Mobile | [FAP_MOBILE_WEB_VERSION](https://github.com/freeanimationpower/FAP_MOBILE_WEB_VERSION) |
| Desktop | [FAP_PC_WEB_VERSION](https://github.com/freeanimationpower/FAP_PC_WEB_VERSION) |

---

## Technical Stack

| Layer | Technology |
|---|---|
| UI | HTML5, CSS3 (Flexbox, CSS Custom Properties) |
| Graphics | Canvas 2D API — fixed 1920×1080 internal resolution |
| Input | **Pointer Events API** (unified mouse + stylus + tablet), `setPointerCapture`, `getCoalescedEvents` |
| Logic | Vanilla JavaScript ES6+ |
| Export | Custom GIF binary encoder (LZW), MediaRecorder API (WebM/MP4) |
| Files | `.fap` JSON format (PNG base64 frames) |
| Dependencies | **Zero** — no libraries, no frameworks, no backend |
| Stylus support | Pressure sensitivity, tilt angle, coalesced events — works with **Wacom, Huion, XP-Pen, and all Wintab/WinTab-compatible tablets** |

---

## Architecture

### Input System — Pointer Events

FAP Desktop uses the **Pointer Events API** as its sole input system, replacing the three separate event systems (touch + mouse + basic pointer) from the mobile version:

```
pointerdown  → setPointerCapture(e.pointerId) → startDraw(e)
pointermove  → getCoalescedEvents() → moveDraw(e) for each coalesced point
pointerup    → endDraw(e) → saveStroke()
pointercancel / lostpointercapture → endDraw(e)
```

**Why Pointer Events:**
- Unifies mouse, touch, and stylus into a single API
- Native `e.pressure` for stroke weight modulation
- Native `e.tiltX` / `e.tiltY` for calligraphy brush angle
- `getCoalescedEvents()` captures all intermediate hardware reports for smooth strokes
- `setPointerCapture()` keeps tracking even when cursor leaves canvas

### Pressure Modulation

```
effectiveSize = brushSize × (0.2 + e.pressure × 0.8)
```

Maps pressure range [0, 1] → [20%, 100%] of configured brush size. Light touch = thin line, hard press = full thickness. Applied to every brush rendering call in both `moveDraw()` and `drawDot()`.

### Rendering Pipeline (loadFrame)

1. Clear `ctx`, fill white background
2. **Onion Skin prev frames** (red tint): extract strokes via `getStrokeLayer()` → `tintAndDraw()` with `source-atop` composite
3. **Onion Skin next frames** (green tint): same technique
4. **Current frame** (solid): draw `getStrokeLayer(current)` on top

### Canvas Sizing

- **Fixed internal resolution**: 1920 × 1080 px — always renders at full HD regardless of window size
- **CSS scaling**: canvas element fills `.canvas-surface` via `width: 100%; height: 100%`
- **Surface fitting**: `fitCanvasSurface()` JavaScript function computes the maximum 16:9 box that fits within the available viewport area, recalculated on every window resize

---

## Features

### 40 Brushes (20 Classic + 20 New)

Brush engine powered by `BRUSHES` config object + `setBrushStyle()`:

**Classic brushes (#1-20):**

| # | Brush | Technique | Pressure |
|---|-------|-----------|----------|
| 1 | **Round** | Standard circle, `lineCap:round` | Yes |
| 2 | **Square** | Square cap, `lineJoin:miter` | Yes |
| 3 | **Pencil** | 1px fixed, no anti-aliasing | No (fixed) |
| 4 | **Soft** | `shadowBlur:3` GPU-native airbrush | Yes |
| 5 | **Calligraphy** | Angled ellipse stamping (via **stylus tilt** or fixed 45° slant, 0.3× ratio) | Yes + Tilt |
| 6 | **Marker** | 1.4× scale, 65% alpha | Yes |
| 7 | **Ink** | 1.05× subtle round variant | Yes |
| 8 | **Chalk** | Jittered random dots along path | Yes |
| 9 | **Glow** | Double stroke: `shadowBlur:5` outer + 0.3× solid inner | Yes |
| 10 | **Pixel** | 1px square, crisp edges | No (fixed) |
| 11 | **Splatter** | Random orbital particles around stroke | Yes |
| 12 | **Watercolor** | 4 translucent overlapping blobs per step | Yes |
| 13 | **Charcoal** | Random squarish particles with variable alpha | Yes |
| 14 | **Crayon** | Wobble offset perpendicular to stroke | Yes |
| 15 | **Hard Eraser** | Rectangular stamping along path | Yes |
| 16 | **Dots** | Spaced circles, no continuous line | Yes |
| 17 | **Crosshatch** | Rotated ±45° rects at each sample point | Yes |
| 18 | **Neon** | Intense glow: `shadowBlur:6` + solid core | Yes |
| 19 | **Fur** | 7 parallel strands with jitter | Yes |
| 20 | **Drip** | Main stroke + vertical drip drops | Yes |

**New brushes (#21-40) — Desktop exclusives:**

| # | Brush | Technique | Pressure |
|---|-------|-----------|----------|
| 21 | **Airbrush** | Random scattered dots in circular cone radius, density decays from center outward | Yes |
| 22 | **Sand** | 25-40 fine grain 1px dots scattered in a narrow band along stroke path | Yes |
| 23 | **Sponge** | 3-6 irregular rotated elliptical blob stamps per step | Yes |
| 24 | **Pastel** | Chunky directional rectangles with soft edges, 2-3 per step | Yes |
| 25 | **Gravel** | Random 3-5 vertex polygon shapes scattered along stroke path | Yes |
| 26 | **Rake** | 5-7 parallel thin lines perpendicular to stroke direction (comb/rake effect) | Yes |
| 27 | **Chain** | Interlocking hollow circles alternating position along stroke | Yes |
| 28 | **Zigzag** | Angular zigzag pattern superimposed on stroke direction | Yes |
| 29 | **Wave** | 3-phase sinusoidal waves (main + 2 offsets at ± amplitude) | Yes |
| 30 | **Lace** | Delicate circular pattern of dots in repeating formation | Yes |
| 31 | **Oil** | 3-layer impasto: overlapping strokes with random offset and variable alpha | Yes |
| 32 | **Dry Brush** | Scratchy broken stroke — only draws ~55% of sample points with shallow marks | Yes |
| 33 | **Smudge** | Wide soft blur — 4 layers of large low-opacity overlapping circles | Yes |
| 34 | **Acrylic** | Bold opaque base + 3-5 directional short parallel texture lines | Yes |
| 35 | **Gouache** | Flat fill circle + darker outline edge (`darkenColor(55%)`) | Yes |
| 36 | **Sparkle** | 4-point star stamps (crossed lines + center dot), 2-3 per step | Yes |
| 37 | **Comet** | Bright head + 5 trailing dots of decreasing size and opacity | Yes |
| 38 | **Graffiti** | Thick stroke in selected color + darker contrasting outline (`darkenColor(30%)`) | Yes |
| 39 | **Spiderweb** | 6-8 thin radial lines from each sample point + solid center dot | Yes |
| 40 | **Smoke** | 5-8 expanding soft circles with increasing radius and decreasing opacity | Yes |

### Onion Skin

- **Red tint** for previous frames, **green tint** for next frames
- Uses `globalCompositeOperation: source-atop` for colorization
- Stroke layer cached via `getStrokeLayer()` — white pixels extracted to transparency
- Opacity decay: 35% (nearest) → 26% → 17.5% (farthest)
- Toggle via onion button or **O key**

### 64-Color Palette

- 32 standard colors (classic web palette)
- 32 **ultra fluor/neon** colors (hot magenta `#FF006E`, neon green `#39FF14`, electric cyan `#00D4FF`, fluorescent violet `#7B00FF`, fluorescent orange `#FF5500`, etc.)
- Scrollable strip with visible scrollbar

### Undo / Redo

- Per-frame history (30 snapshots max per frame)
- Snapshot = `getImageData()` of full frame canvas
- Redo stack clears on new stroke within same frame
- Cross-frame isolated — undo in frame 2 doesn't affect frame 1

### Playback

- `requestAnimationFrame` with delta-time accumulator
- No `setInterval` jitter
- Catch-up limited to 4 frames (prevents spiral on lag spike)
- FPS selector: 8 / 12 / 24 / 30 / 60 fps

### Save / Load (`.fap` files)

Format:

```json
{
  "v": 1,
  "fps": 24,
  "total": 24,
  "frames": {
    "0": "data:image/png;base64,...",
    "1": null
  }
}
```

- Frames serialized as PNG via `canvas.toDataURL('image/png')`
- Load via `FileReader` → `JSON.parse` → `Image.onload` → `drawImage` to frame canvas
- Validation: checks `v`, `total`, `frames` structure
- Empty frames stored as `null`, initialized via `ensureFrameCanvas`

### Export

All exported files follow naming **`FAP_ANIMATION_XX.ext`**, auto-incrementing per session.

#### GIF
- Custom pure-JS GIF encoder (LZW compression, global 256-color palette)
- **Color cube** (32³ precomputed lookup) for O(1) per-pixel quantization
- `async/await` with yields between frames — doesn't freeze UI

#### Video (WebM/MP4)
- `captureStream(state.fps)` + `MediaRecorder`
- Fallback codec chain: VP9 → VP8 → WebM → MP4

---

## Keyboard Shortcuts

A `?` button on the right side of the toolbar opens a dropdown panel with all shortcuts. Press **Esc** or click outside to close it.

### Tools

| Key | Action |
|---|---|
| **B** | Brush tool |
| **E** | Eraser |
| **G** | Fill / bucket |
| **H** | Hand / pan |
| **O** | Toggle onion skin |

### Navigation

| Key | Action |
|---|---|
| **Space** (tap) | Play / Pause |
| **Space** (hold + drag) | Temporary pan (releases tool on Space up) |
| **← →** | Previous / Next frame |
| **Home / End** | First / Last frame |

### Brush

| Key | Action |
|---|---|
| **[ / ]** | Brush size -1 / +1 px |
| **Shift+[ / ]** ({ / }) | Opacity -5% / +5% |
| **1 2 3 … 9 0 - =** | Quick brush select (round → watercolor) |

### Zoom

| Key | Action |
|---|---|
| **+ / -** | Zoom in / out |
| **Ctrl + 0** | Reset zoom |
| **Mouse wheel** on canvas | Zoom towards cursor |

### Edit & Export

| Key | Action |
|---|---|
| **Ctrl + Z** | Undo |
| **Ctrl + Shift + Z** | Redo |
| **Ctrl + S** | Save project (.fap) |
| **Ctrl + O** | Open project (.fap) |
| **Ctrl + Shift + G** | Export GIF |
| **Ctrl + Shift + V** | Export video |

---

## UI Layout

```
┌──────────────────────────────────────────────────────────────┐
│ [logo] | undo redo | ◀ ▶ ■ ▶ | 0/24 + - FPS▾ | | B E G ◐ H | | S▬5px O▬100% | | + - 🔍 | gif mp4 save open │  ← unified toolbar (40px)
├──────────────────────────────────────────────────────────────┤
│ ◉ ■ ◎ │ ◉ ... (40 brushes, scrollable)                       │  ← brush strip (30px)
├──────────────────────────────────────────────────────────────┤
│ ●●●●●●●●●●●●●●●●●●●●... (64 colors, scrollable)               │  ← color strip (30px)
├──────────────────────────────────────────────────────────────┤
│                                                              │
│                    CANVAS 16:9                               │  ← fills remaining space
│               1920 × 1080 px (internal)                      │     auto-fitted to viewport
│                                                              │
├──────────────────────────────────────────────────────────────┤
│ ◀ 1 2 3 4 5 ... 23 24 ▶                                     │  ← timeline (68px)
└──────────────────────────────────────────────────────────────┘
```

---

## Drawing Tablet Compatibility

FAP Desktop uses the standard **Pointer Events API** — supported natively by all modern browsers on Windows, macOS, and Linux. This means it works with:

- **Wacom** (Intuos, Cintiq, Bamboo, One, etc.)
- **Huion** (Kamvas, Inspiroy, H Series, etc.)
- **XP-Pen** (Artist, Deco, Star Series, etc.)
- **Gaomon, Veikk, Parblo, Ugee**
- **Microsoft Surface** (Surface Pen, Surface Slim Pen)
- **iPad + Apple Pencil** (via Safari on macOS/Windows remote... or directly on iPad Safari)
- Any **Wintab-compatible** or **Windows Ink** device

Pressure sensitivity and tilt are available **automatically** — no drivers or plugins needed. The browser exposes `e.pressure`, `e.tiltX`, and `e.tiltY` directly through the Pointer Events API.

---

## Browser Compatibility

| Feature | Chrome | Edge | Firefox | Safari |
|---|---|---|---|---|
| Canvas 2D | ✅ | ✅ | ✅ | ✅ |
| Pointer Events | ✅ | ✅ | ✅ 59+ | ✅ 13+ |
| Pressure sensitivity | ✅ | ✅ | ✅ | ✅ |
| Tilt X/Y | ✅ | ✅ | ✅ | ✅ |
| `getCoalescedEvents` | ✅ | ✅ | ✅ | ✅ |
| `setPointerCapture` | ✅ | ✅ | ✅ | ✅ |
| `requestAnimationFrame` | ✅ | ✅ | ✅ | ✅ |
| `shadowBlur` | ✅ | ✅ | ✅ | ✅ |
| `MediaRecorder` | ✅ | ✅ | ✅ | ✅ 14.1+ |
| `captureStream` | ✅ | ✅ | ✅ | ✅ |

---

## File Structure

```
FAP_PC_WEB_VERSION/
├── index.html          # Single-file app (HTML + CSS + JS)
├── README.md
└── icons/
    ├── brushes/        # 40 brush PNG icons
    └── *.png / *.svg   # 55+ UI icons (shared with mobile version)
```

---

## License

&copy; Todos los derechos reservados. Free Animation Power by Eduardo Fierro Duque, Santiago de Chile 2026.
