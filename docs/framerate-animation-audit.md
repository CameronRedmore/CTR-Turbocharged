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

### Minor

- `Display.c`: blur wave phase.
- `RB_FlameJet.c`: particle wobble phase.
- `MainInit.c`: initial water animation call.

## Not changed

- `RB_FlameJet.c` and other emitters may spawn particles once per rendered frame.
  This affects particle density rather than animation speed, and was not audited.
