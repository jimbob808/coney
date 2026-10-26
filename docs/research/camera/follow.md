---
kind: topic
system: camera
owns:
  functions:
    - "0x00124778"
    - "0x00124be0"
    - "0x00124c28"
    - "0x00124d00"
    - "0x00124f38"
    - "0x001250a8"
    - "0x001254f0"
    - "0x00125588"
    - "0x001255f8"
    - "0x00125870"
    - "0x00125888"
    - "0x001259e8"
    - "0x00125a40"
    - "0x00125ac8"
    - "0x00125b60"
    - "0x00125c50"
    - "0x00125cc0"
    - "0x00125e50"
    - "0x001260e8"
    - "0x00126370"
    - "0x00126450"
    - "0x00126488"
    - "0x00126520"
    - "0x00126558"
    - "0x00126878"
    - "0x00126a30"
    - "0x00126a68"
    - "0x00126a88"
    - "0x00126c40"
    - "0x001276a0"
    - "0x00127ac8"
    - "0x00127c10"
    - "0x00127cd8"
    - "0x00127d48"
    - "0x00127d88"
    - "0x00128298"
    - "0x001282a0"
    - "0x00128678"
    - "0x00128cf0"
    - "0x00129c78"
    - "0x00129f88"
    - "0x0012a400"
    - "0x0012a7d8"
    - "0x0012aae0"
    - "0x0012ac58"
    - "0x0012adf0"
    - "0x0012ae58"
    - "0x0012d3b8"
    - "0x0012d4e8"
    - "0x0012d688"
    - "0x0012d7a8"
    - "0x0012d908"
    - "0x0012da20"
    - "0x0012de48"
    - "0x0012ded8"
    - "0x0012e170"
    - "0x0012e9a8"
    - "0x0012ec50"
    - "0x0012f3e0"
    - "0x0012f858"
    - "0x0012fd20"
    - "0x001303c8"
    - "0x00134ab0"
    - "0x00134b70"
---
# Follow camera tracking

Verified against: `SLUS_212.15` (NTSC-U); findings retain the earlier camera research's evidence levels.

## Summary

How the follow camera tracks a moving target: look-at and aim points, heading, pitch, distance band, falls, climbs,
zoom and height hold. Input controls and world collision are separate topics:
[look controls](look-controls.md) and [collision](collision.md).

## Map

The complete function entries are in the [camera function database](functions.md). Start with:

| Address | Function | Subject | Evidence |
| --- | --- | --- | --- |
| `0x0012ae58` | `Cam_Follow_Update` | the ordered follow update | confirmed (code) |

### Follow-camera entries {#fn-follow}

See the [camera function index](functions.md) for the complete entries.

## Data

### The follow camera object {#the-follow-camera-object}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

0x480 bytes. Fields written by the constructor and `CfgFollowCamera`, confirmed (code); "runtime" values are from
`level99` with the player standing, after the tutorial script's own calls.

| Offset | Meaning | Constructor | `CfgFollowCamera` in `level99` (runtime) |
| --- | --- | --- | --- |
| `+0x180` | look-at point (target position + offset) | | player position + (0, 0, 1.4) |
| `+0x200` | the target's offset (set by `CamSetupFollow`) | | |
| `+0x1e0` / `+0x1f0` | the camera's position and orientation at the end of the last update (copied from `+0x10` / `+0x20`) | | |
| `+0x240` | the [aim point](#aim-point) the view faces | | look-at + 0.36 m along the facing, at the body's height |
| `+0x250` | the wanted position (after the leash and the rules that turn it) | | |
| `+0x260` | the held position: the wanted position after the lag and the hard band, before collision | | |
| `+0x270` | last update's look-at point | | |
| `+0x2b0` | the position the collision step returned on the last update ("allowed") | | |
| `+0x2e0` | the eased copy of the target's rotation the aim point leads along (35% per update) | | |
| `+0x210`-`+0x21c` | look-at offset from the target's feet | (0, 0, 1.4) | (0, 0, 1.4) |
| `+0x300` | minimum distance | 4 | 3 |
| `+0x304` | maximum distance | 9 | 6.6 |
| `+0x308` | default distance (also written to `+0x344` and `+0x32c`) | 6 | 4.8 |
| `+0x30c` (and `+0x3b4`) | pitch, radians | 15° | 0.2269 (13°) |
| `+0x310` (and `+0x394`) | field of view, degrees | 65 | 65 |
| `+0x32c` / `+0x330` | the leash band: default and default + min(0.5, max − min) (also `+0x344` / `+0x348`) | 6 / 6.5 | 3.0 / 3.5 at checkpoint 1; **4.8 / 5.3** in the street, and 3.0 / 3.5 while sprinting ([In the street](#street)) |
| `+0x320` / `+0x3fc` | handle of a human or object the camera keeps in view (`CamSetSecondary`) / its range | | |
| `+0x3cc` | the band's near edge saved by the [combat camera](#combat-camera) (0 for none) | | 0 |
| `+0x46f` | the combat camera is on | | 0; 1 with L1 held at a target |
| `+0x33c` / `+0x340` | the hard band: the leash band widened by max(5%, 0.2 m) and max(6%, 0.35 m) | | |
| `+0x34c` | the band's wanted near edge: `+0x32c` eases toward it ([Sprint zoom](#sprint-zoom)); −1 for none | | 3.0 while sprinting, else −1 |
| `+0x36c` | seconds the target has run or sprinted (zeroed when it stops on the ground) | | |
| `+0x3c8` | a band saved by another zoom (`0x00126558` / `0x00126878`, not the sprint); while it is set the sprint zoom leaves the band alone | 0 | 0 |
| `+0x3e0` / `+0x3e8` / `+0x3e4` | the sprint zoom's saved band near edge, zoom distance and target pitch (`+0x3e4` −FLT_MAX for none) | | 4.8 / 6.6 / 13° |
| `+0x448` | sprint zoom state: 0 off, 1 zoomed in, otherwise the game time (ms) at which to zoom back out | 0 | |
| `+0x45b` / `+0x45c` / `+0x45d` | `+0x45b`: the collision step's main ray was blocked this update; `+0x45c`: a sticky copy of it; `+0x45d`: a sticky "the view was blocked" latch. The latches hold until the player stops, and auto-follow stays off while they are set ([Heading](#heading)) | 0 | 0 in the street; 1 for the whole run at checkpoint 1 |
| `+0x466` / `+0x467` / `+0x468` | sprint zoom latched / armed for this sprint / enabled (`CamEnable(5, on)`) | | 0 / 0 / 1 |
| `+0x474` | auto-centre latch: cleared while the stick points more than 157.5° from up, set again when the target moves | | 1 |
| `+0x350` | heading target (a direction; −FLT_MAX for none) | | |
| `+0x354` | pitch override (−FLT_MAX for none), set only by aiming (`0x00128678`) and `TurnToMatrix` ([update step 16](#update)) | | |
| `+0x370` | hold timer of a reverse button shared with the zoom button (one per pad) ([buttons](look-controls.md#buttons)) | | |
| `+0x47c` / `+0x3c0` | queued hand-over: the camera to make current when the timed move `+0x40c` ends, and the blend time (s); set by `TurnToMatrix` ([mini-game and mugging cameras](special.md#mini-mug)) | 0 / 0 | |
| `+0x3d4` | target pitch saved by [ReactToLedge](#ledge) (−FLT_MAX for none) | | |
| `+0x35c` | band near edge saved while [looking behind](look-controls.md#reverse) | | |
| `+0x455` / `+0x458` / `+0x459` | view pulled out behind / pad holding the reverse button (−1 for none) / drop the reverse hold ([looking behind](look-controls.md#reverse)) | | |
| `+0x453`, `+0x220` / `+0x230`, `+0x324` | [height hold](#height-hold): on, the stored face's normal and point, the held height | | |
| `+0x464` / `+0x471` / `+0x479` | collision: wall push allowed (set with `+0x462` / `+0x463` by `0x0012fd20`) / a given-up turn found one side open / full-block turn latch (0, 1, 2, −1) ([collision](collision.md#collision)) | | |
| `+0x358` | right-stick yaw rate, rad/s | | |
| `+0x368` | stick hold timer: 0.334 s after any camera input | | |
| `+0x380` | recovered distance (eases 10% per update) | | |
| `+0x388` | position lag: share of the wanted move covered per update | 0.22 | |
| `+0x398` | scale of the height hold's 30% ease (1 normally; 0.25 after `0x00125888`) | | 1 |
| `+0x3ac` | an upper pitch limit | 50° | 30° at checkpoint 1; **40°** in the street (band 4.8-5.3) |
| `+0x3b0` | a lower pitch limit: `atan((1 − offset.z) / max)`, at least −20° | −20° | |
| `+0x3b8` | right-stick pitch rate, rad/s | | |
| `+0x400` | zoom step: the preset the zoom button goes to **next** (the default while the band is at the minimum, the maximum while it is at the default, the minimum while it is at the maximum) | 6.5 | 4.8 at checkpoint 1; 6.6 from checkpoint 2 |
| `+0x40c` | timed-move timer, seconds (heading target, pitch override, and the sprint zoom's 0.5 s) | | |
| `+0x434` / `+0x438` | side factors capping the stick yaw: 1, or −1 on the side of a wall push ([right stick](look-controls.md#right-stick)) | | |
| `+0x444` | number of targets | | |
| `+0x418` | the [aim point](#aim-point)'s lag, worked out each update from 0.06 | 0.06 | 0.061 standing, 0.14-0.16 running |
| `+0x440` | the current distance | | 1.85 standing (runtime) |
| slot `+0x1a4` | near plane | 0.1 | 0.1 |
| `0x005148a0` (global) | slow-motion factor | | 0.2 |

The far clip is 115 for this camera ([The streamed world](../world.md#player-camera)).

### `level99`'s values {#level99-values}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

`CameraCreateFollow` passes the `global.lua` constants ([Scripts](../scripting.md#cameras-from-lua)): minimum 3, maximum
6.6, default 4.8, pitch 13°, field of view 65, near plane 0.1, offset (0, 0, 1.4), slow motion 0.2. historical
observation ([checking notes](#follow-inline-observations)) in the object above.

## Behaviour

### The follow update {#update}

`0x0012ae58` runs once per character step (30 Hz; it stores its `dt` argument at `+0x1a0`, where the heading rules read
it) for each follow camera. It is a **leash camera**: the camera does not sit at a fixed spot behind the player but
stays where it is until the player drags it, and a few separate rules turn it. The steps below are **in the order the
code runs them** (`0x0012ae58`-`0x0012d3a0`); confirmed (code) at the cited addresses unless marked. Angles are given
in degrees, the code holds radians. "One target" means one human in the camera's target list (`+0x444` = 1), the
single-player case.

1. **Targets and states.** The target list is gathered (`0x001282a0`); with one target the update reads, for the
   steps below, whether the player is idle, walking, running or sprinting (the run timer `+0x36c` grows by `dt` while
   he runs or sprints), whether he is tagging, in a fight stance, or hidden in shadow, and sets the **aiming** flag
   `+0x454` while his control function is the throw-aim one (`Human_MoveThrowAim`) with his `+0x256` set and none of
   state flags `0x1c00000000`. Two factors passed to the collision step are 2.0 and 0.25 (1.9 and 0.3 × scale with a
   target: a probe height and a sphere-radius factor, inferred).
2. **Right stick and buttons** (`0x00129050`, [below](look-controls.md#right-stick)): yaw rate `+0x358`, pitch rate
  `+0x3b8`, the
   zoom, centre and reverse buttons, and the head-look angles of step 17. Confirmed (code).
3. **Bookkeeping.** `+0x46e` = the player is in a fight stance that counts for the camera; the sprint-zoom latches
   ([Sprint zoom](#sprint-zoom)); and the **saved pitch** `+0x3d4` (kept by [ReactToLedge](#ledge)) is put back into the
   target pitch `+0x3b4` once the player no longer has state flag `0x1800000000` (falling or a long fall,
   [Characters](../characters.md)): after he lands.
4. **Previous look-at** (`+0x180` copied to `+0x270`), then the band eases toward a wanted near edge (`0x0012aae0`,
   the [sprint zoom](#sprint-zoom)) and the stick hold timer `+0x368` counts down by `dt`. Confirmed (code).
5. **Look-at point** (`+0x180`, `0x00127d88`, called with 0 from the update at `0x0012b740`): the target's position
   plus the offset (runtime: feet + 1.4 m). The offset itself eases toward its wanted value by 15% per update
   (`+0x200` toward `+0x210`). Unless the target is jumping or falling (`0x00227f90`), the look-at point's move this
   update is **limited by its length** `d` (in 3D, from the previous look-at point `+0x270`): above 0.8 m it moves
   20% of the way; from 0.4 to 0.8 m it moves `1 − 2 × (d − 0.4)` of the way (100% at 0.4 m, 20% at 0.8 m); below
   0.4 m all of it. So a climb's rise of 1.3-2.6 m is followed at 20% an update until 0.8 m is left, then in two more
   updates, and a jump is followed directly. **While the player is hidden in shadow** (record flag `0x200000`, tested
   by `0x00228168`, set by `Brain_SetHiddenInShadow` `0x0028ee88` → `0x0022ff88`) and not running or sprinting, with
   one target, the look-at height is instead set straight to feet + 1.75 − 0.1 = **1.65 m** (no ease;
   `0x0012b3fc`-`0x0012b7e0`). While aiming the look-at point is the player's aim origin (player record `+0x5750`).
   `level99` has no hiding spot on its path (inferred).
6. **Field of view** eases toward `+0x394` at `+0x39c` degrees per second, at most 7.5, or over the timed move's
   remaining time `+0x40c` when one runs (then `|fov − wanted| / +0x40c` per second).
7. **Reset when lost** (only with one player camera, `0x0050b19c` = 1): when the held position `+0x260` of the last
   update is more than **1.3 × the maximum distance** `+0x304` from the look-at point **across the ground** (x and y
   only; the squares are compared with 1.69 × max², and only with one target), or the camera (its position at
   `+0x10`) is more than **30 m** above or below it, `0x00124d00` puts the camera behind the player as `CameraReset`
   does, with its pitch and field of view step when the target is a player (its byte `+0x1b0` is not −1).
   Confirmed (code) at `0x0012b918`-`0x0012b9c8` and `0x0012aff0`-`0x0012b014`.
8. **Look-behind** (`0x001298c0`, [below](look-controls.md#reverse)) while the reverse button is held, and on the
  update it is let go.
   Confirmed (code).
9. **Framing rules**, only while the stick has been still for 0.334 s (`+0x368` = 0) and the view is not pulled out
   behind (`+0x455` = 0), or while a hand-over is queued (`+0x47c`): the [combat camera](#combat-camera) (player 1
   with the lock-on button, `0x0012e9a8`), else [keep in view](#combat-camera) of the fight target (0.4) or of a
   secondary target (`+0x320`, 0.25); a queued **heading from `+0x43c`** (`+0x462`: the heading target `+0x350` =
   `+0x43c` with a 0.1 s timer, and the band moved to the default distance when it was at the minimum); then
   **auto-follow** (`0x00129c78`, [Heading](#heading)) when nothing above framed the player. Confirmed (code).
10. **Leash** to the distance band `+0x32c`-`+0x330`: when the distance from the wanted position to the look-at point
    leaves the band, the wanted position is moved along that line back to the nearer edge. `level99`'s band is
    3.0-3.5 m at checkpoint 1 and 4.8-5.3 m in the street.
11. **Pitch toward its target** `+0x3b4`, skipped in the height hold: the wanted position's pitch about the look-at
    point is measured (`0x0012d908`, from the unit direction look-at point to wanted position, negative below) and
    `0x0012d4e8` turns the wanted position about the look-at point by the **whole** difference to the target, the
    new pitch kept within [`+0x3b0`, `+0x3ac`] (the call at `0x0012beac`). So the pitch is put back every update, not
    eased: a leash pull along a steep line (a [long fall](#long-fall)) does not carry the wanted position up. See
    [Pitch](#pitch). Confirmed (code).
12. **Height hold** `+0x453` ([below](#height-hold)): the probe `0x0012a7d8` enters or leaves it; while it is set the
    wanted position's height moves 30% (close zoom), 40.5% (default) or 48% (far) of the way per update toward the
    look-at height plus the height it held (`+0x378` + `+0x324`), times `+0x398` (1, or 0.25 after `0x00125888`).
    `0x00125588` reads the zoom level from the zoom step `+0x400` (with one player camera: 2 far when `+0x400` ≤ the
    minimum, 1 default when `+0x400` > the default, else 0 close). Confirmed (code) at `0x0012c0c0`-`0x0012c14c`.
13. **Aiming** ([below](#aiming)): while `+0x454` is set, `0x00128678` sets the heading target, the pitch limits and a
    pitch override every update. On the first update after aiming ends the wanted near edge `+0x34c` goes back to the
    distance saved in `+0x3cc`, the lower pitch limit to max(−20°, an angle of (1 − offset z) / maximum distance,
    `0x004b8d30`, inferred an arcsine), the target pitch into the limits, and the timer `+0x40c` to at least 0.35 s.
14. **Placed heading** `+0x3f0` (inferred: a scene's hand-back): with one target, no stick turn this update and
    `+0x3f0` set (with `+0x465`, only while the player runs), `0x001250a8` puts the camera at that heading round the
    look-at point at the far edge of the band, every update.
15. **Heading target** `+0x350` (the centre button, [ReactToLedge](#ledge), aiming, `TurnToMatrix`, `+0x43c`): when
    set, the camera yaws about the look-at point by the angle `a` between its horizontal forward and the target
    heading, or by `a × dt / +0x40c` when the timer runs and that is smaller, toward it; the target (and `+0x43c`) is
    cleared once `a` is within **0.1°** (`0x3ae4c389` is 0.1° in radians, passed to the cosine). **Without a heading
    target** the right stick's yaw rate turns the camera by `+0x358 × dt` (`0x0012d3b8`) and the rate is cleared. So
    a heading target overrides the right stick. Confirmed (code).
16. **Pitch** ([Pitch](#pitch)): the sprint zoom (`0x00128cf0`) when its latch is due; then, **with a stick pitch
    rate**, the target pitch moves by `+0x3b8 × dt` within the limits (with the slope flag `+0x460` = 1, also kept on
    the side of `+0x3a4` that its sign gives) and the camera pitches by `rate × dt`; **without one**, a **pitch
    override** `+0x354` (when above −FLT_MAX) is reached: with the timer `+0x40c` = 0 at once (when not aiming) or at
    **10°/s** (when aiming, cleared within 0.001°), with the timer by `(override − pitch) × dt / +0x40c`; the target
    pitch follows the camera's pitch meanwhile. With neither, outside the height hold, the pitch returns to its
    target at most **85°/s** (`1.4835` rad/s). Confirmed (code).
17. **Head look** (`0x00128b20`, [below](look-controls.md#head-look)) while the stick gave a head angle, with one
  target, the player
    idle and not in a fight stance. Confirmed (code).
18. **Position lag**, only when there is no heading target and `+0x46b` (set by `PlaceBehind`) is clear, and not
    while the reverse button is held unless the view is pulled out: `wanted = held + (wanted − held) × +0x388`, with
    `held` the position `+0x260` of the last update. `+0x388` is 0.22 normally (0.27 after a blocked collision,
    chosen by the [collision step](collision.md#collision) of the previous update), **0.8 while aiming**, and **0.05**
    while the
    player is in a task of type 11 (inferred: a grapple) with no other input. When the move is under 10⁻⁵ m but the
    stick pitches the camera, only the height moves, by max(`+0x388`, 0.35).
19. **Hard band**: the wanted position is then kept between `+0x33c` and `+0x340` from the look-at point: the leash
    band widened by max(5% of the near edge, 0.2 m) and max(6% of the far edge, 0.35 m); when they shrink they ease
    back by 1% per update. In a task of type 11 the band is the leash band × 0.85 and × 1.2; on the first update the
    player has state flag `0x1000000000` (a long fall) the far edge becomes min(2 × far, 1.1 × the maximum
    distance) and the near edge max(itself, 0.85 × near). The result is the **held position** `+0x260`.
20. **Tilt over camera obstacles** (`0x0012ec50`), then **world collision** (`0x00130990`,
  [below](collision.md#collision)), which
    also moves the [aim point](#aim-point) `+0x240`; the camera is placed (vtable slot `+0x1bc`) at the position it
    returns, that position and the orientation are copied to `+0x1e0` / `+0x1f0` (the camera of the last update,
    which the next update's collision step reads), the shake is applied (`0x00121298`), and the camera is turned to
    face the **aim point** `+0x240` with `Mat_LookAt` (`0x00337028`), not the look-at point. `Mat_LookAt` fails only
    when the eye and the aim point are within 0.005 m; the position is then put back and the orientation kept. There
    is no line-of-sight test here. Then the roll eases (`0x0012ac58`) and the view frustum is rebuilt. Confirmed (code).
21. **Timers**: `+0x40c` counts down by `dt`; at 0 the timed move ends (`+0x475` and `+0x43c` cleared) and a queued
    **hand-over** runs: when `+0x47c` holds a camera, `Camera_MakeActive` (`0x0011ee08`) makes it current for this
    camera's player, blending over `+0x3c0` seconds, and both are cleared (`0x0012d07c`-`0x0012d0c4`, confirmed
    (code); see [Mini-game and mugging cameras](special.md#mini-mug)). `+0x410` and `+0x414` count down. A fade timer
    `+0x424` sets the alpha byte `+0x1b4` of the two objects at `0x005fdeb8` and `0x005fdebc` to
    `min(10 × t, 1) × 225` (inferred: the player models fading when the camera is close). `+0x46b` and `+0x469` are
    cleared.

### The aim point {#aim-point}

The view does not face the look-at point: it faces an **aim point** `+0x240` that leads the player a little along his
facing and rises and falls with his body, eased twice. The collision step moves it each update (`0x00130990`, at
`0x00133320`-`0x00134290`); confirmed (code) there and historical observation
([checking notes](#upstream-earlier-observations)) through the `fence_climb` scenario (below).
With one target that is a player (with any other target, only a 1.0 m sphere push of the old aim point, when switch
`0x0050b1c0` is 0):

1. **Lead.** `+0x2e0`, a copy of the target's rotation, is slerped 35% of the way to it each update (snapped while the
   target does not count for the cameras, while the camera aims, `+0x454`, or while the target plays clip 1). Its
   forward `f`, the view's direction `v` (from the camera of the last update `+0x1e0` to the look-at point) and `L =
   max(0.75, 0.15 × band near edge)`: `L` is scaled by allowed ÷ wanted distance when the world holds the camera in, and
   by 0.4 while the target has state flag `0x40` or `0x80` (mask `0xc0`: he is grabbing someone from the front or the
   rear, [Combat](../combat/index.md#state-flags)). A world ray (`WorldManager_RayCast` `0x0040db88`, at `0x00133604`)
   from the look-at point along `f`, `L − 0.5 × L × |f · v|` long, gives the lead point `P` = look-at + `f` × (the free
   length − 0.01). So running away from the camera the aim point is about 0.36 m ahead of the look-at point (0.375 −
   0.01 at the default band; runtime 0.363 standing, 0.365 running). Confirmed (code); the runtime distances
   historical observation ([checking notes](#upstream-earlier-observations)).
2. **Height.** With `d` = the human's body point height `+0x4e8` − 1.0 (the [body point](#body-point) `+0x4e0` is the
   hips' position in the body's frame, runtime 1.035 m standing, 0.93-1.0 running, 2.37 at the top of a fence vault,
   0.62 landing). **While the target runs or sprints** (below) this step is replaced. Otherwise: nothing (`P` stays at
   the look-at height) while `|d|` ≤ 0.27, unless the human has any of the state flags `0xe2c00`: `0x400` tackling (on
   top of the victim), `0x800` being tackled, `0x2000` (not named; paired attacks read it), `0x20000` arrested,
   `0x40000` knocked out, `0x80000` knocked down ([Combat](../combat/index.md#state-flags),
   [Characters](../characters.md#the-record), `Human_EndDownState` `0x0025e250`), so on the ground or in a tackle the
   height follows the body with no dead band. Otherwise `d` is scaled by `max(0, (0.27 − |d|) × 0.9 / 0.27 + 0.1)` when
   `|d|` ≤ 0.27, by `min(1, (|d| − 0.27) × 0.9 / 0.27 + 0.1)` up to 0.54, and kept beyond; a negative `d` is lowered by
   `(offset z − 1) × 0.6` more when the look-at offset is above 1.25 m (0.24 in `level99`). The height goal `zt` =
   look-at height + `d`; the new height is the old aim point's height moved 35% toward `zt`, but where that does not lie
   between the look-at height and `zt` it is `zt` itself; the result is stored as the look-at point's height `+0x188`
   (`0x001339c0`). (With `+0x465`, the target takeover of `0x00126558`, no dead band and 75%.) Confirmed (code); the
   body point's heights historical observation ([checking notes](#upstream-earlier-observations)).

   **Running and sprinting** (`0x00133674`-`0x001337d0`), confirmed (code): `+0x450` is a switch the constructor sets
   to 1 (`0x00124940`, its only writer), so it is always on. The update (`0x0012b310`-`0x0012b3f4`) passes the
   collision step two flags of this update: **sprinting** (gait `+0x1a8` 5 with no blocking record flags,
   `0x00223a98`) and **running or sprinting** (that, or gait 4, `0x00223a60`); both are 0 under the takeover `+0x465`.
   While either is set the height step above is skipped and the old aim point is moved instead:
   - **running** (gait 4): by 0.22 × (`+0x4e0` − (0, 0, 1)), the body point's offset as it is, not turned by the
     human's heading; the sway `+0x3f8` is set to 0. So the aim point bobs with the hips: 0.22 × (0.93-1.0 − 1), a
     little below its old height each update;
   - **sprinting**: a sideways **sway**. The sway `+0x3f8` moves by `+0x3f4` × 0.028 × a random number in 0.125-0.25
     (`0x00335420`), 0.0035-0.007 m an update; past ±0.028 it is held at `+0x3f4` × 0.028 and the direction `+0x3f4`
     (1.0 at construction) flips. The aim point moves sideways by the sway along the view's horizontal side axis (the
     unit cross product made at `0x00130b68`) and up by (2 × |sway| − 0.028) × 0.6, so between −0.017 and +0.017 m.
     Confirmed (code).
3. **Push.** Unless the camera aims, `P` is pushed out of the world by a sphere of radius 1.0 (0.95 with two targets ×
   the target's scale; `CollisionMesh_SpherePush`).
4. **Lag** `+0x418`, worked out afresh each update from 0.06: × (1 + (1 − `c`) × 0.5) where `c` = `|unit(velocity) ·
   camera forward|`, both in **3D** (the velocity's z too; the forward is the y axis of the orientation `+0x20`,
   `Quat_AxisY` `0x003363b0` called at `0x00133434`, the view as the last update left it, toward the aim point, not
   the look-at point; confirmed (code)), once the target is faster than his walk speed (else `c` =
   1); + `s` × it × ((1 − `c`) × 0.5625 + 0.1875) with `s` = (horizontal speed − walk speed) ÷ sprint speed (≥ 0; `s`
   is scaled by (0.75 − `+0x38c`) × 4/3 while the timer `+0x38c` runs down, set to 0.375 s after a blocked view clears
   and 0.75 s after a sphere push ends); × 2.5 when the target's **3D** speed is above walk + 18% of (run − walk) and
   either the new aim point would end farther from `P` than the look-at point is, or the stick is not his (`+0x1b` of
   the per-player record 0) ([below](#long-fall)); × 1.25 when slower and the latches `+0x45b`/`+0x45d` are set
   on a slope (`+0x460`). Then `+0x418` = last update's value + 3% of the difference; 0.6 while the human has state flag
   `0x1000000000` (a long fall); at least 0.8 when the main ray is blocked while aiming; at least 0.3 during a takeover;
   halved while the target does not count or the game mode is not 1. Runtime: 0.061 standing, 0.14-0.16 running.
5. **Move**: `+0x240` = old + (`P` − old) × `+0x418` (when the main ray is blocked by a wall whose normal is nearly
   level, by `min(1, (3.5 − the feet's distance in front of it) × 0.4) × 0.3` instead); then, if that left it farther
   from the look-at point than `P` is, it is pulled back in plan to `P`'s distance.

**Steps 4 and 5 in detail** (confirmed (code), `0x00133b40`-`0x00134180`; `old` is the aim point `+0x240` before
this update, `P` the pushed lead point):

- **The tests in player terms.** "The target does not count for the cameras" is `Camera_HumanCounts`
  (`0x00123500`) returning 0: he is down or dead (`Human_IsDownOrDead` `0x00227e60`: dying, dead, knocked out or
  tackled, state `0x180050000`; not knocked down), or he is **cuffed** (`Human_IsCuffed` `0x00223b70`, state
  `0x20000`) and cannot free himself (he still counts with upgrade (6, 15) unlocked and a key, item 6, carried,
  [Crimes](../crimes.md)). "The game mode is not 1" is `GameModeStack_TopId` (`0x0015e748`) ≠ 1: another mode is on top
  of gameplay ([Boot](../boot.md)). **The long fall** (state flag `0x1000000000`, read only for a player, human `+0x1b0`
  ≠ −1) is set by `Human_BuildFallClip` (`0x0025d020`) when a fall (anim state 26) starts while he is knocked down,
  knocked out or cuffed (state `0xe0000`), dying or dead, or already falling (held flag `0x400000`): he goes over an
  edge limp, as when knocked or thrown off a ledge, and plays the long-fall cycles 422 / 425. Confirmed (code).
- **The timer `+0x38c`.** Set to **0.375 s** when the main ray hit a face this update (its hit plane is not zero) but
  the hit does not hold the camera (`+0x45b` clear) while the sticky copy `+0x45c` is still set; set to **0.75 s**
  when the main ray hit a face this update and the step-3 push of `P` (the 1.0 m `CollisionMesh_SpherePush`, which
  returns whether it touched a wall) touched nothing this update but touched (or did not run, while aiming) on the
  last (`+0x45a`, this update's result kept at the end of the step). With any other target the push in question is
  the 1.0 m push of the old aim point. It **counts down by `dt` only while `s` > 0** (the target faster than his
  walk), so it waits while he stands.
- **The overshoot test** (the ×2.5), confirmed (code) at `0x00133dcc`-`0x00133ec8` and historical observation
  ([checking notes](#upstream-earlier-observations)) by the
  `fall_high` trace ([Long falls](#long-fall)): the speed it compares is the **3D** speed (`sp+0x35c`, `vmul.xyz` at
  `0x00130f90`; the `s` above uses the horizontal one, `sp+0x358`, `vmul.xy` at `0x00130f68`). With `new` = `old` +
  (`P` − `old`) × the lag being built, it compares squared 3D distances: (|`P` − look-at|² > 10⁻⁵ or the slope latch
  `+0x460`) and |`P` − look-at|² < |`P` − `new`|², so the lag is × 2.5 whenever this update's aim point would still be
  farther from `P` than the look-at point is (about 0.36 m, the lead); only when that fails is the per-player record's
  `+0x1b` read (0: × 2.5). (This page used to read the first distance as |`P` − `old`|, which made the test never hold;
  the subtrahend at `0x00133df4` is the pointer at `sp+0x3b4`, the look-at point `+0x180`.) The ×1.25 needs the target
  at or below walk + 18% of (run − walk), the latch bytes `+0x45b` or `+0x45d` set (`+0x458` & `0xff00ff000000`) and
  the slope latch `+0x460`.
- **The blocked-wall share** replaces the lag in step 5 when the main ray's hit holds the camera (`+0x45b`), the
  camera does not aim, and the hit face is steep: |normal `z`| < **0.6** (`0x005d91cc`, steeper than 53.13°). Its
  "feet's distance in front" is the target's transform position (his feet) against the **hit face's plane**:
  `n · feet − d`, with `n` and `d` the plane of the main ray's hit (the ray from the look-at point toward the
  camera). The share is `min(1, (3.5 − that distance) × 0.4) × 0.3`: 0.3 within 1 m of the wall, falling to 0 at
  3.5 m. (Aiming, the lag is at least 0.8 instead.) Confirmed (code).

The camera's yaw and pitch therefore lag the look-at point: the trace's `cam_pitch` (camera to look-at) is not the
pitch of the view; the view's pitch is camera to `+0x240`.

#### The body point {#body-point}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

A human's `+0x4e0` (`+0x4e8` its height) is **pose bone 2** of his skeleton (HAnim node 0, the hips: the parent of the
spine and of both legs, [Animation](../formats/animation.md)), its position in the model's frame: z up, the model facing
`+y`, relative to the feet and **not turned by the human's heading**, times the human's scale (`+0x65c`, 0.97 for
Rembrandt). Confirmed (code): each update `Human_StateUpdate` (`0x0023fea8`, vtable `+0x13c`, at `0x002400d4`) calls the
human's bone transform (vtable `+0xbc`, `Human_GetBoneTransform` `0x0023bde8`) with bone 2 and the destination
`+0x4e0` (the 32-byte entry: position at `+0x4e0`, rotation at `+0x4f0`); in the same call it fetches bone 0 (the root,
for the root motion), 3 into `+0x540`, 5 into `+0x520` and 6 into `+0x500` (the head look's origin). The bone comes from
the per-update bone cache ([Combat](../combat-grabs.md#grab-posing)), built from the playing pose and already composed
through
the parent table, so it is the animated hips, not the clip's root track. With no model instance (`+0xd8` 0) it is not
written and keeps its old value. Its position is the pelvis channel's: the idle's pelvis height 1.067 × 0.97 = 1.035,
the runtime value standing (inferred from that match).

**When**: in the human's state update, after the state function and the head look and before the root motion and the
move. The cameras update at the end of the same 30 Hz step ([Coney](../../engine/camera.md#coneys-implementation)), so
the
camera reads this
update's value; whether the clip time has already been advanced in this step when the cache is built is not traced.

### Long falls {#long-fall}

User checklist item 48 asked what the follow camera does when the player falls from a great height. **There is no
long-fall camera for a plain fall**: the long-fall state flag `0x1000000000` (aim lag 0.6, update step 19's wider hard
band) is set only when a fall starts limp or while the drop's held flag `0x400000` is still set, and a 40 m fall
does not set it. What keeps the view on the player is the [aim point](#aim-point)'s lag, which climbs from 0.06 toward
about 0.21 because a fall is fast in 3D.

Earlier observation: see the [recorded values and method](#upstream-earlier-observations).

Earlier observation: see the [recorded values and method](#upstream-earlier-observations).

Earlier observation: see the [recorded values and method](#upstream-earlier-observations).

| Rise | Updates falling | Top speed | Highest lag | Aim point above look-at | Camera pitch | View pitch | Look-at off the view's centre |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 20 m | 47 | 24.6 m/s | 0.153 | 4.2 m | 41.9° | 15.5° | 57.5° |
| 40 m | 67 | 35.0 m/s | 0.189 | 4.8 m | 51.5° | 14.0° | 65.6° |
| 70 m | 89 | 46.5 m/s | 0.193 | 6.2 m | 59.1° | 36.9° | 96.1° |
| 120 m | 119 | 50.0 m/s | 0.193 | 7.6 m | 61.8° | 55.2° | 117.1° |

The columns are each the largest value of the fall: the camera's pitch above the look-at point, the view's pitch
(camera to aim point, positive up) and the angle between the view and the look-at point. The fall caps at 50 m/s.
So **the original's camera rises too**: it stays 4.5-5.2 m from the look-at point and climbs above it to about 62°,
and in a fall of 20 m or more the player leaves the frame below (the look-at point is more than 57° off the view's
centre); from about 70 m the view also tilts up, to 55° at 120 m. What it never does is face straight up: at 40 m the
view stays within 15° of level and shows the horizon and the buildings (a PCSX2 frame at 56 updates, 14 m above the
street, checked by eye). After the landing the lag eases back to 0.061 by about 140 updates.

Earlier observation: see the [recorded values and method](#upstream-earlier-observations).

### Climbs {#climbs}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

The follow camera has no climbing mode: the climb start (`Climb_TryStart` `0x002826f0`,
[Characters](../characters.md#climb)) calls no camera function (confirmed (code) for its direct callees), and the only
traversal hooks are the drop ([Ledges](#ledge)), the landing shake and the long-fall far edge (update step 19). There
are no ladders or drainpipes: the executable holds no such names, and climbing is the fence and wall climb (inferred
from the string search and the climb code). A climb is followed by the ordinary rules: the look-at point's distance
limit (step 5) and the aim point.

Does the original's camera jump when the player climbs? **No**, the climb itself moves it smoothly; one level spot
pulls it in once. historical observation ([checking notes](#follow-inline-observations)), PCSX2 2.9.94, slot 7
(`level99` checkpoint 3, the street fence), scenario
`fence_climb` (left stick 100 % straight ahead, triangle every 2 updates from 4.6 m before the fence), every update:

| Updates | Body | Camera |
| --- | --- | --- |
| run (410) | 7.80 m/s, body point 0.93-1.0 m | moves 0.260 m per update with the player, 5.65 m from the look-at point (the hard band's far edge), pitch 10.8° |
| 440, the 2 snap updates | feet 1.11 m, then 1.17 m forward in one update each | look-at point moves 0.22 m, then 0.41 m (20% of the gap, [step 5](#update)); the camera 0.22 m, then 0.41 m: **no jump** |
| 440, to the fence | the clip carries the body 0.18 m per update | camera 0.37, 0.33, 0.30, … 0.22 m per update as the look-at point catches up; its distance stays 5.65 m |
| 440's last update | the body point rises to 1.49 m | the camera moves **0.91 m in one update** (5.65 → 4.93 m from the look-at point) |
| 441, over the fence | body point 1.59 → 2.37 → 1.79 m; feet stay at ground height | aim point height 1.67 → 2.23 m (eased: 0.03-0.05 m per update); camera moves 0.07-0.11 m per update and creeps back out (4.93 → 5.15 m) |
| 442, landing | body point 2.37 → 0.62 → 0.98 m | aim point height eases back down to 1.60 m over 10 updates |

The one 0.91 m move is the street's **disabled panel** ([In the street](#street)): the camera, held 5.65 m behind, is
then 0.27 m behind the 2.65 m panel at `y` = 31.17 and comes to rest 0.25 m in front of it; from the next update
the main ray hits the panel (`+0x290` read its plane, `y` = 31.17), because the camera of the last update is now in
front of it ([World collision](collision.md#collision)), and the camera stays held 4.9-5.4 m away, creeping out at
about 3% of the
gap per update with the 0.27 lag. It comes from the level's data at that spot: in the same run without the climb
(stopping against the fence) the camera crossed the panel at 0.26 m per update with no pull, then moved 0.34 m
once, pulled in to 5.47 m as the player stopped. Which test pulled it in on 440's last update (the main ray was
clear on that update, `+0x45b` 0, while the blocked-view latch `+0x45d` was set) is not traced. What does move with
a climb is the **aim point**, which rises 0.6 m with the body during the vault and comes
back down over a third of a second, and the look-at point's 20% follow of the snap. A Coney camera that jumps on
every climb is missing these two eases, or pulls in on faces the original ignores.

### Heading {#heading}

The yaw rotation is done by `0x0012d688(angle)`, about the look-at point. Four rules call it, confirmed (code):

- **Auto-follow** (`0x0012a400`), the default: let *a* be the angle between the camera's horizontal forward and the
  player's facing. Nothing happens below 22.5° or above 157.5°, so running towards the camera does not spin it. The
  rate is `(a − 22.5°) × 2.667` per second below 45° (0 to 60°/s), 60°/s from 45° to 135°, and above 135° it falls
  from 120°/s back to 60°/s at 157.5°. Each update turns by `min(a, rate × dt)` toward the facing. Confirmed (code).
- **Auto-centre option** (`0x00129f88`, used instead when the per-pad option bytes `0x0050b240` and `0x0050b248` are
  both set; they are 1 in the `level99` save used at runtime and 1 by default,
  [Feel](../feel.md#details-behind-the-table)): nothing below 22.5°; from 22.5° to 90° the rate is `(a − 45°) × 2.444 +
  45°` per second (negative below 26.6°, so the step is then a small turn the other way; 45°/s at 45°, 155°/s at 90°);
  200°/s (`3.4907` rad/s) from 90° to 100°; from 100° to 157.5° it falls linearly (`(157.5° − a) × 2.435 + 60°`) from
  200°/s to 60°/s, unless the call's fourth argument is set, which keeps 200°/s up to 157.5°. That argument is the
  update's local at `sp + 0x1d4`, set at `0x0012b2ac`: 1 when there is one target, he is in a fight stance
  (`Human_IsInFightStance` `0x00228340`, state flags `0x3`) and he still counts for the cameras (`0x00123500`), else 0;
  confirmed (code). So a player squaring up gets the fast swing over the whole range. Beyond 157.5° the rule turns only
  when the target's record has state flag 4 (`0x002265f0(target, 4)`, "has a target" in
  [Combat](../combat/index.md#state-flags), inferred there; the same flag can stand in for "moving").
  `a` is the angle between the target's facing (its rotation in the transform table) and the camera's own horizontal
  forward: vtable slot `+0x224` returns the orientation `+0x20`, whose y axis (`Quat_AxisY` `0x003363b0`), flattened and
  normalised, is the view as placed at the end of the previous update, toward the [aim point](#aim-point) `+0x240`, not
  the look-at point. The facing's y axis is used flattened but not normalised. The angle is `π/2 − atan(c / √(1 − c²))`
  of the cosine `c`, with an atan (`0x003356a0`, a range-reduced rational form) good to about 10⁻⁵° (inferred: one of
  its constants sits in memory Ghidra cannot read). Confirmed (code); historical observation
  ([checking notes](#upstream-earlier-observations)) in [In the street](#street).
  Each update turns by `min(a, rate × dt)` (`dt` = `+0x1a0`) toward the facing. The cosine of each threshold is computed
  with `0x004b8a70` (cosine, inferred from the thresholds' use).
- **When either rule runs** (`0x0012ae58`, the call at `0x0012bd80`): one target (`+0x444` = 1), the target passes
  `0x00123500` (not in states `0x180050000` of record word `+0x00`), no camera flags in `+0x460 & 0xffff0000`, nothing
  watched (`+0x320` = −1), no yaw from earlier steps this update, and no camera input in the last 0.334 s (`+0x368` =
  0). Inside `0x00129c78` (arguments read at the call, `0x0012bd30`-`0x0012bd84`), confirmed (code): the default rule
  needs "running": the target's gait `+0x1a8` is 4 or 5 with no blocking record flags (`0x00223a60`, `0x00223a98`). The
  auto-centre rule needs "**moving**": gait **2, 4 or 5** (walk, run, sprint; `0x00223a40` adds the walk), or state flag
  4 of the record; **not 0, 1 or 3**, so not while the body moves slower than a walk or at a jog's speed. It also needs
  `+0x474` = 1 (cleared on an update whose stick vector in the per-player record, `+0x00` / `+0x04`, points more than
  157.5° from +y, and set again at the start of any update whose gait is not 0, so it blocks only the updates with the
  stick pulled back that far; whether that vector is the camera-turned stick is not traced), `+0x455` = 0, no
  right-stick input, the top animation task's clip without descriptor flag `0x8000` (`0x00175be8`), none of the human
  flags `0x18003ff0`, and the collision bytes `+0x45b` and `+0x45d` clear: the collision step sets `+0x45b` when its
  main ray from the look-at point is blocked ([World collision](collision.md#collision)), so **auto-follow stops on
  the update after
  one in which the view was blocked**. `+0x45d` is a latch: it is set at `0x00132fa4` when the step's local "view
  blocked" (`sp + 0x364`, set at `0x001327a0` from the blocked local `sp + 0x368`, which `0x0013207c`, `0x001321e4` and
  the swing-away give-up `0x001325a4` set) is true, and `+0x45c` copies `+0x45b` the same way (`0x00132fb4`). Both are
  cleared by `0x00124778` (`0x00124980`) and at the end of the collision step (`0x00132fd0`-`0x0013314c`) on an update
  whose view is not blocked (`sp + 0x364` = 0) when either the camera is no longer held in (its wanted distance from the
  look-at point, `sp + 0xc0` to `sp + 0x20`, is within 10⁻⁵ of the distance the step allows, `sp + 0x348`), or the
  allowed distance plus 3% of that gap reaches the wanted distance the step started from (`sp + 0x34c`), or the zoom
  button was pressed this update (the stick step's flag, the step's stack argument `0x4c8`). Otherwise the wanted
  position is moved out to the allowed distance plus 3% of the gap and the latches stay. Here "allowed distance" is the
  distance from the last look-at point `+0x270` to `+0x2b0`, the point the step returned on the last update, and the 3%
  step is capped at the distance the step started from. While the view is blocked (`+0x45b` or the local set) the wanted
  position is instead drawn to the allowed distance plus **4%** of the gap, and the hold timer `+0x414` starts at 1.5 s
  if it is not running; on the clearing path it starts at 1.5 s (3 s while `+0x460` is set, 0.1 s with `+0x463` and a
  timed move). This is the final form of the rule (`0x00132f5c`-`0x00133250`, read in full). The **recovered distance**
  `+0x380` moves 10% of the way to the allowed distance at the end of every step (`0x00134a18`). Nothing else reads it:
  the constructor zeroes it and the activation sets it to the band's near edge, so it can be left out. Confirmed (code);
  the meaning of the locals is inferred. While the player runs, the leash keeps the wanted position at the band's far
  edge beyond what the walls allow, so the latches hold until he stops (runtime). So **one blocked update keeps
  auto-follow off until the player stops** ([Runtime checks](#runtime-checks)). historical observation
  ([checking notes](#upstream-earlier-observations)), slot 1, stick 100 %
  sideways: the rule turned 110-129°/s every update at gait 4; with `+0x45b` written to 1 before each of 21 updates it
  turned 0 on each of them, and 129°/s again on the next. The gaits explain what [In the street](#street) saw: no turn
  in the walk and run start clips (gait 0-1 while the walk start moves at 0.76 m/s, 3 in the run start's middle) or the
  landing clip (4.23 m/s, gait 3), and a turn during the run start's first five updates and the run stop's slower
  updates (gait 2).
- **Keep the target in view** (`0x0012e170(factor, range)`, called with 0.25 and `+0x3fc` for the human
  `CamSetSecondary` gives, `+0x320`, and with 0.4 in one fight case): when the target's direction from the camera is
  more than `fov × factor` from the view's, the camera yaws 35% of the excess per update, at most 270°/s
  (`4.712` rad/s), toward it. With a range above 0 it acts only while the target is within the range and a ray
  (mask `0x200`) from the look-at point reaches it; `level99` passes 0, so no range and no ray. It runs only when
  nothing else turned the camera this update, `+0x463` and `+0x454` are clear, and it replaces auto-follow, which
  needs `+0x320` to be NilHandle. Confirmed (code).
- **The right stick** (rate `+0x358`) and the **heading target** (`+0x350`, step 15 above).

### Pitch {#pitch}

- The target pitch `+0x3b4` is 13° in `level99` (`CfgFollowCamera`).
- The **lower limit** `+0x3b0 = atan((1 − offset.z) / far)`, at least −20° (`0x0012d7a8`, run when the distance band
  moves); the target pitch is raised to it. Confirmed (code).
- The **upper limit** `+0x3ac` follows the zoom step (`0x001254f0`): 50° at the minimum distance, 40° above the
  default, and 30° at the default when `0x0050b19c` is 1 (50° otherwise). `0x0050b19c` is not an option: it is the
  **number of player cameras**, counted by `0x00122ed0` (written at `0x00123248`) over the camera slots from
  `0x005d9148` up to the player count (`*(0x0051489c) + 0x224`), so 1 means one player and the 30° limit is the
  single-player case. Beside it, `0x0050b198` is the number of split-screen views in use and `0x0050b1a8` /
  `0x0050b1aa` the views' grid (from `0x0050b1a0`). Confirmed (code). The same function sets the
  zoom distance `+0x400` to the minimum, maximum or default.
- Without input the pitch is driven back to the target at most 85°/s (step 16).

### Sprint zoom {#sprint-zoom}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

While the player sprints, the camera pulls in to the minimum distance and lowers its target pitch to 7°, and both go
back 250 ms after the sprint ends. Confirmed (code) at the cited addresses and historical observation ([checking
notes](#follow-inline-observations)) in the street save
(slot 1, stick 100 % and L2, every field below read every update; PCSX2 2.9.94):

1. **Detecting the sprint** (in the update, `0x0012b310`-`0x0012b3f4`): the target's stored gait `+0x1a8` is 5
   with no blocking record flags (`0x00223a98`) → "sprinting" this update (a local, `sp + 0x1e8`). On the first such
   update of a sprint (`+0x36c`, the run time, still 0) the switch `+0x468` is copied to the arm byte `+0x467`.
2. **Latching** (in the update, between the right-stick step and the band's ease, `0x0012b504`-`0x0012b5d8`): while
   sprinting and armed, when the player's **brain** (`0x0021d408`) has no enemies (byte `+0x152`, the one `BrHasEnemies`
   reads) or the nearest of its up to 16 enemies (handles at `+0x164`, `0x004db5b0` with the squared distance of
   `0x00336ce0`, FLT_MAX `0x00548ad8` for none) is closer than **12 m** (squared distance < 144), `+0x467` is cleared
   and `+0x466` set; with `+0x466` set and `+0x448` 0, `+0x448` = 1. On the first update **not** sprinting with `+0x466`
   set: `+0x466` = 0 and `+0x448` = the game time + **250 ms**. (A time left in `+0x448` by an earlier sprint, as in the
   save, also lets the function run; it then has nothing to do until the next sprint.)
3. **The zoom function** `0x00128cf0(camera, sprinting, 0)` runs every update once the game time has passed `+0x448`
   (so at once for 1, after 250 ms for a time; call at `0x0012c5c8`).
    - **Sprinting**, with no other zoom's band saved (`+0x3c8` = 0): the first time, it saves the band's near edge in
      `+0x3e0`, the zoom distance in `+0x3e8` and sets the timer `+0x40c` to **0.5 s**; with one player camera
      (`0x0050b19c` = 1, [Pitch](#pitch)) it sets the wanted near edge `+0x34c` to the **minimum distance** (`+0x300`,
      3.0) and steps the zoom distance to the default (`0x001254f0`, which also sets the upper pitch limit to 30°);
      otherwise (two players) it leaves the band and sets `+0x3e0` to the maximum − 0.5 and `+0x3e8` to the default.
      It then saves the target pitch in `+0x3e4` and moves the target pitch toward **7°** (0.122173 rad): on the
      update the timer reads 0.5 nothing; then by `|7° − pitch| / +0x40c × dt` per update, which is a straight line
      arriving as the timer runs out (without a timer, 30°/s).
    - **Not sprinting**: `+0x34c` = the saved `+0x3e0`, `+0x3e0` = 0, the zoom distance back to `+0x3e8`, `+0x40c` =
      0.5 s; the target pitch moves back to `+0x3e4` the same way while the timer runs. Once both the band and the
      pitch are within 10⁻⁵ of their goals, `+0x448` = 0 and `+0x3e4` = −FLT_MAX: the zoom is over.
4. **The band's ease** (`0x0012aae0`, every update while `+0x34c` > 0): with `d` = `+0x34c` − `+0x32c` and `T` =
   `+0x40c`, the near edge moves by `d × |d| / T × dt` (or `d × 3.5 × dt`, `d × 4.5 × dt` with `+0x448` 0, when no
   timer runs), clamped at `+0x34c`; the far edge is the near edge + 0.5; when within 10⁻⁵ the edge snaps and `+0x34c`
   = −1. The zoom step follows the band's near edge as `CamSetFollowZoom` sets it ([Script
   calls](scripted.md#script-calls)): at
   or below `min + 0.4 × (default − min)` (3.72 m) the step is the default (30° with one player camera), at or below
   `default + 0.6 × (max − default)` (5.88 m) the maximum (40°), else the minimum (50°) (`0x001254f0`). historical
   observation ([checking notes](#follow-inline-observations)): 40° until the near edge passed 3.72 m, then 30°
   (sprint and [combat camera](#combat-camera)).

So the band moves 4.8 → 4.569 (`1.8² / 0.4667 / 30` = 0.231), 4.379, 4.221, … 3.216, then 3.0 when the timer's last
float (about 1.5 × 10⁻⁸) makes the step reach the goal: 14 updates, the measured curve to 0.001 m; the target pitch
falls 6° / 14 = 0.4286° per update. At runtime `+0x34c` read 3 and `+0x40c` 0.4667 on the first update at gait 5,
`+0x466` was 1 from then until the run stop's first update, when `+0x448` became that time + 250 ms; 8 updates later
(267 ms) `+0x34c` read 4.8 and the band and pitch went back over the next 14 updates, and on the 15th `+0x448` read 0
and `+0x3e4` −FLT_MAX. The saved zoom distance was 6.6 (the maximum): the upper pitch limit read 30° in the sprint and
40° again from the 6th update of the way back.

`CamEnable(5, on)` (`Camera_EnableFeature`, `0x0011de58` → `0x00126a30`) sets the switch `+0x468`; turning it off
also clears `+0x34c`, `+0x3e4`, `+0x448`, `+0x466` and `+0x467`. The constructor sets it to 1 (it read 1 in the
save). Confirmed (code).

### Ledges {#ledge}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

When the player walks off an edge, the camera turns to show the drop. Confirmed (code):

- **Who calls it.** `Human_StartFall` (`0x0023dc58`) calls `ReactToLedge` (`0x0012da20`) with the heading of the
  player's velocity (his facing when he is not moving), −1 and time 0, on his follow camera, only when: he is a
  player (`+0x1b0` ≠ −1), not jumping (state `0x400000000`), the number of players equals the number of views
  (`0x0050b198`), he is on the camera's target list, the current camera is the follow camera (type 2), and a world
  ray straight down from his body point (vtable `+0x9c`) raised by his height (vtable `+0x5c`) and moved **0.75 m**
  along that heading, as long as the height + 1.5 m, hits nothing: there is a drop ahead.
- **`ReactToLedge(heading, −1, time, camera)`** does nothing unless the camera has a target, the target has state flag
  `0x800000000` (falling), and the stick has been still for 0.334 s (`+0x368` = 0). Then:
    1. The heading target `+0x350` = the heading. When the target does not hold flag 6 and moves at least
       `walk + 0.18 × (run − walk)` speed, that is all: the camera simply turns to face the way he is going.
    2. Otherwise (walking, or slower) it looks for the open side: two rays along ± the heading's sideways direction,
       **9.5 m** long, from the target's x, y at the height `+0x208`, and again from a point 1 m along the heading
       dropped by a 5 m ray to its hit − 0.3 m. If the room to the left is smaller than to the right the heading is
       turned by **−92°**, else by **+92°** (`0x3fcd87ad` = 92° in radians), and wrapped to 0-2π: the camera swings
       round to look along the edge from the side with more room.
    3. The target pitch is saved in `+0x3d4` (if not already saved) and set to the **upper pitch limit − 5°**, so the
       camera looks down over the edge; `+0x448` = 0, `+0x3e4` = −1.7 × 10³⁸.
    4. The timer `+0x40c` = time, **0.35 s** when 0 (always from `Human_StartFall`).
- **Afterwards**: the turn runs as any heading target (update step 15) and the saved pitch is put back when the
  player no longer has state flag `0x1800000000` (step 3), so after he lands.
- **The second argument** −1 is read only as the heading target's timer fallback: unused here (inferred).

**Landing** (`Human_Land` `0x0023e090`, confirmed (code)): when a player lands with a vertical speed below **−1.5
m/s**, his player camera's shake slot `+0x15c` is called with (human, level, 1): level **1** above −14.9 m/s (no
damage), **2** from −14.9 to −20.5 m/s (fall damage), **3** below −20.5 m/s (a fatal fall). For the follow camera
that is `0x001263e8`, the follow camera's [shake](effects.md#shake) (only for a human on its target list), so the
levels are
the shake's levels 1-3; the last argument 1 (`+0x1d7`) lets the shake move the view, not only the rumble
([Shake](effects.md#shake)).

### Height hold {#height-hold}

The height hold drops a raised camera under an overhang (a beam, an awning's edge, a bridge's underside) and keeps
it there instead of following the leash's height. Confirmed (code) in `0x0012ae58` (`0x0012bf14`-`0x0012c078`) and
`0x0012a7d8`; the player-terms reading is inferred. Below, `feet` = `+0x378` − the offset's z (`+0x208`), where
`+0x378` is the look-at height, refreshed every update while not holding and frozen while holding; `H` is the probe
height of update step 1 (2.0 m; with a target 1.9 m × his scale, changed in some animations, not traced); the near
plane is the camera's vtable `+0x5c`.

- **Probe** (update step 12): runs while the main ray did not block last update (`+0x45b` = 0) and the held position
  `+0x260` is higher than `feet` + `H` − 0.5 m (1.4 m for a normal-sized player): a ray with mask `0x200` from the
  wanted position's x, y at the height `feet` + `H` + 0.75 m + the near plane, level toward the look-at point
  (`0x00336da8`), as long as their distance in plan (`0x00336d68`). Confirmed (code).
- **Entering** (`0x0012a7d8`, called with `feet`, `feet` + `H` and a table of two heights at `0x00548ad0`): needs a
  hit more than 10⁻⁵ along the ray, on a steep face (its normal's |z| under cos 53.13° = 0.6, `0x005d91cc`), while not
  holding. The face's plane goes to `+0x220` and the hit point to `+0x230`. The same level ray is then cast lower,
  at `feet` + **2.395 m** and then `feet` + **2.148 m**: forward, and when that is clear, backward from its far end.
  The first height clear both ways is taken. If both hit, a last try at `feet` + `H`; if that hits either way there
  is no hold. Otherwise the hold starts: `+0x453` = 1 and `+0x324` = the height − the near plane − `+0x378`, so
  step 12 eases the wanted position's height to that height less the near plane. Confirmed (code).
- **In player terms**: the camera stands higher than the player's head, something steep sits between it and the
  look-at point about 0.75 m above head height, and there is clear space 2.0-2.4 m above his feet: the camera comes
  down into that space and stays there.
- **Leaving** (`0x0012c048`): when the slope flag `+0x460` is set, the probe did not run or hit nothing, or (holding)
  the wanted position is no longer in front of the stored plane (its distance in plan under 10⁻⁵): `+0x453` = 0 and
  the plane is cleared. So the hold also needs the probe ray to keep hitting something every update. Confirmed (code).
- **While holding**: update step 12 eases the height and step 11 (the pitch toward its target) and the 85°/s pitch
  return of step 16 are skipped.

### Aiming {#aiming}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

While the player aims a throw (`+0x454`, update step 1), `Cam_Follow_AimView` (`0x00128678`) runs every update,
confirmed (code):

- On the first update the wanted near edge `+0x34c` is saved in `+0x3cc` and set to **0.8 m**, so the camera comes
  in close; the upper pitch limit becomes **50°** and the lower one max(−20°, the angle of (0.075 − offset z) /
  maximum distance), and the old look-at point is copied to `+0x240` and `+0x270`.
- The look-at point is the aim origin (player record `+0x5750`) moved 0.05 m toward the aim target (`+0x5760`).
- **Heading target**: the heading from the player's body point raised by the look-at offset to the aim target,
  **−15°**, with no timer, so the camera turns there at once.
- **Pitch override**: when the view's pitch is more than `fov × 0.1875°` from the pitch toward the aim target, the
  override `+0x354` = that pitch clamped to the limits (reached at 10°/s, update step 16), and the yaw rate is raised
  by up to 3 × its value (`× (1 + 3 × min(1, (off − fov × 0.1875°) / (fov × 0.3375° − fov × 0.1875°)))`).
- The lag is 0.8 (update step 18). When aiming ends see update step 13.

### Combat camera {#combat-camera}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

While the player fights with a lock-on, the follow camera pulls in and frames the enemy. Confirmed (code) in the
update at `0x0012bb00`-`0x0012bbe8`; historical observation ([checking notes](#follow-inline-observations)) as noted.

- **When**: one target, the target is player 1, the player is pad-controlled (per-player `+0x1b`), and
  `0x00233c50` holds: the mode `0x00510228` is not 0 (1 by default; `CfgAutoCloseMode` sets it, no script calls it),
  one player, the player rides nothing (`+0xc4`), and the player has a fight target (`0x00226e60`) with record flags
  `0x8` and `0x4` both set (L1 held at a target; with mode 2, also any update in the lock-on movement state
  `0x00241b90`). historical observation ([checking notes](#follow-inline-observations)), slot 6 copy, L1 held facing
  the pedestrian: flags `0xd`, `+0x46f` 1 on the
  third update after the press, 0 on the second after the release. In `level99_combat.lua`'s sparring L1 is disabled
  (`EnableCommand(player, 6/8, 0)`) and the camera stayed off.
- **On entry** (`+0x46f` set): the band's wanted near edge is saved in `+0x3cc` (the sprint zoom's saved edge if
  one is held, else a wanted edge in progress, else the near edge) and set to **2.4 m**; unless `+0x3d0` holds a
  pitch, the **target pitch becomes 15°**. The band eases by `d × 4.5 × dt` per update (no timer, [Sprint
  zoom](#sprint-zoom) step 4), so 4.8 → 4.44, 4.134, 3.874, …, 2.4 in about 50 updates (runtime, to 0.001 m); the
  zoom step follows it (30° below 3.72 m).
- **Each update while on**: `0x0012e9a8` takes the enemy's point (its position plus half its [body point](#body-point)
  `+0x4e0`, the hips in the body's frame, not turned by his heading, so about 0.5 m up; an object's position + 0.5 m)
  and the angle at the look-at point between the camera's horizontal view and the direction to the enemy; outside
  25°-29° it yaws by `(angle − 27°) × 0.455` toward 27°; the turn is capped at 640°/s (`11.17` rad/s) when the enemy is
  more than 29° off, not when it is under 25°. So **the enemy is held 27° off the view's centre**, beside the player.
  This counts as this update's turn: auto-follow and keep-in-view do not run. Confirmed (code).
- **On exit**: the wanted near edge is set back to the saved `+0x3cc`, which is cleared; the band eases back the same
  way (runtime: 2.4 → 2.76, 3.066, …, 4.8). **The target pitch stays 15°** until the next `CameraReset` or
  `CfgFollowCamera` (runtime: 15° 150 updates later).
- Shakes started while it is on are 0.66 as strong, and the pad's rumble thresholds are lowered by a quarter
  ([Shake](effects.md#shake)).

## In Coney

### Long-fall port notes {#long-fall-coney}

The Coney changes and remaining landing comparison are described in
[Camera implementation](../../engine/camera.md#long-fall-coney).

## Open questions

The landing update after a long fall and the other outstanding comparisons remain in the system's
[open questions](index.md#open-questions). The retained runtime measurements below have no new run records.

## How this was checked

The earlier PCSX2 methods and observations below predate run-record ids. This restructure preserves their evidence
without claiming a fresh executable read or runtime comparison.

### Runtime checks {#runtime-checks}

PCSX2 2.9.94, `level99` checkpoint 1, Rembrandt, read over PINE once per update; stick magnitudes by the
[stick-table method](../../guides/research-workflow.md#driving-pcsx2), right stick by the keyboard (full deflection).
"Wanted position" is `+0x250`, the camera's own position `+0x10`. Confirmed (runtime) unless marked:

- **Position lag.** With the player moved 2 m away in one write, the gap between the camera and its wanted position
  shrank by a factor of 0.78 per update (0.120, 0.094, 0.074, 0.057, 0.045 m), with `+0x388` reading 0.226: 22% per
  update. `+0x388` was 0.157 standing at the start spot (pulled in by a wall) and climbed to 0.227 within 0.5 s of
  running into the open, the 0.5%-per-update drift of [World collision](collision.md#collision).
- **Leash band.** While running, the wanted position stays 3.50 m from the look-at point (the band's far edge);
  standing, it stays wherever it was inside the band (3.00 m at the start). At the start spot the camera itself is
  pulled in to 1.85 m by the walls and recovers toward 3.5 m over about 1.5 s of running.
- **Right stick yaw.** Full deflection right (raw x = 255): the wanted position turns by exactly 5.00° per update
  (150°/s) from the first update; the camera's own yaw follows with the position lag, reaching about 5° per update
  after 0.4 s, and coasts for a few updates after release. The 0.334 s hold timer `+0x368` read 0.301 (one update
  counted down) while the stick was held.
- **Right stick pitch.** Full deflection up: `+0x3b8` = 85°/s; the pitch target `+0x3b4` stopped at the upper limit
  `+0x3ac` = 30°, which is the one-player (`0x0050b19c` = 1) case at the close zoom (zoom step 4.8). The view's pitch
  eased from 15° to 30° behind it.
- **Auto-follow was not seen.** Running at 7.8 m/s (gait 4) with the facing held 63-78° away from the view for
  1.5 s, in the open, the wanted position turned only as the moving look-at point dragged it: no rotation of its own,
  with the auto-centre option on (as saved) and with `0x0050b240` written to 0 (the default rule). By the code either
  rule should then turn about 3.4° (auto-centre) or 2° (default) per update. Every gate listed under
  [Heading](#heading) that can be read over PINE passed (`+0x444` = 1, record words `+0x00` and `+0x08` zero,
  `+0x460` = 0, `+0x320` = −1, `+0x368` = 0, `+0x455` = 0, `+0x474` = 1, the run clip's descriptor flags 0); the
  condition that blocks it was not found then (the update's locals cannot be read without breakpoints).
- **Why: the blocked-view latch.** A later run from the same spot (left stick 100 % up for 35 updates, then 70 %
  up and 70 % left, then released) read `+0x458`-`+0x45f` every update. `+0x45b` was 1 for the first 19 updates,
  while the walls held the camera at 1.85 m, and 0 after. `+0x45c` and `+0x45d` were 1 from the start and stayed 1
  through the whole run, clearing on the second update after the player stopped (gait 0). The camera's wanted
  position turned 0° of its own on every update (its angle about the new look-at point, before and after the
  update). The same run with `+0x45c` and `+0x45d` written to 0 before every update: they came back while `+0x45b`
  was 1, stayed 0 after, and once the stick went diagonal the wanted position turned **0.6-1.6° per update** of its
  own (18-47°/s), the auto-follow rule at work. So a blocked view at the start of a run keeps auto-follow off until
  the player stops. In the street the run starts in the open, nothing is latched, and the rule turns. Confirmed
  (runtime).

### In the street {#street}

PCSX2 2.9.94, the street saves (in `level99`'s world, [Feel comparison](../feel.md)), Rembrandt, the camera read every
update while a scripted pad played; the per-pad option bytes `0x0050b240` / `0x0050b248` were 1 (the auto-centre
rule) and `0x0050b19c` was 1. Confirmed (runtime) unless marked:

- **Band and distance.** The leash band was **4.8-5.3 m** (`CfgFollowCamera`'s default 4.8 and + 0.5), the hard band
  as described (far edge + 0.35 m): the camera stood 5.30 m from the look-at point, 5.49 m walking and 5.65 m
  running, at pitches of 13°, 12.6° and 11.1°.
- **Auto-follow runs.** The wanted position turned about the look-at point by the auto-centre rule's rate
  (`(a − 45°) × 2.444 + 45°` per second above 22.5°, 200°/s from 90° to 100°, falling to 60°/s at 157.5°) when `a`
  is the angle between the player's new facing and the camera's view at the start of the update (its position then
  to its look-at point): within 5°/s on average from 30° to 90°, over 656 updates of walks, runs, sprints and turns;
  just above 22.5° it turned slightly the other way, as the rule's negative rate says. It turned while walking,
  running, sprinting and in the air, and not while standing, during the walk and run start clips or during the
  landing clip 436 (the gaits, [Heading](#heading)). Because the stick is turned by the camera, a stick held 90° to
  the side makes the player run in a circle. **Which camera turns the stick** (confirmed (code)): `Pads_Update`
  (`0x001454a8`, step 2 of [`Humans_Update`](../tasks.md#humans-update), before the locomotion; the cameras update
  last) builds the matrix from the player camera's orientation (vtable slot `+0x224`, the `+0x20` that faces the
  aim point) as left by the previous update, stores it at `0x005ddc90 + player × 0x10`, and `Pad_Update`
  (`0x00144fb0`) turns the stick `(x, y)` by its x and y axes and restores the stick's length (reversed while
  `+0x455` is set). The stored quaternion is reused instead only while the follow camera's `+0x473` is 0; the
  constructor, `Cam_Follow_Deactivate` and a stick at rest set it to 1, and only the co-op view change
  (`0x001276a0`, its third argument) can clear it. So in one-player play the stick's frame is exactly the orientation
  the auto-centre rule measures `a` from. Inferred check, with the locomotion's
  turn ([Characters](../characters.md#locomotion),
  18° limit, ease to 2.0 rad, carry 0.8; steady step = 5 × 18° × ease(`e`)): at the run's circle the original turns
  6.37° an update, so `e` = 90° − 76.6° + 6.37° = 19.8° before the step, and the rule gives 6.47° (194°/s).
  **The circling rate** (re-measured, slot 1, stick held 90°
  to the side for 120 updates after 30 updates straight up; rates averaged over updates 100-158):

  | Stick | Gait | Player's turn | Rule's turn about the look-at point | Leash drag | `a` |
  | --- | --- | --- | --- | --- | --- |
  | 35 %, 60 % or 80 % sideways | walk, 1.63 m/s | **143°/s** | 127°/s | 16°/s | 77° |
  | 100 % sideways | run, 7.80 m/s | **191°/s** | 122°/s | 72°/s | 72° |
  | 70 % / 70 % (a full diagonal) | run | 61°/s | 24°/s | 42°/s | 34° |

  "Rule's turn" is the wanted position's rotation about the **new** look-at point (what the auto-centre rule adds;
  the leash moves it only along that line), "leash drag" the rotation of the old wanted position from the old to
  the new look-at point (the target's sideways move); they add up to the camera's turn, and the player's facing
  turns with the camera, `a` steady. The 122°/s and 127°/s an earlier reading gave as the circling rate are the
  rule's share alone: the original circles at **about 190°/s** at a run. The `a` in that table is measured to the
  look-at point, which is why the rule's turn came out 3-10°/s above the formula. The rule measures `a` against the
  camera's orientation, which faces the aim point. Re-recorded (the `run_circle` scenario, with the
  orientation `+0x20`, the aim point and the turn rate `+0x360` that `0x0012d688` writes): with `a` taken from the
  previous update's orientation and this update's facing, the formula gives the written rate to 0.1°/s on every
  update except the three on the 90° boundary itself. At the run's steady circle that `a` is 76.6° (122.3°/s), while
  the view to the look-at point gives 74.8°: the aim point sits 0.17-0.40 m off the look-at point. Confirmed
  (runtime).
- **Sprint zoom.** From the first update at the sprint gait the band's near edge went 4.8 → 4.569, 4.379, 4.221,
  4.085, 3.968, 3.863, 3.770, 3.686, 3.607, 3.533, 3.462, 3.391, 3.315, 3.216, 3.0 (the far edge 0.5 more), and the
  target pitch `+0x3b4` fell by 0.4286° per update from 13° to **7°**: both over 14 updates. In the sprint the camera
  settled 4.70 m from the look-at point at a pitch of 5.2°. Both went back the same way over 14 updates, starting
  8 updates after the run stop began (the body at about 2.5 m/s). A run (gait 4) did not change them.
- **Right stick.** Yaw at 30, 60 and 100 % to the right (raw x 189, 217, 255): 74.8, 106.7 and 150.0°/s applied to
  the wanted position from the first update, as the table above gives. Pitch up at 100 %: the target rose 85°/s to
  the upper limit **40°** and stayed there after release.
- **Look-at height after a climb's rise** (slot 8, the feet rising 1.28 m onto a trash can and 2.64 m onto a roof):
  the look-at point moved **20 % of the way** to feet + 1.4 m per update for 4-7 updates, then covered the last
  0.5-0.6 m in 2 updates, so the view's pitch stayed at 3.6° or more. During a jump it follows the feet directly.
- **Fences.** Through a running fence climb (slot 7, material 30) the camera stayed 4.9-5.3 m away while the fence stood
  between it and the player, and passed through the fence afterwards without pulling in: its collision does not see that
  fence, because every ray of the collision step excludes its material ([World collision](collision.md#collision)).
  Nor did the face
  that pulled Coney's camera in 0.19 m behind the player: it is not a low face but a **disabled** two-sided panel of
  `level99`'s mesh (triangle 27, material 91, flags `0xf442`: bit 0 clear, 4.1 m wide and 2.65 m tall across the run's
  path at `y` = 31.17), read from the save's RAM; slot 7 has 8 disabled triangles (that panel, two of material 187
  `STOREDOOR_GLASS`, four of material 2 `GLASS`) and slot 1 none, while every triangle on the disc is enabled, so the
  game switched them off ([Collision](../collision.md#chunks), `0x0034fba0`). The main ray tests disabled triangles but
  recasts without them when the look-at point is within 0.5 m of the plane
  ([World collision](collision.md#collision)), which is the
  case here (inferred; the panel's data and the disabled counts are confirmed (runtime), read from the saves' RAM).

### Earlier inline observations {#follow-inline-observations}

The earlier inline passages are preserved below as checking notes. Their list numbers refer to the original
surrounding steps, not a new sequence of instructions.

```text
These passages retain the earlier inline runtime labels and measurements. Links in the body point here rather than
asserting a newly recorded comparison.

The view does not face the look-at point: it faces an **aim point** `+0x240` that leads the player a little along his
facing and rises and falls with his body, eased twice. The collision step moves it each update (`0x00130990`, at
`0x00133320`-`0x00134290`); confirmed (code) there and confirmed (runtime) through the `fence_climb` scenario (below).
With one target that is a player (with any other target, only a 1.0 m sphere push of the old aim point, when switch
`0x0050b1c0` is 0):

   **Running and sprinting** (`0x00133674`-`0x001337d0`), confirmed (code): `+0x450` is a switch the constructor sets
   to 1 (`0x00124940`, its only writer), so it is always on. The update (`0x0012b310`-`0x0012b3f4`) passes the
   collision step two flags of this update: **sprinting** (gait `+0x1a8` 5 with no blocking record flags,
   `0x00223a98`) and **running or sprinting** (that, or gait 4, `0x00223a60`); both are 0 under the takeover `+0x465`.
   While either is set the height step above is skipped and the old aim point is moved instead:
   - **running** (gait 4): by 0.22 × (`+0x4e0` − (0, 0, 1)), the body point's offset as it is, not turned by the
     human's heading; the sway `+0x3f8` is set to 0. So the aim point bobs with the hips: 0.22 × (0.93-1.0 − 1), a
     little below its old height each update;
   - **sprinting**: a sideways **sway**. The sway `+0x3f8` moves by `+0x3f4` × 0.028 × a random number in 0.125-0.25
     (`0x00335420`), 0.0035-0.007 m an update; past ±0.028 it is held at `+0x3f4` × 0.028 and the direction `+0x3f4`
     (1.0 at construction) flips. The aim point moves sideways by the sway along the view's horizontal side axis (the
     unit cross product made at `0x00130b68`) and up by (2 × |sway| − 0.028) × 0.6, so between −0.017 and +0.017 m.
     Confirmed (code).
3. **Push.** Unless the camera aims, `P` is pushed out of the world by a sphere of radius 1.0 (0.95 with two targets ×
   the target's scale; `CollisionMesh_SpherePush`).
4. **Lag** `+0x418`, worked out afresh each update from 0.06: × (1 + (1 − `c`) × 0.5) where `c` = `|unit(velocity) ·
   camera forward|`, both in **3D** (the velocity's z too; the forward is the y axis of the orientation `+0x20`,
   `Quat_AxisY` `0x003363b0` called at `0x00133434`, the view as the last update left it, toward the aim point, not
   the look-at point; confirmed (code), re-read 2026-10-21), once the target is faster than his walk speed (else `c` =
   1); + `s` × it × ((1 − `c`) × 0.5625 + 0.1875) with `s` = (horizontal speed − walk speed) ÷ sprint speed (≥ 0; `s`
   is scaled by (0.75 − `+0x38c`) × 4/3 while the timer `+0x38c` runs down, set to 0.375 s after a blocked view clears
   and 0.75 s after a sphere push ends); × 2.5 when the target's **3D** speed is above walk + 18% of (run − walk) and
   either the new aim point would end farther from `P` than the look-at point is, or the stick is not his (`+0x1b` of
   the per-player record 0) ([below](#long-fall)); × 1.25 when slower and the latches `+0x45b`/`+0x45d` are set
   on a slope (`+0x460`). Then `+0x418` = last update's value + 3% of the difference; 0.6 while the human has state flag
   `0x1000000000` (a long fall); at least 0.8 when the main ray is blocked while aiming; at least 0.3 during a takeover;
   halved while the target does not count or the game mode is not 1. Runtime: 0.061 standing, 0.14-0.16 running.
5. **Move**: `+0x240` = old + (`P` − old) × `+0x418` (when the main ray is blocked by a wall whose normal is nearly
   level, by `min(1, (3.5 − the feet's distance in front of it) × 0.4) × 0.3` instead); then, if that left it farther
   from the look-at point than `P` is, it is pulled back in plan to `P`'s distance.

- **The tests in player terms.** "The target does not count for the cameras" is `Camera_HumanCounts`
  (`0x00123500`) returning 0: he is down or dead (`Human_IsDownOrDead` `0x00227e60`: dying, dead, knocked out or
  tackled, state `0x180050000`; not knocked down), or he is **cuffed** (`Human_IsCuffed` `0x00223b70`, state
  `0x20000`) and cannot free himself (he still counts with upgrade (6, 15) unlocked and a key, item 6, carried,
  [Crimes](../crimes.md)). "The game mode is not 1" is `GameModeStack_TopId` (`0x0015e748`) ≠ 1: another mode is on top
  of gameplay ([Boot](../boot.md)). **The long fall** (state flag `0x1000000000`, read only for a player, human `+0x1b0`
  ≠ −1) is set by `Human_BuildFallClip` (`0x0025d020`) when a fall (anim state 26) starts while he is knocked down,
  knocked out or cuffed (state `0xe0000`), dying or dead, or already falling (held flag `0x400000`): he goes over an
  edge limp, as when knocked or thrown off a ledge, and plays the long-fall cycles 422 / 425.
- **The timer `+0x38c`.** Set to **0.375 s** when the main ray hit a face this update (its hit plane is not zero) but
  the hit does not hold the camera (`+0x45b` clear) while the sticky copy `+0x45c` is still set; set to **0.75 s**
  when the main ray hit a face this update and the step-3 push of `P` (the 1.0 m `CollisionMesh_SpherePush`, which
  returns whether it touched a wall) touched nothing this update but touched (or did not run, while aiming) on the
  last (`+0x45a`, this update's result kept at the end of the step). With any other target the push in question is
  the 1.0 m push of the old aim point. It **counts down by `dt` only while `s` > 0** (the target faster than his
  walk), so it waits while he stands.
- **The overshoot test** (the ×2.5), confirmed (code) at `0x00133dcc`-`0x00133ec8` and confirmed (runtime) by the
  `fall_high` trace ([Long falls](#long-fall)): the speed it compares is the **3D** speed (`sp+0x35c`, `vmul.xyz` at
  `0x00130f90`; the `s` above uses the horizontal one, `sp+0x358`, `vmul.xy` at `0x00130f68`). With `new` = `old` +
  (`P` − `old`) × the lag being built, it compares squared 3D distances: (|`P` − look-at|² > 10⁻⁵ or the slope latch
  `+0x460`) and |`P` − look-at|² < |`P` − `new`|², so the lag is × 2.5 whenever this update's aim point would still be
  farther from `P` than the look-at point is (about 0.36 m, the lead); only when that fails is the per-player record's
  `+0x1b` read (0: × 2.5). (This page used to read the first distance as |`P` − `old`|, which made the test never hold;
  the subtrahend at `0x00133df4` is the pointer at `sp+0x3b4`, the look-at point `+0x180`.) The ×1.25 needs the target
  at or below walk + 18% of (run − walk), the latch bytes `+0x45b` or `+0x45d` set (`+0x458` & `0xff00ff000000`) and
  the slope latch `+0x460`.
- **The blocked-wall share** replaces the lag in step 5 when the main ray's hit holds the camera (`+0x45b`), the
  camera does not aim, and the hit face is steep: |normal `z`| < **0.6** (`0x005d91cc`, steeper than 53.13°). Its
  "feet's distance in front" is the target's transform position (his feet) against the **hit face's plane**:
  `n · feet − d`, with `n` and `d` the plane of the main ray's hit (the ray from the look-at point toward the
  camera). The share is `min(1, (3.5 − that distance) × 0.4) × 0.3`: 0.3 within 1 m of the wall, falling to 0 at
  3.5 m. (Aiming, the lag is at least 0.8 instead.)

**The trace** (confirmed (runtime), PCSX2 2.9.94, 2026-10-10, scenarios `repo:research/traces/scenarios/fall_high.toml`
and `fall_high_stick.toml`, field set `fall` of `repo:research/traces/fields.toml`): slot 1, Rembrandt in `level99`'s
street; at frame 5 his feet are put 40 m up and he falls for 71 updates (2.4 s), to 34 m/s. Every update:

**The trace** (historical observation ([checking notes](#follow-inline-observations)), PCSX2 2.9.94, 2026-10-10,
scenarios `repo:research/traces/scenarios/fall_high.toml`
and `fall_high_stick.toml`, field set `fall` of `repo:research/traces/fields.toml`): slot 1, Rembrandt in `level99`'s
street; at frame 5 his feet are put 40 m up and he falls for 71 updates (2.4 s), to 34 m/s. Every update:

**Several heights** (confirmed (runtime), the maintainer's question on issue #111: does the original's camera fly up
too?). PCSX2-MCP 2.9.129, 2026-10-20, scenario `repo:research/traces/scenarios/fall_heights.toml` recorded with
`--let rise=20`, `40`, `70` and the default 120: the same spot, slot 1, the stick at rest, 300 updates sampled every
update. While he falls (vertical speed under −0.5 m/s):

**Several heights** (historical observation ([checking notes](#follow-inline-observations)), the maintainer's question
on issue #111: does the original's camera fly up
too?). PCSX2-MCP 2.9.129, 2026-10-20, scenario `repo:research/traces/scenarios/fall_heights.toml` recorded with
`--let rise=20`, `40`, `70` and the default 120: the same spot, slot 1, the stick at rest, 300 updates sampled every
update. While he falls (vertical speed under −0.5 m/s):

How it was done (a recipe for `coney-tools pcsx2`): the scenario's `setup` adds `rise` to the player's transform
height (`tf(player) + 0x8`, `f32`) at frame 5, once; for a single frame, launch the scenario's patched state copy
with `pcsx2 launch`, pause through the debug server, write the scripted pad's rest bytes (`0x005de3aa`-`0x005de3af`:
`ff ff 80 80 80 80`), find the player's transform as the entry of the table at `0x00714b00` (0x20 each) whose
position is the trace's first row (entry 2 in slot 1), add the height at `+0x8`, and `pcsx2_frame_advance` two
frames per update before `pcsx2_screenshot`. Confirmed (runtime).

Does the original's camera jump when the player climbs? **No**, the climb itself moves it smoothly; one level spot
pulls it in once. Confirmed (runtime), PCSX2 2.9.94, slot 7 (`level99` checkpoint 3, the street fence), scenario
`fence_climb` (left stick 100 % straight ahead, triangle every 2 updates from 4.6 m before the fence), every update:

- **Auto-follow** (`0x0012a400`), the default: let *a* be the angle between the camera's horizontal forward and the
  player's facing. Nothing happens below 22.5° or above 157.5°, so running towards the camera does not spin it. The
  rate is `(a − 22.5°) × 2.667` per second below 45° (0 to 60°/s), 60°/s from 45° to 135°, and above 135° it falls
  from 120°/s back to 60°/s at 157.5°. Each update turns by `min(a, rate × dt)` toward the facing.
- **Auto-centre option** (`0x00129f88`, used instead when the per-pad option bytes `0x0050b240` and `0x0050b248` are
  both set; they are 1 in the `level99` save used at runtime and 1 by default,
  [Feel](../feel.md#details-behind-the-table)): nothing below 22.5°; from 22.5° to 90° the rate is `(a − 45°) × 2.444 +
  45°` per second (negative below 26.6°, so the step is then a small turn the other way; 45°/s at 45°, 155°/s at 90°);
  200°/s (`3.4907` rad/s) from 90° to 100°; from 100° to 157.5° it falls linearly (`(157.5° − a) × 2.435 + 60°`) from
  200°/s to 60°/s, unless the call's fourth argument is set, which keeps 200°/s up to 157.5°. That argument is the
  update's local at `sp + 0x1d4`, set at `0x0012b2ac`: 1 when there is one target, he is in a fight stance
  (`Human_IsInFightStance` `0x00228340`, state flags `0x3`) and he still counts for the cameras (`0x00123500`), else 0;
  confirmed (code). So a player squaring up gets the fast swing over the whole range. Beyond 157.5° the rule turns only
  when the target's record has state flag 4 (`0x002265f0(target, 4)`, "has a target" in
  [Combat](../combat/index.md#state-flags), inferred there; the same flag can stand in for "moving").
  `a` is the angle between the target's facing (its rotation in the transform table) and the camera's own horizontal
  forward: vtable slot `+0x224` returns the orientation `+0x20`, whose y axis (`Quat_AxisY` `0x003363b0`), flattened and
  normalised, is the view as placed at the end of the previous update, toward the [aim point](#aim-point) `+0x240`, not
  the look-at point. The facing's y axis is used flattened but not normalised. The angle is `π/2 − atan(c / √(1 − c²))`
  of the cosine `c`, with an atan (`0x003356a0`, a range-reduced rational form) good to about 10⁻⁵° (inferred: one of
  its constants sits in memory Ghidra cannot read). Confirmed (code); confirmed (runtime) in [In the street](#street).
  Each update turns by `min(a, rate × dt)` (`dt` = `+0x1a0`) toward the facing. The cosine of each threshold is computed
  with `0x004b8a70` (cosine, inferred from the thresholds' use).
- **When either rule runs** (`0x0012ae58`, the call at `0x0012bd80`): one target (`+0x444` = 1), the target passes
  `0x00123500` (not in states `0x180050000` of record word `+0x00`), no camera flags in `+0x460 & 0xffff0000`, nothing
  watched (`+0x320` = −1), no yaw from earlier steps this update, and no camera input in the last 0.334 s (`+0x368` =
  0). Inside `0x00129c78` (arguments read at the call, `0x0012bd30`-`0x0012bd84`), confirmed (code): the default rule
  needs "running": the target's gait `+0x1a8` is 4 or 5 with no blocking record flags (`0x00223a60`, `0x00223a98`). The
  auto-centre rule needs "**moving**": gait **2, 4 or 5** (walk, run, sprint; `0x00223a40` adds the walk), or state flag
  4 of the record; **not 0, 1 or 3**, so not while the body moves slower than a walk or at a jog's speed. It also needs
  `+0x474` = 1 (cleared on an update whose stick vector in the per-player record, `+0x00` / `+0x04`, points more than
  157.5° from +y, and set again at the start of any update whose gait is not 0, so it blocks only the updates with the
  stick pulled back that far; whether that vector is the camera-turned stick is not traced), `+0x455` = 0, no
  right-stick input, the top animation task's clip without descriptor flag `0x8000` (`0x00175be8`), none of the human
  flags `0x18003ff0`, and the collision bytes `+0x45b` and `+0x45d` clear: the collision step sets `+0x45b` when its
  main ray from the look-at point is blocked ([World collision](collision.md#collision)), so **auto-follow stops on
  the update after
  one in which the view was blocked**. `+0x45d` is a latch: it is set at `0x00132fa4` when the step's local "view
  blocked" (`sp + 0x364`, set at `0x001327a0` from the blocked local `sp + 0x368`, which `0x0013207c`, `0x001321e4` and
  the swing-away give-up `0x001325a4` set) is true, and `+0x45c` copies `+0x45b` the same way (`0x00132fb4`). Both are
  cleared by `0x00124778` (`0x00124980`) and at the end of the collision step (`0x00132fd0`-`0x0013314c`) on an update
  whose view is not blocked (`sp + 0x364` = 0) when either the camera is no longer held in (its wanted distance from the
  look-at point, `sp + 0xc0` to `sp + 0x20`, is within 10⁻⁵ of the distance the step allows, `sp + 0x348`), or the
  allowed distance plus 3% of that gap reaches the wanted distance the step started from (`sp + 0x34c`), or the zoom
  button was pressed this update (the stick step's flag, the step's stack argument `0x4c8`). Otherwise the wanted
  position is moved out to the allowed distance plus 3% of the gap and the latches stay. Here "allowed distance" is the
  distance from the last look-at point `+0x270` to `+0x2b0`, the point the step returned on the last update, and the 3%
  step is capped at the distance the step started from. While the view is blocked (`+0x45b` or the local set) the wanted
  position is instead drawn to the allowed distance plus **4%** of the gap, and the hold timer `+0x414` starts at 1.5 s
  if it is not running; on the clearing path it starts at 1.5 s (3 s while `+0x460` is set, 0.1 s with `+0x463` and a
  timed move). This is the final form of the rule (`0x00132f5c`-`0x00133250`, read in full). The **recovered distance**
  `+0x380` moves 10% of the way to the allowed distance at the end of every step (`0x00134a18`). Nothing else reads it:
  the constructor zeroes it and the activation sets it to the band's near edge, so it can be left out. Confirmed (code);
  the meaning of the locals is inferred. While the player runs, the leash keeps the wanted position at the band's far
  edge beyond what the walls allow, so the latches hold until he stops (runtime). So **one blocked update keeps
  auto-follow off until the player stops** ([Runtime checks](#runtime-checks)). Confirmed (runtime), slot 1, stick 100 %
  sideways: the rule turned 110-129°/s every update at gait 4; with `+0x45b` written to 1 before each of 21 updates it
  turned 0 on each of them, and 129°/s again on the next. The gaits explain what [In the street](#street) saw: no turn
  in the walk and run start clips (gait 0-1 while the walk start moves at 0.76 m/s, 3 in the run start's middle) or the
  landing clip (4.23 m/s, gait 3), and a turn during the run start's first five updates and the run stop's slower
  updates (gait 2).
- **Keep the target in view** (`0x0012e170(factor, range)`, called with 0.25 and `+0x3fc` for the human
  `CamSetSecondary` gives, `+0x320`, and with 0.4 in one fight case): when the target's direction from the camera is
  more than `fov × factor` from the view's, the camera yaws 35% of the excess per update, at most 270°/s
  (`4.712` rad/s), toward it. With a range above 0 it acts only while the target is within the range and a ray
  (mask `0x200`) from the look-at point reaches it; `level99` passes 0, so no range and no ray. It runs only when
  nothing else turned the camera this update, `+0x463` and `+0x454` are clear, and it replaces auto-follow, which
  needs `+0x320` to be NilHandle. Confirmed (code).
- **The right stick** (rate `+0x358`) and the **heading target** (`+0x350`, step 15 above).

While the player sprints, the camera pulls in to the minimum distance and lowers its target pitch to 7°, and both go
back 250 ms after the sprint ends. Confirmed (code) at the cited addresses and confirmed (runtime) in the street save
(slot 1, stick 100 % and L2, every field below read every update; PCSX2 2.9.94):

While the player fights with a lock-on, the follow camera pulls in and frames the enemy. Confirmed (code) in the
update at `0x0012bb00`-`0x0012bbe8`; confirmed (runtime) as noted.

- **When**: one target, the target is player 1, the player is pad-controlled (per-player `+0x1b`), and
  `0x00233c50` holds: the mode `0x00510228` is not 0 (1 by default; `CfgAutoCloseMode` sets it, no script calls it),
  one player, the player rides nothing (`+0xc4`), and the player has a fight target (`0x00226e60`) with record flags
  `0x8` and `0x4` both set (L1 held at a target; with mode 2, also any update in the lock-on movement state
  `0x00241b90`). Confirmed (runtime), slot 6 copy, L1 held facing the pedestrian: flags `0xd`, `+0x46f` 1 on the
  third update after the press, 0 on the second after the release. In `level99_combat.lua`'s sparring L1 is disabled
  (`EnableCommand(player, 6/8, 0)`) and the camera stayed off.
- **On entry** (`+0x46f` set): the band's wanted near edge is saved in `+0x3cc` (the sprint zoom's saved edge if
  one is held, else a wanted edge in progress, else the near edge) and set to **2.4 m**; unless `+0x3d0` holds a
  pitch, the **target pitch becomes 15°**. The band eases by `d × 4.5 × dt` per update (no timer, [Sprint
  zoom](#sprint-zoom) step 4), so 4.8 → 4.44, 4.134, 3.874, …, 2.4 in about 50 updates (runtime, to 0.001 m); the
  zoom step follows it (30° below 3.72 m).
- **Each update while on**: `0x0012e9a8` takes the enemy's point (its position plus half its [body point](#body-point)
  `+0x4e0`, the hips in the body's frame, not turned by his heading, so about 0.5 m up; an object's position + 0.5 m)
  and the angle at the look-at point between the camera's horizontal view and the direction to the enemy; outside
  25°-29° it yaws by `(angle − 27°) × 0.455` toward 27°; the turn is capped at 640°/s (`11.17` rad/s) when the enemy is
  more than 29° off, not when it is under 25°. So **the enemy is held 27° off the view's centre**, beside the player.
  This counts as this update's turn: auto-follow and keep-in-view do not run. Confirmed (code).
- **On exit**: the wanted near edge is set back to the saved `+0x3cc`, which is cleared; the band eases back the same
  way (runtime: 2.4 → 2.76, 3.066, …, 4.8). **The target pitch stays 15°** until the next `CameraReset` or
  `CfgFollowCamera` (runtime: 15° 150 updates later).
- Shakes started while it is on are 0.66 as strong, and the pad's rumble thresholds are lowered by a quarter
  ([Shake](effects.md#shake)).

`CameraCreateFollow` passes the `global.lua` constants ([Scripts](../scripting.md#cameras-from-lua)): minimum 3, maximum
6.6, default 4.8, pitch 13°, field of view 65, near plane 0.1, offset (0, 0, 1.4), slow motion 0.2. Confirmed
(runtime) in the object above.

**Several heights** (historical observation ([checking notes](#follow-inline-observations)), the maintainer's question
on issue #111: does the original's camera fly up
too?). PCSX2-MCP 2.9.129, scenario `repo:research/traces/scenarios/fall_heights.toml` recorded with
`--let rise=20`, `40`, `70` and the default 120: the same spot, slot 1, the stick at rest, 300 updates sampled every
update. While he falls (vertical speed under −0.5 m/s):

1. **Detecting the sprint** (in the update, `0x0012b310`-`0x0012b3f4`): the target's stored gait `+0x1a8` is 5
   with no blocking record flags (`0x00223a98`) → "sprinting" this update (a local, `sp + 0x1e8`). On the first such
   update of a sprint (`+0x36c`, the run time, still 0) the switch `+0x468` is copied to the arm byte `+0x467`.
2. **Latching** (in the update, between the right-stick step and the band's ease, `0x0012b504`-`0x0012b5d8`): while
   sprinting and armed, when the player's **brain** (`0x0021d408`) has no enemies (byte `+0x152`, the one `BrHasEnemies`
   reads) or the nearest of its up to 16 enemies (handles at `+0x164`, `0x004db5b0` with the squared distance of
   `0x00336ce0`, FLT_MAX `0x00548ad8` for none) is closer than **12 m** (squared distance < 144), `+0x467` is cleared
   and `+0x466` set; with `+0x466` set and `+0x448` 0, `+0x448` = 1. On the first update **not** sprinting with `+0x466`
   set: `+0x466` = 0 and `+0x448` = the game time + **250 ms**. (A time left in `+0x448` by an earlier sprint, as in the
   save, also lets the function run; it then has nothing to do until the next sprint.)
3. **The zoom function** `0x00128cf0(camera, sprinting, 0)` runs every update once the game time has passed `+0x448`
   (so at once for 1, after 250 ms for a time; call at `0x0012c5c8`).
    - **Sprinting**, with no other zoom's band saved (`+0x3c8` = 0): the first time, it saves the band's near edge in
      `+0x3e0`, the zoom distance in `+0x3e8` and sets the timer `+0x40c` to **0.5 s**; with one player camera
      (`0x0050b19c` = 1, [Pitch](#pitch)) it sets the wanted near edge `+0x34c` to the **minimum distance** (`+0x300`,
      3.0) and steps the zoom distance to the default (`0x001254f0`, which also sets the upper pitch limit to 30°);
      otherwise (two players) it leaves the band and sets `+0x3e0` to the maximum − 0.5 and `+0x3e8` to the default.
      It then saves the target pitch in `+0x3e4` and moves the target pitch toward **7°** (0.122173 rad): on the
      update the timer reads 0.5 nothing; then by `|7° − pitch| / +0x40c × dt` per update, which is a straight line
      arriving as the timer runs out (without a timer, 30°/s).
    - **Not sprinting**: `+0x34c` = the saved `+0x3e0`, `+0x3e0` = 0, the zoom distance back to `+0x3e8`, `+0x40c` =
      0.5 s; the target pitch moves back to `+0x3e4` the same way while the timer runs. Once both the band and the
      pitch are within 10⁻⁵ of their goals, `+0x448` = 0 and `+0x3e4` = −FLT_MAX: the zoom is over.
4. **The band's ease** (`0x0012aae0`, every update while `+0x34c` > 0): with `d` = `+0x34c` − `+0x32c` and `T` =
   `+0x40c`, the near edge moves by `d × |d| / T × dt` (or `d × 3.5 × dt`, `d × 4.5 × dt` with `+0x448` 0, when no
   timer runs), clamped at `+0x34c`; the far edge is the near edge + 0.5; when within 10⁻⁵ the edge snaps and `+0x34c`
   = −1. The zoom step follows the band's near edge as `CamSetFollowZoom` sets it ([Script
   calls](scripted.md#script-calls)): at
   or below `min + 0.4 × (default − min)` (3.72 m) the step is the default (30° with one player camera), at or below
   `default + 0.6 × (max − default)` (5.88 m) the maximum (40°), else the minimum (50°) (`0x001254f0`). Confirmed
   (runtime): 40° until the near edge passed 3.72 m, then 30° (sprint and [combat camera](#combat-camera)).

Fixed (issue #111); this paragraph is how Coney differed before. Inferred, from `repo:src/camera/follow_camera.cpp`
(`stepAim()`) and `repo:src/human/player.cpp` (`followTargetOf()`): Coney built the lag from the **horizontal** speed
only. `c` was 1 while the horizontal speed was under the walk speed, and the ×2.5 needed `!stickOwned` and a
horizontal speed above walk + 18% of (run − walk), with no overshoot test. In a straight fall the horizontal speed is
about 0, so the lag stayed **0.06**, and at 35 m/s the aim point settled about 1.167 × 0.94 / 0.06 ≈ **18 m** above
the look-at point (11 m at 20 m/s) instead of under 5 m. With the camera 3.5-5 m from the look-at point the view then
faced nearly straight up: the user's "camera flies up". Coney's `longFall` (airborne while not counting or grounded)
is not the cause: the original does not set it on a plain fall either. (Coney's headless `--trace` could not show
this: with the release build of 2026-10-10 the trace's camera columns stayed constant even in the `walk60` scenario.)
```

### Earlier long-fall port comparison {#long-fall-port-comparison}

The earlier page recorded the following comparison and development history. It is retained as historical checking
notes, not a newly performed comparison or a newly verified implementation status.

```text
### Coney's difference {#long-fall-coney}

Fixed (issue #111); this paragraph is how Coney differed before. Inferred, from `repo:src/camera/follow_camera.cpp`
(`stepAim()`) and `repo:src/human/player.cpp` (`followTargetOf()`): Coney built the lag from the **horizontal** speed
only. `c` was 1 while the horizontal speed was under the walk speed, and the ×2.5 needed `!stickOwned` and a
horizontal speed above walk + 18% of (run − walk), with no overshoot test. In a straight fall the horizontal speed is
about 0, so the lag stayed **0.06**, and at 35 m/s the aim point settled about 1.167 × 0.94 / 0.06 ≈ **18 m** above
the look-at point (11 m at 20 m/s) instead of under 5 m. With the camera 3.5-5 m from the look-at point the view then
faced nearly straight up: the user's "camera flies up". Coney's `longFall` (airborne while not counting or grounded)
is not the cause: the original does not set it on a plain fall either. (Coney's headless `--trace` could not show
this: with the release build of 2026-10-10 the trace's camera columns stayed constant even in the `walk60` scenario.)

The fix, now in `stepAim()`: it takes `c` from the 3D velocity against the view's 3D direction as the last update
left it (from the camera's place before the update to the old aim point; **Coney's reading** of the orientation
`+0x20`), with its gate on the 3D speed; keeps `s` on the horizontal speed; gates the ×2.5 on the 3D speed and
applies it when |`P` − look-at|² > 10⁻⁵ and |`P` − look-at|² < |`P` − `new`|² (`new` = the old aim point + (`P` − the
old aim point) × the lag being built, which the original keeps in `+0x418` while it builds it: written at
`0x00133be8` and `0x00133d4c`, read for the
test at `0x00133e20`; confirmed (code), re-read 2026-10-20), else when the stick is not the player's. **Open
question**: the slope latch `+0x460`, which also passes the first test, is not modelled (Coney has no slope latches
yet), a marked stand-in in the code. A first version took `c` against the direction to the look-at point, which in
a fall points down along the velocity (`c` near 1), and the synthetic test's lag only reached 0.141 with the aim point
8.8 m above the look-at point; read against the view toward the aim point it passes.

The camera's own height was a second difference: Coney's pitch step turned the wanted position toward its target at
most 85°/s, so in a fall the leash dragged it up its line to the falling look-at point and the camera ended almost
overhead (84-87° above the look-at point, against the original's 52-62°). The original's update step 11 turns the
wanted position the whole way to the target pitch every update (`0x0012d4e8` called at `0x0012beac`, the pitch measured
by `0x0012d908`; confirmed (code), [update](#update)); the 85°/s is only step 16's return without a stick. Coney now
does the same (`FollowCamera::update()`).

After both fixes (Coney's `--trace`, 2026-10-21, the same four falls from `level99` checkpoint 1's yard with the noclip
rise, the stick at rest; the original's from the table above). The view pitch is the highest before the last three
updates in the air, in both traces:

| Rise | Highest lag (Coney / original) | Aim point above look-at | View pitch | Camera pitch |
| --- | --- | --- | --- | --- |
| 20 m | 0.172 / 0.153 | 3.6 / 4.2 m | 8.5° / 15.3° | 43.1° / 41.9° |
| 40 m | 0.192 / 0.189 | 4.6 / 4.8 m | 10.5° / 11.4° | 52.1° / 51.5° |
| 70 m | 0.196 / 0.193 | 6.0 / 6.2 m | 31.3° / 32.9° | 58.9° / 59.1° |
| 120 m | 0.196 / 0.193 | 7.5 / 7.6 m | 53.9° / 54.8° | 61.3° / 61.8° |

Before the fixes, the 40 m fall gave a lag of 0.060, the aim point 14.4 m up and the view 84.8° up. Over the 40 m
fall the lag and the aim point's height stay within 0.01 and 0.15 m of the original's at every update. **Open
question** (inferred from the traces): in the last one or two updates before he lands, Coney's view tips up to
43-68° while the original's stays where it was (14° at 40 m); what the original does to the camera on the landing
update is unread.

The tests: in `repo:tests/camera/follow_camera_test.cpp`, a synthetic follow camera with the target falling straight
down from rest, gaining 0.52 m/s an update up to 35 m/s, for 70 updates: the lag passes 0.18 and the aim point stays
under 5 m above the look-at point, and the wanted position keeps the target pitch, so the camera stays under 65° above
the look-at point. A disc test, `repo:tests/platform/disc_fall_camera_test.cpp`, plays the same 40 m fall in `level99`
checkpoint 3 through the game session, once the follow camera has the view back from the checkpoint's opening scene
and the street end's locked camera (0.192 and 4.70 m). The overshoot test also applies at a run, so the `walk60` and
`run_circle` diffs are worth rerunning (the runtime lag there is 0.14-0.16).
```

### Earlier observations retained from upstream {#upstream-earlier-observations}

These passages came from upstream 8ecd4cb8 without surviving run identifiers. Their original labels and
context are retained as earlier checking notes, not a new runtime comparison.

```text
The view does not face the look-at point: it faces an **aim point** `+0x240` that leads the player a little along his
facing and rises and falls with his body, eased twice. The collision step moves it each update (`0x00130990`, at
`0x00133320`-`0x00134290`); confirmed (code) there and confirmed (runtime) through the `fence_climb` scenario (below).
With one target that is a player (with any other target, only a 1.0 m sphere push of the old aim point, when switch
`0x0050b1c0` is 0):

1. **Lead.** `+0x2e0`, a copy of the target's rotation, is slerped 35% of the way to it each update (snapped while the
   target does not count for the cameras, while the camera aims, `+0x454`, or while the target plays clip 1). Its
   forward `f`, the view's direction `v` (from the camera of the last update `+0x1e0` to the look-at point) and `L =
   max(0.75, 0.15 × band near edge)`: `L` is scaled by allowed ÷ wanted distance when the world holds the camera in, and
   by 0.4 while the target has state flag `0x40` or `0x80` (mask `0xc0`: he is grabbing someone from the front or the
   rear, [Combat](../combat/index.md#state-flags)). A world ray (`WorldManager_RayCast` `0x0040db88`, at `0x00133604`)
   from the look-at point along `f`, `L − 0.5 × L × |f · v|` long, gives the lead point `P` = look-at + `f` × (the free
   length − 0.01). So running away from the camera the aim point is about 0.36 m ahead of the look-at point (0.375 −
   0.01 at the default band; runtime 0.363 standing, 0.365 running). Confirmed (code); the runtime distances
   confirmed (runtime).
2. **Height.** With `d` = the human's body point height `+0x4e8` − 1.0 (the [body point](#body-point) `+0x4e0` is the
   hips' position in the body's frame, runtime 1.035 m standing, 0.93-1.0 running, 2.37 at the top of a fence vault,
   0.62 landing). **While the target runs or sprints** (below) this step is replaced. Otherwise: nothing (`P` stays at
   the look-at height) while `|d|` ≤ 0.27, unless the human has any of the state flags `0xe2c00`: `0x400` tackling (on
   top of the victim), `0x800` being tackled, `0x2000` (not named; paired attacks read it), `0x20000` arrested,
   `0x40000` knocked out, `0x80000` knocked down ([Combat](../combat/index.md#state-flags),
   [Characters](../characters.md#the-record), `Human_EndDownState` `0x0025e250`), so on the ground or in a tackle the
   height follows the body with no dead band. Otherwise `d` is scaled by `max(0, (0.27 − |d|) × 0.9 / 0.27 + 0.1)` when
   `|d|` ≤ 0.27, by `min(1, (|d| − 0.27) × 0.9 / 0.27 + 0.1)` up to 0.54, and kept beyond; a negative `d` is lowered by
   `(offset z − 1) × 0.6` more when the look-at offset is above 1.25 m (0.24 in `level99`). The height goal `zt` =
   look-at height + `d`; the new height is the old aim point's height moved 35% toward `zt`, but where that does not lie
   between the look-at height and `zt` it is `zt` itself; the result is stored as the look-at point's height `+0x188`
   (`0x001339c0`). (With `+0x465`, the target takeover of `0x00126558`, no dead band and 75%.) Confirmed (code); the
   body point's heights confirmed (runtime).

- **The tests in player terms.** "The target does not count for the cameras" is `Camera_HumanCounts`
  (`0x00123500`) returning 0: he is down or dead (`Human_IsDownOrDead` `0x00227e60`: dying, dead, knocked out or
  tackled, state `0x180050000`; not knocked down), or he is **cuffed** (`Human_IsCuffed` `0x00223b70`, state
  `0x20000`) and cannot free himself (he still counts with upgrade (6, 15) unlocked and a key, item 6, carried,
  [Crimes](../crimes.md)). "The game mode is not 1" is `GameModeStack_TopId` (`0x0015e748`) ≠ 1: another mode is on top
  of gameplay ([Boot](../boot.md)). **The long fall** (state flag `0x1000000000`, read only for a player, human `+0x1b0`
  ≠ −1) is set by `Human_BuildFallClip` (`0x0025d020`) when a fall (anim state 26) starts while he is knocked down,
  knocked out or cuffed (state `0xe0000`), dying or dead, or already falling (held flag `0x400000`): he goes over an
  edge limp, as when knocked or thrown off a ledge, and plays the long-fall cycles 422 / 425. Confirmed (code).
- **The timer `+0x38c`.** Set to **0.375 s** when the main ray hit a face this update (its hit plane is not zero) but
  the hit does not hold the camera (`+0x45b` clear) while the sticky copy `+0x45c` is still set; set to **0.75 s**
  when the main ray hit a face this update and the step-3 push of `P` (the 1.0 m `CollisionMesh_SpherePush`, which
  returns whether it touched a wall) touched nothing this update but touched (or did not run, while aiming) on the
  last (`+0x45a`, this update's result kept at the end of the step). With any other target the push in question is
  the 1.0 m push of the old aim point. It **counts down by `dt` only while `s` > 0** (the target faster than his
  walk), so it waits while he stands.
- **The overshoot test** (the ×2.5), confirmed (code) at `0x00133dcc`-`0x00133ec8` and confirmed (runtime) by the
  `fall_high` trace ([Long falls](#long-fall)): the speed it compares is the **3D** speed (`sp+0x35c`, `vmul.xyz` at
  `0x00130f90`; the `s` above uses the horizontal one, `sp+0x358`, `vmul.xy` at `0x00130f68`). With `new` = `old` +
  (`P` − `old`) × the lag being built, it compares squared 3D distances: (|`P` − look-at|² > 10⁻⁵ or the slope latch
  `+0x460`) and |`P` − look-at|² < |`P` − `new`|², so the lag is × 2.5 whenever this update's aim point would still be
  farther from `P` than the look-at point is (about 0.36 m, the lead); only when that fails is the per-player record's
  `+0x1b` read (0: × 2.5). (This page used to read the first distance as |`P` − `old`|, which made the test never hold;
  the subtrahend at `0x00133df4` is the pointer at `sp+0x3b4`, the look-at point `+0x180`.) The ×1.25 needs the target
  at or below walk + 18% of (run − walk), the latch bytes `+0x45b` or `+0x45d` set (`+0x458` & `0xff00ff000000`) and
  the slope latch `+0x460`.
- **The blocked-wall share** replaces the lag in step 5 when the main ray's hit holds the camera (`+0x45b`), the
  camera does not aim, and the hit face is steep: |normal `z`| < **0.6** (`0x005d91cc`, steeper than 53.13°). Its
  "feet's distance in front" is the target's transform position (his feet) against the **hit face's plane**:
  `n · feet − d`, with `n` and `d` the plane of the main ray's hit (the ray from the look-at point toward the
  camera). The share is `min(1, (3.5 − that distance) × 0.4) × 0.3`: 0.3 within 1 m of the wall, falling to 0 at
  3.5 m. (Aiming, the lag is at least 0.8 instead.) Confirmed (code).

**The trace** (confirmed (runtime), PCSX2 2.9.94, scenarios `repo:research/traces/scenarios/fall_high.toml`
and `fall_high_stick.toml`, field set `fall` of `repo:research/traces/fields.toml`): slot 1, Rembrandt in `level99`'s
street; at frame 5 his feet are put 40 m up and he falls for 71 updates (2.4 s), to 34 m/s. Every update:

- **State**: the drop starts at once (clip 428, anim state 26 by `Human_BuildFallClip` `0x0025d020`, held flag
  `0x400000` for 4 updates, state `0x800000000` falling). The state word's `0x1000000000` (`state_hi` `0x10`) is
  **never** set, with the stick at rest or at 45 %: the fall clip is built once. The landing gives state `0x80001`
  (knocked down) and clip 294 with the stick at rest, the roll 436 with it held. Confirmed (runtime).
- **Look-at point** `+0x180`: follows the feet + 1.4 m exactly, every update ([step 5](#update)).
- **Aim lag** `+0x418`: from the update the fall passes about 3 m/s, the value built each update is 0.06 × (1 + (1 −
  `c`) × 0.5) × **2.5** = 0.211-0.224 (`c` 0.03-0.2: the fall runs across the view; the ratio to 0.06 × (1 + (1 − `c`)
  × 0.5) is 2.50 ± 0.03 at every sampled update); eased 3% an update, the stored lag reads 0.061 (step 12), 0.077 (18),
  0.121 (30), 0.161 (48), 0.188 (72). The ×2.5 is the overshoot test's: the per-player record's `+0x1b` is 1
  throughout.
- **Aim point** `+0x240`: trails above the look-at point by 0.23 m (step 12), 1.6 (24), 2.7 (36), 3.4 (48), 4.1 (60)
  and 4.7 m at 34 m/s (72): about v / 30 × (1 − lag) / lag.
- **Camera, the stick at rest**: 4.4-5.4 m from the look-at point, pitched 13° above it at the start, 25° (step 30),
  38° (48) and 51° (74), as its 22 % position lag leaves it above the falling look-at point; the view's pitch (camera
  to aim point) goes from −13° to +10° (+14° on the last update in the air). No reset behind the player (step 7's 30 m
  limit is not reached once falling), and the hard band only eases in (far edge 5.65 to 5.01 m).
- **Camera, the stick at 45 %**: the target pitch drops to 7° and the camera keeps 3.5 m from the look-at point, near
  its height, so the aim point's trail tilts the view up: +16° (step 30), +38° (42), +46° (54), +55° (72). The original
  looks up a good way too in a long fall with the stick held, but its view stays under about 55°.

**Several heights** (confirmed (runtime); does the original's camera fly up too?). PCSX2-MCP 2.9.129, scenario
`repo:research/traces/scenarios/fall_heights.toml` recorded with
`--let rise=20`, `40`, `70` and the default 120: the same spot, slot 1, the stick at rest, 300 updates sampled every
update. While he falls (vertical speed under −0.5 m/s):

How it was done (a recipe for `coney-tools pcsx2`): the scenario's `setup` adds `rise` to the player's transform
height (`tf(player) + 0x8`, `f32`) at frame 5, once; for a single frame, launch the scenario's patched state copy
with `pcsx2 launch`, pause through the debug server, write the scripted pad's rest bytes (`0x005de3aa`-`0x005de3af`:
`ff ff 80 80 80 80`), find the player's transform as the entry of the table at `0x00714b00` (0x20 each) whose
position is the trace's first row (entry 2 in slot 1), add the height at `+0x8`, and `pcsx2_frame_advance` two
frames per update before `pcsx2_screenshot`. Confirmed (runtime).

- **Auto-follow** (`0x0012a400`), the default: let *a* be the angle between the camera's horizontal forward and the
  player's facing. Nothing happens below 22.5° or above 157.5°, so running towards the camera does not spin it. The
  rate is `(a − 22.5°) × 2.667` per second below 45° (0 to 60°/s), 60°/s from 45° to 135°, and above 135° it falls
  from 120°/s back to 60°/s at 157.5°. Each update turns by `min(a, rate × dt)` toward the facing. Confirmed (code).
- **Auto-centre option** (`0x00129f88`, used instead when the per-pad option bytes `0x0050b240` and `0x0050b248` are
  both set; they are 1 in the `level99` save used at runtime and 1 by default,
  [Feel](../feel.md#details-behind-the-table)): nothing below 22.5°; from 22.5° to 90° the rate is `(a − 45°) × 2.444 +
  45°` per second (negative below 26.6°, so the step is then a small turn the other way; 45°/s at 45°, 155°/s at 90°);
  200°/s (`3.4907` rad/s) from 90° to 100°; from 100° to 157.5° it falls linearly (`(157.5° − a) × 2.435 + 60°`) from
  200°/s to 60°/s, unless the call's fourth argument is set, which keeps 200°/s up to 157.5°. That argument is the
  update's local at `sp + 0x1d4`, set at `0x0012b2ac`: 1 when there is one target, he is in a fight stance
  (`Human_IsInFightStance` `0x00228340`, state flags `0x3`) and he still counts for the cameras (`0x00123500`), else 0;
  confirmed (code). So a player squaring up gets the fast swing over the whole range. Beyond 157.5° the rule turns only
  when the target's record has state flag 4 (`0x002265f0(target, 4)`, "has a target" in
  [Combat](../combat/index.md#state-flags), inferred there; the same flag can stand in for "moving").
  `a` is the angle between the target's facing (its rotation in the transform table) and the camera's own horizontal
  forward: vtable slot `+0x224` returns the orientation `+0x20`, whose y axis (`Quat_AxisY` `0x003363b0`), flattened and
  normalised, is the view as placed at the end of the previous update, toward the [aim point](#aim-point) `+0x240`, not
  the look-at point. The facing's y axis is used flattened but not normalised. The angle is `π/2 − atan(c / √(1 − c²))`
  of the cosine `c`, with an atan (`0x003356a0`, a range-reduced rational form) good to about 10⁻⁵° (inferred: one of
  its constants sits in memory Ghidra cannot read). Confirmed (code); confirmed (runtime) in [In the street](#street).
  Each update turns by `min(a, rate × dt)` (`dt` = `+0x1a0`) toward the facing. The cosine of each threshold is computed
  with `0x004b8a70` (cosine, inferred from the thresholds' use).
- **When either rule runs** (`0x0012ae58`, the call at `0x0012bd80`): one target (`+0x444` = 1), the target passes
  `0x00123500` (not in states `0x180050000` of record word `+0x00`), no camera flags in `+0x460 & 0xffff0000`, nothing
  watched (`+0x320` = −1), no yaw from earlier steps this update, and no camera input in the last 0.334 s (`+0x368` =
  0). Inside `0x00129c78` (arguments read at the call, `0x0012bd30`-`0x0012bd84`), confirmed (code): the default rule
  needs "running": the target's gait `+0x1a8` is 4 or 5 with no blocking record flags (`0x00223a60`, `0x00223a98`). The
  auto-centre rule needs "**moving**": gait **2, 4 or 5** (walk, run, sprint; `0x00223a40` adds the walk), or state flag
  4 of the record; **not 0, 1 or 3**, so not while the body moves slower than a walk or at a jog's speed. It also needs
  `+0x474` = 1 (cleared on an update whose stick vector in the per-player record, `+0x00` / `+0x04`, points more than
  157.5° from +y, and set again at the start of any update whose gait is not 0, so it blocks only the updates with the
  stick pulled back that far; whether that vector is the camera-turned stick is not traced), `+0x455` = 0, no
  right-stick input, the top animation task's clip without descriptor flag `0x8000` (`0x00175be8`), none of the human
  flags `0x18003ff0`, and the collision bytes `+0x45b` and `+0x45d` clear: the collision step sets `+0x45b` when its
  main ray from the look-at point is blocked ([World collision](collision.md#collision)), so **auto-follow stops on
  the update after
  one in which the view was blocked**. `+0x45d` is a latch: it is set at `0x00132fa4` when the step's local "view
  blocked" (`sp + 0x364`, set at `0x001327a0` from the blocked local `sp + 0x368`, which `0x0013207c`, `0x001321e4` and
  the swing-away give-up `0x001325a4` set) is true, and `+0x45c` copies `+0x45b` the same way (`0x00132fb4`). Both are
  cleared by `0x00124778` (`0x00124980`) and at the end of the collision step (`0x00132fd0`-`0x0013314c`) on an update
  whose view is not blocked (`sp + 0x364` = 0) when either the camera is no longer held in (its wanted distance from the
  look-at point, `sp + 0xc0` to `sp + 0x20`, is within 10⁻⁵ of the distance the step allows, `sp + 0x348`), or the
  allowed distance plus 3% of that gap reaches the wanted distance the step started from (`sp + 0x34c`), or the zoom
  button was pressed this update (the stick step's flag, the step's stack argument `0x4c8`). Otherwise the wanted
  position is moved out to the allowed distance plus 3% of the gap and the latches stay. Here "allowed distance" is the
  distance from the last look-at point `+0x270` to `+0x2b0`, the point the step returned on the last update, and the 3%
  step is capped at the distance the step started from. While the view is blocked (`+0x45b` or the local set) the wanted
  position is instead drawn to the allowed distance plus **4%** of the gap, and the hold timer `+0x414` starts at 1.5 s
  if it is not running; on the clearing path it starts at 1.5 s (3 s while `+0x460` is set, 0.1 s with `+0x463` and a
  timed move). This is the final form of the rule (`0x00132f5c`-`0x00133250`, read in full). The **recovered distance**
  `+0x380` moves 10% of the way to the allowed distance at the end of every step (`0x00134a18`). Nothing else reads it:
  the constructor zeroes it and the activation sets it to the band's near edge, so it can be left out. Confirmed (code);
  the meaning of the locals is inferred. While the player runs, the leash keeps the wanted position at the band's far
  edge beyond what the walls allow, so the latches hold until he stops (runtime). So **one blocked update keeps
  auto-follow off until the player stops** ([Runtime checks](#runtime-checks)). Confirmed (runtime), slot 1, stick 100 %
  sideways: the rule turned 110-129°/s every update at gait 4; with `+0x45b` written to 1 before each of 21 updates it
  turned 0 on each of them, and 129°/s again on the next. The gaits explain what [In the street](#street) saw: no turn
  in the walk and run start clips (gait 0-1 while the walk start moves at 0.76 m/s, 3 in the run start's middle) or the
  landing clip (4.23 m/s, gait 3), and a turn during the run start's first five updates and the run stop's slower
  updates (gait 2).
- **Keep the target in view** (`0x0012e170(factor, range)`, called with 0.25 and `+0x3fc` for the human
  `CamSetSecondary` gives, `+0x320`, and with 0.4 in one fight case): when the target's direction from the camera is
  more than `fov × factor` from the view's, the camera yaws 35% of the excess per update, at most 270°/s
  (`4.712` rad/s), toward it. With a range above 0 it acts only while the target is within the range and a ray
  (mask `0x200`) from the look-at point reaches it; `level99` passes 0, so no range and no ray. It runs only when
  nothing else turned the camera this update, `+0x463` and `+0x454` are clear, and it replaces auto-follow, which
  needs `+0x320` to be NilHandle. Confirmed (code).
- **The right stick** (rate `+0x358`) and the **heading target** (`+0x350`, step 15 above).
```
