# Camera Zoom & Pan in Phaser 3

## Key Concepts

### Scroll Coordinates Are Unzoomed
Phaser docs: *"scrollX and scrollY are unzoomed."* This means:
- `cam.scrollX/scrollY` are in **world coordinates**, not screen pixels
- The viewport shows from `(scrollX, scrollY)` to `(scrollX + width/zoom, scrollY + height/zoom)`
- The viewport center in world coords: `(scrollX + width/(2*zoom), scrollY + height/(2*zoom))`

### cam.width/height vs Zoom
- `cam.width` and `cam.height` = viewport size in **screen pixels** (unaffected by zoom)
- With `Scale.RESIZE`, `cam.width/height` match the canvas CSS pixel size
- `cam.zoom` is independent — changing zoom does NOT change `cam.width/height`
- The visible world area = `cam.width / cam.zoom` × `cam.height / cam.zoom`

---

## Zoom-to-Center Pattern (Recommended)

When zooming, keep the viewport center fixed. Capture all values BEFORE calling `setZoom`:

```typescript
private zoomKeepCenter(newZoom: number): void {
  // Capture BEFORE setZoom — in RESIZE mode, cam.width could theoretically
  // change if setZoom triggers an internal resize cycle
  const w = this.cam.width
  const h = this.cam.height
  const oldZoom = this.cam.zoom

  // Current world point at viewport center
  const cx = this.cam.scrollX + w / (2 * oldZoom)
  const cy = this.cam.scrollY + h / (2 * oldZoom)

  // Set zoom, then recompute scroll so (cx,cy) stays at viewport center
  this.cam.setZoom(newZoom)
  this.cam.scrollX = cx - w / (2 * newZoom)
  this.cam.scrollY = cy - h / (2 * newZoom)
}
```

**Why not `cam.centerOn()`?** — `centerOn` internally reads `this.width` and `this.zoom` at call time. In RESIZE mode, if `cam.width` changed between `setZoom` and `centerOn`, the center drifts. Manually setting `scrollX`/`scrollY` with captured values is safer.

---

## Scale.RESIZE + Multi-Scene Gotchas

### Resize Events Reset Zoom
The `scale.on("resize")` event fires when the canvas/parent changes size. **Never call `applyFitZoom()` or reset zoom in the resize handler** — this destroys the user's zoom state. Common triggers:
- React re-render that changes layout (overlay appearing/disappearing)
- Browser toolbar show/hide (mobile `100dvh` changes)
- HMR/Vite hot reload

```typescript
// WRONG — resets zoom on every layout change
private onResize = (): void => {
  this.updateZoomLimits()
  this.applyFitZoom()  // ← DESTROYS user's zoom
}

// CORRECT — only clamp to new limits
private onResize = (): void => {
  this.updateZoomLimits()
  const z = Phaser.Math.Clamp(this.cam.zoom, this.minZoom, this.maxZoom)
  if (z !== this.cam.zoom) this.cam.setZoom(z)
}
```

### Two-Scene Input (globalTopOnly)
When using parallel scenes (game + UI overlay), Phaser's `game.input.globalTopOnly` (default `true`) causes the top scene's interactive objects to block input from reaching lower scenes. Fix:

```typescript
// In the game scene's create():
this.game.input.globalTopOnly = false
```

### UI Scene Camera Independence
Each scene has its own camera. The UI scene camera (zoom=1, no scroll) is completely independent from the game scene camera. Zooming the game camera does NOT affect UI elements.

---

## Mouse Wheel Zoom

### Listen on `window`, Not Canvas
DOM overlays (React HUD, navbar) sit above the Phaser canvas. Wheel events may fire on the overlay instead of the canvas. Listen on `window` and bounds-check:

```typescript
window.addEventListener("wheel", (event) => {
  const rect = canvas.getBoundingClientRect()
  if (event.clientX < rect.left || event.clientX > rect.right ||
      event.clientY < rect.top  || event.clientY > rect.bottom) return

  event.preventDefault()
  // ... zoom logic
}, { passive: false })
```

### Trackpad Sensitivity
Mac trackpad fires many small `deltaY` values (1-8 per event). Mouse wheels fire larger values (100+). Use proportional zoom:

```typescript
const SENSITIVITY = 0.001
const clampedDelta = Phaser.Math.Clamp(event.deltaY, -50, 50)
const delta = -clampedDelta * SENSITIVITY * cam.zoom  // proportional to current zoom
```

**Do NOT use a fixed zoom step** like `±0.1` — trackpad fires 30+ events per gesture and zooms way too fast.

---

## Camera Clamping (No setBounds)

Avoid `cam.setBounds()` with `Scale.RESIZE` — it has known bugs when the viewport resizes. Use manual clamping:

```typescript
private clampCamera(): void {
  const viewW = this.cam.width / this.cam.zoom
  const viewH = this.cam.height / this.cam.zoom

  // Soft clamp: allow 50% viewport padding beyond world edges
  const padX = viewW * 0.5
  const padY = viewH * 0.5

  const minX = -padX
  const maxX = this.worldW - viewW + padX

  if (minX > maxX) {
    // Viewport larger than world: center
    this.cam.scrollX = this.worldCenterX - viewW / 2
  } else {
    this.cam.scrollX = Phaser.Math.Clamp(this.cam.scrollX, minX, maxX)
  }
  // ... same for Y
}
```

**Important:** Don't force-center when viewport > world bounds — this LOCKS the camera and prevents all panning on that axis.

---

## CameraController Lifecycle

Always store the controller reference and call `destroy()` on scene shutdown. Otherwise, `window` event listeners leak and accumulate across HMR reloads:

```typescript
// In scene:
private cameraController: CameraController | null = null

create(): void {
  this.cameraController = new CameraController(this, config)
}

shutdown(): void {
  this.cameraController?.destroy()
  this.cameraController = null
}
```

The `destroy()` must remove ALL listeners:
- `scene.input.off("pointerdown/move/up")`
- `scene.scale.off("resize")`
- `window.removeEventListener("wheel")`
- `window.removeEventListener("keydown")`
