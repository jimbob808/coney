---
kind: topic
system: camera
owns:
  functions:
    - "0x001210f8"
    - "0x00121298"
    - "0x001263e8"
    - "0x0041ab30"
    - "0x0041ab60"
---
# Camera shake, rumble and slow motion

Verified against: `SLUS_212.15` (NTSC-U); this page preserves the earlier camera research's findings.

## Summary

The camera's shake slots and pad rumble, world-driven shake and the character-step scale used by slow motion.
The camera manager and view selection remain on [Camera](index.md#fn-manager).

## Map

| Address | Function | Subject | Evidence |
| --- | --- | --- | --- |
| `0x001210f8` | `Cam_StartShake` | shake and rumble | confirmed (code) |
| `0x00121298` | `Cam_UpdateShake` | shake and rumble | confirmed (code) |
| `0x001263e8` | `Cam_Follow_StartShake` | shake and rumble | confirmed (code) |
| `0x0041ab30` | `SlowMotion_On` | slow motion | confirmed (code) |
| `0x0041ab60` | `SlowMotion_Off` | slow motion | confirmed (code) |

## Data

Shake slot fields, strength levels, timers and the character-step globals are recorded with their operations below.
The common camera layout stays on [the base camera object](index.md#the-base-camera-object).

## Behaviour

### Shake and rumble {#shake}

A shake is started on a camera through its vtable slot `+0x15c` (`0x001263e8` for the follow camera, `0x001210f8` in
the base) as `(human, level, view)`, confirmed (code). The last argument goes to `+0x1d7`: 1 lets the shake move the
view, 0 makes it a rumble only. The callers:

- a hit reaction (`Human_PlayReaction`, `0x0026a6d0`), with the attack's strength bits `(flags & 0x30) >> 4` as the
  level: on the attacker's player camera with view 1 (`0x0026a8fc` / `0x0026a924`), the level raised by one (at most 3)
  while the attacker has bit `0x80000` in his word `+0xe0` (raging, inferred); and, from strength 2, on the victim's
  camera with view 0 (`0x0026a988`): **the one hit only feels it in the pad**. Confirmed (code);
- rage start (`Human_StartRageMode`, `0x00236e90`): level 1, view 1 (confirmed (code));
- a landing (`Human_Land`, `0x0023e090`, under the follow camera above): view 1; Confirmed (code).
- the animation event of **type `0x20`** (`Anim_FireFrameEvents` `0x00101dd8`, at `0x0010354c`): level = the
  event's value, view 1, on a player's camera. The human is the clip's own unless he has a state of `0x82830`, when it
  is his foe or his grab partner (or the human of handle `+0xc4`); a target whose `+0xcc` kind is one of `0x77`,
  `0x78`, `0x6a`, `0x80`, `0xa0`, `0xa1`, `0xb2`, `0xb3`, `0x4d`-`0x4f`, `0xed` gives the shake back to the clip's
  human (the kinds' meaning not traced). Confirmed (code).

Levels: **1**: amplitude 0.5, 0.10 s, rumble base 0; **2**: 0.75, 0.15 s, `0x30`;
**3**: 1.0, 0.18 s, `0x60`; 0 stops it. The update (`0x00121298`, every camera kind): the amplitude eases 65% per
update toward the level's while its time lasts; the time counts down by `dt × min(1, 54 × step)` (so slower in [slow
motion](#slow-motion)); the pad's rumble byte (`+0x41` of the pad record) gets `255 × current / level amplitude` when
that exceeds `(base >> 2) + 0x28`, capped at `base + 0x60`. Confirmed (code).

**Slots and the view offset** (`0x00121298`, confirmed (code)). A camera keeps three shakes, slot `player + 1` (0 for
a shake with no human): level amplitude `+0x1a8`, time `+0x1b4`, current amplitude `+0x1c0`, rumble base `+0x1d0`.
While a slot's time is positive its current eases 65% toward the level; once the time is negative it decays × 0.6 per
update and is zeroed under 0.001; it is never above the level. When a slot's current reaches 0 the slot, the offset
and the pad's rumble bytes (`+0x40`, `+0x41`) are cleared. Each update, only in game mode 1, with `A` = min(the sum of
the currents, the largest level), the offset `o` (`+0x190`) is, per axis,

`o = A × (0.5 × o_last + 0.5 × (u − 0.5))`, with `u` a fresh uniform draw in [0, 1) (`Random_Unit`, `0x003354e0`, three
draws for x, y, z; confirmed (code)),

so each axis stays within ±`A` / 2 m and changes smoothly (half the last offset carried over). It is added only when
`A` > 0, `+0x1d7` is set and switch 6 (`0x0050b1d0`) is on. Each caller passes the point it moves: the follow camera
passes its **aim point** `+0x240` (`0x0012cf94`, just before `Mat_LookAt`), so its shake turns the view and does not
move the camera; with no point given the base adds it to the look-at point `+0x180`. Confirmed (code).

#### Shakes from the world: the passing train {#world-shake}

Set dressing shakes the cameras through **slot 0**, the slot of a shake with no human, with any amplitude and time, not
through the three levels. Two one-instruction methods of the base camera (every camera class keeps them, `0x00535510`
and `0x00535d50` alike): vtable `+0x1d4` (`swc1 f12, 0x1a8(a0)`) sets slot 0's level amplitude `+0x1a8`,
and `+0x1dc` (`swc1 f12, 0x1b4(a0)`) its time `+0x1b4`; the script-object helpers
`ScriptObj_SetCameraWorldShakeLevel` (`0x003a4608`) and `ScriptObj_SetCameraWorldShakeTime` (`0x003a4668`) call them on
a player's current camera. Neither touches the rumble base `+0x1d0` or the view switch `+0x1d7`. Confirmed (code).

**The train** (`part_train_sound`, `PartTrainSound_Update` `0x003db690`,
[Script types](../script-types.md#subway-train)):
the sound object that travels with a subway or elevated train. While it is within 60 m of a camera, each of its
updates (every 5 updates there; every 60 when farther than 120 m) does, for every view (confirmed (code)):

- level amplitude = **(60 − d) × 0.004237288** (= 0.25 / 59), `d` the distance in metres from that view's camera
  (vtable `+0x21c`) to the train sound, so 0 at 60 m and 0.25 at 1 m; **0.25** flat when within 1 m of any camera;
- time = **0.95 s**.

So nothing triggers it but distance: no script call, no sound event, no animation event. The shake then runs through
the usual update ([above](#shake)): the current amplitude eases 65 % a camera update toward the level, the time
counts down, and because the train re-arms the time every 5 updates (about 0.17 s) it never runs out while the train
is in range; the level follows the distance, rising as the train comes and falling as it goes. Once out of range the
last level holds for the rest of its 0.95 s, then the current decays × 0.6 an update to 0. The view offset is the
shake's own (each axis `A × (0.5 × o_last + 0.5 × (u − 0.5))`, so within ±`A` / 2 m, smoothed by half the last
offset), added to the follow camera's aim point, so **the view turns slightly; the camera does not move**. It needs
`+0x1d7` set (the camera constructor `0x00120868` sets it to 1; a shake started with view 0 clears it until the next
view-1 shake) and switch 6 on. Confirmed (code).

**No pad rumble**: `Cam_UpdateShake` (`0x00121298`) finds the pad of slot *k* from the player list only for *k* ≥ 1
(slot 0 keeps pad −1), so slot 0 never writes the rumble bytes. Confirmed (code); historical observation
([checking notes](#upstream-earlier-observations)) below.

Earlier observation: see the [recorded values and method](#upstream-earlier-observations).

Earlier observation: see the [recorded values and method](#upstream-earlier-observations).

**The other world shakes** on slot 0 (confirmed (code), the arguments at the cited calls): `PartPelhamSubway_Update`
(`0x003da744`) and `PartLevel2Subway_Update` (`0x003d9bcc`) set a level of **0.02** for **4 s** on each view's
camera while within 30 m and 50 m of one; also `PartExplosionScreenShake_Init`, `PartMotorbikeGang`'s message and
update, `PartPlasterDrop_OnMessage`, `SubPlasterDrop_Init` and `SubShk_Update` (arguments not traced).

The world-shake port boundary and implementation steps are on
[Camera implementation](../../engine/camera.md#shake).

### Slow motion {#slow-motion}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

`0x005148a0` (`CfgFollowCamera`'s last argument, 0.2 in `global.lua`) scales **the characters' step**: an animation
event of type `0x2e` on a player's clip (`0x00101dd8`, one player only) calls `0x0041ab30`, which sets the step
`0x005102cc` to `0x005148a0 / 30` (1/150 s) and marks the player; type `0x2f` (`0x0041ab60`) unmarks it and, when no
player is marked, sets the step back to 1/30 s. Every character update then advances `dt` = 1/150 s, so humans,
animation and the cameras' character-step logic run at 20% speed while frames keep their rate. Confirmed (code).

**Which clips** (a survey of the disc's animation clips, counts only): 52 clip files (44 distinct names) carry the
pair, always one `0x2e` and one `0x2f`, 1 to 8 clip frames apart (4.4 on average), so a slow-motion moment is short.
They are all **finishing moves**: the grab power moves (`gen_grab_front_power*`, 18 files, plus 2 reactions) and the
mount power moves (`gen_mount_power*`, 16), the tandem attacks' ends (`gen_tandem*` and `samo_bo_tandem*`, 13 files,
one begin), the bottle smash (`bottle_smash_attacker`) and one boss counter (`bopp_bo_grab_counter_victim`, 2).
No rage, running or ordinary attack clip has one. Since the event fires only on a player's clip, a non-player's
finisher does not slow time. Inferred: counted from the clips' frame events, not read from the code.

The other writers of the step, confirmed (code): `HuSetSlowMo(fraction)` (`Human_SetSlowMo`, `0x0023b778`), step =
fraction / 30 for 0 < fraction < 1, else 1/30; and Luther's boss tactic: `BossLutherTactic_StartSlowMotion`
(`0x0030c8c8`) sets 1/60 s for 1 s of game time, then `BossLutherTactic_UpdateSlowMotion` (`0x0030c8f8`) 1/120 s for
4 s, then 1/30 s again.

**Game time does not slow down.** `GameTimer_Update` (`0x00145a10`) advances game time (`+0x40`, milliseconds `+0x48`)
by its own fixed step (`+0x4c`, 1/30 s) × its scale (`+0x50`, 1.0 from the reset; no other writer among the timer's
functions) every update and never reads `0x005102cc`. So everything timed in game milliseconds (the combat timers such
as stun and ground; the missions' and Luther's own timers above) keeps counting frames at 30 a second while the
characters move at 20%: a stun of 1 s ends after 30 frames, in which a slowed character has lived only 0.2 s (inferred
from the code; not seen at runtime). The cameras are not slowed either: `Humans_Update` runs `Cameras_Update`
(`0x0011e878`) with max(step, 1/30) (`0x00249b50`), so 1/30 s.

## In Coney

Coney's shake implementation and world-shake gaps are described in [Camera
implementation](../../engine/camera.md#shake).

## Open questions

The character-step effect on game-time timers below is inferred from static code, not a new runtime comparison.
The system's [open questions](index.md#open-questions) retain other unresolved camera behaviour.

## How this was checked

The earlier PCSX2 observation below has no surviving run-record id. It is retained as historical checking notes.

**Runtime** (confirmed (runtime), PCSX2 2.9.94, a copy of quick-save slot 6 in `level99`, player standing, the
player camera `*(0x005d9150)` read every 33 ms, 2026-10-10). `level99`'s subway (six `dyn_s_train_med_full` cars and
the `part_train_sound`, moved by the scene `Coney_Sub1` or `Coney_Sub2`; each run's return function schedules the
next 30 s later) passed three times, at game times 557.4 s, 604.2 s and 650.8 s (46.6 s apart: the scene plus the
30 s wait). Each pass:

- slot 0's level rose from 0.004 to a peak of **0.106** (the closest approach, 60 − 0.106 / 0.004237 = **35 m**) and
  fell back, with the time at 0.883-0.950 s throughout (re-armed every 5 updates); the train was within 60 m for about
  **3.2 s**, then the last level held for 0.95 s and decayed; about 4.2 s of shake in all;
- the current amplitude followed the level within one or two updates (0.0027 on the first update at a level of
  0.0041, then 65 % of the gap each update);
- the view offset (`+0x190`) stayed within **±0.026 m** per axis (the bound at the peak is ±0.053 m), changing every
  update (30 Hz);
- `+0x1d7` was 1, switch 6 (`0x0050b1d0`) 1, and the pad's rumble bytes (`0x005dd810 + 0x40` / `+0x41`) **0 on every
  sample**, with the pad's actuators enabled (`+0x4c` = 1).

### Recorded world-shake setter locators {#world-setter-locators}

The earlier notes recorded `0x004db180` for the store to slot 0's amplitude and `0x004db188` for the store to its
time. Neither resolves in the current function listing or image-section map. They remain unresolved provenance;
this restructure does not invent replacement addresses or claim new executable analysis.

### Earlier observations retained from upstream {#upstream-earlier-observations}

These passages came from upstream 8ecd4cb8 without surviving run identifiers. Their original labels and
context are retained as earlier checking notes, not a new runtime comparison.

```text
**No pad rumble**: `Cam_UpdateShake` (`0x00121298`) finds the pad of slot *k* from the player list only for *k* ≥ 1
(slot 0 keeps pad −1), so slot 0 never writes the rumble bytes. Confirmed (code); confirmed (runtime) below.

**Runtime** (confirmed (runtime), PCSX2 2.9.94, a copy of quick-save slot 6 in `level99`, player standing, the
player camera `*(0x005d9150)` read every 33 ms). `level99`'s subway (six `dyn_s_train_med_full` cars and
the `part_train_sound`, moved by the scene `Coney_Sub1` or `Coney_Sub2`; each run's return function schedules the
next 30 s later) passed three times, at game times 557.4 s, 604.2 s and 650.8 s (46.6 s apart: the scene plus the
30 s wait). Each pass:

- slot 0's level rose from 0.004 to a peak of **0.106** (the closest approach, 60 − 0.106 / 0.004237 = **35 m**) and
  fell back, with the time at 0.883-0.950 s throughout (re-armed every 5 updates); the train was within 60 m for about
  **3.2 s**, then the last level held for 0.95 s and decayed; about 4.2 s of shake in all;
- the current amplitude followed the level within one or two updates (0.0027 on the first update at a level of
  0.0041, then 65 % of the gap each update);
- the view offset (`+0x190`) stayed within **±0.026 m** per axis (the bound at the peak is ±0.053 m), changing every
  update (30 Hz);
- `+0x1d7` was 1, switch 6 (`0x0050b1d0`) 1, and the pad's rumble bytes (`0x005dd810 + 0x40` / `+0x41`) **0 on every
  sample**, with the pad's actuators enabled (`+0x4c` = 1). Confirmed (runtime).
```
