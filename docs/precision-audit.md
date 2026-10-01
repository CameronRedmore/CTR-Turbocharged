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
| Physics (`game/Vehicle/VehPhysForce.c`, `VehPhysGeneral.c`, `VehPhysProc.c`, `VehPhysCrash.c`) | Integer velocity/acceleration, fixed point integration, integer square roots, rotation through s16 GTE inputs, and conversion of positions to whole units for collision searches. | Audited; physics calculations remain as they were. |

Transform shadows are keyed by matrix address, checked against their integer
contents, and expire at frame boundaries. Direct control-register writes clear
loaded transform shadows. Unknown or changed transforms use their integer
values. The audit also found an out-of-bounds PGXP input lookup for MVMVA's IR
vector (vector index 3); it now falls back safely. Frame changes also clear old
PGXP register/input results.

## Scope and remaining integer calculations

This is a native visual precision improvement, not a complete conversion of the
engine to floating point. Camera mode state, zoom/height smoothing, collision
constraints, authored fly-in/path samples, axis-angle camera smoothing, model
animation/scale decoding, and special split/reflection model matrix construction
still contain integer calculations. Their integer outputs can still limit
smoothness before the new projection path receives them. Physics and binary
structure layouts retain their original behavior. PGXP Off uses the existing
integer rendering path; Vita currently has PGXP disabled and keeps its existing
renderer.

## Verification

Build with the existing CMake configuration and run `ctest --test-dir build
--output-on-failure`. `ctr_native_precision` checks compound camera rotation
order, fractional-angle orthonormality, fractional translation and projection
depth, fractional rotation in rotation/light banks, invalidation after direct
register writes, changed matrix contents, frame expiry, PGXP Off, and the IR
vector lookup. Assertions are enabled even in Release builds.

A gameplay visual check still requires the user's disc asset (`assets/ctr-u.bin`),
which was absent from this checkout during the audit.
