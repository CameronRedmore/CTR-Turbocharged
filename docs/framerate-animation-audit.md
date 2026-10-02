# Frame-rate animation audit

Animations that advanced by a fixed amount per rendered frame, and so ran
faster above 30 FPS. All items below have been fixed.

Tools used for the fixes:

- `CTR_FRAME_STEP(x, timer)`: spreads a per-30Hz step evenly across faster frames.
- `CTR_RETAIL_FRAME_TICK(timer)`: true on one in every N frames, giving a 30 Hz tick.
- `FPS_DOUBLE(n)` / `FPS_HALF(n)`: scale a frame count or a frame-counter phase.
- `INSTANCE_AnimFramesScaled(inst, anim)` / `INSTANCE_ScaleAnimFrames(inst, anim, n)`:
  whether an animation's `animFrame` counts frame-rate-scaled frames, and how to
  convert a 30 FPS frame count into those units.

## Animation frame units

`animFrame` uses one of two units:

- **Scaled:** normal anims on models that pass `INSTANCE_Use60FpsAnimation`.
  Frame counts are `FPS_DOUBLE(n - 1) + 1` and the renderer interpolates, so game
  code steps `animFrame` once per rendered frame.
- **30 FPS frames:** half-rate anims (`numFrames & 0x8000`) and models excluded
  from 60 FPS animation. Game code must step these on `CTR_RETAIL_FRAME_TICK`.
  Half-rate anims seen so far:
  - player driver anims 1 (stationary) and 2 (crash fall)
  - `ukamouth`
  - `scan` (hub save station)

Cutscenes (overlay 233) skip the scaled path entirely and advance by elapsed time.

## Fixed

### 3D / model animation

- `SelectProfile.c` `SelectProfile_ThTick`: load/save icon spin uses `CTR_FRAME_STEP`.
- `VehFrame.c` `VehFrameInst_GetNumAnimFrames`:
  - Was hard-coded to `(n << 1) - 1`, which is only correct at 60 FPS.
  - Now matches `INSTANCE_GetNumAnimFrames`.
  - Fixes the Tiny Arena teeth door and the driver steering lean above 60 FPS.
- `VehFrame.c`: half-rate driver anims step at 30 Hz. Their baked matrix index is
  `animFrame`, not `FPS_HALF(animFrame)`, which matches the mask-grab code in
  `VehStuckProc.c`.
- `VehPhysProc.c` `VehPhysProc_SlamWall_Animate`: the wall-crash fall anim is half-rate,
  so `animFrame` now steps on `CTR_RETAIL_FRAME_TICK` (it was stepping every
  rendered frame, so the crash ended far too early above 30 FPS).
- `VehPhysForce.c` `VehPhysForce_TranslateMatrix`:
  - Wheelie start/recover and landing-squish recover stepped `matrixIndex` once per
    rendered frame; they now step on `CTR_RETAIL_FRAME_TICK`.
  - Jump squash/stretch decay and height smoothing run on the retail tick, and the
    kart scale interpolation speed uses `CTR_FRAME_STEP`.
- `VehTalkMask.c`: mouth frames are converted from 30 FPS units into `animFrame` units.
- `RB_Orca.c`:
  - The cooldown counts down at 30 Hz.
  - Path offsets, splash frames and the initial `animIndex` are scaled to animation units.
- `RenderBucket_QueueExecute.c` `RenderBucket_AdvanceInstanceAnimWord`: renderer
  auto-advance (instance flags 0x10/0x20) steps unscaled anims at 30 Hz.
- `FLARE.c`: lifetime, size breakpoints and spin are scaled.
- `RB_Follower.c`: lifetime and scale-up step at 30 Hz. Position still updates every frame.

### Menu / UI animation

- `SelectProfile.c`: profile/"saved" blink uses `FPS_HALF(frameCounter)`, and the
  "save complete" timer is `FPS_DOUBLE(0x3c)`.
- `MM_Characters.c`:
  - The border colour pulse phase uses `FPS_HALF`.
  - Stat bars fill with `CTR_FRAME_STEP`.
- `MM_TransitionInOut`:
  - Now takes frame counts in rendered frames and scales `headStart` and the swish
    frame itself.
  - Callers scale their counters and limits: character select, track select, cup
    select, battle, high score (both versions). The title menu already did.
- `MM_TrackSelect.c`: the track list scroll is scaled.
- `MM_HighScore.c`, `MM_HighScore_Online.c`: row/track slides step at 30 Hz,
  because their off-screen checks rely on whole-step offsets.
- `MM_Battle.c`: error colour flash uses `FPS_HALF(frameCounter)`.
- `MainFreeze.c`: pause blink and the analog config wobble use `FPS_HALF`.

- `UI_RenderFrame.c`: the turbo-count slide in/out steps on the retail tick.
- `222.c`: C-T-R letter fly-in/out and time-display fly-out frame counts are scaled,
  and the "press to continue" delay is scaled.
- `MainFrame.c`, `UI_VsQuip.c`: the VS end-of-race timer and its thresholds are scaled.

### Minor

- `Display.c`: blur wave phase.
- `RB_FlameJet.c`: particle wobble phase.
- `MainInit.c`: initial water animation call.

## Step functions

`FPS_HALF(step)` truncates, so a per-frame step such as `FPS_HALF(0x40)` was 13
instead of 13.33 at 144 FPS (and could round to 0 for small steps). Per-frame
accumulating steps now use `CTR_FRAME_STEP(step, timer)`, which spreads the 30 Hz
step exactly across rendered frames: object/hub/UI spins and scale steps, the
blasted-camera lerp, and the `InterpBySpeed` speeds in `VehStuckProc.c`,
`VehPhysProc.c` and `VehPhysGeneral.c`. Where a frame count multiplies a step
(`AH_Pause.c`, `AH_MaskHint.c`), `FPS_HALF` wraps the whole product.

`FPS_HALF` remains for converting a frame counter into a 30 FPS phase
(`FPS_HALF(gGT->timer) & 1`, texture/water animation, table indices) and for
rate or threshold values that are not accumulated per frame.

## Particles

- Rescale: `Particle_RescaleNewParticles` (called from `MainFrame.c`) scales each new particle's
  velocities, accelerations and lifespan once, after thread ticks, for both the
  ordinary and heat-warp lists. `PARTICLE_SET_COLOR_FLAG_NATIVE_FRAME_RATE_SCALED`
  marks done particles and `Particle_Init` clears it, since pool slots are reused
  without clearing. Special-line particles skip the last axis, whose
  velocity/accel hold the packed line colour.
- Acceleration: the rescaled accel is still a per-30-FPS-frame velocity change,
  so `Particle_UpdateList` spreads it with `Particle_FrameStep` (an `accel / k`
  per frame made exhaust fall k times too fast). Each particle's own 30 FPS
  frames are counted from `framesLeftInLife`, not the global timer, because
  bursts spawn mid-frame; `Particle_IsRetailFrameEnd` marks the update that
  completes one. The rescale also offsets the start
  velocity by `-accel * (k - 1) / (2k²)` so positions follow retail's discrete
  path at each 30 FPS frame, where k = rate / 30.
- Particle callbacks work in retail units through helpers in `Particle.c`
  (`Particle_GetRetailVelocity`, `Particle_SetRetailVelocity`,
  `Particle_FrameStep`, `Particle_FrameCount`): potion shatter's Y-speed
  threshold (checked only at the particle's 30 FPS frame ends, as retail does),
  random velocities and colour fade; spit tire bounce velocities and shrink;
  underwater exhaust's pop threshold, with the pop shown, frozen in place, for
  one 30 FPS frame and the callback cleared so it pops once.
- Emission rate: `Particle_Init` only spawns on the first rendered frame of each
  30 FPS frame (`CTR_RETAIL_FRAME_START`), so emitters called every rendered frame
  emit at the retail rate.
- `PARTICLE_SPAWN_UNGATED_BEGIN/END` bypass that gate for one-shot bursts (potion
  shatter, orca splash, mask hint leave/vanish, mask grab, landing sparks, mud
  landing splash, wake entry burst) and for callers that pace themselves to 30 FPS
  frames (warp-pad dust, which runs on `CTR_RETAIL_FRAME_TICK` frames). Older call
  sites set `sdata->UnusedPadding1` directly for the same effect.
- Emitter timer phases use `CTR_RETAIL_FRAME_INDEX(gGT->timer)` rather than the
  rendered-frame timer: exhaust driver interleave, terrain odd/even emitters,
  flame-jet rotation flip and warp-pad dust.
