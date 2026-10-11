# Porcelain — codebase map for AI agents

Single-file browser app: all logic lives in `index.html`. No build step, no dependencies, no backend. Keep it that way unless the user asks otherwise.

## Architecture (index.html, top to bottom)

| Section | What it does |
|---|---|
| `<style>` | Dark single-theme UI, tokens in `:root` |
| markup | header actions / `#stage` (canvas + zoom gestures) / `#panel` (sliders + filters) |
| i18n block | `isZH` detect; Chinese markup is the source, English applied as overrides |
| `initLandmarker()` | MediaPipe Face Landmarker from `vendor/` (GPU, falls back to CPU) |
| landmark constants | `OVAL/EYE_L/EYE_R/BROW_*/LIPS` = MediaPipe FaceMesh indices |
| `buildMaskCanvas()` | RGBA mask at half res: **R=skin, G=eyes, B=fold band** |
| `buildOps()` | slider values → warp op list (push/zoom) for the shader |
| `FS_MAIN` | the one fragment shader: warp → smooth → fold-lift → whiten → eye light → color grade |
| `loadFile()` | decode (EXIF-aware), cap 2048px, detect face, build masks, render |
| `doSave()` | claude.ai `downloads` capability → real-browser `<a download>` → long-press overlay |
| zoom block | pinch/wheel/double-tap on `#gl` via CSS transform; UI never moves |

## Tuning table (the numbers users ask about)

| Ask | Where | Current | Safe range |
|---|---|---|---|
| smoothing strength cap | `FS_MAIN`: `mix(col, blur, uSmooth*m.r*keep)` + detail re-add `0.22` | 0.22 re-add | 0.1–0.35 |
| smoothing edge protection | `smoothstep(0.09, 0.28, diff)` | — | lower first edge = more protection |
| slim jaw strength | `buildOps()`: `params.slim/100 * 0.13` | 0.13 | 0.08–0.2 |
| cheek strength | `params.cheek/100 * 0.135` | 0.135 | 0.08–0.2 |
| eye zoom | `params.eye/100 * 0.22` | 0.22 | 0.1–0.3 |
| nose | `params.nose/100 * 0.14` | 0.14 | 0.08–0.2 |
| fold band width | `buildMaskCanvas()`: `faceW*0.055/2` lineWidth | 0.055 | 0.04–0.08 |
| fold band position | stroke from lm 98→61 (left), 327→291 (right), extended −0.12/+0.10, lateral offset 1.5% faceW | — | extend end to +0.25 to reach marionette lines |
| fold gain | `FS_MAIN`: `clamp(uFold*m.b*1.35, …)` | 1.35 | 1.0–1.8 |
| fold side-sample distance | `loadFile()` `mkOff`: `* 0.042` | 0.042 | 0.03–0.06 |
| filters | `FILTERS` array: `[exposure(stops), contrast, saturation, temp, tint, fade]` | 6 presets | add entries + a gradient swatch in the chip builder |
| whitening | `FS_MAIN`: `uWhiten*(0.26*m.r + 0.05)` | — | keep global term ≤ 0.08 |

Adding a slider: markup `.srow` with id `s-<key>` → add key to `params` → consume in `buildOps()` (warp) or as a shader uniform (color) → add to the auto/zero/reset presets and the no-face disable list → English label in the i18n block.

## Test loop

```sh
python3 -m http.server 8788   # from repo root
```

Open `http://localhost:8788/index.html`. Debug hooks on `window`:

- `__ketuParams` — mutate slider values directly (0–100)
- `__ketuRender()` — **synchronous** render; use this in automation (rAF stalls in background tabs and gives false "no effect" results)
- `__ketuMask` — the mask canvas; draw it or read channel stats to verify band placement

Generate synthetic test portraits rather than using anyone's real photo. Verify warps at slider = 100 (artifacts show at extremes) and check that background/hair edges don't tear.

## Gotchas

- Shaders are WebGL1 GLSL; uniform arrays are fixed at 16 ops — bump both the GLSL `[16]`/loop bound and the JS `Float32Array(64)` together
- The model file is named `face_landmarker.task.wasm` — the `.wasm` extension is a lie required by claude.ai artifact hosting; do not "fix" it
- Bump `CACHE` in `sw.js` whenever any cached asset changes, or returning users get stale code
- `DEV-SKELETON` comment blocks exist so the same file doubles as a claude.ai artifact (which injects its own skeleton); GitHub Pages serves the file as-is. Don't remove the markers
- Keep Chinese as the source language in markup; English lives in the i18n override block
