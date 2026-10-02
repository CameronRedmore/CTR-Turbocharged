# Camera, rendering, and physics precision audit

The engine uses PS1 compatible integer and fixed point structures. Fixed point
already stores fractional values: driver world positions have eight fractional
bits, and rotation matrices use Q12 coefficients. Replacing these structures
with floats would break their asserted offsets and change physics, collision,
and replay calculations. The native renderer instead carries floating point
values alongside the original structures when PGXP Geometry or Perspective is
selected.

| Area | Integer precision loss found | Changes |
| --- | --- | --- |
| Normal follow camera (`game/CAM.c`) | Driver positions lose their eight fractional bits; rotated eye and target offsets round through GTE MAC/IR; pitch/yaw use integer angle lookup; the output adds integer position corrections. | Preserve driver fractions and transform residuals for the visual eye, calculate visual pitch/yaw with `atan2` and `hypot`, and bypass the integer correction in the visual position. Resolved terrain and mask height constraints retain their existing height. |
| Camera transitions (`CAM_ProcessTransition`) | Position and shortest-path rotation interpolation round to integer units and angle steps. | Carry fractional position and angle interpolation into the view matrix, including when inputs and outputs alias. |
| View matrix (`game/PushBuffer.c`) | Trigonometric lookup, matrix multiplication, inverse translation, vertical aspect correction, and widescreen scaling round before projection. | Construct a double precision rotation, transpose, translation and aspect scaling alongside the integer camera matrices. |
| Projection and clipping (`platform/native_gte_core.c`) | PGXP already preserved projection fractions but used rounded matrix coefficients and translations. | Use validated precise transforms for RTPS/RTPT and MVMVA, including clipping transformations. Preserve the existing saturation and overflow rules. |
| Model rendering (`RenderBucket_QueueExecute.c`) | Camera-relative translation and view/model matrix composition round before model vertices are projected. | Preserve precise translation and compose the ordinary model MVP in double precision. Retain the integer model scale and depth decisions. |
| Track, stars and weather rendering | Several paths write GTE registers directly and bypass standard matrix-loading macros. | Bind precise transforms after the corresponding direct camera matrix loads. |
| Physics (`game/Vehicle/VehPhysForce.c`, `VehPhysGeneral.c`, `VehPhysProc.c`, `VehPhysCrash.c`) | Integer velocity/acceleration, fixed point integration, integer square roots, rotation through s16 GTE inputs, and conversion of positions to whole units for collision searches. | Original mode retains these calculations. The optional Smoothed mode uses double precision for player acceleration, jump impulses, gravity, friction, speed/vector conversion and integration, with fixed point exports at collision and gameplay boundaries. |

Transform shadows are keyed by matrix address, checked against their integer
contents, and expire at frame boundaries. Direct control-register writes clear
loaded transform shadows. Unknown or changed transforms use their integer
values. The audit also found an out-of-bounds PGXP input lookup for MVMVA's IR
vector (vector index 3); it now falls back safely. Frame changes also clear old
PGXP register/input results.

Model camera-relative translations preserve the GTE's signed 16-bit input
wrapping and IR saturation before near-instance and DRAW_HUGE scaling. Portal
reward animation can retain high bits from unsigned trigonometric shifts in its
32-bit positions; ignoring that wrapping sent precise reward geometry outside
the portal. Fractional camera residuals are applied after the integer input wraps.

## Scope and remaining integer calculations

This is a native visual precision improvement, not a complete conversion of the
engine to floating point. Camera mode state, zoom/height smoothing, collision
constraints, authored fly-in/path samples, axis-angle camera smoothing, model
animation/scale decoding, and special split/reflection model matrix construction
still contain integer calculations. Their integer outputs can still limit
smoothness before the new projection path receives them. Original physics and binary
structure layouts retain their original behavior. PGXP Off uses the existing
integer rendering path; Vita currently has PGXP disabled and keeps its existing
renderer.

## Verification

Build with the existing CMake configuration and run `ctest --test-dir build
--output-on-failure`. `ctr_native_precision` checks compound camera rotation
order, fractional-angle orthonormality, fractional translation and projection
depth, fractional rotation in rotation/light banks, invalidation after direct
register writes, changed matrix contents, frame expiry, PGXP Off, and the IR
vector lookup. It also checks portal reward positions with unsigned animation
offsets through projection, camera-relative wrapping across signed boundaries,
and model translation saturation. Assertions are enabled even in Release builds.

The native tests run without a disc image. Gameplay checks require extracted
disc assets.

For development builds (`CTR_INTERNAL`), launch with
`CTR_PGXP_RETAIL_TRANSFORMS=1` to bypass the precise rotation and translation
shadows while retaining PGXP subpixel projection and the selected texture
mapping mode. This diagnostic is read on the first transform and logs when active.
Compare the same scene with and without the variable to isolate transform
shadows from the rest of PGXP. Options > Enhancements > Integer NCLIP controls
the winding calculation independently; End toggles it in development builds.

## Optional smoothing modes

On PC and Web, Options > Enhancements groups PGXP, detail level, and four
independent Original / Smoothed settings. Original is the default for player physics, AI and steering; collisions default to Smoothed.
The native configuration saves `smoothed_physics`, `smoothed_ai`,
`smoothed_collisions`, and `smoothed_steering` separately. Internal desktop builds
can toggle player physics on Delete release, like the existing Home and Insert
shortcuts. PGXP remains an independent rendering setting; Vita keeps Original.

Continuous state uses double precision shadows outside the PS1 structures:

- Player physics evaluates acceleration, jump impulses, gravity, local-space
  friction, trigonometric speed conversion and movement integration without
  integer truncation.
- AI retains fractional navigation progress, path interpolation, speed,
  acceleration, airborne motion and rotation. Path selection, timers, flags and
  authored path samples retain their original representations.
- Collisions use floating-point swept sphere / triangle face, edge and vertex
  tests, ray intersection, contact fractions, normals, wall response and kart
  separation / weighted bounce. The integer BSP is a conservative broad phase;
  authored track geometry and exported contact metadata keep their binary layout.
- Steering evaluates normal and drift controls, spin rate, turn and wobble
  interpolation, yaw and terrain rotation in floating point. Input buttons and
  kart state transitions remain discrete.

External writes replace affected shadow fields. Teleports, driver creation,
arena resets and mode changes discard continuous state. Mixed configurations
are supported: each option selects its own solver; shared movement and velocity
exports preserve fractional results from the selected domains. Original modes
retain the legacy arithmetic at their respective entry points.

Native checkpoint bundles capture all four settings and continuous state. The
native state bundle is version 3; quick saves and development replay checkpoints
from older builds have a different format and are rejected. Ordinary game saves
and ghost layouts are unchanged.

`ctr_native_physics` checks fractional and negative movement at 30/60 FPS,
external writes, speed/direction round trips, gravity, friction, acceleration,
jumps, all 16 mode combinations, checkpoint restoration, original AI wrapping,
fractional AI arithmetic, normal/drift steering, fractional terrain rotation,
face, edge, vertex, tangent, fast and degenerate collision cases, wall crash
stops, and fractional kart bounce. Higher framerate checks cover displacement,
gravity, AI acceleration/damping and steering scale at 30, 60, 90, 120, 144 and
240 FPS, including forced 30 FPS and ghost overrides. Smoothed
changes handling; sustained race testing should cover ramps, drifts, walls,
weapons and split screen.

The Enhancements submenu was checked in a running native build on an isolated
virtual display, including its six rows, independent saved toggles and return
to Options. The main checkout's higher framerate changes are integrated with
the continuous solvers. High-rate movement uses fractional elapsed time rather
than alternating integer millisecond steps; fixed-rate AI impulses and damping
and steering interpolation scale with the selected simulation rate. Discrete
state timers keep the shared integer clock.
