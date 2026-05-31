# Bathroom 3D Model — Project Documentation

## File
`/var/www/promofeatures/public/bathroom-3d.html`  
Standalone HTML file, no Laravel routing needed. Open directly at `/bathroom-3d.html`.

## Technology
- **Three.js r0.165.0** via importmap CDN
- `OrbitControls` for camera (rotate / scroll / right-drag pan)
- No bundler, no build step — pure browser JS module

---

## Room Geometry

### Fixed constants (cannot be changed without code edit)
| Constant | Value | Description |
|----------|-------|-------------|
| `W` | 2.34 m | Room width (X axis) |
| `H0` | 2.64 m | Ceiling height at entrance + flat section |
| `H1` | 1.4 m | Ceiling height at back wall |
| `flatZ` | 1.5 m | Z where flat ceiling ends and slope begins |
| `doorW` | 1.0 m | Door width |
| `doorH` | 2.05 m | Door height |
| `doorX` | W/2 − 0.30 | Door centre X (30 cm left of room centre, away from shower) |

### Dynamic params (controllable via UI inputs)
| Variable | Default | UI input id | Description |
|----------|---------|-------------|-------------|
| `roomL` | 3.21 m | `#room-length` | Room length (Z axis) |
| `roomPC` | 0.40 m | `#pipe-chase` | Pipe chase depth (back wall → gypsum wall) |
| `roomGW` | 1.10 m | `#gypsum-width` | Width of upper gypsum extension (mirror wall) |
| `roomSinkX` | 1.17 m | `#sink-x` | Sink / mirror wall centre X position |

Changing any of these and pressing **Rebuild room** (or just editing the number) calls `applyRoomParams()` → `buildRoom()`.

### Ceiling
- Flat section: z = 0 … flatZ, height = H0
- Sloped section: z = flatZ … L, height goes linearly from H0 → H1
- Formula at any z > flatZ: `H0 - (H0−H1) * (z−flatZ) / (L−flatZ)`

### Axes convention
- X: left (0) → right (W = 2.34 m)
- Y: floor (0) → ceiling
- Z: entrance (0) → back wall (L = 3.21 m)

---

## Pipe Chase (Короб для труб)

The hidden plumbing chase sits between the back wall and the gypsum board wall.

| Derived value | Formula |
|---------------|---------|
| `gypsumZ` | `L − PC` — gypsum wall centre Z |
| `gypsumFace` | `gypsumZ − GYP/2` — front face of gypsum wall |
| `GYP` | 0.0125 m (12.5 mm) — gypsum board thickness |
| `ceilAtGypsum` | ceiling height at gypsumZ (using slope formula) |
| `gypsumTopH` | `ceilAtGypsum − 1.0` — height of upper mirror-wall extension |

### Gypsum board components
1. **Lower panel** — full width W × 1.0 m high (tiled, `wallBoxMat`)
2. **Upper extension (mirror wall)** — `roomGW` wide × `gypsumTopH` high (tiled, `wallBoxMat`)
3. **Horizontal cap** — W wide × GYP thick, at y = 1.0, spans full pipe chase depth PC (plain grey)
4. **Side panels** — GYP wide × `(H1−1.0)` high × PC deep, one on each side of mirror extension (plain grey)

---

## Fixtures

All fixture Z positions are derived from `gypsumFace` so they stay flush when room length / pipe chase depth changes.

| Fixture | X centre | Z centre | Notes |
|---------|----------|----------|-------|
| Bathtub | `W − 0.4` | `gypsumFace − 0.8` | Flush right wall + gypsum wall; 0.8×1.6×0.55 m |
| Toilet | 0.38 | `gypsumFace − 0.26` | Left side, flush gypsum wall |
| Sink | `roomSinkX` | `gypsumFace − 0.21` | Cabinet 0.5×0.78×0.42 m; mirror above |
| Shower | `W − shW/2` | `shD/2` | Right-front corner; 0.85×0.85×2.0 m, glass walls |
| Towel rail | `W − 0.06` | 1.5 | Chrome rail on right wall |

---

## Walls (tiled surfaces)

| Surface | Material | Notes |
|---------|----------|-------|
| Floor | `floorMatB` (FloorTex) | Full W × L |
| Side walls (left & right) | `wallFrontMat` (ShapeGeometry + ShapeTex) | Sloped trapezoid shape |
| Entrance wall (around door) | `wallBoxMat` | 3 boxes: left, right, top-above-door |
| Gypsum wall lower | `wallBoxMat` | W × 1.0 m |
| Gypsum wall upper (mirror) | `wallBoxMat` | roomGW × gypsumTopH |
| Back wall top strip | `wallBoxMat` | W × min(H1, 1.0) = W × 1.0 m |
| Back wall bottom strip | plain `0xd4c9b8` | W × max(0, H1−1.0) = W × 0.4 m |

### UV / texture note
- **ShapeGeometry** (side walls): UV = raw world coords → `repeat = 1/tileSizeMetres`
- **BoxGeometry** (all other walls): UV = 0…1 per face → `repeat = surfaceSize/tileSizeMetres`

---

## Tile System

### Size selects
- `#floor-size` — floor tile W×H in metres (comma-separated value)
- `#wall-size` — wall tile W×H

Available presets: 60×120, 60×60, 80×80, 120×120, 20×120, 15×90, 7.5×15, 5×25, 10×30, 10×10, 15×15 cm

### Image
- Default image loaded from `/6836051_1.webp`
- Custom URL via `#tile-url` + **Apply** button (or Enter key)
- Canvas texture: tile image + 2px white grout border (`_buildTileCanvas`)
- `_lastTileCanvas` — stored so tiles survive `buildRoom()` rebuilds

### Texture tracking arrays (cleared on rebuild)
| Array | Purpose |
|-------|---------|
| `_tileTextures` | All textures (for image reload) |
| `_floorTexRefs` | `{tex, sw, sh}` — BoxGeometry floor faces |
| `_wallBoxRefs` | `{tex, sw, sh}` — BoxGeometry wall faces |
| `_wallShapeRefs` | `tex[]` — ShapeGeometry side walls |

---

## Estimate Widget (`#estimate-panel`)

Located bottom-left, below room info. Recalculates automatically on every `buildRoom()` or tile size change.

### Tile counts (+10% waste)
- **Floor**: `floorArea = W × L`
- **Walls** = 2× side walls (trapezoid shoelace area) + entrance wall (W×H0 − door) + gypsum wall face + back wall top strip
- Tile count = `ceil(area × 1.10 / tileArea)`

### Tile costs
- `#floor-tile-price` (₴/m²) → floor cost
- `#wall-tile-price` (₴/m²) → wall cost
- **Плитка разом** — floor + wall cost total (gold)

### Substrate boards
| Product | Sheet size | Price |
|---------|-----------|-------|
| Knauf Aquapanel Indoor | 2400×900 mm = 2.16 m² | 2 198 ₴ |
| Rigips PRO Hydro | 2500×1200 mm = 3.0 m² | 499 ₴ |

- **Всі стіни** — based on `totalWallArea × 1.10`
- **Мокра зона** (Aquapanel) — based on `W × L` (entire bathroom floor area = entire bathroom is wet zone)
- **Суха зона** (Rigips) — `totalWallArea − W×L`

### Board overlay visualisation
Checkboxes `#show-aqua` / `#show-rigips` toggle green sheet overlays on all wall surfaces:
- Green = full sheet, yellow-green = cut piece, white lines = joints
- Meshes in `_aquaMeshes` / `_rigipsMeshes`, cleared + rebuilt on every `buildRoom()`
- Covered surfaces: gypsum wall (lower + mirror), back wall (full H1), entrance wall (3 pieces), left + right side walls

---

## UI Panels

### Left column (`#left-column`, pointer-events: none except estimate)
- `#info` — static room dimensions
- `#estimate-panel` — live estimate (pointer-events: auto so checkboxes work)

### Right panel (`#tile-panel`)
All inputs trigger `applyRoomParams()` or `updateEstimate()` via `input` event (no button needed for room params — live update).

| Input id | Purpose |
|----------|---------|
| `#floor-size` | Floor tile size select |
| `#floor-tile-price` | Floor tile ₴/m² |
| `#wall-size` | Wall tile size select |
| `#wall-tile-price` | Wall tile ₴/m² |
| `#tile-url` | Tile image URL |
| `#tile-apply` | Load tile image |
| `#room-length` | Room length (roomL) |
| `#pipe-chase` | Pipe chase depth (roomPC) |
| `#gypsum-width` | Mirror wall width (roomGW) |
| `#sink-x` | Sink X position (roomSinkX) |
| `#room-apply` | Rebuild room button |

---

## Key Functions

| Function | Description |
|----------|-------------|
| `buildRoom()` | Clears and rebuilds entire scene; calls board overlays at end |
| `clearRoom()` | Removes all roomMeshes, roomLights, tile textures, board overlays |
| `applyRoomParams()` | Reads UI inputs → updates roomL/PC/GW/SinkX → buildRoom() → updateTiles → updateEstimate |
| `updateEstimate()` | Recomputes all m² and costs; reads tile price inputs |
| `updateFloorTiles(w,h)` | Updates floor tile repeat on existing textures |
| `updateWallTiles(w,h)` | Updates wall tile repeat (BoxGeometry + ShapeGeometry) |
| `_buildTileCanvas(img)` | Renders tile image + grout to canvas; applies to all textures |
| `_makeBoardCanvas(surfW,surfH,sheetW,sheetH)` | Canvas with green sheet grid + white joints |
| `_addBoardSurface(arr, ...)` | Creates PlaneGeometry overlay, adds to scene hidden |

---

## Rebuild Flow

```
User edits input → applyRoomParams()
  └─ buildRoom()
       ├─ clearRoom()                  // dispose all meshes, textures, overlays
       ├─ build geometry (floor, ceiling, walls, gypsum, fixtures, lights)
       ├─ build board overlays          // _aquaMeshes, _rigipsMeshes
       └─ reapply _lastTileCanvas       // tile image survives rebuild
  └─ updateFloorTiles / updateWallTiles // reapply tile sizes
  └─ updateEstimate()                  // recalculate costs
```

---

## Notes for Future Sessions
- The back wall tile strip is only the **top 1.0 m** of H1=1.4 m (bottom 0.4 m is plain wall colour — windows look onto back wall)
- Gypsum board thickness is exactly **12.5 mm** (`GYP = 0.0125`)
- All fixture positions depend on `gypsumFace`; change `roomPC` or `roomL` and everything repositions
- Side walls use `ShapeGeometry` — UV coords are raw world coordinates, NOT 0-1
- `#left-column` has `pointer-events: none`; only `#estimate-panel` has `pointer-events: auto`
- Tile image default: `/6836051_1.webp` (served from Laravel `public/` folder)
