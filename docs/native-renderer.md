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

## Current status (2026-10-02)

The main level, model, sky and effect paths are implemented. Menu, HUD and
cutscene models are also converted through the common instance handlers,
including screen-space models, shared-range title models and the talking mask.
The latest executable builds and is installed as `ctr_native`; its preceding
version is saved as `ctr_native.before-native-menu-hud-cutscenes`.

Implementation coverage does not imply visual verification. Tiger Temple's
black-ring fix is confirmed by the user in game. The original Adventure hub
level comparison was checked in game. The subsequent model, tire/decal,
sky/effect and menu/HUD/cutscene conversions still need in-game checks.

| Area | Implementation | Verification still needed |
| --- | --- | --- |
| Native layer/GPU pipeline | Implemented: transforms, clipping, perspective, depth, materials and blending | Full running-game regression after the latest conversions |
| Level geometry | Implemented: BSP visibility, textures, mosaics, LOD morph and water | Track coverage and non-Max detail quirks |
| Tiger Temple sky order | Fixed: level marker `0x3fc`, lower sky `0x3fd` | User confirmed the fix; include it in future regressions |
| Instance models | Normal, depth-split, water-split, reflection and ghost handlers converted | Animation, split surfaces, reflections and ghost blending |
| Tires and ground decals | Solid/reflected tires, shadows and skids converted | Slopes, fading, water reflections and coplanar depth bias |
| Sky and effects | Sky, stars, world particles, weather, beaker rain, confetti, flares, webs and heat distortion converted | Appearance, line widths, blend order and framebuffer feedback |
| Menu, HUD and cutscene 3D | Common model handlers, screen models, title models, hint mask and cutscene line converted | Transitions, placement, offscreen wumpa, cutscene animation and overlay depth |
| Screen-space driver particles | Native sprite quads and line streaks with isolated depth at their retail OT slots | Placement, blending, mirror mode, split screen and overlay performance |
| Screen-space/owned-buffer tires | Native wheel quads with isolated overlay depth | Placement, sprite selection, blending and offscreen rendering |
| Warp-pad beam and transition flag | Native warp strips and animated flag mesh | Warp colours/order/mirroring, flag transitions and loading-screen layering |
| Ordinary 2D UI | Native homogeneous screen geometry for sprites, text, lines, rectangles and UI polygons; existing packets carry state/order | Appearance and ordering around native 3D overlays |

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
  VRAM/CLUT sampling, STP and blend code with retail packets. Ghost triangles
  retain their mask/body order through an ordered blend mode: all passes,
  including non-STP texture texels, run after opaque visibility with depth
  testing and no depth writes.
- The vertex shader passes clip-space vertices through. Clip w is camera
  depth and NDC depth is `1 - 64 / depth`, the PGXP encoding, so native and
  retail geometry share one depth buffer. The host clips at depth 32.
  Native triangles shade in perspective because screen-linear interpolation
  across a clipped corner behind the camera is undefined (Mesa draws black).
- The level marker uses OT slot `0x3fc`, after the sky at `0x3ff` and
  its negative face offsets. Tiger Temple's black lower sky uses a `-8`
  byte offset (`0x3fd`); placing the level at `0x3fe` let that sky paint
  over the road and walls as a large black ring.
- Retail OT draw-order bias becomes a small relative depth offset
  (`depthBias`), which keeps coplanar decals over the surface below them.
- Screen-space models, owned-push-buffer models and the talking mask use
  isolated depth at their existing OT position. Their colour goes into the
  active target, while their depth stays separate from the world. PS1 mask
  stencil is preserved; screen and offscreen targets and MSAA are supported.
- Layer storage holds up to 16384 layers per frame and resets with the OT
  before game logic can submit effects.

## Converted

- Level geometry (`game/RenderLevel/NativeDrawLevel.c`): the retail BSP
  render lists and visibility bits, face permutation table (R226), UV flip,
  triangle faces, winding, texture LOD thresholds, animated textures,
  semi-transparency (tpage blend 3 = opaque), draw-order bias, super turbo
  tint, reverse-track turbo UVs, far low-LOD quads and env-mapped water with
  its distance fade. Retail near-camera subdivision is not needed for
  clipping, only where it changes the picture:
  - Mosaic textures: a near face whose fourth texture entry points to records
    is split twice, and each cell takes its own record (UVs, CLUT, tpage). The
    render list slot sets the grid: 4x4 (quad and dynamic lists, centre or
    diagonal midpoint), 4x2 and 4x1, four columns along the 0-1 edge, with a
    second grid for UV-swapped faces. Without Max detail, a part whose corners
    don't pick retail's full subdivision handler keeps the near texture.
  - Full-dynamic LOD morph: when a far quadblock's corner is inside the morph
    distance, the middle vertices slide to their edge midpoints and fade to
    the low colour past the fade start. Retail then draws the four regular
    faces when its full handler is picked, otherwise one of eight low-texture
    partial topologies.

- Instance models (`game/RenderBucket/RenderBucket_QueueExecute.c`), normal,
  depth-split, water-split and reflection draw handlers: decoded animation vertices (including interpolated frames and
  triangle-strip continuation), precise MVP, near/far and DRAW_HUGE depth unit
  conversion, per-vertex and flat colours, UV/CLUT/tpage, winding, native
  near-plane clipping, normal alpha/depth fades, and lit keys/relics/tokens.
  Water uses the retail Y-plane clipping, generated UVs/colours and side
  selectors. Reflected passes mirror input Y about the split line, reverse
  winding, retain reflection colour masks and use the secondary depth bias.
  The dimmed water side keeps its three-pixel screen offset after mirror
  projection. Ghost materials preserve the subtractive flat mask followed
  by the additive flat/textured body, DPCT fade colours, texture STP behaviour
  and generated split UVs. Zero-alpha ghosts use the normal opaque path.
  Existing primitive writers provide temporary colour/material records; their
  screen coordinates never become native geometry and their packets are not
  linked into the OT. Each instance gets a marker in its own OT range to retain
  the correct viewport and order around retail UI/effects. Unconverted handlers
  and exhausted layer storage fall back to the classic path.

## Look options

- Colour depth: 24-bit, or 15-bit PS1 framebuffer precision (5 bits per
  channel after dithering).
- Textures: nearest or bilinear.
- Dithering (Options) applies to both renderers.

## Outstanding implementation

- Validate callback/emitter coverage in game. The source audit accounts for
  all seven setup/primitive selector rows, the four water-side selectors and
  both split-colour helpers. Direct projected emitters now include the
  warp-pad beam and transition flag. The apparent eighth setup/primitive
  row was adjacent table data copied into scratch, not another callback;
  native queueing explicitly rejects selector 7 before claiming draw success.
  Runtime coverage of loaded assets and unusual flag combinations remains
  pending.
- Decide whether to reproduce two non-Max-detail retail level quirks: a
  deepest frame under a partial parent handler reads a stale mosaic slot word,
  and partial full-dynamic faces subdivide further near the camera. Neither
  changes a Max detail frame.
- Remove the PGXP geometry-recovery dependency only after remaining 3D paths
  are converted and verified. Precise transform tracking currently uses
  `NativePgxp_*` helpers even when the PGXP option is disabled. Classic remains
  available, and Vita/Web retain Classic.

Model/effect layer exhaustion and unusable native model setup can retain Classic
packets. Unknown model handlers may be skipped; level layer exhaustion skips the
level, and triangle exhaustion drops further native triangles.
An unavailable isolated-depth framebuffer draws the affected overlay model
without depth. Ordinary 2D draws use native screen geometry with depth disabled.
Their existing packets remain the transport for coordinates, materials, draw
environment commands and OT order; they do not become world-space 3D layers.

## Outstanding in-game checks

These checks are pending; the latest continuations were built, not visually
verified. Earlier automated coverage is described below and does not establish
visual correctness of the new paths.

1. Compare Native and Classic across race tracks and the Adventure hub,
   including Tiger Temple, water, mosaic textures and both detail settings.
2. Check animated models, depth/water splits, reflections, ghost mask/body
   blending, tires, shadows and skid marks on flat ground and slopes.
3. Check sky/stars and each converted effect, especially transparent ordering,
   rain anchors, one-pixel lines and heat-distortion feedback.
4. Check menu transitions, title trophy/models, character selection, HUD 3D
   placement, multiplayer offscreen wumpa, the talking mask and cutscene actors
   and line effects. Confirm overlays preserve the track's depth and draw order.
5. Repeat relevant checks in split screen, mirror/reverse-track modes and each
   anti-aliasing mode, including MSAA and offscreen rendering.
6. Check performance and fallback behaviour in scenes with many particles and
   model overlays. The new isolated-depth passes have not been profiled in game.

## Recommended next step

Check the newly installed menu/HUD/cutscene changes in game first, then check
the particle, overlay tire, warp-pad beam and transition flag conversions from
`build/ctr_native`. Complete the
broader visual checks before removing PGXP recovery from the native path.

## Minimap precision

The **Precise Minimap** enhancement (Off by default, saved as
`precise_minimap=0/1`) lets minimap markers retain fractional
screen positions through world-to-map projection and widescreen scaling.
The Adventure player arrow also retains fractional rotated corners. Race dots,
ghosts and tracking icons keep their existing sprite size and shape. These
explicit HUD vertices carry no world depth or perspective divisor. This toggle
works independently of 3D PGXP and the depth buffer. Turning it Off restores
the original rounded positions and arrow corners.

`ctr_native_minimap` (with `CTR_NATIVE_RENDERER_TESTS=ON`) checks all map
orientations, the three-player offset, subpixel dot movement, arrow corners,
GPU vertex recovery and frame expiry without starting SDL or OpenGL.

## Verification tools and historical evidence

The commands below describe available checks, not tests run for the latest
continuation. This sandbox rejects execution of the 32-bit game and test
executables with SIGSYS. Recent continuations have successful build results;
menu/HUD/cutscene and isolated-depth changes have no new runtime test results.

`ctest --test-dir build --output-on-failure` runs `ctr_native_draw3d`
(layers, culling, frustum, transforms, marker) and, with
`CTR_NATIVE_RENDERER_TESTS=ON`, `ctr_native_depth_renderer`: native versus
retail depth in both submission orders, the implied depth buffer,
translucency, near-plane clipping, mirror mode and draw-order bias across
every anti-aliasing and PGXP mode, plus 15-bit quantisation. The sky
regression uses `DrawSky_Full` and the actual level marker helper, checking
that the native foreground survives sky faces at offsets `0`, `-4`, `-8`.

With `CTR_NATIVE_RENDERER_TESTS=ON`, `ctr_native_level_sky_order` checks the
same packet chains without starting SDL or OpenGL, across all PGXP modes and
both mirror settings. Run it directly with
`build/ctr_native_depth_renderer_test --sky-order-only`.

Internal builds: Page Down toggles the renderer and Scroll Lock freezes game
logic while rendering continues, so both renderers can be compared on the
same frame. Home toggles Max detail, End toggles the depth buffer and F6
shows the debug overlay (merged from fix-fireball-flash).

The following comparison predates the model/effect conversions and documents
the level-geometry milestone, not the current executable.

In the Adventure hub with logic frozen (1280x720, 921,600 pixels), Classic
against Classic differs by 0 pixels. Native against Classic, counting pixels
whose largest channel difference is over 8 / over 32:

| Detail | Before mosaics and morph | Now |
| --- | --- | --- |
| Max | 201,592 / 2,849 | 1,510 / 237 |
| Off | 182,379 / 2,221 | 382 / 12 |

Disabling only the morph (Max detail off) gives 2,682 / 1,294, mostly a far
rock face. What remains with Max detail is along the trophy statue's edges
(an instance model, still Classic). Below 8 levels, about 12% of pixels
differ from perspective-correct sampling and interpolation. Count with PIL;
ImageMagick 7.1's AE metric doesn't report a pixel count.

## Implementation history (2026-10-02)

- Phase 1 (native layer pipeline, look options, tests) is complete.
- Phase 2 (level geometry) works in game, including mosaic textures and the
  full-dynamic LOD morph. Broad race-track coverage remains pending; the
  Tiger Temple marker fix was later confirmed by the user.
- fix-fireball-flash (frame-rate-correct particles, F6 overlay) is merged
  into this branch. The overlay now lists Renderer and Colour depth, and
  reports the effective depth buffer setting implied by Native 3D.
- Phase 3 conversions include normal, depth-split, water-split and reflected
  instance models, ghost mask/body passes, tires, kart shadows and skid marks.
  All submit native geometry. Sky and the main world effect emitters are now
  converted as well. The subsequent menu/HUD/cutscene continuation also
  converts the common model paths; see current status and outstanding work above.

### Tiger Temple follow-up (2026-10-02)

The playable executable still contained the old `0x3fe` marker despite the
source fix. The executable has been rebuilt with `0x3fc`. Source-level packet
ordering checks pass with the fix and fail with the old slot. Draw3D and
precision checks also pass as 64-bit host builds. This sandbox rejects 32-bit
execution with SIGSYS, so the normal 32-bit CTest targets and fresh in-game
screenshots could not be verified here.

Earlier `/tmp/ctr-native-test/tracks/` screenshots have incorrect track names
(the frame labelled `tiger` shows a different track). Do not use that strip as
evidence of track coverage. The actual Tiger Temple captures are in `tt/`;
they show the sky obstruction before this executable update. The user confirmed Tiger
Temple is fixed in game after installing the executable.

### Instance model follow-up (2026-10-02)

The common model path and the depth-split handler are converted. Generated
split intersections submit their own positions, colours and UVs, rather than
reusing the original triangle's FIFO. The existing split interpolation rules
are retained. At this milestone, layer capacity was increased to 4096 for up to 2048
queued instance/viewport entries plus level and future effect layers (now 8192
after the sky/effects continuation). Precise
model MVP tracking is active for Native 3D even with PGXP disabled.

`ctr_native_model_submission` (with `CTR_NATIVE_RENDERER_TESTS=ON`) checks
animation-coordinate submission independently of bogus packet screen positions,
quarter-scale depth units, generated split geometry, G3/GT3/FT3 colours and
materials, winding/instance
flags/start-banner double flips, near-plane crossings, classic fallback, and
layer capacity. Run the CPU portion directly with
`build/ctr_native_depth_renderer_test --model-submission-only`.

The 32-bit executable and renderer test build successfully. The source-level
submission checks pass as a 64-bit host harness, as do Draw3D and precision
checks. In-game model appearance and the normal 32-bit test run still require
verification outside this sandbox (32-bit execution is rejected with SIGSYS).

The subsequent menu/HUD/cutscene continuation converts screen-space models
and the hint mask while retaining their retail overlay positions (see below).

### Water and reflection continuation

The water/reflection model bridge and post-mirror material screen offset build
successfully. This continuation has not been checked in a running race; the
sandbox still prevents executing the 32-bit game. The reflected and original
passes currently share one instance layer and the native depth buffer.

### Ghost continuation

Ghost compound packets now contribute camera-space mask/body triangle pairs,
including generated split triangles. The GPU keeps both passes in the late
transparent phase and depth tests them against opaque geometry without writing
depth. Equal-depth sort ties preserve pair submission order. Layer storage drops
an entire pair when only one triangle slot remains. This continuation is build
checked; rendered appearance still requires a running game outside the sandbox.

### Tires and ground decals continuation

Solid and reflected tires now submit their generated 3D billboard corners with
retail sprite selection, UV corner permutations, water-side visibility and
per-wheel ordering bias. Camera-relative positions use the same quarter-scale
depth normalization as models. Native submission reads no projected packet XY.
Retail GTE operations still generate the wheel axes and sprite selection.

Kart shadows retain the four-quad footprint and distance/height fading. Skid
marks use both frames' stored 3D edges and preserve segment connectivity, fading
and normal/alternate blend modes. Both decal types are depth tested in the late
transparent phase without writing depth, including non-STP texels. Their native
depth bias pulls them ahead of coplanar ground; their markers keep retail OT
placement. Classic drawing remains available when native layer storage fills.

These changes build successfully. No tests or in-game visual checks were run
for this continuation; the sandbox blocks executing the 32-bit game. Tire
reflections, decals on slopes, mirrored tracks and split-screen appearance need
in-game verification.

### Sky and effects continuation

Sky faces now submit the original 3D positions and colours, using the same four
selected sectors and signed face OT offsets as retail. A native background
pass preserves their order while disabling depth testing and depth writes.
The level marker remains at `0x3fc`, after Tiger Temple's `0x3fd` lower sky.
Stars use the same background pass with one-pixel native point ribbons.

The shared particle path now submits animated sprite quads and coloured line
streaks with retail scale/rotation, driver-local placement, fades, raw-texture
colours and blend modes. Transparent particles run after opaque visibility and
do not write depth. The common native line API clips endpoints to the near
plane, interpolates endpoint colours, and builds a one-pixel camera-space ribbon.
Weather and red-beaker rain use it too. Beaker rain derives its current cloud
anchor and depth units directly, before this frame's instance queue is built.

Confetti uses its generated 3D corners. Spider webs use their world endpoints.
Flare quads retain their spinning four-part gradient. Native storage now resets
with the OT in `MainFrame_ResetDB`, before game-logic flare submission, rather
than erasing those layers at the start of `MainFrame_RenderFrame`. Layer capacity
is 8192 to accommodate the additional per-particle and decal submissions.

Heat distortion submits camera-space rings from particle positions and radii.
Its framebuffer UV lookup, clamping and texture-page addressing retain retail
behaviour, with mirrored source lookup for mirrored native geometry. The effect
runs late without replacing world depth. Traffic lights and other HUD drawings
remain screen-space overlays.

The build succeeds. The existing sky-order check was adapted to count native
background triangles; no tests were added or run for this continuation. In-game
sky/particle appearance, one-pixel line rasterisation, feedback distortion,
mirroring and split-screen behaviour remain unverified: the sandbox blocks
executing the 32-bit game.

### Menu, HUD and cutscene continuation

The common model bridge now accepts screen-space instances, shared-range title
models, and the talking hint mask. It covers the existing normal, split,
reflection, special, colour and ghost primitive handlers used by menu models,
HUD models and cutscene actors. Title models reuse their owner's current OT
range; native submission checks that the marker slot belongs to this frame.
Screen-space translation retains the instance's camera-space coordinates, and
custom facing matrices preserve the precise translation cache.

Screen-space instances, instances with an owned push buffer (including the
multiplayer offscreen wumpa pass), and the talking mask are marked as native
model overlays. Each model draws at its existing OT position with private depth:
its opaque geometry establishes visibility before its translucent geometry.
The renderer shares the active target's colour attachment, allocates matching
private depth/stencil storage, and copies only the PS1 mask stencil in and out.
The track depth stays intact. This works with both screen and offscreen targets
and matches the main target's MSAA sample count. Unsupported framebuffer setup
falls back to drawing the model without depth. Ordinary menu/cutscene world
models continue to share scene depth; screen models remain unmirrored.

The cutscene interpolated-frame line effect submits its world endpoints through
the native line path, retaining retail colour fade, additive blending and OT
placement. Text, HUD sprites, traffic lights and menu rectangles remain ordinary
2D draws.

The executable builds successfully and is installed with a backup at
`ctr_native.before-native-menu-hud-cutscenes`. No tests were added or run for this
continuation. Menu transitions, HUD placement (including multiplayer wumpa),
cutscene animation, hint-mask overlay order and MSAA/offscreen behaviour need
in-game checking; this sandbox cannot execute the 32-bit game.

### Screen-space driver particle continuation

The shared particle bridge now accepts particles attached to screen-space
driver instances, including both textured quads and coloured line streaks.
It retains the particle renderer's camera-relative placement, scale, rotation,
mirror projection, colours, materials, visibility gates and driver OT range.
Their native materials select isolated overlay depth, preserving world depth
and the retail order around models and ordinary 2D UI. Geometry still comes
from particle positions and generated corners/endpoints, never packet XY.
Layer exhaustion continues to use the existing Classic fallback.

The initial callback audit flagged an apparent eighth setup/primitive table
row and the screen-space tire fallback. The follow-up below resolves both.

The executable builds successfully at `build/ctr_native`; the installed
`ctr_native` still contains the preceding menu/HUD/cutscene version. No tests
or in-game checks were run for this continuation. Screen-space particle
placement, line appearance, blend order, mirror/split-screen behaviour and
the cost of per-particle isolated depth passes remain pending.

### Callback and remaining emitter continuation

The setup and primitive tables each occupy seven words (`0x8008a428` through
`0x8008a440`, and `0x8008a444` through `0x8008a45c`). Retail's eight-word
scratch copies include the next table's first word. Thus the apparent eighth
row (`0x8006ad88` / `0x8006d55c`) is not a missing callback. All seven valid
rows have native model support. Native queueing rejects selector 7 and clears
draw success rather than interpreting the adjacent word as a callback.

Screen-space and owned-push-buffer tire passes now use native wheel quads with
isolated overlay depth. Their generated camera-relative corners, UV selection,
colours, mirror projection and retail marker slots are retained.

The emitter audit found two additional projected 3D paths. Warp-pad beam strips
now submit their jittered world-space endpoints and signed-halfword edge offsets
directly, retaining the paired coloured strips, additive blend and per-segment
OT slots. Their mirror setting is selected during game logic, before the render
phase. The waving transition flag retains both columns of generated vertices,
checker colours, brightness, animation and retail screen rejection. It submits
one native mesh layer with isolated depth at the flag's existing OT position;
loading text remains in its original packet order.

The audit also checked projection users and direct polygon writers outside the
level/model paths. Fruit pickup coordinates, battle arrows and weapon tracking
remain 2D HUD anchors; missile targeting is a gameplay visibility query. Sky
glow is a screen gradient. Menus/stat bars, pause backgrounds, offscreen texture
blits, text, sprites and fade rectangles remain ordinary 2D work. Matrix-only
particle emission and actor placement feed existing converted draw paths.
This is source coverage, not evidence that every loaded asset has been exercised.

`cmake --build build --target ctr_native -j4` succeeds. The new executable is
`build/ctr_native`; the installed executable is unchanged. No tests or in-game
checks were run for this continuation. Warp-pad appearance, flag/loading/menu
transitions, overlay tires, split screen and MSAA/offscreen behaviour remain
pending. PGXP recovery remains until visual verification is complete.

### Native 2D continuation

When Native is selected, the shared GPU frontend now emits homogeneous screen
geometry for ordinary triangle/quad polygons, sprite tiles (including font
glyphs and framebuffer blits), solid rectangles and line ribbons. Clip W is 1
and clip Z is 0, so these use the native shader input without scene depth,
perspective recovery or world mirroring. Draw-environment offsets, clipping,
offscreen targets, UV wrapping, texture windows, palettes, raw-texture colour,
STP blending, dithering and mask stencil retain the shared backend behaviour.
The existing packets and OT traversal continue to carry material/state changes
and preserve ordering around native 3D markers. Clear and image-transfer
commands continue through the existing native backend operations.

Polygon corners with explicit Precise Minimap positions retain their fractions,
independently of the PGXP setting. Address/value-validated metadata is consulted
to distinguish those coordinates from GTE-backed corners: a polygon containing
a GTE-backed corner retains the 3D recovery/fallback path. Ordinary screen
polygons use their supplied coordinates and never recover world geometry.
Classic, Vita and Web retain their existing vertex paths.

The executable builds at `build/ctr_native`. The installed executable remains
unchanged. In-game checks are pending for fonts, sprites, gradients, lines,
minimap precision, transparency/masks, menu/HUD layering, framebuffer feedback,
offscreen rendering and all anti-aliasing modes.

### Capacity and geometry recovery diagnostics

PC layer capacity is now 16,384 per frame (previously 8,192). Primitive arenas
are 6 MiB per DB (previously 4 MiB), with the existing 256-byte guard: usable
space is 6,291,200 bytes per buffer. Both arenas occupy 12 MiB of the 15 MiB
registered GPU-link address budget, leaving space for ordering tables and
other registered ranges. Two 8 MiB arenas would exceed that budget. Triangle
capacity remains 262,144; GPU vertex capacity remains 524,288 with packet
expansion headroom. Vita keeps its existing smaller arena and Classic path.
The larger PC arenas also expand the PGXP shadow map by 24 MiB, in addition
to 4 MiB of extra packet storage and about 2 MiB of extra layer storage.

Capacity usage sampling, periodic capacity reports and capacity-exhaustion
warnings have been removed. Buffer bounds checks and the allocated limits
remain in place.

While Native is active, geometry recovery and non-capacity fallback reasons
log their first occurrence immediately, with a source label and detail value.
Repeats produce interval/session counts every ten seconds when events occur.
Reasons cover model setup/handler/primitive rejection, invalid depth scale,
missing or stale OT ranges, missing level input, isolated-depth target failures,
and projected polygon packets entering geometry recovery. Counts are failed
operations/packet draws, not affected pixels, triangles or distinct objects.
Model detail values are model IDs or retail callback labels; queue selector
details are flags. The projected-packet counter records GTE-backed inputs,
including ones without usable depth; it does not prove recovered vertices
are valid.
