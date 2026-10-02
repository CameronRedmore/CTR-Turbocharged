# Ramp adhesion and collision recovery at high frame rates

Three surface calculations applied their full 30 FPS strength once per
simulation frame:

- `VehPhysGeneral_JumpAndFriction` and its Smoothed counterpart add an authored
  force along the kart's local up axis. `QuadBlock.mulNormVecY` is signed; the
  level structure documents `-127` as Sewer Speedway's anti-gravity multiplier.
  At a fixed approach speed of 4096, this produces a local impulse of -2032
  per frame. The unscaled force runs twice as often at 60 FPS and eight times
  as often at 240 FPS. On a banked surface it also has a sideways component.
- `VehPhysForce_CollideDrivers` adds velocity to recover from an obstructing
  surface when `DRIVER_COLL_FLAG_SURFACE_PUSHBACK` is set and the penetration
  check is negative. This recovery acceleration was also unscaled.
- Mud's minimum friction removes 1/8 of lateral speed and 1/2 of excess forward
  speed (or 1/2 of forward speed when braking against the direction of travel).
  These are proportional drags applied after the ordinary time-scaled friction.
  Their full per-frame fractions made mud grip much stronger at high FPS.

Original physics now distributes the signed, rotated adhesion impulse and
surface recovery impulse using `CTR_FRAME_STEP`. At 30 FPS the original
arithmetic is preserved. Smoothed adhesion and recovery use
`NativePhysics_ElapsedMS(elapsedTimeMS) / 32.0`; recovery retains fractional
position and velocity. Jump impulses and single collision impacts remain
discrete events.

Original mud retains its signed shifts and evaluates its combined constant and
proportional drag on `CTR_RETAIL_FRAME_TICK`. Sampling them together preserves
the maximum of the two drags rather than adding extra constant friction between
mud ticks. Smoothed mud uses fractional powers of the authored 7/8 and 1/2
retention factors, scaled by elapsed time. Other terrain's constant friction
still runs each simulation step. Mask protection still bypasses mud damping.

## Track data audit

The installed `assets/BIGFILE.BIG` was read without modification. All 18 race
tracks were checked at each of their four stored geometry variants (1P, 2P,
3P/4P and time trial), following `LOAD_GetBigfileIndex` and the binary Level,
mesh and QuadBlock layouts. Quad array extents and vertex indices were checked.
The relevant surface counts agree across all four variants:

| Track | Relevant surfaces | Finding |
| --- | --- | --- |
| Sewer Speedway | 32 quads with adhesion -127 | Original adhesion fix applies. |
| Tiger Temple | 3 quads with adhesion -64 | Uses the same adhesion path; covered by the same fix. |
| Tiny Arena | 52 mud quads | Speed-dependent mud drag had the separate frame-rate bug; now corrected. |
| Blizzard Bluff | 36 ice quads | Regular ice friction is already time-scaled. The stronger friction on leaving ice is a transition response. |
| Polar Pass | 34 ice quads | Same ice rules as Blizzard Bluff. |
| Oxide Station | 90 low-gravity quads | Gravity's 41% multiplier already feeds a time-scaled force; no matching unscaled-force bug. |

Among these 18 tracks, only Sewer Speedway and Tiger Temple have nonzero
adhesion multipliers, and only Tiny Arena has mud. The earlier collision
recovery fix applies to all tracks when its penetration condition is met.
Turbo-pad reserves are latched on entry rather than added every frame, and
the barrel/snowball path advances on retail ticks. This audit does not establish
that every hazard, timer or geometric contact is frame-rate independent.

## Validation

- `ctr_native_physics` exercises the actual Smoothed solver on a banked surface
  with Sewer's multiplier, in both driving directions, at 30, 60, 90, 120, 144
  and 240 FPS. The total force over one second is identical at each rate.
  It also checks persistent penetration recovery, fractional velocity, leaving
  the ramp, absent contacts and recovery on the allowed side of a surface.
  Tiger Temple's -64 multiplier and Oxide Station's low gravity are also checked
  at all supported rates.
- Mud regressions cover positive and negative motion, forward excess speed,
  braking against the direction of travel, and mask protection at every rate.
  With ordinary friction disabled to isolate the proportional drag, Smoothed
  lateral speed starting at 8192 previously fell to 149.152842466 after one
  second at 30 FPS but approximately 0.000000000099 at 240 FPS. After the fix,
  it falls to 149.152842466 at every rate. The Original game-object probe
  previously produced 153 at 30 FPS and 7 at 240 FPS; it now produces 153 at
  every rate. The remaining difference between modes is integer rounding.
  A second game-object probe with nonzero constant friction checked the combined
  drag over five retail steps: every rate produced (4202, 4224) lateral/forward
  speed in Original and (4201.75, 4224) in Smoothed.
- A local probe linked against the built game object called the actual Original
  `VehPhysGeneral_JumpAndFriction` and `VehPhysForce_CollideDrivers` functions.
  At every supported rate, the measured local adhesion total was -60960 and
  the recovery total was (-1920, 1920, -3840) for a fixed penetration.
- The native game builds and all four non-renderer CTest checks pass.

These checks establish the frame-rate scaling bug and its correction. The
specific reported Sewer Speedway ramp has not been reproduced in a live race;
additional track geometry or collision issues may still affect that location.

## Smoothed collision hit position

Surface recovery compares the driver's position after movement with the
exported hit position. Retail exports the triangle point nearest the requested
end of the sweep. Smoothed collisions exported the contact point instead. A
kart resting on an uphill road touches it at the start of each step, so the
whole step read as penetration along the direction of travel. Recovery then
added about a quarter of the kart's speed per 30 FPS frame until the speed cap,
and launched it over crests. Larger steps made it worse at lower frame rates.

Smoothed collisions now export the triangle point nearest the sweep end, and
the push-out position on the face plane, as retail does. A traced drive over the
reported hills at 30 FPS previously reached a speed of 31,844 (Original: 16,627).
After the fix, it peaked at 17,441 at 30 FPS and 17,065 at 60 FPS, with no
per-frame speed jumps. `ctr_native_physics` reproduces the traced slope contact
at every supported rate and checks that recovery leaves only integer rounding.
