# Native 3D renderer (depth-first)

Retail draw code projects every vertex to integer screen coordinates and
orders polygons with the OT. PGXP recovers precision afterwards by matching
packet words against shadowed GTE results, and anything it cannot match falls
back to retail order. The native renderer instead submits exact camera-space
triangles, so the depth buffer decides visibility and nothing has to be
recovered. It runs beside the classic path: converted draw code submits
native layers, everything else keeps its retail packets (with PGXP).

Options > Enhancements > Renderer selects Classic or Native 3D (PC only;
Vita and Web keep Classic). Native implies the depth buffer.

## Pipeline

- `platform/native_draw3d.c`: per-frame layers of camera-space triangles.
  A layer holds a view (world-to-camera transform, GTE H/OFX/OFY, viewport,
  mirror flag) and an object transform for its input. Back faces are culled
  with the camera-space determinant, which matches retail NCLIP including
  mirror mode; triangles outside one frustum plane are dropped.
- A `DR_PSYX_DRAW3D` marker packet links a layer into the OT, so it draws
  in its viewport's draw environment and in order with retail 2D work.
- `NativeGpu_EmitDraw3DLayer` (native_gpu.c) expands the layer into
  homogeneous `GrVertex` data (`clipSpace` set): opaque triangles grouped by
  shader state, translucent ones far to near. They share the batch, shader,
  VRAM/CLUT sampling, STP and blend code with retail packets.
- The vertex shader passes clip-space vertices through. Clip w is camera
  depth and NDC depth is `1 - 64 / depth`, the PGXP encoding, so native and
  retail geometry share one depth buffer. The host clips at depth 32.
  Native triangles shade in perspective because screen-linear interpolation
  across a clipped corner behind the camera is undefined (Mesa draws black).
- Retail OT draw-order bias becomes a small relative depth offset
  (`depthBias`), which keeps coplanar decals over the surface below them.

## Converted

- Level geometry (`game/RenderLevel/NativeDrawLevel.c`): the retail BSP
  render lists and visibility bits, face permutation table (R226), UV flip,
  triangle faces, winding, texture LOD thresholds, animated textures,
  semi-transparency (tpage blend 3 = opaque), draw-order bias, super turbo
  tint, reverse-track turbo UVs, far low-LOD quads and env-mapped water with
  its distance fade. Retail near-camera subdivision is not needed.

## Look options

- Colour depth: 24-bit, or 15-bit PS1 framebuffer precision (5 bits per
  channel after dithering).
- Textures: nearest or bilinear.
- Dithering (Options) applies to both renderers.

## Known gaps

- Mosaic textures: with Max detail, retail subdivides near faces into 2x2,
  4x2 or 4x4 grids with their own high-resolution textures. Native stops at
  the near texture level, so close surfaces look softer than Classic.
- Full-dynamic LOD morph (only with Max detail off): the transition quads
  pop instead of morphing their middle vertices.
- Not yet converted: models, tires, shadows, skids, particles, weather,
  stars, cutscene and menu 3D. These keep PGXP.
- Split screen and mirror mode are covered by tests but not yet checked in
  a running race.

## Next phases

1. Mosaic textures and LOD morph, then in-game checks across tracks.
2. Instance models (RenderBucket), with their split line, ghosts and fades.
3. Tires, shadows and skids as depth-biased decals.
4. Effects, sky, weather, cutscenes and menu 3D.
5. Remove PGXP once every 3D path is native; Vita keeps Classic.

## Verification

`ctest --test-dir build --output-on-failure` runs `ctr_native_draw3d`
(layers, culling, frustum, transforms, marker) and, with
`CTR_NATIVE_RENDERER_TESTS=ON`, `ctr_native_depth_renderer`: native versus
retail depth in both submission orders, the implied depth buffer,
translucency, near-plane clipping, mirror mode and draw-order bias across
every anti-aliasing and PGXP mode, plus 15-bit quantisation.

Internal builds: Page Down toggles the renderer and Scroll Lock freezes game
logic while rendering continues, so both renderers can be compared on the
same frame. End toggles the depth buffer and F6 shows the debug overlay
(merged from fix-fireball-flash).

In the Adventure hub with logic frozen, Classic against Classic differs by
0 pixels and Native against Classic by under 1%, all of it the missing mosaic
textures on near surfaces.

## Status (2026-10-02)

- Phase 1 (native layer pipeline, look options, tests) is complete.
- Phase 2 (level geometry) works in game. Mosaic textures and the LOD morph
  are next.
- fix-fireball-flash (frame-rate-correct particles, F6 overlay) is merged
  into this branch. The overlay does not yet list the Renderer or Colour
  depth options.
