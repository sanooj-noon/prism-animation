# Prism Animation

A self-contained WebGL2 homepage loading animation: a glass prism disperses a
white ray into a VIBGYOR spectrum, then transitions into a homepage UI. It is a
faithful port of the vgpu.sh hero (WebGPU/WGSL -> WebGL2/GLSL) with exact
optical constants.

## Layout
- `index.html` — EVERYTHING lives here: CSS, homepage DOM, all GLSL shaders
  (template strings), CPU spectral ray tracer, and the render loop. ~2000 lines.
- `serve.py` — static server with no-cache headers. Run `python3 serve.py`,
  open http://localhost:8321. Always use this (browsers heuristically cache
  index.html otherwise).
- `Prism Chunks/` — the original vgpu.sh production bundle the physics was
  extracted from. Reference only; not loaded by the page.
- `home page.png` — reference screenshot of the target homepage (the UI is now
  real DOM, ported from a React recreation).

## Architecture (single frame)
1. CPU: `buildLightMesh()` traces 64 wavelengths x 24 beam slices through the
   2D prism cross-section (refraction, TIR up to 3 bounces, Fresnel), writing a
   vertex buffer (white beam + internal strips + outgoing fan).
2. GPU passes: background HDR (glass back interface + light sheet, 4x MSAA,
   supersampled) -> scene HDR (copy + glass front interface, refracting the
   background) -> bloom pyramid (threshold 0.1, 4 levels) -> present
   (Karis-weighted resolve + ACES + sRGB) -> canvas (reveal mix + FXAA-era
   copy) -> dust + ambient particles (additive).
3. The homepage is a DOM layer *under* the canvas; the canvas uses
   `mix-blend-mode: screen`, so the UI appears through the light.

## Key constants (top of the script in index.html)
- Optics: IOR 1.645, absorption (1,1,0.54), dispersion n(l)=1.245+0.06/l^2.
- `PRISM_SCALE` sizes the prism; the edge bevel deliberately does NOT scale
  (sub-pixel bevels render as broken dashes).
- Timeline lives in `heroRevealProgress()` — fade-in, ray sweep (quintic),
  dispersion open, camera charge-back, homepage reveal, loop fade. The loop is
  `elapsed % REVEAL_CYCLE`.
- Controls: click/R = replay, F/double-click = fullscreen.

## Hard-won lessons (do not regress)
- Never point-sample the refracted background sharply: the refraction lookup
  must blur where the uv sweeps fast (bevels) or edges break into fragments.
- Bright thin rims need luminance-weighted (Karis) resolves; plain averaging
  flickers regardless of MSAA.
- Any per-frame work must be smooth-paced: full-res generateMipmap or chunky
  update thresholds read as visible stutter/stepping.
- Rotation and light updates should be continuous functions of time, not
  lerp-chased targets (catch-up lurches) or coarse thresholds (stepping).

## Deploy
Push to `main` — GitHub Pages serves https://sanooj-noon.github.io/prism-animation/
