# FAP Mobile — 2D Animation Studio (PWA)

**FAP** (Free Animation Power) is a mobile-first, zero-backend 2D animation web application. Draw frame-by-frame, use professional onion skinning, choose from 20 brushes, and export your animations as GIF or video — all from your phone's browser.

---

## Credits

**Design & Development:** Eduardo Fierro Duque

- Website: [www.fierroduque.com](https://www.fierroduque.com)
- LinkedIn: [eduardofierroduque](https://www.linkedin.com/in/eduardofierroduque/)
- GitHub: [eduardofierroduque-sudo](https://github.com/eduardofierroduque-sudo)

**Repository:** [FAP_MOBILE_WEB_VERSION](https://github.com/freeanimationpower/FAP_MOBILE_WEB_VERSION)

---

## Technical Stack

| Layer | Technology |
|-------|-----------|
| UI | HTML5, CSS3 (Flexbox, CSS Custom Properties) |
| Graphics | Canvas 2D API |
| Logic | Vanilla JavaScript ES6+ |
| Export | Custom GIF binary encoder (LZW), MediaRecorder API (WebM/MP4) |
| Files | `.fap` JSON format (PNG base64 frames) |
| Dependencies | **Zero** — no libraries, no frameworks, no backend |

---

## Architecture

### Data Model

```
state                  → currentFrame, totalFrames, fps, activeTool, brushType, onionEnabled
frameCanvases {}       → offscreen canvas per frame (white bg + strokes)
strokeCanvases {}      → cached transparent stroke layer per frame (lazy-built for onion skin)
undoStacks {}          → per-frame undo snapshots (ImageData, max 30)
redoStacks {}          → per-frame redo snapshots (ImageData, max 30)
```

### Rendering Pipeline (loadFrame)

1. Clear `ctx`, fill white background
2. **Onion Skin prev frames** (red tint): extract strokes via `getStrokeLayer()` → `tintAndDraw()` with `source-atop` composite
3. **Onion Skin next frames** (green tint): same technique
4. **Current frame** (solid): draw `getStrokeLayer(current)` on top

### Drawing Flow

```
startDraw → getPos() → drawDot()
moveDraw  → getPos() → stroke lines on ctx + fbCtx
endDraw   → saveStroke() → fbCtx copied to frameCanvases[key] + undo snapshot
```

**Coordinate mapping** (`getPos`): purely via `getBoundingClientRect()` — no manual zoom/pan math needed. CSS `transform` is already absorbed by the bounding rect.

---

## Features

### 🎨 20 Brushes

Brush engine powered by `BRUSHES` config object + `setBrushStyle()`:

| # | Brush | Technique |
|---|-------|-----------|
| 1 | **Round** | Standard circle, `lineCap:round` |
| 2 | **Square** | Square cap, `lineJoin:miter` |
| 3 | **Pencil** | 1px fixed, no anti-aliasing |
| 4 | **Soft** | `shadowBlur:3` GPU-native airbrush |
| 5 | **Calligraphy** | Angled ellipse stamping (45° slant, 0.3× ratio) |
| 6 | **Marker** | 1.4× scale, 65% alpha |
| 7 | **Ink** | 1.05× subtle round variant |
| 8 | **Chalk** | Jittered random dots along path |
| 9 | **Glow** | Double stroke: `shadowBlur:5` outer + 0.3× solid inner |
| 10 | **Pixel** | 1px square, crisp edges |
| 11 | **Splatter** | Random orbital particles around stroke |
| 12 | **Watercolor** | 4 translucent overlapping blobs |
| 13 | **Charcoal** | Random squarish particles with variable alpha |
| 14 | **Crayon** | Wobble offset perpendicular to stroke |
| 15 | **Hard Eraser** | Rectangular stamping along path |
| 16 | **Dots** | Spaced circles, no continuous line |
| 17 | **Crosshatch** | Rotated ±45° rects at each sample point |
| 18 | **Neon** | Intense glow: `shadowBlur:6` + solid core |
| 19 | **Fur** | 7 parallel strands with jitter |
| 20 | **Drip** | Main stroke + vertical drip drops |

### 🧅 Onion Skin

- **Red tint** for previous frames, **green tint** for next frames
- Uses `globalCompositeOperation: source-atop` for colorization
- Stroke layer cached via `getStrokeLayer()` — white pixels extracted to transparency
- Opacity decay: 35% (nearest) → 26% → 17.5% (farthest)
- Toggle via onion button in tool panel

### ⏪ Undo / Redo

- Per-frame history (30 snapshots max per frame)
- Snapshot = `getImageData()` of full frame canvas
- Redo stack clears on new stroke within same frame
- Cross-frame isolated — undo in frame 2 doesn't affect frame 1

### ▶️ Playback

- `requestAnimationFrame` with delta-time accumulator
- No `setInterval` jitter
- Catch-up limited to 4 frames (prevents spiral on lag spike)
- FPS selector: 8 / 12 / 24 / 30 / 60 fps

### 💾 Save / Load (`.fap` files)

Files saved as `FREEANIMATIONPOWER_XX.fap` with sequential numbering (`01`, `02`, ...). Format:

```
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
- File input reset (`value=''`) allows re-opening the same file

### 📤 Export

All exported files follow the naming convention **`FREEANIMATIONPOWER_XX.ext`**, where `XX` auto-increments from `01` to `99` per session (persists across sequential exports, resets on page reload).

#### GIF
- Custom pure-JS GIF encoder (LZW compression, global 256-color palette)
- **Color cube** (32³ precomputed lookup) for O(1) per-pixel quantization
- `async/await` with yields between frames — doesn't freeze UI
- Delay per frame: `100 / state.fps` centiseconds

#### Video (WebM / MP4)
- `captureStream(state.fps)` + `MediaRecorder`
- `recorder.start(100)` timeslice for periodic data delivery
- `requestData()` before `stop()` to flush pending frames
- `setTimeout`-chained frame drawing with correct frame duration
- Fallback codec chain: VP9 → VP8 → WebM → MP4

---

## UI Layout (top to bottom)

```
┌─────────────────────────┐
│      CANVAS AREA        │  flex:1
├─────────────────────────┤
│ ◀ ▶⏸ ⏹ ▶  0/24 FPS  ... │  toolbar (playback, export, save/load)
├─────────────────────────┤
│ ●●●●●●●●●●●●...         │  color strip (32 colors, scrollable)
├─────────────────────────┤
│ ◉ ■ │ ◎ ...            │  brush strip (20 brushes, scrollable)
├─────────────────────────┤
│ 🖌 ✏ 🪣 ◐ ✋  S= O=     │  tool panel (brush, eraser, fill, onion, hand, size, opacity, zoom)
├─────────────────────────┤
│ ◀ 1 2 3 ... 24 ▶       │  timeline
└─────────────────────────┘
```

### Viewport Lock (Mobile)

- `body: position:fixed; overflow:hidden` — no scroll, no reflow
- `--vh` CSS variable set via `window.innerHeight` — fixes 100vh bug on mobile browsers
- `touch-action: manipulation` on key elements
- Toolbar at bottom (thumb zone ergonomics)

---

## Key Functions Reference

| Function | Responsibility |
|----------|---------------|
| `getPos(e)` | Screen → canvas coordinate mapping via bounding rect |
| `loadFrame(fn)` | Full render: white bg + onion skin + current frame |
| `drawDot(x,y)` | Single point stamp (brush-aware) |
| `moveDraw(e)` | Stroke interpolation between last and current position |
| `saveStroke()` | Snapshot undo → copy `fbCtx` to `frameCanvases` → invalidate stroke cache |
| `getStrokeLayer(key)` | Lazy-build transparent stroke canvas from full frame |
| `tintAndDraw(ctx, src, color, alpha)` | Apply color tint + alpha to a canvas via `source-atop` |
| `setBrushStyle(ctx, br, color, alpha, lw)` | Configure ctx properties for a brush type |
| `ensureFrameCanvas(key)` | Create offscreen canvas for frame if missing |
| `exportGIF()` | Async GIF encoder with color cube |
| `exportVideo()` | Async video export via `MediaRecorder` |
| `saveProject()` | Serialize all frames to `.fap` JSON |
| `loadProjectFile(file)` | Parse `.fap`, load PNGs, rebuild timeline |
| `resizeCanvas()` | Resize + `clearAllFrameData()` |

---

## Browser Compatibility

| Feature | Chrome | Safari | Firefox | iOS Safari |
|---------|--------|--------|---------|------------|
| Canvas 2D | ✅ | ✅ | ✅ | ✅ |
| `requestAnimationFrame` | ✅ | ✅ | ✅ | ✅ |
| `shadowBlur` | ✅ | ✅ | ✅ | ✅ |
| `MediaRecorder` | ✅ | ✅ 14.1+ | ✅ | ✅ 14.5+ |
| `captureStream` | ✅ | ✅ | ✅ | ✅ |
| `globalCompositeOperation` | ✅ | ✅ | ✅ | ✅ |
| `getBoundingClientRect` | ✅ | ✅ | ✅ | ✅ |

---

## File Structure

```
FAP_MOBILE_WEB_VERSION/
├── index.html          # Single-file app (HTML + CSS + JS)
├── README.md
└── icons/
    ├── brushes/        # 20 brush PNG icons
    │   ├── round.png
    │   ├── square.png
    │   ├── pencil.png
    │   ├── ...
    │   └── drip.png
    ├── brush.png
    ├── eraser.png
    ├── export gif.png
    ├── export video.png
    ├── save project.png
    ├── open project.png
    ├── LUPA.png
    └── ...              # 50+ UI icons
```

---

## License

&copy; Todos los derechos reservados. Free Animation Power by Eduardo Fierro Duque, Santiago de Chile 2026.
