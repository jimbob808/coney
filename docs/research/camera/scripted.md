---
kind: topic
system: camera
owns:
  functions:
    - "0x00123ea0"
    - "0x00123f08"
    - "0x00123f30"
    - "0x00123fb8"
    - "0x00124000"
    - "0x00124248"
    - "0x00124690"
    - "0x001352f0"
    - "0x00135390"
    - "0x001353b8"
    - "0x001353e0"
    - "0x001355f8"
    - "0x00135680"
    - "0x001358d0"
    - "0x00135960"
    - "0x00135ca8"
    - "0x0013b118"
    - "0x0013b290"
    - "0x0013b2b8"
    - "0x0013b358"
    - "0x0013b508"
    - "0x0013b5d8"
    - "0x0013b698"
    - "0x0013b6f8"
    - "0x0013b770"
    - "0x0013b780"
    - "0x0013b7b8"
    - "0x0013ba68"
    - "0x0013bad0"
    - "0x0013bfe0"
    - "0x0013c590"
    - "0x0013c5a8"
    - "0x0013c710"
    - "0x0013ccd8"
    - "0x0013ce78"
    - "0x0013cf20"
    - "0x0013d010"
    - "0x0013d6b0"
    - "0x0013dcc8"
    - "0x0013e708"
    - "0x0013f930"
    - "0x0013f9b0"
    - "0x0013fac8"
    - "0x0013fb30"
    - "0x0013fba0"
    - "0x00140308"
    - "0x00140708"
    - "0x00140830"
    - "0x00140ad0"
    - "0x00140d68"
    - "0x00142260"
    - "0x001422e0"
    - "0x00142308"
    - "0x00142368"
    - "0x00142578"
    - "0x00142690"
    - "0x001426c0"
    - "0x00142a58"
    - "0x00142ac8"
    - "0x00142d28"
    - "0x00142db8"
---
# Scripted camera views

Verified against: `SLUS_212.15` (NTSC-U); findings retain the earlier camera research's evidence levels.

## Summary

Fixed and locked shots, scripted angles, path and rail cameras, and the level scripts that configure them.
The camera manager and blends remain on [Camera](index.md#fn-manager).

## Map

The full entry list is in the [camera function database](functions.md). The camera classes are grouped there as
fixed, locked, rail and path cameras. Update entry points include:

| Address | Function | Subject | Evidence |
| --- | --- | --- | --- |
| `0x00124690` | `CamFixed_Update` | scripted camera update | confirmed (code) |
| `0x00135680` | `CamLocked_Update` | scripted camera update | confirmed (code) |
| `0x0013d010` | `CamRail_Update` | scripted camera update | confirmed (code) |
| `0x001426c0` | `PoizoCam_Update` | scripted camera update | confirmed (code) |

### Fixed-camera entries {#fn-fixed}

See the [camera function index](functions.md) for the complete entries.

### Locked-camera entries {#fn-locked}

See the [camera function index](functions.md) for the complete entries.

### Rail-camera entries {#fn-rail}

See the [camera function index](functions.md) for the complete entries.

### Spline-camera entries {#fn-spline}

See the [camera function index](functions.md) for the complete entries.

## Data

The level-script configuration values, rail control points and path records are described with their corresponding
operations below. Common camera fields stay on [the base camera object](index.md#the-base-camera-object).

## Behaviour

### Script calls in `level99` {#script-calls}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

Every camera call of `level99.lua`, `level99_combat.lua`, `level99_lesson1.lua`, `level99_lesson2.lua` and the
`global.lua` helpers they reach (read from the scripts' bytecode; the bindings: [mission 1
coverage](../../references/bindings/mission1.md)). Confirmed (code) at the cited functions.

| Call (where) | Effect |
| --- | --- |
| `CameraCreateFollow("follow", player)` (`AddCameras`) | `CamSetupFollow` + `CfgFollowCamera(3, 6.6, 4.8, 13, 65, 0.1, {0, 0, 1.4}, 0.2)` ([Setting up](index.md#setting-up)): band 3.0-3.5 m, zoom step 4.8, upper pitch limit 30° |
| `CameraMakeActive(MainCam, 0)` then `CameraReset(MainCam)` (`AddCameras`) | current at once; then placed behind the player at 3.0 m, pitch 13° |
| `CamEnable(3/4, false)` (combat setup), `true` again (lesson 1) | no effect with one player ([Switches](index.md#switches)) |
| `CamTarget(1, MainCam, player)` / `CamTarget(0, …)` (lesson 1) | removes the player from the shared target list, later adds it back. The follow camera takes its targets from the list's first two entries and falls back to its last target (`+0x31c`) when the list is empty (`0x001282a0`), so with one player nothing changes |
| `CamSetFollowZoom(1)` (lesson 1 setup; lesson 2 after `DealerPoizo` and the dealer's respawn) | **band 4.8-5.3 m**, zoom step 6.6, upper pitch limit 40° |
| `CameraCreateLocked(name, pos, fov, heading, pitch, 0, 0.1, far)` + `CameraMakeActive(name, 0)` | a cut to a [locked camera](#locked-cameras) (nine in the tutorial: fov 50 or 65, far 72.6-150) |
| `CameraReset(MainCam)` + `CameraMakeActive(MainCam, 1)` | back to the follow camera, placed behind the player, with a **1 s blend** ([Blends](index.md#blends)); after `VerminWait` with 0 s, a cut |
| `CamSetFollowAngle(-10)` (lesson 2 `P3.BreakFence`, between the reset and the blend) | target pitch −10° clamped to the limits, so the lower limit `atan(−0.4 / 5.3)` = **−4.3°** (the camera looks up at the fence); reached at once; it stays until the next `CameraReset` (after `DealerPoizo`) |
| `CamSetSecondary(Teacher.Vermin, 0, p)` / `(NilHandle, 0, p)` (lesson 2, both players) | keep Vermin in view ([Heading](follow.md#heading), factor 0.25, no range) instead of auto-follow; NilHandle ends it |
| `CamEnable(0, false/true)` (lesson 2 `P3.Player2Jumps` … `P3.VerminJumped`) | right stick and zoom buttons off while the camera watches Vermin's jump |
| `CamEnable(9, false/true)` (lesson 2 `P3.DealerHit` … `P3.CheckForFlash`) | power-move cameras off while the dealer respawns |
| `ScreenQueueEffect(2, 1)` / `(3, 1)` around `VerminCar`, `PedCam`, `ClimbPoizo`, `JumpCam`, `VerminFencePoizo` | letterbox in / out over 1 s ([Screen effects](../../references/screen-effects.md)); `VerminWait`, `FenceCam` and `DealerPoizo` have none |

`CamSetFollowAngle(degrees)` (`0x0011c3b8`): snaps player 0's look-at point, then for every follow camera sets the
target pitch to `degrees` clamped to `[+0x3b0, +0x3ac]`, turns the view to it at once (the wanted, previous and own
positions all set to the result) and clears the sprint latch `+0x466`. Confirmed (code).

`CamSetFollowHeading(degrees)` (`0x0011c2f0`, 14 calls in `level5`, `level80` and `level87`, all after a scene:
170-200, once 0) is **relative to the target's facing, not a world heading**. It snaps player 0's look-at point and
calls `Cam_Follow_PlaceBehind(degrees, −1)` (`0x00124f38`) on every follow camera, the same placement as
`CameraReset` with 180 (above). `Cam_Follow_PlaceAtHeading` (`0x001250a8`) builds the direction `D` = the first
listed target's forward (`Quat_AxisY` of its transform-table quaternion, `0x00714b10` + index × `0x20`; world +y
when there is no target) turned by `degrees` about **−z** (`Quat_FromAxisAngle` `0x00335ea0` uses the axis
(0, 0, −1) at `0x00511710`), so a positive angle turns **clockwise** seen from above, as the characters' headings do.
The camera's orientation is turned so that its backward axis lies along `D`, then the camera is put at the look-at
point + distance × `D` and pitched to the target pitch. So 180 stands it behind the player looking where he faces,
190 behind and 10° round to his left (clockwise from his facing, seen from above), 0 in front looking at his face.
Confirmed (code).

In detail, confirmed (code):

1. Player 0's follow camera: the look-at point is snapped (`0x00127d48`) and copied to the previous look-at points
   `+0x240` and `+0x270`.
2. For each player's follow camera, `Cam_Follow_PlaceBehind(degrees, −1)` (`0x00124f38`), the same call as
   `CameraReset`'s:
    - **Distance**: the current one (`+0x3e0`, the sprint zoom's saved edge, when it is set; else the camera's distance
      from the look-at point, `+0x260` to `+0x180`) clamped to the band `[+0x32c, +0x330]`, then rounded to a preset: at
      most `min + (default − min) / 2` → the minimum (zoom step the default); at most `default + (max − default) / 2` →
      the default (zoom step the maximum); else the maximum − 0.5 (zoom step the minimum). The **band** is moved to
      that distance .. distance + min(0.5, max − min), and the lower pitch limit follows (`0x001250a8` with its last
      argument 1).
    - **Heading**: `degrees` is **relative to the target's facing**, not a world heading: the target's forward (the
      y axis of its transform-table rotation, `0x00714b10`) is turned by `degrees` about **−z** (`Quat_FromAxisAngle`
      `0x00335ea0` uses the axis at `0x00511710`, (0, 0, −1)), so a positive angle turns clockwise seen from above;
      the camera is put on that side of the look-at point: 180 is behind, 90 on the target's right, −90 on his left.
      With no target in the list the world's +y is turned instead.
    - **Pitch**: the view is turned to the **current target pitch** `+0x3b4` (the target pitch is not reset).
    - The camera is placed at the look-at point − forward × distance (the held position `+0x260`), and the wanted
      `+0x250`, last-update `+0x1e0` and allowed `+0x2b0` positions are all set to it (no lag, no collision memory);
      `+0x46b` is set for one update (it keeps the next update's lag from running).

`Cam_Follow_SetHeading` (`0x001255f8`, which stands the camera at the band's near edge and runs a 1/60 s update) is
not this binding's: only the activation step and the co-op view change call it.

### Script calls in `level95` {#level95-calls}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

Every camera call of `level95.lua`, `level95_clubhouse.lua`, `level95_coney.lua` and `level95_workout.lua` (read from
the scripts' bytecode, as for `level99`). The hub plays the same follow camera at every checkpoint. Confirmed (code):

| Call (where) | Effect |
| --- | --- |
| `CameraCreateFollow("follow", player)`, `CameraMakeActive(follow, 0)`, `CameraReset(follow)` (`AddCameras`, from `RunLevel`) | as `level99`: `CfgFollowCamera(3, 6.6, 4.8, 13, 65, 0.1, {0, 0, 1.4}, 0.2)`, band **3.0-3.5 m**, zoom step 4.8, upper pitch limit 30°, behind the player |
| `CamSetFollowZoom(0)` (`SetupClubhouseEnvironment` from `StartLevel`, and `SetupConeyEnvironment`) | band stays at the minimum, 3.0-3.5 m: unlike `level99` from checkpoint 2 (4.8-5.3 m) |
| `CamSetFollowAngle(10)` (`StartTheWalk`, after `StartDoorWalk`; `FinalStartLevel`, the opening walk's callback, run by `WalkFinish` after `ExitDoorCam`) | target pitch **10°**, reached at once; any later `CameraReset` puts back the configured 13° (`ExitDoorCam` is also the door trigger's message-4 handler) |
| `CameraCreateLocked("Door1EnterCam" … "Door5EnterCam", …)` (`SetupClubhouseEnvironment`) | stored as `env.Doors[1..5] = {door, cam}`; the previous `env.Doors` cams are `CamDelete`d first; field of view 50, far 50 |
| `CameraCreateLocked("Door1LeaveCam" … "Door5LeaveCam", …)` (`SetupConeyEnvironment`) | the same table for the outside; far 90. Only `env.FinishLeaveClubhouse` calls this set-up, so no `*LeaveCam` exists until the player first leaves the clubhouse |
| `StartDoorWalk`: `CameraMakeActive(WalkTable.cam, 0)` when `WalkTable.cam` is set, else `CameraReset(MainCam)` | a cut to the door's locked camera, or the follow camera put behind the player at the start flag |
| `StartDoorWalk`, leaving only, with a second player: `CamEnable(3, false)`, `CameraReset(MainCam)` | switch 3 is the two-player view rule ([switches](index.md#switches)): nothing with one player |
| `ExitDoorCam(flag, human)`: `CamEnable(3, true)`, `CameraMakeActive(MainCam, 0.75)`, `CameraReset(MainCam)` | back to the follow camera, behind the player, with a **0.75 s** blend; ignored when `human` is given and is not a player; with a flag it also clears the flag's message-4 handler and turns its sphere off |
| `CameraSetClipping(MainCam, 0.1, 50 / 90 / 350)` | far clip 50 in the clubhouse, 90 on Coney Island, 350 in one scene set-up |
| `CamEnable(0, true / false)` (`ActivateWarchief` / `DeactivateWarchief`) | right stick and zoom buttons on / off around the warchief menu |
| the clubhouse menus' locked cameras (`MenuCam`, `name`, `options`, `replay`, `rumble`, `exit1`, `Radio`, `Cleon`, `Ajax`, `Vermin`, the workout's three) | cuts at fields of view 50-70, far 50 (299.6 for one workout shot) |

So in the hub the follow camera stands **3.0-3.5 m** from the look-at point (the hard band 2.80-3.85 m) at a 10° target
pitch after the opening walk (13° after a later reset), with the upper pitch limit 30°.

#### The door walks {#door-walks}

A door walk is one scripted walk of the player through a clubhouse door, with an optional locked camera. Confirmed
(code), from the bytecode of `level95.lua` (functions `PrepareDoorWalk`, `StartDoorWalk`, `WalkFinish`, `SkipWalk`,
`SkipWalkFinish`, `ExitDoorCam`, and the anonymous `env.FinishEnterClubhouse` / `env.FinishLeaveClubhouse`):

1. **`PrepareDoorWalk(enter, loc, callback, cam)`** only fills the global `WalkTable`: `loc`, `enter`, `start` and
   `term` (`fWchiefStart[loc]` and `fWchiefEnd[loc]` when `enter` is true, else `fRoamStart[loc]` and
   `fRoamEnd[loc]`), `door = env.Doors[loc].door`, `callback`, and `cam` (the fourth argument, unchanged).
2. **Who calls it**, three places:

    | Caller | `enter` | `loc` | `callback` | `cam` |
    | --- | --- | --- | --- | --- |
    | `StartLevel` (the level's opening, when unlock 6/3 is not set and `LoadLight` is false; otherwise `DoQuickMap` and no walk) | 2 | `WCLoc` (5 on a first visit, while unlock 6/4 is not set) | `FinalStartLevel` | **nil** |
    | `env.FinishEnterClubhouse` (30 ms, or 5 s while `env.BuildCac`, after `env.EnterClubhouse`) | true | `env.flagIndex` | nil | `env.Doors[env.flagIndex].cam`, a `DoorNEnterCam` |
    | `env.FinishLeaveClubhouse` (30 ms after `env.LeaveClubhouse`, the message-0 handler of each `fClubexit_N` flag, the door's action prompt) | false | `env.flagIndex` | `coney.CheckTutorial` | a `DoorNLeaveCam` |

    The two `env` callers first run `SetupClubhouseEnvironment` / `SetupConeyEnvironment`, which make the cams, so
    the cam handed over is the fresh one. `StartLevel` passes **no camera**: the opening walk has no door shot.
3. **`StartDoorWalk`** (from `StartTheWalk` 100 ms after `StartLevel`, or straight after `PrepareDoorWalk` in the `env`
   callers unless `LoadLight` is set): teleports the player to `start`, cuts to `cam` (or, with no cam,
   `CameraReset(MainCam)`), registers `SkipWalk` as the skip callback, opens `door` to −90°, and gives the player
   `GoalMoveToFlag(player, term, 2, 0, 0, 0.5, 0, true)` (a walk, gait 2, to within 0.5 m, then facing the flag). It
   sets `SetMsgHandler(term, 8, "WalkFinish")`. On the opening walk `StartTheWalk` then sets `CamSetFollowAngle(10)`.
4. **What ends it**: the flag's message 8, which the move-to-flag goal sends **once** when the player is inside its
   radius (0.5 m) and turned within 15° of the flag's heading ([ai.md](../ai-scripting.md#move-to-flag) End,
   `0x002da4a0`);
   the goal plans again each time a move ends short of the radius, which is not a new walk. Or the skip button:
   `SkipWalk` teleports to `term`, resets the follow camera and 100 ms later (`SkipWalkFinish`) calls `WalkFinish`.
5. **`WalkFinish(flag, human)`** first clears the message-8 handler and the skip callback, so it runs once; at the
   end it sets `WalkTable = nil`. Its camera step, only when `WalkTable.cam` is set:
    - **entering**: `ExitDoorCam(flag, human)` at once: the 0.75 s blend back to the follow camera, reset behind
      the player.
    - **leaving**: the `DoorNLeaveCam` stays on. `TriggerSphereCfg(term, true, r, 0, 200)` with r = 1 m (with a
      second player: 3, 2, 2.5, 1.5, 2 m for doors 1-5) and `SetMsgHandler(term, 4, "ExitDoorCam")`: the blend
      back starts when the player walks out of that sphere (message 4 is "left or died",
      [TriggerSphereCfg](../../references/bindings/world.md#triggerspherecfg)).

    Then it closes the door, gives control back (`ActivateWarchief`, a 0.5 s fade effect 3) and calls `callback`
    (`FinalStartLevel` on the opening walk: `CamSetFollowAngle(10)` again).

So the opening walk of `level95` is played on the **follow camera**: no `DoorNEnterCam`, no blend, no switch change;
the camera is reset behind the player at the start flag, pitched to 10° and follows the walk. The walk happens once.
Coney's repeated-walk investigation is on
[Camera implementation](../../engine/camera.md#door-walks).

### Locked cameras {#locked-cameras}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

A locked camera (type 1, `CameraCreateLocked`) stays where it was put, looking along its heading and pitch. Its update
(`0x00135680`) places it, aims it at a point 3 m ahead along its forward, keeps it out of walls with a line-of-sight
test from its position, runs `0x00135ca8` and applies the [shake](effects.md#shake). Its orientation comes from
three angles
in degrees, by the conventions of [Scripted camera angles](#scripted-angles). `0x00135ca8` does not
track: each human `CamLockLocked` lists (none in `level99`) that comes within 0.3 m of the view's left or right edge
is pushed back inside, an invisible wall at the frame's sides (confirmed (code)). The test point is 1.4 m above the
physics body; the camera's planes 0 and 1 (camera `+0x70`, inward normals; left and right inferred) are tried in
turn, and the first one the point is within 0.3 m of pushes it to exactly 0.3 m inside; the sideways part of the
step is clipped by a world ray, and the feet are snapped to the ground (ray from 2.4 m up, 3.4 m long). Details in
[`CamLockLocked`](../../references/bindings/camera.md#camlocklocked). Confirmed (code).

### Scripted camera angles {#scripted-angles}

`CameraCreateLocked(name, {x, y, z}, fov, heading, pitch, roll, near, far)` (binding `0x00365d38`: Lua arguments 4,
5 and 6 go to `+0x200`, `+0x204`, `+0x208`) and `CamAddPoizoPoint({x, y, z}, heading, pitch, roll, time,
callback)` (binding `0x00366e18`) take the same three angles in degrees and turn them into the camera's
orientation quaternion the same way: `CamLocked_BuildOrientation` (`0x001353e0`) and `CamSpline_AddPointAngles`
(`0x00142ac8`) are the same code. Confirmed (code) at both:

1. Three axis quaternions, each `(axis × sin(a/2), cos(a/2))` with `a` = angle × π/180 (`sinf` `0x004b8c40`,
   `cosf` `0x004b8a70`): `qh` about world +z (`0x00511700`) by the heading, `qp` about +x (`0x005116e0`) by the
   pitch, `qr` about +y (`0x005116f0`) by the roll. Confirmed (code).
2. `q = qh ⊗ qp ⊗ qr`, Hamilton products (the vector-unit sequence at `0x00135568` computes
   `(w1·v2 + w2·v1 + v1 × v2, w1·w2 − v1·v2)` with `qh` on the left, then the result times `qr`). With column
   vectors that is `R = Rz(h) · Rx(p) · Ry(r)`: the pitch turns about the camera's own right axis after the
   heading, and the roll about its own forward axis; the pitch is not about world x. Confirmed (code).
3. `q` (x, y, z, w) is stored at `+0x20` (and `+0x1f0` for a locked camera, the orientation it keeps). It maps the
   camera's local axes to the world ([Axes](index.md#the-base-camera-object): +x right, +y forward, +z up), as
   `Quat_AxisY` (`0x003363b0`: `(2(xy − wz), 1 − 2(x² + z²), 2(yz + wx))`) reads it. Confirmed (code).

So, with `h`, `p`, `r` the script's angles:

| Axis | World vector |
| --- | --- |
| forward (+y) | `(−sin h · cos p, cos h · cos p, sin p)` |
| up (+z) | `(cos h · sin r + sin h · sin p · cos r, sin h · sin r − cos h · sin p · cos r, cos p · cos r)` |
| right (+x) | `forward × up`; with `r` = 0 it is `(cos h, sin h, 0)` |

- **Heading** 0 looks along world +y and **grows anticlockwise** seen from above (90 looks along −x). This is the
  opposite way round from the characters' headings and `Vec_FromHeading` (`(sin h, cos h)`, clockwise,
  [Maths](../maths.md)): a camera heading `h` looks where a character heading `−h` faces.
- **Pitch positive looks up**, negative looks down: every `level99` locked camera that looks down at the street has
  a negative pitch (−2.7° to −27.7°).
- **Roll positive** tips the camera's top toward its right (+x). Of the 362 `CameraCreateLocked` and
  `CamAddPoizoPoint` calls in the disc's scripts (constant arguments), **7 pass a roll**, all locked cameras and all
  small: `level87`'s tag cameras `one` −0.1°, `ten` **−4.6°**, `eleven` 0.9°, `thirteen` 0.4°; `level95`'s two
  new-item shots 0.4°; `level84_final`'s `MercyCam` 0.4°. At 0.4° the horizon tips about 2 pixels across half the 640
  width (320 × tan 0.4°), so only `ten` shows clearly. The other rolls in the game: the **mugging shot's Dutch tilt**
  of 15° or 11° ([Mini-game and mugging cameras](special.md#mini-mug)), and the follow camera's eased roll
  (`Cam_Follow_EaseRoll` `0x0012ac58`: `q` = orientation ⊗ (0, sin(r/2), 0, cos(r/2)), a turn about the camera's
  own +y, its view axis), whose target `+0x404` only the constructor writes (0), so it stays 0.
  Confirmed (code) for the roll's maths and `+0x404`; the counts are a survey of the disc's scripts.
- **What the view needs for a roll**: nothing beyond the camera's own axes. The original draws through the current
  camera's orientation quaternion (`+0x20`): the view matrix below takes **up = `Quat_AxisZ(q)`** with right and
  forward from the same `q`, so a roll tips the picture with no separate step. A port that rebuilds the view from
  the eye, the look-at point and the world's +z drops it; one that carries `Quat_AxisZ(q)` (or `q` itself) to the
  renderer keeps it. To see it, use `level87`'s camera `ten`.
- The locked update (`0x00135680`) then rebuilds the same orientation each frame: the eye is the position
  (`+0x1e0`, also `+0x10`), the target the eye + 3 m × `Quat_AxisY(q)`, the up hint `Quat_AxisZ(q)` (`0x00336458`),
  through `Mat_LookAtUp` (`0x00337168`) and `Mat_ToQuat`, so the roll survives (the default up hint of
  `Mat_LookAt` would drop it). Its look-at point is that target. Confirmed (code).
- The **view matrix** (world to camera) is the inverse of `[right, forward, up]` placed at the eye: camera-space
  `(x, y, z) = (right · (P − eye), forward · (P − eye), up · (P − eye))`, with +y the depth; the lens is on
  [The streamed world](../world.md#player-camera).

Earlier observation: see the [recorded values and method](#upstream-earlier-observations).

| Camera | Position | h, p, r | fov / far | Quaternion `+0x20` (x, y, z, w) | Forward |
| --- | --- | --- | --- | --- | --- |
| `VerminCar` | 60.925, 43.425, 2.146 | 151.47183, −11.52631, 0 | 65 / 107.3 | −0.02474, −0.09732, 0.96427, 0.24515 | −0.4680, −0.8609, −0.1998 |
| `PedCam` | 44.246, 39.242, 1.752 | 49.16566, −2.67624, 0 | 50 / 150 | −0.02124, −0.00971, 0.41589, 0.90911 | −0.7558, 0.6532, −0.0467 |
| `VerminWait` | 50.613, 43.172, 1.936 | 21.78949, −6.02318, 0 | 50 / 72.6 | −0.05159, −0.00993, 0.18874, 0.98062 | −0.3691, 0.9234, −0.1049 |
| `FenceCam` | 47.497, 37.941, 3.507 | 177.78334, −11.85450, 0 | 50 / 150 | −0.00200, −0.10325, 0.99447, 0.01924 | −0.0379, −0.9779, −0.2054 |
| `ClimbPoizo` | 64.084, 15.313, 9.375 | 143.57188, −27.69311, 0 | 50 / 150 | −0.07480, −0.22733, 0.92229, 0.30348 | −0.5258, −0.7124, −0.4647 |
| `JumpCam` | 57.213, −1.465, 6.173 | 88.01634, 0.31346, 0 | 50 / 150 | 0.00197, 0.00190, 0.69476, 0.71924 | −0.9994, 0.0346, 0.0055 |
| `VerminFencePoizo` | 25.282, 6.984, 2.012 | −90.95051, −3.46206, 0 | 50 / 150 | −0.02118, 0.02154, −0.71262, 0.70090 | 0.9980, −0.0166, −0.0604 |
| `BreakFencePoizo` | 23.674, 0.423, 1.087 | 178.03990, 4.13726, 0 | 50 / 150 | 0.00062, 0.03609, 0.99920, 0.01709 | −0.0341, −0.9968, 0.0721 |

Earlier observation: see the [recorded values and method](#upstream-earlier-observations).

### Path cameras {#path-cameras}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

A path camera (`CamSetupPoizo`, points added with `CamAddPoizoPoint` / `CamAddPoizoPointCam`) holds at most 8 points
and moves along a Catmull-Rom curve through them, slerping between the points' orientations; a point's time is the
time to the **next** point. Its update (`PoizoCam_Update`, `0x001426c0`) runs only while it is player 0's current
camera, so a script makes it active itself. Confirmed (code); details per call on
[Cameras](../../references/bindings/camera.md#camsetuppoizo).

### Rail cameras {#rail}

A rail camera (type 9, one per player, `Camera_GetPlayerRail` `0x0011fbb0`) slides along a polyline of up to 16
points while it frames its targets: the chases of `level2`, `level3` and 17 later levels. Confirmed (code) at the
addresses cited; the bindings' arguments are on [Cameras](../../references/bindings/camera.md#camsetuprail).

**Set-up.** [`CamSetupRail`](../../references/bindings/camera.md#camsetuprail) (`Camera_SetupRail` `0x0011cce8`) names
the
camera, adds the human to the camera target list (`CameraTargets_Add(human, 1)`), sets the field of view, near and far
planes (far at most 150) and the look-at offset (`+0x320`, its z also the start of setting 9), then `CamRail_Reset`
(`0x0013b2b8`): no points (`+0x350` count, `+0x352` current segment), mode 0 (`+0x3ed`), the distance, setting 3 and 4
targets 0, the height target −1 (off), the angle targets off (−FLT_MAX), all ease times 0, the field-of-view target the
current one, and switches 7 and 8 back on. [`CamAddRailPoint`](../../references/bindings/camera.md#camaddrailpoint)
appends
points (`+0x1e0`, 16 bytes each). The camera does nothing until a script makes it current (`CameraMakeActive(rail,
seconds, nil, player)`). Confirmed (code).

**Settings.** [`CamModifyRail(setting, value, seconds, player)`](../../references/bindings/camera.md#cammodifyrail)
(`Camera_ModifyRail` `0x0011d228`) stores a target and an ease time; every update `CamRail_EaseValue` (`0x0013f930`)
moves each current value toward its target by `(target − current) × dt / time left` and takes the frame's time off,
so the value arrives linearly in `seconds` (0: at once). Confirmed (code), as is the table.

| Setting | Target / time / current | Unit | Used by mode 0 as | Evidence |
| --- | --- | --- | --- | --- |
| 0 | `+0x378` / `+0x3a8` / `+0x3c8` | m, 0 = off (negative stored as 0) | the most the camera stays from its target point in plan | confirmed (code) |
| 1 | `+0x37c` / `+0x3a4` / `+0x358` | m, negative = off | the most the camera stands above the first target's feet | confirmed (code) |
| 2 | `+0x374` / `+0x3a0` / the lens | degrees, negative ignored | field of view | confirmed (code) |
| 3 | `+0x380` / `+0x3ac` / `+0x360` | m | the target point's shift along the rail | confirmed (code) |
| 4 | `+0x384` / `+0x3b0` / `+0x364` | m | the look-at point's further shift along the rail | confirmed (code) |
| 5 | `+0x388` / `+0x3b4` / `+0x368` | degrees, below −360 = off | switches the rail to mode 3 (`0x0013fac8`) and is that mode's view pitch ([mode 3](#rail-mode3)) | confirmed (code) |
| 6 | `+0x38c` / `+0x3b8` / `+0x3d0` | | eased (`0x0013d35c`) but never read: nothing else touches `+0x3d0` after the constructor | confirmed (code) |
| 7 | `+0x390` / `+0x3bc` / `+0x3d4` | m | not read by modes 0-2; in mode 3 the margin the keep-in-frame step (`0x00140d68`, at `0x00141814` and `0x001418b8`) keeps each target inside the view's two side planes, in place of the target's radius (a player's scale × 0.5) | confirmed (code) |
| 8 | `+0x394` / `+0x3c0` / `+0x36c` | degrees, below −360 = off | a fixed pitch for the look-at (below) | confirmed (code) |
| 9 | `+0x398` / `+0x3c4` / `+0x328` | m | the look-at offset's height above the targets | confirmed (code) |

Settings 0 and 1 start from the camera's present distance and height when they are switched on from off, and a
height set negative eases back to the present height before it switches off. Setting 2 eases the lens itself.

**One update** (`CamRail_Update`, `0x0013d010`, skipped when the frame time is 0; confirmed (code)), in order:

1. Ease the settings (above) and the lead (`+0x35c` toward `+0x370`,
   [`CamLeadRail`](../../references/bindings/camera.md#camleadrail)).
2. **Targets** (`CamRail_GatherTargets`, `0x0013b7b8`): only entries 0 and 1 of the shared target list
   (`0x005d91a8`, up to 4) are read, players or not, in order (`+0x340`, `+0x344`, count `+0x348`). A resolved entry
   is kept at once when switch 3 is on, or when switch 12 is on, the rail is in mode 0 and switch 8 is on (both
   default on); otherwise it must pass `Camera_HumanCounts` (`0x00123500`: not down or dead, and not cuffed unless
   unlock 6/15 and an item count say so), which has no player test, so a standing non-player counts. With switch 3
   on, unless player 0's current camera is type 5 or (one player camera, or as many views as cameras) and both view
   flags are set, each rail keeps only the entry of its own player index (none when the list is shorter). When the
   count grows, `+0x3e5` and (not in mode 3) `+0x3e7` are set and the split timer is armed. None: the update stops
   here. So a one-player list [player 1, Diego] with switch 3 off gives two targets, the player first. Confirmed (code).
3. **The target point** (`CamRail_UpdateTargetPoint`, `0x0013bad0`) `P` = the plain mean of the targets'
   transform-table positions (their feet), each weighted equally.
   In mode 0, with `d` the current segment's direction flattened to the ground plane and normalised (confirmed (code)):
   - the look-at point starts as `P`; `P.z` += the offset's z (setting 9);
   - setting 3 non-zero: `P` += `d` × setting 3, and the look-at point = `P` + `d` × setting 4;
   - setting 3 zero, one target: the look-at point += the whole offset `(x, y, z)` when setting 4 is 0, else its z and
     then `d` × setting 4;
   - setting 3 zero, two targets: the look-at point = `P`.

   In modes 1-3 `P` += the offset and the look-at point = `P` (with two targets, modes 1 and 2 first pull `P` back
   toward the camera when the targets are spread wider than the camera's distance to them). With a split screen
   (`0x0050b1a4` = 0 and the view's `+0x44` ≠ `+0x48`) settings 3 and 4 and the offset's x and y are halved.
   In modes 0-2, unless `+0x3e8` is set or the first target has a state flag of `0x1c00000000`, `0x0013bfe0` then
   **damps** `P` and the look-at point toward last frame's values. With `Δ` = new − old, `h` its length in plan and
   `dz` its rise (the rise tests are skipped when the segment has no length in plan):
   `dz` > 0.5 → old + `Δ` × 0.5, then its z keeps only a quarter of its rise; else `h` > 1 → old + `Δ` × 0.25;
   else `dz` > 0.1 → with `f` = 1 − 1.875 (`dz` − 0.1), old + `Δ` × min(2`f`, 1), then its z keeps `f` of its
   rise; else `h` > 0.4 → old + `Δ` × (1 − 1.25 (`h` − 0.4)); else no damping. While `+0x3ec` is 1 the look-at
   point moves only 40% of the way each frame, until it arrives (then `+0x3ec` = 0). Confirmed (code).
4. **Placement** by mode (`+0x3ed`): 0 `CamRail_UpdatePosition` (`0x0013d6b0`, below), 1 and 2 `CamRail_PlaceLeading`
   (`0x0013e708`, set by `CamLeadRail`: the camera stands the lead ahead of or behind the target's place on the
   rail, [below](#rail-leading)), 3 `0x0013dcc8` ([below](#rail-mode3)). Confirmed (code).
5. Split-screen extras (`0x0013cf20`, only with two player cameras); a timer at `+0x3e0` fades the co-op flash,
   alpha 225 on both colour controllers' byte `+0x1b4` ([Two players](index.md#two-players)); then the camera's
   distance to
   `P` is kept at `+0x3d8` and the flags
   `+0x3e5`, `+0x3e7`, `+0x3e8`, `+0x3e9` cleared. Confirmed (code).

**Mode 0 placement** (`CamRail_UpdatePosition`):

1. Unless the camera is **held** (`+0x3e4`), choose the segment (`CamRail_ChooseSegment`, `0x0013fba0`): the rail
   point nearest `P` (3D) starts the segment, the last point giving the last segment (a tie between two points takes
   the later one only when `P` projects inside its segment); then, when `P` projects inside that segment
   (0 ≤ t ≤ length) it stays, else it moves to the previous or the next segment, whichever `P` projects inside. A
   change of segment sets a blend `+0x3cc` = 0.1. Confirmed (code).
2. **Project** `P` on the segment (`CamRail_ProjectOnSegment`, `0x00140308`): `Q` = the foot of the perpendicular,
   clamped to the segment's ends. With setting 0 on, when `Q` is farther than it from `P` in plan, `Q` moves toward
   `P` (in plan) until it is exactly that far: the camera leaves the rail to stay within reach. Confirmed (code).
3. While the blend is on, `Q` = camera + (`Q` − camera) × blend and the blend grows by 5% a frame until it reaches
   1 or the step is under 1 cm: a soft hand-over between segments. `Q` += the vector at `+0x330`, which is always zero:
   only the constructor (zero, `0x0013b25c`) and `CamRail_CopySettings` (`0x0013b3d0`) write it (mode 3 also
   subtracts it, `0x0013e4a8`). Confirmed (code).
4. **Height**: with setting 1 on, `Q.z` is lowered to the first target's feet + setting 1 when it is higher (a
   ceiling, it never raises the camera). The first target is entry 0 of the gathered list (`+0x340`, read at
   `0x0013d848`): the first kept entry of the shared list, so player 1 on a [player, Diego] rail. The camera stands at
   `Q`.
   Confirmed (code).
5. **Look-at**: with setting 8 on, the look-at point is replaced by the point 6 m from `Q` toward the targets in plan,
   raised by 6 × tan(setting 8). The camera then faces the look-at point.
6. **Ends**: when `Q` is the rail's first point and the look-at point lies behind the rail's start, or `Q` is the
   last point and the look-at point lies beyond the end, the hold `+0x3e4` is set (`0x0013dafc`) and the camera is
   put back as it was at the start of the update (position, orientation, `P`, look-at point), then faces the
   look-at point again. This hold lasts one update: step 7 clears it. So the camera stands still while the look-at
   point is past an end and follows again on the first update it is not. Confirmed (code).
7. **Collision** (`0x0013dbbc`-`0x0013dc1c`): when the camera is held (by step 6 or by `CamLockRail`), or with
   switch 3 off, switch 8 on and two or more targets, `CamRail_CollideTwoTargets` (`0x00140708`) runs and the hold
   is then set back to its value at the start of placement. Otherwise `CamRail_Collide` (`0x00140830`) runs for the
   first target ([below](#rail-collide)); it does not touch the hold. Confirmed (code).

**The hold** (`+0x3e4`, confirmed (code) at the addresses cited):

- **Mode 0 held** at the start of placement (the value reaches `CamRail_UpdatePosition` as its fifth argument, `t0` =
  `+0x3e4` ≠ 0 at `0x0013d498`): steps 1-6 are skipped. The camera's position, orientation, `P` and look-at point are
  put back to their values at the start of the update (last update's), the camera faces the look-at point and the
  frustum is rebuilt. Step 7 then runs `CamRail_CollideTwoTargets`: when a target is out of frame (`0x00140ad0`) it
  pushes the targets back into the frame (`0x00140d68`, [below](#rail-frame)); a held camera is not put back, it stays.
  The hold is written back as 1, so a `CamLockRail(true)` lock lasts until `CamLockRail(false)`. Confirmed (code).
- **Every writer**: the constructor (0, `0x0013b1fc`); `Camera_LockRail` (`0x0011d180`, the script's value);
  `CamRail_CopySettings` (`0x0013b498`, copies player 1's, so a joining player inherits his lock); the mode 0 ends
  test (1) and step 7 (the start value); mode 3 (1, below). `CamRail_Reset` (`0x0013b2b8`), `CamRail_Activate`
  (`0x0013b508`), `CamSetupRail` and switching modes never write it: a lock survives `CamSetupRail` and
  `CameraMakeActive`. Confirmed (code).
- **Modes 1 and 2** (`CamRail_PlaceLeading`, `0x0013e708`) never read it: a lock does not stop them. Confirmed (code).
- **Mode 3** (`0x0013dcc8`) sets it when the camera reaches the rail's first or last point (squared distance under
  10⁻¹⁰, `0x0013e260`, `0x0013e2ec`) and, while it is set, no longer moves along the rail toward the target: the
  camera keeps its position and its look-at point, still reframing (`0x00140d68`) and shaking. Mode 3 never clears it,
  so a mode-3 camera that reaches an end stays there until `CamLockRail(false)` (inferred from the list of writers).

#### Collision {#rail-collide}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

`CamRail_Collide` (`0x00140830`, confirmed (code))
does no world or physics test.
Its one caller is mode 0 (`0x0013dc18`). Let `A` be the first target's position in the transform table
(`0x00714b00` + index × `0x20`) plus the look-at offset `+0x320`, and `B` his position as copied at the start of
his own update (human `+0x10`, vtable `+0x9c`) plus the same offset. The helper `0x004dbb68` then acts by switch 2
(`+0x3e6`):

- **Switch 2 off** (the default): the camera keeps its distance from `P`. With `m` = max(2, 0.4 × (camera z − `P`.z)),
  when the camera is nearer than `m` to `P` in plan, it moves along the current segment's direction by `m` minus
  that distance: toward the segment's end, or toward its start when it is within `m` of the end (segment length −
  `m` ≤ its distance from the segment's first point). One step, with no easing or timer; it can leave the camera
  still nearer than `m`. The player is not moved. The camera then faces the look-at point and the frustum is rebuilt.
- **Switch 2 on** (`level83_c5_apartment.lua`): the **player** is kept at least 2 m in plan from the vertical planes
  through the previous, current and next segments. Each plane is unbounded, built from the segment and world up,
  with its positive side the one `B` is on. In turn, when `A` is nearer than 2 m to a plane on that side, `A` is
  pushed along the plane's normal to 2 m (plus 10⁻⁵). If any plane pushed, the player is placed at `A` minus the
  offset with his current rotation (`Human_SetTransform` `0x0023d440`, his vtable `+0x6c`). When he is jumping or in a
  long fall (state flags `0x400000000` or `0x1000000000`) he is landed first (`Human_Land` `0x0023e090`). The camera
  is neither moved nor turned. Confirmed (code).
- `CamRail_Collide` also builds a vertical plane 2 m in front of the camera along its level forward and passes it
  with a 0.3 m margin. The helper uses that plane only for a rail in modes 1-3 (it would push `A` to 0.3 m beyond
  it), which never call `CamRail_Collide`: the 0.3 m and 2 m have no effect.

#### Keeping the targets in frame {#rail-frame}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

(confirmed (code) at the addresses cited). Two helpers,
reached only through `CamRail_CollideTwoTargets` (mode 0, step 7) and mode 3's placement (`0x0013e088`). Planes are the
camera's frustum planes from `Camera_BuildFrustum` (`0x00120c50`), read through vtable `+0x174(i)`, normals inward:
0 left, 1 right, 2 near, 3 bottom, 4 top, 5 far. For each target, `A` is his transform-table position (feet) plus the
look-at offset `+0x320`, `B` his start-of-update position (human `+0x10`) plus the same offset, and `r` his radius:
a player's scale × 0.5, or for a non-player (`+0x1b0` = −1) his capsule radius (`Human_GetCapsuleRadius`).

- **`CamRail_TargetsInFrame`** (`0x00140ad0`) is a test, one pass over the targets: false as soon as one has
  `n · A − d < r` for plane 0, 1 or 3 (left, right, bottom; never the top), or, with switch 2 (`+0x3e6`), for the
  vertical plane 2 m in front of the camera along its forward axis. The 0.3 its caller passes is the radius only when
  its last argument is 0; `CamRail_CollideTwoTargets` passes 1, so the targets' own radii are used and the 0.3 is
  dead. Confirmed (code).
- **`CamRail_CollideTwoTargets`** (`0x00140708`): when that test fails and the camera is not held, the camera's
  position, orientation, target point `P` (`+0x180`) and look-at point (`+0x2f0`) go back to their values at the start
  of the update and it faces the look-at point again (frustum rebuilt); held or not, it then calls the keeper. The
  camera is never pulled back or up. Confirmed (code).
- **`CamRail_KeepTargetsInFrame`** (`0x00140d68`) **moves the targets, not the camera**: one pass, one correction of
  each kind per target, no loop. Before it, `zmax` / `zmin` are the highest and lowest of all the targets' feet and
  start-of-update heights, and `zmax` += max(offset z `+0x328`, 1 m). Per target, in order
  (confirmed (code) unless marked):
    1. With switch 2 on: in mode 0, the same 2 m push off the previous, current and next segments' vertical planes as
       `CamRail_Collide` ([above](#rail-collide)); in modes 1-3, `A` is pushed beyond the plane 2 m in front of the
       camera by `r`.
    2. **Sides**: if `A` is within the margin of the left plane (else the right; only one per update), its move
       from `B` is split into the part along the plane's normal and the rest; the normal part becomes the margin
       (+ 10⁻⁵) inside the plane, and the rest is cut short by a ray from `B` (mask `0x200`, as long as the rest + the
       margin) so it stops the margin before a wall. The margin is `r`, or setting 7 (`+0x3d4`) in mode 3.
    3. **Bottom** (both callers pass this on): when `A` is within `r` of the bottom plane it is moved across it by the
       shortfall, keeping its distance from the camera (inferred from the vector steps), again cut short by a ray.
    4. **Line of sight**, when any step moved him (or in mode 3 on player 0's current camera while `+0x3e5` is
       set): a ray (mask `0x200`, plus bit 0 in mode 3) from the point 2 m in front of the camera, at `A`'s height
       (the camera's in mode 3), toward `A`, as long as min(that distance, 0.7 × the far plane) + 3`r`; on a hit `A`
       is put `r` (along the face's normal) before the face.
    5. When a step moved him or the ray hit: `A` − offset is his new place; his height is found by a ray straight down
       from `zmax`, `zmax − zmin + 1` m long, and set to the hit + 0.01 m (kept when nothing is hit). If he is jumping
       or in a long fall (state flags `0x1400000000`) he is landed (`Human_Land(1, 1)`) and his height moves only 30%
       of the way. He is placed there with his current rotation (`Human_SetTransform`, vtable `+0x6c`).

  So on a two-target rail every target, **non-players included** (`level5`'s Diego), is pushed back inside the left,
  right and bottom edges of the view and kept out of walls between him and the camera; the camera only stops.

#### Mode 3 placement {#rail-mode3}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

(`CamRail_PlaceMode3`, `0x0013dcc8`; confirmed (code) at the
addresses cited unless marked). `CamModifyRail(5, angle)` at or above −360 sets the mode (`+0x3ed` = 3) and the angle
(`0x0013fac8`); below −360 it goes back to mode 0. Only the five Armies of the Night levels (`level60`-`level64`) use
it, each with `CamSetupRail("rail", player, 22, {0, −0.25, 1.25}, 0.5, 90)`, then `CamModifyRail(5, −7, 0)` and
`CamModifyRail(7, 0.5)`; their gang scripts (`level60_mothers`, `level61_hihats`, `level63_orphans`,
`level64_riffs`) later set setting 7 to −10 or −15 (a survey of the disc's scripts, values only). The mode's pieces:

- **The angle is a pitch.** The look-at point `+0x2f0` is always put 2 m from the camera in plan and
  2 × tan(angle) above it, so the camera looks at a fixed pitch: −7° in those levels, looking down. The camera
  itself stays on the rail (setting 1 aside, below); nothing raises it.
- **Start**, once, on the first update after `CamRail_Activate` (`0x0013b544` sets `+0x3e7`; the update clears it at
  its end, `0x0013d678`, and `CamRail_GatherTargets` sets it again only outside mode 3, `0x0013b9a0`): the base
  speed `+0x1a4` = 1 m/s; `P` = the targets' mean (`CamRail_UpdateTargetPoint`), segment chosen and `P` projected
  (`CamRail_ChooseSegment`, `CamRail_ProjectOnSegment`); the camera is placed at that point `Q`; the look-at point at
  2 m from `Q` toward `P` in plan, at the pitch; the goal `+0x310` = the look-at point. `+0x3ea` = 1 when the first
  target stands to the left of the segment's direction (z of the cross product > 0). Switching setting 5 on later
  does not restart: the next start is the next activation. Confirmed (code).
- **Projection in mode 3** (`0x00140308`): along the segment **in plan** (the plan distance divided by the plan
  length of the segment's unit vector, so the height follows the rail), and clamped only before the rail's first
  point and past its last; on an inner segment the point may run past the segment's end. Setting 0's reach is
  applied in mode 0 only. Confirmed (code).
- **Each later update**, with at least one target (none: the camera keeps its place, still shakes and faces the
  look-at point):
    1. **Which target.** With two or more, the one **farthest back** along the current segment's direction (smallest
       signed distance of his feet from the segment's start along it); with one, entry 0.
    2. `CamRail_KeepTargetsInFrame` (`0x00140d68`, [above](#rail-frame)) moves the targets back into frame, with the
       margin setting 7 and the line of sight from the 2 m point while `+0x3e5` is set (the update a target joined).
       Mode 3 passes `+0x3ea` and its negation in `a2` / `a3`, but the keeper overwrites both registers before
       reading them (`0x00140dd0`-`0x00140dd4`): `+0x3ea` is dead. Confirmed (code).
    3. `P` (`+0x180`) = that target's feet + (0, 0, offset z `+0x328`): only the offset's height; in mode 3 the
       mean and the damping of update step 3 are replaced.
    4. **Goal.** Segment chosen for `P`, `P` projected to `Q'`; the goal = `Q'` + 2 m toward `P` in plan, at the
       pitch. It is taken (`+0x310` = goal) only when it lies **ahead** of the look-at point along the segment's
       direction (plan dot product > 0): the camera never goes back along the rail.
    5. **Ends.** On segment 0, a camera within 10⁻⁵ m of the first point, or on the last segment within 10⁻⁵ m of
       the last point, sets the hold `+0x3e4` ([the hold](#rail)). Inferred: a camera that starts clamped at the
       first point holds at once, so these levels must start the player ahead of it.
    6. **Move**, when the goal was taken, the hold was off at step 4 and is still off: with `v` = the target's
       **stored velocity** (`Task_GetVelocity` `0x003a1f00`: the task's `+0x30`, zero while its flags `+0x54` have
       `0x600`; for a human what the locomotion writes, so 0 while a clip alone moves the body, and not the move
       made by a push) along the segment's direction in plan, the look-at point moves toward the goal by
       min(0.4 × the gap, (max(`v`, 0) + 1 m/s) × `dt`). So it moves 1 m/s faster than the target goes forward,
       never more than 40% of the gap in one update. The segment is then chosen again for the moved look-at point
       (so it **crosses joints** on its own) and the camera = the look-at point projected on the rail. Confirmed (code).
    7. **Segment blend.** While `+0x3cc` > 0 (set to 0.1 by a change of segment): the camera =
       old position − `+0x330` + `+0x3cc` × (new − old), and `+0x3cc` × 1.15 each update, until it reaches 1 or the
       step is under 0.3 m in plan (then the new position is taken at once). `+0x330` is always zero (only the
       constructor and `CamRail_CopySettings` write it), so the subtraction does nothing.
    8. The look-at point is put again 2 m from the camera in plan, toward where it was, at the pitch.
    9. Without a move (no goal ahead, or held) the goal is set to the look-at point and the camera keeps its place.
- **Finish**: `+0x2e0` = the position; with setting 1 above 0 the height is capped at the first target's feet +
  setting 1; the camera is placed; the shake (`0x00121298`, given the look-at point `+0x2f0`) moves the look-at
  point, so it turns the view; the camera faces the look-at point (`Mat_LookAt`) and the frustum is rebuilt.
  Confirmed (code).

So the Armies levels' camera trails the hindmost fighter along a one-way rail at a fixed downward pitch, catching up
at his forward speed + 1 m/s, and pushes everyone back inside the view (0.5 m inside the sides; with −10 or −15
during the gang fights, inferred: up to that far outside, so the side push stops).

#### Leading placement, modes 1 and 2 {#rail-leading}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

(`CamRail_PlaceLeading`, `0x0013e708`, set by
[`CamLeadRail(lead, seconds, ahead)`](../../references/bindings/camera.md#camleadrail); confirmed (code) at the cited
addresses unless marked). `s` = +1 in mode 1 (ahead), −1 in mode 2 (behind); the lead is `+0x35c`. The target point
`P` comes from update step 3 as in mode 0 (mean, offset, damping); the hold is never read.

1. **Segment.** `CamRail_ChooseSegment(P)` as in mode 0, then, when `P`'s plan distance from the segment's start is
   more than the segment's plan length, step forward while `P` lies beyond the next point along the segment (plan
   dot product > 0, up to the last segment); then step back while `P` lies before the segment's start.
2. **Foot and lead point.** `u` = the segment's direction in plan; `F` = start + `u` × (`P`'s plan distance along
   it), at the start point's height; `B` = `F` + `s` × lead × `u`. With two or more targets, `B` moves on by `s` ×
   the farthest any target's feet are ahead of `P` along `u` (positive values only; in mode 2 the first pass never
   adds one, as decompiled, so trailing adds nothing there).
3. **Lead point's segment.** The same forward and back steps for `B`. If they change the segment, `F` and `B` are
   worked out again on the new one (the farthest-ahead target added again, in either mode) and `B` is the mean of
   the old and new `B`: the camera does not jump at a joint.
4. **Rail point.** `t` = `B`'s plan distance from the segment's start ÷ the segment's plan length, a **length**
   (the plan length of `B` − start, never negative) and not clamped on an inner segment. So a lead point
   before the start gives the same `t` as one that far after it: in mode 2 with `P` less than the lead past the
   first point, the camera stands (lead − `P`'s distance) **past** the start, mirrored, not before it; and with `P`
   up to the lead behind the start, lead + that distance past it. Ends: on segment 0 with `P` more than the lead
   behind the start, `t` = 0 (the first point); on
   the last segment, unless `P` is still before the last point and `F` is at least the lead from it, the camera
   takes the last point (`t` = 1). `C` = the 3D lerp of the segment's points by `t`, so the height follows the rail.
5. **Switch 7** (`0x0050b2ac`, default 1, set to 1 again by every `CamSetupRail`, `0x0013b2b8`): when on, the
   camera leaves the rail: `X` = `P` + (lead + the farthest-ahead extra) × the plan direction from `F` to `C`, at
   `C`'s height. So it stands exactly the lead ahead of (or behind) the target in plan, beside the rail point. Off,
   `X` = `C`, on the rail. `level54_park` turns it off (`CamEnable(7, nil, 0)`). Confirmed (code).
6. **Collision**, only when `X` ≠ `C`: a world ray (`CollisionMesh_RayCast`, mask `0x200`) from `C` toward `X`; on
   a hit `X` is pulled back toward `C` to stop short of the face (by max(0.5, 2 × the value of vtable slot `+0x5c`)
   plus that value; inferred: the near plane), and a second ray back from `X` can shorten it again. Then `P` is pushed
   out of the world by a 0.5 m sphere (`CollisionMesh_SpherePush`) and the look-at point `+0x2f0` = `P`. With the
   switch off the look-at point is update step 3's.
7. **Finish**: `+0x2e0` = `X`; with setting 1 above 0, `X`'s height **is set** to the first target's feet +
   setting 1 (a fixed height here, not mode 0's ceiling); `Camera_PushOutOfHumans` (`0x00122728`, 3 m); placed;
   faces the look-at point; with `+0x3e8` set, the position and orientation are copied to the caller; frustum; with
   two or more targets `CamRail_CollideTwoTargets` (`0x00140708`, [frame](#rail-frame)). While a shake runs the
   view is faced again after it and `+0x3ec` = 2; on the update it stops `+0x3ec` goes 2 → 1, so the look-at point
   eases back at 40% an update (update step 3). Confirmed (code).

Levels calling `CamLeadRail` (survey of the disc's scripts, values only): `level2` (3 m), `level9` (4 m),
`level51` chapters 8 and 9 (12 m, ahead), `level54` and its `alley` / `park` scripts (7 and 8 m, `ahead` false,
and the chase's own `chase.CUR_LEAD_DIST`), `level81_chase` (14 and 15 m), `level84_chase` (12 m, ahead).

**In `level3`'s chase** ([`level3`](../scripting.md#level3)) the rail runs straight along y = 349 at z = 29.97, beside
the
rooftops the player runs along; the main framing (setting 3 = −8, 4 = 6, 0 = 5.5, 1 = 4, 9 = 2, field of view 78°)
puts the camera 8 m behind the player along the rail, at most 5.5 m from him in plan and 4 m above his feet, looking
at a point 2 m behind him and 2 m up. Inferred from the code above with the script's values.

### Fixed and third-person cameras {#fixed-third}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

- **The fixed camera** (type 0, [`CameraCreateFixed`](../../references/bindings/camera.md#cameracreatefixed)) stands at
  one point and looks at the average of the humans on the shared target list plus an offset; its target is put on
  that list. Confirmed (code) at `0x00124690`.
- **The third-person camera** (type 16, [`CameraCreateThird`](../../references/bindings/camera.md#cameracreatethird))
  sits a distance behind its target's facing, eased 10% an update toward it, a height above the look-at point and
  swung by an angle. Confirmed (code) at `0x001205e8`.

## In Coney

The port notes and door-walk troubleshooting are in [Camera implementation](../../engine/camera.md#door-walks).

## Open questions

The original page's outstanding camera comparisons remain in the system's
[open questions](index.md#open-questions). Historical observations below do not establish fresh run records.

## How this was checked

The following earlier PCSX2 observations are preserved with their methods and measurements. Their source page
predates run-record ids; this restructure does not fabricate those records or claim a new runtime comparison.

### Level 99 checking notes {#level99-camera-checks}

`CamSetFollowZoom(preset, player = −1)` (`0x0011c470`), for each player's follow camera (or one): the look-at point is
snapped (`0x00127d48`, its previous values set to it, so no ease), then the band is moved so that its near edge is
the preset distance (kept 0.5 m deep, clamped to `[min, max]`, and saved in `+0x344`/`+0x348`), the lower pitch limit
recomputed from the new far edge, and the zoom step set to the next preset: **0** → near edge the minimum, step the
default (30°); **1** → the default, step the maximum (40°); **2** → the maximum (band max − 0.5 .. max), step the
minimum (50°). With two player cameras 0 acts as 1. The camera itself is not moved: the leash drags it into the new
band over the next updates. Confirmed (runtime): checkpoint 1 states read 3.0 / 3.5, step 4.8, 30°; a run left to
itself from `l99-warriors-fight-start` until lesson 1, and the street saves, read 4.8 / 5.3, step 6.6, 40°.

### At runtime in the clubhouse {#level95-runtime}

Reference numbers to compare Coney with. Confirmed (runtime), PCSX2 2.9.94: from a `level99` state
(slot 7, only read) with the `scripted-pad` and `right-stick` patches, `level95` was loaded by writing its level
index (29 in the level table at `W_GameState + 0x14e8`) to mode 8's `+0x20` (`0x005e5db0`) and 3 to
`W_GameState + 0x14c`, the two writes `MenuLoadLevel` makes ([Front end](../frontend.md)). The level opened in the
clubhouse with the opening walk done. The camera object (`*0x005d9158`) was read over PINE about every 0.1 s, so the
times are wall-clock and approximate:

| Moment | Camera |
| --- | --- |
| standing after the opening walk, feet at (−184.92, 95.46, −194.38) | 3.50 m from the look-at point (feet + 1.4 m), pitch **10.0°**, band 3.0-3.5 m, hard band 2.80-3.85 m, zoom step 4.8, upper pitch limit 30°, lower −6.58°, lag 0.22 |
| walking, left stick 50 % straight up (raw y 47), about 1.6 m/s | the distance grows to **3.69 m** and the pitch drops to 9.5° while he walks; back to 3.50 m and 10.0° about 0.5 s after he stops |
| right stick 50 % right (raw x 208) for 1.0 s, player still | the view turns clockwise at about **93-96°/s** (the formula gives 96.5°/s), about 100° in all; during the turn the distance dips to **3.42 m** and the pitch rises to 10.24°, both back within 0.5 s of letting go |
| a run, left stick 100 % up, into a wall after 0.5 s | while he pressed into the wall the camera stood **3.85 m** away (the hard band's far edge) and stayed there after he stopped, although the wanted and held positions (`+0x250`, `+0x260`) were 3.50 m away; the cause is not traced |

### The first visit's opening, at runtime {#level95-first-visit}

The same load (the profile of slot 7 has not done the hub tutorial, unlockable `(6, 4)` locked, so `WCLoc` is 5),
recorded from the moment the level starts. Headings are the game's (`Quat_Heading`, clockwise from +y,
[Maths](../maths.md)); a recorder heading (2 atan2(q.z, q.w), anticlockwise) is 360° less this. Times are game time from
the level's start (`*(0x0050b734) + 0x48`, which the load sets back to 0) and updates at 30 a second; the player was
read every one or two updates, so each moment is good to about 2 updates. Confirmed (runtime), PCSX2 2.9.94;
what each step is comes from the scripts (`level95.lua`: `StartLevel`, `FinalStartLevel`, and the `tutorial` table's
`RunTutorial`, `ShowANewItem`, `StopWalk` and `Setup`), confirmed (code) for the bytecode:

| Game time (update) | What happens |
| --- | --- |
| 0 | the player stands at `fWchiefStart_1` (−188.6, 95, −194.38), where `AddWarchief` makes him |
| 0.10 s (3) | teleported to `fWchiefStart_5` (−185.2, 112.7, −193.79), facing the flag's 182°; brain "dead" (`+0x09` = 1) |
| 0.43-0.70 s (13-21) | turns in place towards the end flag |
| 0.73 s (22) | **walk 1** starts (gait 1, then 2 from 1.23 s), straight down −y at about 1.6 m/s |
| 4.07 s (122) | stops at (−185.48, 107.63): **5.07 m** walked, 0.43 m short of **`fWchiefEnd_5` (−185.5, 107.2, −193.8), heading 182°**, inside the goal's 0.5 m radius |
| 4.17 s (125) | turned to 177.4° (recorded 182.6°; within the goal's 15° of the flag's 182°): message 8, so **`WalkFinish`** runs between 4.10 and 4.27 s (updates 123-128), then `FinalStartLevel` |
| 4.27 s (128) | **walk 2** starts at once: `FinalStartLevel` sees the tutorial locked, puts the player back under script control (`DeactivateWarchief`) and gives him `GoalMoveToFlag(player, fWchiefEnd_1, walk, 0.5 m)` |
| about 7.6 s (229) | he enters the volume box `vbTutorialStart` (centre (−187.8, 98, −194.1), size (6, 4, 4); entered at y ≈ 102, so the size is a half-extent, inferred): `tutorial.RunTutorial` gives the **same** `GoalMoveToFlag` again (its Resume clears the running move; the new move action waits a random 0-250 ms first, the brain asking for speed 0 meanwhile, [AI](../ai-moving.md#move-action)); the positions, read every 1-2 updates, show no stop here, so the walk goes on, and starts the tutorial text and shots |
| 12.21 s (366) | walk 2 ends at (−184.92, 95.47), 0.47 m from `fWchiefEnd_1` (−184.9, 95, −194.3), inside the goal's 0.5 m radius: **12.2 m** in 7.9 s, facing 178° (the game's clockwise heading; the recorder's 182° is anticlockwise) |
| 12.24-12.44 s (367-373) | the goal's face-the-flag turn (`faceFlag` true; facing 178°, he is 89° off the flag's 89°): a turn action at the standing 12° per update, which ends within 15°, at **97.6°** (recorded as 262.4°); then the goal arrives, its End sends the flag message 8, and `tutorial.StopWalk` (the flag's message-8 handler, installed by `RunTutorial`) flushes the player's goals and removes the handler; he stands |
| **25.05 s (751)** | brain "dead" back to 0: **control returns** (`ActivateWarchief`, at the end of the tutorial's last item) |

The tutorial (`ShowANewItem`, rescheduled after each item's time) is seven items, one skipped: item 1, 4.5 s, the
text `LEVEL95.ACT[11]` on the follow camera; items 2-5, 2.5 s each, a cut (blend 0) to the locked cameras `name`,
`options`, `replay` and `rumble` with the flags `fVermin` and `fMenu` flashing; item 6 is not defined and is
skipped; item 7, 2.5 s, the `exit1` camera on `fClubexit_4`. That is **17.0 s**; then a cut back to `MainCam` (no blend
and no reset) and `ActivateWarchief` (`BrDead(player, false)`, right stick on). From the box at about 7.6 s that
gives about 24.6 s, as seen (25.05 s).

So in the original the opening has **two walks, not three**: the door walk (5.1 m, 4.1 s) and one walk to `fWchiefEnd_1`
(12.2 m), which the tutorial's second `GoalMoveToFlag` continues rather than restarts, and which ends as walk 1 does:
arrival inside the radius, the turn to the flag's heading (to within 15°), message 8. Control comes back **about 25 s**
after the level starts (21 s after `WalkFinish`). A walk 1 that ends at y 108.1 after 3.7 m is 0.9 m from
`fWchiefEnd_5`, outside the 0.5 m radius, so the original would not have sent message 8 there.

### Scripted-angle checking notes {#scripted-angle-checks}

**Runtime check** (confirmed (runtime), PCSX2 2026-10-07, a copy of the user's slot 10 in `level99` checkpoint 3,
every locked camera object read from the handle table `0x006ebd38` slots `0x3c`-`0x6b`). The angles at
`+0x200`-`+0x208` are the script's; `+0x20` equals the formula above to 5 decimals for all eight, and each
camera made current (player 1's current-camera word `0x005d9150` pointed at it) showed its shot:

| Camera | Position | h, p, r | fov / far | Quaternion `+0x20` (x, y, z, w) | Forward |
| --- | --- | --- | --- | --- | --- |
| `VerminCar` | 60.925, 43.425, 2.146 | 151.47183, −11.52631, 0 | 65 / 107.3 | −0.02474, −0.09732, 0.96427, 0.24515 | −0.4680, −0.8609, −0.1998 |
| `PedCam` | 44.246, 39.242, 1.752 | 49.16566, −2.67624, 0 | 50 / 150 | −0.02124, −0.00971, 0.41589, 0.90911 | −0.7558, 0.6532, −0.0467 |
| `VerminWait` | 50.613, 43.172, 1.936 | 21.78949, −6.02318, 0 | 50 / 72.6 | −0.05159, −0.00993, 0.18874, 0.98062 | −0.3691, 0.9234, −0.1049 |
| `FenceCam` | 47.497, 37.941, 3.507 | 177.78334, −11.85450, 0 | 50 / 150 | −0.00200, −0.10325, 0.99447, 0.01924 | −0.0379, −0.9779, −0.2054 |
| `ClimbPoizo` | 64.084, 15.313, 9.375 | 143.57188, −27.69311, 0 | 50 / 150 | −0.07480, −0.22733, 0.92229, 0.30348 | −0.5258, −0.7124, −0.4647 |
| `JumpCam` | 57.213, −1.465, 6.173 | 88.01634, 0.31346, 0 | 50 / 150 | 0.00197, 0.00190, 0.69476, 0.71924 | −0.9994, 0.0346, 0.0055 |
| `VerminFencePoizo` | 25.282, 6.984, 2.012 | −90.95051, −3.46206, 0 | 50 / 150 | −0.02118, 0.02154, −0.71262, 0.70090 | 0.9980, −0.0166, −0.0604 |
| `BreakFencePoizo` | 23.674, 0.423, 1.087 | 178.03990, 4.13726, 0 | 50 / 150 | 0.00062, 0.03609, 0.99920, 0.01709 | −0.0341, −0.9968, 0.0721 |

All eight have near 0.1. In `level80` (checkpoint 1, the "Let's Go" tutorial) the current camera is `StartCam1`
((−194.746, 130.353, 2.699), h −163.123, p −3.893, fov 65, far 150): `+0x20` reads (0.00498, −0.03360, 0.98860,
−0.14667), the formula's quaternion negated (the same rotation; `Mat_ToQuat` picks the sign), and `level2`'s first
camera `StartLock` ((453.389, 203.602, 15.861), h −111.904, p −19.474) reads (−0.09469, 0.14013, −0.81663, 0.55182),
the formula's to 4 decimals; confirmed (runtime).
**Worked example**, `FenceCam` (`level99_lesson2`, checkpoint 3, the shot of Vermin going
over the chain-link fence): `CameraCreateLocked("FenceCam", {47.497, 37.941, 3.507}, 50, 177.78334, −11.85450, 0,
0.1, 150)`. `qh` = (0, 0, sin 88.892°, cos 88.892°) = (0, 0, 0.99981, 0.01934); `qp` = (sin −5.927°, 0, 0, cos
−5.927°) = (−0.10326, 0, 0, 0.99465); `q = qh ⊗ qp` = (−0.00200, −0.10325, 0.99447, 0.01924), as read. Forward =
(−sin 177.78° · cos 11.85°, cos 177.78° · cos 11.85°, sin −11.85°) = (−0.0379, −0.9779, −0.2054): south, 11.9°
down. Up = (−0.0079, −0.2053, 0.9787), right = (−0.9993, 0.0387, 0); look-at point (47.383, 35.007, 2.891), as
read at `+0x180`. View translation (right, forward, up rows times −eye) = (45.994, 39.622, 4.733). The frame
looks down the alley at the fence from above head height; read with the pitch's sign flipped it would look 11.9°
up, over the fence into the buildings.

### Earlier observations retained from upstream {#upstream-earlier-observations}

These passages came from upstream 8ecd4cb8 without surviving run identifiers. Their original labels and
context are retained as earlier checking notes, not a new runtime comparison.

```text
**Runtime check** (confirmed (runtime), PCSX2, a copy of the user's slot 10 in `level99` checkpoint 3,
every locked camera object read from the handle table `0x006ebd38` slots `0x3c`-`0x6b`). The angles at
`+0x200`-`+0x208` are the script's; `+0x20` equals the formula above to 5 decimals for all eight, and each
camera made current (player 1's current-camera word `0x005d9150` pointed at it) showed its shot:

All eight have near 0.1. In `level80` (checkpoint 1, the "Let's Go" tutorial) the current camera is `StartCam1`
((−194.746, 130.353, 2.699), h −163.123, p −3.893, fov 65, far 150): `+0x20` reads (0.00498, −0.03360, 0.98860,
−0.14667), the formula's quaternion negated (the same rotation; `Mat_ToQuat` picks the sign), and `level2`'s first
camera `StartLock` ((453.389, 203.602, 15.861), h −111.904, p −19.474) reads (−0.09469, 0.14013, −0.81663, 0.55182),
the formula's to 4 decimals; confirmed (runtime).
**Worked example**, `FenceCam` (`level99_lesson2`, checkpoint 3, the shot of Vermin going
over the chain-link fence): `CameraCreateLocked("FenceCam", {47.497, 37.941, 3.507}, 50, 177.78334, −11.85450, 0,
0.1, 150)`. `qh` = (0, 0, sin 88.892°, cos 88.892°) = (0, 0, 0.99981, 0.01934); `qp` = (sin −5.927°, 0, 0, cos
−5.927°) = (−0.10326, 0, 0, 0.99465); `q = qh ⊗ qp` = (−0.00200, −0.10325, 0.99447, 0.01924), as read. Forward =
(−sin 177.78° · cos 11.85°, cos 177.78° · cos 11.85°, sin −11.85°) = (−0.0379, −0.9779, −0.2054): south, 11.9°
down. Up = (−0.0079, −0.2053, 0.9787), right = (−0.9993, 0.0387, 0); look-at point (47.383, 35.007, 2.891), as
read at `+0x180`. View translation (right, forward, up rows times −eye) = (45.994, 39.622, 4.733). The frame
looks down the alley at the fence from above head height; read with the pitch's sign flipped it would look 11.9°
up, over the fence into the buildings.
```
