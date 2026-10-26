---
kind: system
system: camera
owns:
  ranges: ["0x0011b770-0x00143ea0"]
---
# Camera

Verified against: `SLUS_212.15` (NTSC-U); methods and earlier runtime provenance are retained in checking notes.

## Purpose

The camera behind the player in normal play: how a level script creates it, what it is configured with, how it is
made current, and what its update does as far as it has been read. It is what the first playable milestone needs to
show the player walking. The lens (field of view, clip planes, view window) is on
[The streamed world](../world.md#player-camera). The other camera kinds ([Types](index.md#types)) are covered from
[Death camera](special.md#death-camera) on, and every function of `Camera/` is in the
[function map](index.md#fn-script).

In one paragraph: there is one **`Cam_Follow`** object per player, a singleton made on first use. `level99.lua` creates
the follow camera with the `global.lua` helper `CameraCreateFollow("follow", player)`, which sets it up on the player
(`CamSetupFollow`) and configures it (`CfgFollowCamera`): distance 3 to 6.6 m (4.8 by default), a pitch of 13°, a 65°
field of view, a near plane of 0.1 and a look-at point 1.4 m above the player's feet. `CameraMakeActive` makes it
current with no blend. It is a leash camera: each update it keeps its look-at point on the player, is dragged back into
a band 0.5 m deep (3.0-3.5 m after `CfgFollowCamera`, which starts it at the minimum; 4.8-5.3 m from checkpoint 2, where
`CamSetFollowZoom(1)` moves it to the default, [Script calls](scripted.md#script-calls)) when the player moves away,
covers 22% of
its wanted move per update, swings round toward the player's facing once the angle passes 22.5° (seen in the street, not
at checkpoint 1, [Runtime checks](follow.md#runtime-checks)), holds a 13° pitch, pulls in to 3.0-3.5 m and lowers its
pitch to 7°
while the player sprints, turns with the right stick at 60-150°/s, and swings or pulls in when the world is in the way,
using ray casts and sphere pushes against the collision mesh. While the player holds a lock-on in a fight it pulls in to
2.4-2.9 m and frames the enemy 27° off centre ([Combat camera](follow.md#combat-camera)). The tutorial's cut-aways are
locked
cameras reached and left with 1 s blends ([Blends](index.md#blends)).

## Map {#original-structure}

`Camera/` (`0x0011b770`-`0x00143ea0`, [Source map](../source-map.md)): the script helpers, then the camera manager and
each camera class in turn, in the order of their vtables (`0x00535250`-`0x005378d0`, `0x2c0` bytes apart, 8-byte
`{delta, function}` slots). Function names are ours unless a class string gives them; every address below has the
same name and a plate comment in the Ghidra project. "Vtable `+0xNN`" is the slot whose function word is at that
offset (the call reads its delta at `+0xNN − 4`). Unless a row says otherwise its evidence is the code at that address.
Confirmed (code).

Outside `Camera/`, the functions this page relies on:

| Address | Name | Role | Evidence |
| --- | --- | --- | --- |
| `0x00367580` → `0x00353818` | `ScenePlayCinematic` (binding) | starts a scene; stores `BlendCam` at the scene's `+0x94` | confirmed (code) |
| `0x0039d870` / `0x0039f450` | scene start / scene end | take the camera over, give it back | confirmed (code) |
| `0x001562c8` | cameras to the device | the lens and draw distance each frame ([The streamed world](../world.md#player-camera)) | confirmed (code) |
| `0x00233c50` | combat camera test | [Combat camera](follow.md#combat-camera) | confirmed (code) |
| `0x0041ab30` / `0x0041ab60` | slow motion on / off | the characters' step `0x005102cc` | confirmed (code) |

### Script helpers {#fn-script}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

The functions the camera bindings call ([camera bindings](../../references/bindings/camera.md)),
`0x0011b770`-`0x0011e1b0`.
Confirmed (code).

See the [camera function index](functions.md) for individual entries.

### The camera manager {#fn-manager}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

`Cam_ICamera.cpp`: the factory, the per-player slots ([Globals](index.md#globals)), the update, the camera stack and the
visibility tests, `0x0011e1b0`-`0x001204f0`. Confirmed (code).

See the [camera function index](functions.md) for individual entries.

### `Cam_3rdPerson` (type 16) {#fn-3rdperson}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See the [camera function index](functions.md) for individual entries.

### The base class `Cam_ICamera` {#fn-base}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

The base camera's own methods, the frustum, the views and the shake, `0x00120868`-`0x00123630`. Its fields are in
[The base camera object](index.md#the-base-camera-object). Confirmed (code).

See the [camera function index](functions.md) for individual entries.

### `Cam_Failed`, the death camera (type 12) {#fn-failed}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See the [camera function index](functions.md) for individual entries.

### `Cam_Fixed` (type 0) {#fn-fixed}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See the [camera function index](functions.md) for individual entries.

### `Cam_Follow` (type 2) {#fn-follow}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

`Cam_Follow.cpp`, `0x00124778`-`0x00134b90`; the behaviour is under [Behaviour](index.md#setting-up). Confirmed (code).

See the [camera function index](functions.md) for individual entries.

### `Cam_Hood` (type 11) {#fn-hood}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See the [camera function index](functions.md) for individual entries.

### `Cam_Locked` (type 1) {#fn-locked}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

[Locked cameras](scripted.md#locked-cameras).

See the [camera function index](functions.md) for individual entries.

### `Cam_Mini`, the mini-game camera (type 8) {#fn-mini}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

`Cam_Mini.cpp`: the shots of the mount and stereo-theft mini-games
([Mini-game and mugging cameras](special.md#mini-mug)).

See the [camera function index](functions.md) for individual entries.

### `Cam_Mug`, the mugging camera (type 7) {#fn-mug}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

`Cam_Mug.cpp`, [Mini-game and mugging cameras](special.md#mini-mug).

See the [camera function index](functions.md) for individual entries.

### `Cam_Power`, the power-move camera (type 6) {#fn-power}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

`Cam_Power.cpp`, [Power-move camera](power.md#power-camera).

See the [camera function index](functions.md) for individual entries.

### `Cam_Rail` (type 9) {#fn-rail}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

[Rail cameras](scripted.md#rail).

See the [camera function index](functions.md) for individual entries.

### `Cam_Scene` (type 4) {#fn-scene}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See the [camera function index](functions.md) for individual entries.

### `Cam_Spline`, the path camera (type 3) {#fn-spline}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

[Path cameras](scripted.md#path-cameras).

See the [camera function index](functions.md) for individual entries.

### `Cam_Transition`, the blend camera (type 5) {#fn-blend}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

[Blends](index.md#blends).

See the [camera function index](functions.md) for individual entries.

### `Cam_Win` (type 13) {#fn-win}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

[Rumble: win camera](../rumble.md#win-camera).

See the [camera function index](functions.md) for individual entries.

### Camera subjects {#subjects}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

| Subject | Page |
| --- | --- |
| Follow tracking | [Follow camera](follow.md) |
| Input and head look | [Look controls](look-controls.md) |
| World probes and recovery | [Collision](collision.md) |
| Fixed, locked, path and rail views | [Scripted cameras](scripted.md) |
| Death, mini-game, mugging and vehicle views | [Special cameras](special.md) |
| Finisher event sequences | [Power camera](power.md) |
| Shake, rumble and character-step scale | [Effects](effects.md) |

## Data

### Types {#types}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

Every camera is one of fourteen classes; each class's vtable function word `+0x1ec` (the `+0x1e8` slot) returns its
type, confirmed (code) for all fourteen ([the list](../../references/cameras.md#type), with tags, vtables and sizes).
The factory `0x0011e1b0(type, name, player)` allocates types 0, 1, 4 and 0x10 afresh; for 2, 3, 5, 7 and 8 it calls a
getter that makes the camera on first use and keeps it (per player in `0x005d9158`-`0x005d918c`, or one in
`0x0050b16c`); for 0xc and 0xd it calls the getter with "do not make" and so only returns an existing one. Types 6,
9 and 0xb are never made by the factory, only by their getters. No class returns 10, 14 or 15. confirmed (code).

### Switches {#switches}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

`CamEnable(switch, on, player)` (`Camera_EnableFeature`, `0x0011de58`) writes one of fourteen flags; a level's camera
reset `0x00122b80` sets them back (switches 1 and 13 to 0, the rest to 1), confirmed (code). The flags, scopes,
defaults and readers are on [the list](../../references/cameras.md#switch). What the readers show, inferred from the
code at the cited addresses:

- **Two-player views.** `0x00121888` decides each update whether each player's view is shown. With switch 3 on and
  two players, a view is kept only while its player still counts (`0x00123500`: 0 while `0x00227e60` holds for the
  human, and for one state of `0x00223b70` unless unlock 6/15 is set), so the other player gets the whole screen;
  switch 13 keeps both views. With switch 3 off both views always show. Switches 4 and 12 apply the same test to
  the follow and rail cameras' targets.
- **Shake** (`0x00121298`): the shake amount drives the pad's rumble byte (pad record `+0x41`) in every case; the
  random view offset is added only while switch 6 is on. Inferred, as above.
- **Right stick** (`0x00129050`): returns at once while `0x0050b1b0[player]` or switch 0 (`0x0050b1b8[player]`) is
  0, so neither the stick nor the zoom buttons act.
- **Look-behind** (switch 11, `0x0050b23c`, confirmed (code) at `0x0012b4b4`): passed to `0x00129050` as its eighth
  argument (forced to 0 unless the player runs or sprints, and while he is tagging:
  [The right stick](look-controls.md#right-stick)). With it 0 the reverse-camera button
  (`0x0050b230`) never turns the view round, and a press on a button shared with the zoom counts as a zoom tap.
  `Human_SetWheelchairControl` clears it.
- **Power camera** (switch 9, `0x0050b1d4`, one flag for both players): read only by `Anim_FireFrameEvents`
  (`0x00101dd8`, at `0x00102da8`, `0x00102f84` and `0x00103274`), which ignores the power camera's begin (`0x39`),
  cut (`0x3a`) and add-shot (`0x3b`) events while it is 0; the end event (`0x3d`) does not read it. Confirmed (code).
  It is 1 from every level's camera reset (`Cameras_ResetLevel` `0x00122b80`) and nothing but `CamEnable` changes it:
  no option, cutscene state or player count. `level99_lesson2.lua` turns it off while the flash dealer respawns, and
  `level55_subway.lua` calls it too ([the list](../../references/cameras.md#cam-switch-9)). What the events do:
  [Power-move camera](power.md#power-camera).
- Switches 2, 7 and 8 are read only by the rail camera (type 9), which `level99` never makes; switch 7 lets the
  leading modes stand the camera off the rail, the lead from the target in plan ([leading](scripted.md#rail-leading),
  confirmed (code)).
- Switches 3 and 4 change nothing with one player: both only gate the second player's view and target
  (`0x00121888`, `0x001282a0`; inferred). `level99` turns them off around its cut-aways and fights.

### Globals {#globals}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

The camera manager's state, confirmed (code) at the functions cited ([The camera manager](index.md#fn-manager)):

| Address | Meaning |
| --- | --- |
| `0x005d9148[2]` | each player's base camera, the one returned to (`0x0011f9c8`) |
| `0x005d9150[2]` | each player's current camera (`0x0011f9b0`) |
| `0x005d9158[2]` / `0x005d9160[2]` / `0x005d9168[2]` | each player's follow / rail / hood camera |
| `0x005d9170[2]` / `0x005d9178[2]` / `0x005d9180[2]` / `0x005d9188[2]` | each player's mug / mini-game / blend / power camera |
| `0x0050b16c` / `0x0050b170` / `0x0050b174` | the one path, death and win camera |
| `0x005d9190[2]`, `0x0050b180` | the camera stack (top index, −1 empty): the camera a scene returns to |
| `0x005d9198[4]` | objects registered with `CamRegisterObject`; message `0x12` goes to each (`0x00123428`) |
| `0x005d91a8[4]` | the shared target list (`CamTarget`); `0x0050b194` its count, refreshed each update |
| `0x005d91b8` | the last target (`CamGetLastTarget`), set by the view decision `0x00121888` |
| `0x0050b198` / `0x0050b19c` | views shown / player cameras, counted by `0x00122ed0` |
| `0x0050b1a0` / `0x0050b1a4` | split mode (`CamSetSplitMode`) / the mode in use (2 with three or more views) |
| `0x0050b1a8` / `0x0050b1aa` | the view grid: columns and rows (mode 0: 2 × 1, 1: 1 × 2, 2: 2 × 2) |
| `0x0050b1e8[2]` | per player: his view may show (`0x00121e30`) |
| `0x0050b200` | field-of-view scale when the screen splits (at least 40°) |
| `0x0050b204` / `0x0050b208` / `0x0050b20c` | game aspect, overlay aspect, overlay scale ([Graphics](../graphics.md)) |
| `0x0050b230` / `0x0050b234` | the reverse-camera and zoom buttons |
| `0x00715280[31]`, `0x0050b238` | the moving vehicles (kind 29) the follow camera tilts over (`0x0012ec50`); `WorldObject_Init` adds them |

The [switches](../../references/cameras.md#switch) sit beside them (`0x0050b1b0`-`0x0050b2b4`).

### The base camera object {#the-base-camera-object}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

Every camera class starts with `Cam_ICamera` (constructor `0x00120868`). Confirmed (code) at the constructor and the
cited functions; the lens fields are also on [The streamed world](../world.md#player-camera).

| Offset | Meaning | Constructor |
| --- | --- | --- |
| `+0x00` | vtable | `0x00535510` |
| `+0x10` | position (`float[4]`) | 0 |
| `+0x20` | orientation quaternion (x, y, z, w); rotates the camera's local axes into the world: +y is the view direction, +z up, +x right (see below) | identity |
| `+0x30` | name (15 characters) | |
| `+0x40` | the camera's handle (a slot from `0x3c` to `0x6b`) | |
| `+0x44` / `+0x48` | field of view, degrees / the one used (+ `0x0050b178`, halved in split mode 0) | 60 |
| `+0x4c` | aspect | `0x0050b204` |
| `+0x50` / `+0x54` / `+0x58` | near plane / own far clip / draw distance | 0.3 / 60 / 60 |
| `+0x5c`-`+0x68` | view window (`0x00120a98`) | |
| `+0x70`-`+0xc0` | six frustum planes (normal, distance), inward | |
| `+0xd0`-`+0x14c` | near and far corner points | |
| `+0x150` / `+0x17c` | bounding sphere centre / radius | |
| `+0x180` | look-at point | 0 |
| `+0x1a0` | the last update's `dt` | |
| `+0x1cc` | a script reference (`0x001221a0`, inferred: the end callback) | 0 |
| `+0x1d4` / `+0x1d5` | the ground under the view has type bit `0x20` / `0x10` (`0x00121720`) | 0 |
| `+0x1d6` | the view may show | 1 |
| `+0x1d8` | the player it belongs to (−1 none) | −1 |

**Axes.** Confirmed (code). Every camera class places itself with `Mat_LookAt` (`0x00337028`, eye, target), which
calls `Mat_LookAtUp` (`0x00337168`). That writes a row-per-axis matrix:

| Row | Axis | Value |
| --- | --- | --- |
| `+0x00` | local +x, right | normalise(forward × up) |
| `+0x10` | local +y, forward | normalise(target - eye) |
| `+0x20` | local +z, up | right × forward |

The up hint is world +z (`0x00511700`), or (1, 1, 1) normalised when the forward's z is beyond ±0.98; eye and
target within 0.005 m return 0 and leave the camera as it was. The rows' w and the matrix's position are zeroed.
`Mat_ToQuat` (`0x003365c8`) turns that matrix into the quaternion at `+0x20`. It treats the rows as the rotated
axes (x = (m12 - m21) / 4w with `m[row][col]`), so the quaternion maps local +y to the view direction. This is the
same convention as the [scene cameras](../scenes.md).

Class fields after `+0x1e0`, confirmed (code) at the constructors and setters:

| Class | Fields |
| --- | --- |
| `Cam_3rdPerson` (`0x230`) | `+0x1e0` target; `+0x1f0` angle quaternion; `+0x200` eased orientation; `+0x210` distance (5.1); `+0x214` height (1.8); `+0x220` look-at offset (0, 0, 1.5); field of view 50, near 0.5, far 75 |
| `Cam_Failed` (`0x200`) | `+0x1e0` the human; `+0x1e4` the camera before it; `+0x1e8` height (5); `+0x1ec` turn rate (1, × 15°/s); `+0x1f0`-`+0x1fc` values `CamUseDeathCamera` saves; `+0x1fe` rotating |
| `Cam_Fixed` (`0x210`) | `+0x1e0` offset added to the look-at point (`CameraCreateFixed`'s offset, w = 1); `+0x1f0` the **aim point**: `CamFixed_ComputeLookAt` (`0x00124000`) copies the new look-at point there, `CamFixed_SmoothLookAt` (`0x00124248`) damps it against last update's on its own, the shake moves it, and the camera faces it (`0x00124690`); `+0x200` target; `+0x204` snapped this update (no damping) |

### The follow camera object {#the-follow-camera-object}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#the-follow-camera-object).

### `level99`'s values {#level99-values}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#level99-values).

## Behaviour

### Setting up {#setting-up}

1. `CamSetupFollow(name, target)` (`0x0011bfa8`): get the player's `Cam_Follow`, store the target (slot `+0x1bc`),
   place the camera at the target's position plus the offset at `+0x200`, call `0x00124f38(180, -1)` (a reset of its
   heading and state, inferred) and return the camera's handle. Confirmed (code).
2. `CfgFollowCamera(min, max, default, pitchDegrees, fov, near, {offset}, slowmo)` (`0x0011c0b8`, player 0's follow
   camera only): the three distance setters (each clamps against the others; the default also sets the band to
   default .. default + min(0.5, max − min); the maximum recomputes the lower pitch limit `+0x3b0`), the pitch as both
   the configured and the target pitch, the field of view as both the configured and the wanted one, then
   `0x00125888`: the view is turned to the pitch at once and the field of view eases to the new value over 1 s
   (`+0x39c` = the difference per second). Then the offset (`+0x210`), the near plane (slot `+0x1a4`) and the
   slow-motion factor `0x005148a0`. **Last, with one player camera, the band is moved to the minimum** (3.0-3.5 m) and
   the zoom step to the default, exactly as `CamSetFollowZoom(0)` ([Script calls](scripted.md#script-calls)); with
   two, as
   `CamSetFollowZoom(1)`. Confirmed (code); historical observation ([checking notes](#upstream-earlier-observations)):
   every checkpoint 1 state read band 3.0 / 3.5, zoom
   step 4.8, upper pitch limit 30°.
3. `CameraMakeActive(camera, seconds, ...)` (`0x0011b770` → `0x0011ee08`): with 0 seconds the camera becomes current
   at once; otherwise the blend camera (type 5, `0x0011fac8`) runs between the old and the new one ([Blends](#blends)).
   `CameraReset` (`0x00365a10`) then calls the camera's slot `+0x13c`. Confirmed (code).
4. **Activation** (slot `+0x144`, `0x00125cc0`, whenever the follow camera becomes current directly, including at the
   end of a blend): it updates the glass tasks and lays out the views, then runs the activation step `0x00125e50`. That
   step gathers the targets and snaps the look-at point onto **where the target stands now** (no ease,
   `Cam_Follow_RefreshLookAt` `0x00127d48`), so a camera made current right after a teleport looks at the new place;
   sets the **field of view to the wanted one at once** (slot `+0x194`, `ICamera_SetFov`, with `+0x394`); clears the
   wanted near edge `+0x34c` (−1); and keeps the camera's direction from the look-at point but puts it at its distance
   clamped to the band. Then, with fewer than two player cameras (`0x0050b19c`), a camera whose player index (slot
   `+0x27c`, inferred) is not below their count is reset (`0x00124d00`) and only then gets its hard band
   `+0x33c`/`+0x340` from `+0x334`/`+0x338`, so player 1's camera with one player camera keeps its hard band; with two
   player cameras the lost test (one target and more than 1.3 × the maximum distance across the ground, or more than
   30 m in height) resets it. Last, the recovered distance `+0x380` is set to the band's near edge. Confirmed (code) at
   `0x00125cc0` and `0x00125e50`.

**`CameraReset` on the follow camera** (`0x00124d00(camera, 1)`), confirmed (code): `0x00124f38(180, −1)` places the
camera **behind the target** (heading 180° from its facing, inferred from `0x001250a8`) at a distance picked from the
current one clamped to the band: the minimum when it is at most halfway from the minimum to the default (zoom step then
the default), the default when at most halfway from the default to the maximum (zoom step the maximum), else the maximum
− 0.5 (zoom step the minimum); then the previous look-at points are set to the current one, the wanted near edge
cleared, **the target pitch set back to the configured pitch** (13°) and reached at once, the wanted field of view set
back to the configured one (over 1 s). It moves the band to that distance (`0x00124f38` passes 1 as `0x001250a8`'s last
argument, which sets `+0x32c` to it and `+0x330` to it + min(0.5, max − min); this page said before that it did not move
the band). Last, it turns the camera to look from its new position at the look-at point (`Mat_LookAt` `0x00337028`, then
`Mat_ToQuat` `0x003365c8`) and copies the position and orientation into the last update's camera (`+0x1e0`). **It casts
no ray**: its only callees are `0x00124f38`, `Cam_Follow_ArmSplitTimers`, `Cam_Follow_GetViewPitch`, `Cam_Follow_Pitch`,
`Mat_LookAt`, `Mat_ToQuat` and `Quat_AxisX`/`Quat_AxisY`, and the camera's position and orientation slots (`+0x1bc`,
`+0x1c4` of the vtable `0x00535d50`) are bare stores into `+0x10` and `+0x20` (one `lq`, one `sq`; Ghidra has no
function there). Confirmed (code) at `0x00124d00`. So a camera reset into a wall stays there until the next update's
[collision step](collision.md#collision) (`0x00130990`) moves it, inferred. For a follow camera the other player's
follow camera is
reset too. The placement reads the target where it stands at that moment: `Cam_Follow_PlaceAtHeading` first gathers the
targets and snaps the look-at point onto the target's transform (`Cam_Follow_RefreshLookAt` `0x00127d48`), and its
heading is the target's facing then, so a `CameraReset` right after a teleport goes behind the new place. Confirmed
(code). The distance it starts from is the held position's (`+0x260`) from the look-at point before that snap.

### The follow update {#update}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#update).

### The aim point {#aim-point}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#aim-point).

#### The body point {#body-point}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#body-point).

### Long falls {#long-fall}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#long-fall).

#### Coney's difference {#long-fall-coney}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#long-fall-coney).

### Climbs {#climbs}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#climbs).

### Heading {#heading}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#heading).

### Pitch {#pitch}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#pitch).

### Sprint zoom {#sprint-zoom}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#sprint-zoom).

### The right stick {#right-stick}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](look-controls.md#right-stick).

#### Buttons {#buttons}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](look-controls.md#buttons).

#### Looking behind {#reverse}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](look-controls.md#reverse).

### Ledges {#ledge}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#ledge).

### Height hold {#height-hold}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#height-hold).

### Aiming {#aiming}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#aiming).

### Head look {#head-look}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](look-controls.md#head-look).

### World collision {#collision}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](collision.md#collision).

#### The step's helpers {#collision-helpers}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](collision.md#collision-helpers).

### Runtime checks {#runtime-checks}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#runtime-checks).

### In the street {#street}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#street).

### Scenes take the camera and give it back {#scenes}

The rest of scene playback (records, roles, letterbox, skipping) is on [Scenes](../scenes.md).

`level99` starts with `SuperRunScene(IntroScene)` at checkpoint 1 ([Scripts](../scripting.md)); `IntroScene` is a table
(`SceneId` `l99_c1`, the humans and objects that take part, `ReturnFunc` = `P1.StartTraining`). The `global.lua`
helpers fill in defaults and call the engine. Confirmed (code) for the script (`global.lua`, read as bytecode) and the
engine at the cited addresses:

1. **`SuperRunScene(t)`** hides the HUD, clears the gang's wanted level, blacks the screen at once
   (`ScreenQueueEffect(1, 0)`), revives the scene's humans, and preloads the scene (`ScenePreload(id,
   "gPlayCutScene")`), keeping `t` in `tblScene[id]`.
2. **`gPlayCutScene(id)`** sets defaults: `Bars` true, `Delay` 0, and when `BlendCam` is not given, **`BlendCam` = −1
   and `FadeIn` true** (a given `BlendCam` sets `FadeIn` false). It joins each human to the scene (`GoalJoinCinematic`,
   or the animation / fixed-scene variants), adds the objects, and calls `ScenePlayCinematic(id, Delay,
   "PreCashTheWorld", Bars, not NoSkip, Looping, Freeze, BlendCam, Final, Chain)`.
3. **Scene start** (`0x00353818` → `0x0039d870`). When the scene has its own camera (scene data `+0x22`), a scene
   camera (type 4, `0x0011e1b0(4, …)`) is made or reused, given the current camera's view (slot `+0xac` → `+0xb4`),
   and **the current camera is pushed** on the camera stack (`0x00120450`; for a blend, locked or other wrapper camera,
   types 5-8, the camera inside it). The scene camera then becomes current at once (`0x0011ee08` with 0 seconds).
   A scene without a camera keeps the current one, saves its position (scene `+0x80`, the camera at `+0x90`) and
   moves it to the scene's anchor. Confirmed (code).
4. **While a scene camera is current**, `CameraMakeActive` for player 1 does not switch: it replaces the camera on the
   stack (`CameraStack_Pop` then `CameraStack_Push`), so the script changes what the scene returns to. The test in
   `Camera_MakeActive` (`0x0011ee08`) is the byte `0x0051489c + 0x410` non-zero, player index 0, and player 0's
   current camera of type 4. That byte is the **cinematic scene state** of [Scenes](../scenes.md) (start step 4: 0 → 1 →
   2 → 3, back to 0 at the end), so only a cinematic scene, not an in-game one, redirects the call. Confirmed (code).
5. **Scene end** (`0x0039f450`): the camera is **popped** (`0x00120488`), reset (slot `+0x13c`, as `CameraReset`) and
   made current with **`BlendCam` seconds**: above 0 the blend camera (type 5) runs between the scene camera and it;
   0 or −1 is a cut. In one game-mode case (the mode object at `0x0015e718` reporting 8) it first takes the scene
   camera's view, field of view and near plane. The scene camera is then released (`0x0011e440`) and **the cameras'
   update runs once with dt = 0.17 s** (`0x0011e878(0.17)`), so the follow camera settles before the next frame. A
   cinematic **given up** before it starts (`SceneTask_Abort`, `0x0039ec60`, after 10 s,
   [Scenes](../scenes.md#starting))
   takes none of these steps: after placing the scene's humans and objects at their end poses it puts the kept camera
   of step 3 (task `+0x90`, when there is one) back at its saved position (task `+0x80`, through the camera's slot
   `+0x1bc`) and clears the pointer, then resets player 1's camera, and player 2's when there is one, through the same
   slot `+0x13c`, with no pop, blend or settling update.
   Confirmed (code) at `0x0039ec60`.
6. **The script's end callback** (`global.lua`, run when the scene ends) calls `ReturnFunc` (for the intro,
   `P1.StartTraining`), turns gang spotting back on unless `BlendCam` was 0, and with `FadeIn` fades the screen in over
   0.5 s (`ScreenQueueEffect(0, 0.5)`).

Earlier observation: see the [recorded values and method](#upstream-earlier-observations).

### Script calls in `level99` {#script-calls}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](scripted.md#script-calls).

### Script calls in `level95` {#level95-calls}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](scripted.md#level95-calls).

#### The door walks {#door-walks}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](scripted.md#door-walks).

#### At runtime in the clubhouse {#level95-runtime}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](scripted.md#level95-runtime).

#### The first visit's opening, at runtime {#level95-first-visit}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](scripted.md#level95-first-visit).

### Blends between cameras {#blends}

`CameraMakeActive(camera, seconds > 0)` makes the blend camera (type 5) current when there is a previous camera and
the new one is not itself a blend (`0x0011ee08` → `0x00143078`). Its update (`0x00143590`), each step, confirmed
(code):

1. Runs the destination camera's own update (and the source's, when it is another camera), so the follow camera keeps
   leashing to the player during the blend.
2. `t = min(elapsed / seconds, 1)`, elapsed counting by `dt`.
3. Look-at point = `lerp(start look-at, destination's look-at, t)`; orientation = a slerp of the start matrix toward
   the destination's by `t` (`0x00336a00`); position = `lerp(start position, destination's position, t)`, then pushed
   out of the collision mesh (`CollisionMesh_SpherePush`). The start values are the source's view when the blend
   began: linear in time, no easing. Confirmed (code).
4. The field of view is the destination's; a second lens value (slot `+0x20c`, inferred the far clip) is
   `min(current, lerp(start, destination, t))`.
5. When `elapsed ≥ seconds`, the destination becomes current directly (`0x0011ee08` with 0 s), which runs its
   activation ([Setting up](#setting-up), step 4). Confirmed (code).

### Locked cameras {#locked-cameras}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](scripted.md#locked-cameras).

### Scripted camera angles {#scripted-angles}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](scripted.md#scripted-angles).

### Path cameras {#path-cameras}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](scripted.md#path-cameras).

### Rail cameras {#rail}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](scripted.md#rail).

### Fixed and third-person cameras {#fixed-third}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](scripted.md#fixed-third).

### Combat camera {#combat-camera}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](follow.md#combat-camera).

### Death camera {#death-camera}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](special.md#death-camera).

### Mini-game and mugging cameras {#mini-mug}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](special.md#mini-mug).

#### Shot tables {#mini-mug-shots}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](special.md#mini-mug-shots).

### Power-move camera {#power-camera}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](power.md#power-camera).

#### The events {#power-events}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](power.md#power-events).

#### Fields {#power-fields}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](power.md#power-fields).

#### Begin {#power-begin}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](power.md#power-begin).

#### Shots {#power-shots}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](power.md#power-shots).

#### Choosing the shots {#power-choice}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](power.md#power-choice).

#### The cut {#power-cut}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](power.md#power-cut).

#### Holding a shot {#power-hold}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](power.md#power-hold).

#### The end and the hand-back {#power-end}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](power.md#power-end).

#### Slow motion and shake {#power-slowmo}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](power.md#power-slowmo).

#### Which clips {#power-clips}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](power.md#power-clips).

#### Two players and other humans {#power-players}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](power.md#power-players).

#### At runtime {#power-runtime}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](power.md#power-runtime).

#### For the implementer {#power-coney}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](../../engine/camera.md#power-coney).

### Hood and third-person cameras {#hood-third}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](special.md#hood-third).

### Two players {#two-players}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

With two player cameras (`0x0050b19c` = 2) the follow and rail cameras decide each update whether the players share one
view or split the screen. Confirmed (code) at `0x00126c40` and `0x0013c710`; the purpose of each test is inferred:

- **Merging** needs both players inside the shared view, within 11 m of each other and neither colour controller
  fading; **splitting** follows when that stops holding. Either waits until its timer has run 0.72 s (follow,
  `+0x41c` / `+0x420`) or 0.5 s (rail, `+0x3dc`).
- On a change (`0x001276a0`) the views are re-laid out (`0x00122ed0`: one view, or the grid of the split mode), the
  band and zoom step re-applied for the new count, the target pitch set to 25°, and one or two updates later a short
  shake and a white flash (alpha 225, fading over 0.2 s) on the colour controller (`0x00126a88`). Confirmed (code).
- When the screen splits, each view's field of view is scaled by `0x0050b200` (at least 40°), and split mode 0 halves
  the view window's width ([The camera manager](index.md#fn-manager)). Confirmed (code).

### Shake and rumble {#shake}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](effects.md#shake).

#### Shakes from the world: the passing train {#world-shake}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](effects.md#world-shake).

### Slow motion {#slow-motion}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See [Camera subject](effects.md#slow-motion).

## In Coney

Camera ownership, current port behaviour and implementation checks are on
[Camera implementation](../../engine/camera.md#coneys-implementation).

### Notes for implementers {#notes-for-implementers}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

See the [implementation checks](../../engine/camera.md#notes).

## Open questions

- **The landing update after a long fall**: in the last one or two updates in the air Coney's view tips up to
  43-68°, while the original's stays where it was ([Long falls](../../engine/camera.md#long-fall-coney)). Inferred
  from the traces; what the
  original's follow camera does on the landing update is unread.
- **Look-behind in a fight** (answered, [Looking behind](look-controls.md#reverse)): R1 turns the view round only
  while the player
  runs or sprints (gait 4 or 5, record `+0x08` clear), so never while he blocks or walks locked on. Confirmed (code)
  at `0x0012b4a0`; a runtime check (R1 held at a standstill, at a walk and at a run) is still to be made.
- **The size of `Cam_Follow_RightStick`** (answered): 2,160 bytes, `0x00129050`-`0x001298bf`, ending with its only
  `jr ra`; nothing calls or jumps to `0x00129100` (176 bytes in, the `addiu` that completes the address of switch 0
  `0x0050b1b8` after a `lui`), so that address is not a function start. Confirmed (code).
- **R1 at a run: block or look-behind?** (answered, [Looking behind](look-controls.md#reverse)): the block's fight
  test fails at a
  run or sprint, so R1 keeps gait 4, the 7.80 m/s and the run clip and turns the view; the block starts the update
  the gait drops below 4. Confirmed (code) at `0x00224f28`; confirmed (runtime).
- **The look-behind's band** at a run: the near edge `+0x32c` read 6.1 m while R1 was held (from 4.8), not the 8.5 m
  `0x0012d7a8` is given; which field holds the 8.5 m is open.
- **The circling rate** (answered): the original circles at about 190°/s at a run and 143°/s at a walk; the 122°/s
  and 127°/s first given were the rule's share ([In the street](follow.md#street)). The rule's `a` is measured from the
  camera's orientation, which faces the aim point. Measured that way, the formula matches the turn exactly.
- **The sprint zoom** (answered, [Sprint zoom](follow.md#sprint-zoom), with its gates: no enemies or the nearest
  within 12 m,
  and `0x0050b19c` the number of player cameras).
- **The collision step** (answered, [World collision](collision.md#collision)): the table of fractions is never read,
  and the side
  angle history (`+0x428`-`+0x430`) drives the centring in a gap.
- **Slow motion's clips** (answered, [Slow motion](effects.md#slow-motion)): grab and mount power moves, tandem ends,
  the
  bottle smash and one boss counter.
- **The combat camera's 0.4 keep-in-view** (answered): `0x0012e170(0.4)` keeps the player's brain target (brain
  `+0x124`) in view while he is in a fight stance that counts for the camera and no
  secondary target is watched ([update step 9](follow.md#update)).
- **Teleports** (answered): `Teleport` / `TeleportToFlag` do not touch the cameras. Confirmed (code): neither
  `Object_Teleport` (`0x00385bb8`) nor `Object_TeleportToFlag` (`0x00385db0`) calls the camera code. Neither do the
  human's set-transform slot (`Human_SetTransform` `0x0023d440`: pair break, ground ray, brain heading, "moved" marks
  `+0x1b3` and `+0x54` bit `0x40000000`) or slot `+0x14c` (`Human_ResetMotion` `0x0023f158`: velocities and turn
  zeroed, transform re-applied), and the only reader of `+0x1b3` is `Human_GetNavPolygon`. So the follow camera sees
  only the target's new position. Its look-at point follows a move of more than 0.8 m at 20% of the gap per update
  ([update step 5](follow.md#update)), and once the held position is more than 1.3 × the maximum distance from the
  look-at
  point (or 30 m above or below it), step 7 puts it behind the player as `CameraReset` does, with one player camera
  only. A script that wants a cut calls `CameraReset` itself (inferred: the camera's path for a long move follows from
  those two steps; not seen at runtime). Coney builds both steps and no longer cuts after a placement
  ([Placing a human](../teleports.md#camera)).
- **The power camera's events and shots** (answered, [Power-move camera](power.md#power-camera)): begin `0x39`, add shot
  `0x3b`, cut `0x3a`, end `0x3d`; eight shots at most, in the followed human's frame at the begin; the first cut
  tests them with two sphere sweeps and picks a pair from a counter; every cut and the hand-back are hard cuts.
- **The power camera, still open**: a boss's counter in one-player play places the shots around the player, not the
  boss (code); whether that looks right on screen, and a blocked shot, mode 1 and two players, are not seen at
  runtime. The begin stores the saved camera's fov (`+0x324`) and nothing in the class reads it. The behaviour
  classes that hand their events to the partner (`0x4d`-`0x4f`, `0x6a`, `0x77`, `0x78`, `0x80`, `0xa0`, `0xa1`,
  `0xb2`, `0xb3`, `0xed`) are not identified.

### Coney's repeated `level95` walk {#q-level95-walk}

Which of the causes in [The door walks](scripted.md#door-walks) makes Coney play the opening walk three times. Found
by logging
message 8 and the calls of `StartDoorWalk` in a Coney run of the level.

## How this was checked {#checked}

The executable read is `SLUS_212.15` (NTSC-U, SHA1 `e9cb2cc49aa046b9e494313dce2f5038ed17b2f4`). A runtime claim
on this page that cites no run id predates the run records; it was made in PCSX2 and its method is here or beside
it:

- Most were made in PCSX2 2.9.94 on 2026-10-04 by reading the camera object over PINE in `level99`, checkpoint 1.
- Those of [In the street](follow.md#street) (2026-10-05) in the street saves of `level99`'s world, read every update
  while
  a scripted pad played ([Feel comparison](../feel.md)); the circling rate was re-measured that day and re-recorded on
  2026-10-07 (the `run_circle` scenario).
- Those of 2026-10-06 from the `level99` checkpoint 1 state files' memory and a copy of slot 6 driven over PINE (L1
  held, no stick).
- [Long falls](follow.md#long-fall): the trace of 2026-10-10 (PCSX2 2.9.94) and the several heights of 2026-10-20
  (PCSX2-MCP 2.9.129), asked on [#111](issue:111); Coney's side was re-traced on 2026-10-21.
- The clubhouse walk ([At runtime in the clubhouse](scripted.md#level95-runtime)) and the locked cameras' angles on
  2026-10-07;
  the power camera ([At runtime](power.md#power-runtime)) on 2026-10-11; the passing train's shake on 2026-10-10.

### Earlier inline system observations {#system-inline-observations}

These passages predate the upstream activation/reset corrections. They are retained as
provenance; use [Setting up](#setting-up) for the current reading.

These earlier passages retain their original runtime labels and surrounding list numbering. They are historical
checking notes, not new run records.

```text
1. `CamSetupFollow(name, target)` (`0x0011bfa8`): get the player's `Cam_Follow`, store the target (slot `+0x1bc`),
   place the camera at the target's position plus the offset at `+0x200`, call `0x00124f38(180, -1)` (a reset of its
   heading and state, inferred) and return the camera's handle. Confirmed (code).
2. `CfgFollowCamera(min, max, default, pitchDegrees, fov, near, {offset}, slowmo)` (`0x0011c0b8`, player 0's follow
   camera only): the three distance setters (each clamps against the others; the default also sets the band to
   default .. default + min(0.5, max − min); the maximum recomputes the lower pitch limit `+0x3b0`), the pitch as both
   the configured and the target pitch, the field of view as both the configured and the wanted one, then
   `0x00125888`: the view is turned to the pitch at once and the field of view eases to the new value over 1 s
   (`+0x39c` = the difference per second). Then the offset (`+0x210`), the near plane (slot `+0x1a4`) and the
   slow-motion factor `0x005148a0`. **Last, with one player camera, the band is moved to the minimum** (3.0-3.5 m) and
   the zoom step to the default, exactly as `CamSetFollowZoom(0)` ([Script calls](scripted.md#script-calls)); with two, as
   `CamSetFollowZoom(1)`. Confirmed (code); confirmed (runtime): every checkpoint 1 state read band 3.0 / 3.5, zoom
   step 4.8, upper pitch limit 30°.
3. `CameraMakeActive(camera, seconds, ...)` (`0x0011b770` → `0x0011ee08`): with 0 seconds the camera becomes current
   at once; otherwise the blend camera (type 5, `0x0011fac8`) runs between the old and the new one ([Blends](index.md#blends)).
   `CameraReset` (`0x00365a10`) then calls the camera's slot `+0x13c`. Confirmed (code).
4. **Activation** (slot `+0x144`, `0x00125cc0`, whenever the follow camera becomes current directly, including at the
   end of a blend): it recounts the targets, snaps the look-at point (no ease), keeps the camera's direction from the
   look-at point but puts it at its distance clamped to the band, sets the hard band to the band, clears the wanted
   near edge `+0x34c` (−1) and sets the recovered distance `+0x380` to the band's near edge. Confirmed (code).

So the intro hands back to the follow camera with a **cut hidden by a 0.5 s fade-in**, the follow camera having been
reset and run for 0.17 s. Confirmed (runtime): after the intro the stack index `0x0050b180` is −1, its slot 0 still
holds the popped follow camera, and the follow camera is the current and previous camera of player 1.

- **The landing update after a long fall**: in the last one or two updates in the air Coney's view tips up to
  43-68°, while the original's stays where it was ([Long falls](../../engine/camera.md#long-fall-coney)). Inferred from the traces; what the
  original's follow camera does on the landing update is unread.
- **Look-behind in a fight** (answered, [Looking behind](look-controls.md#reverse)): R1 turns the view round only while the player
  runs or sprints (gait 4 or 5, record `+0x08` clear), so never while he blocks or walks locked on. Confirmed (code)
  at `0x0012b4a0`; a runtime check (R1 held at a standstill, at a walk and at a run) is still to be made.
- **The size of `Cam_Follow_RightStick`** (answered): 2,160 bytes, `0x00129050`-`0x001298bf`, ending with its only
  `jr ra`; nothing calls or jumps to `0x00129100` (176 bytes in, the `addiu` that completes the address of switch 0
  `0x0050b1b8` after a `lui`), so that address is not a function start. Confirmed (code).
- **R1 at a run: block or look-behind?** (answered, [Looking behind](look-controls.md#reverse)): the block's fight test fails at a
  run or sprint, so R1 keeps gait 4, the 7.80 m/s and the run clip and turns the view; the block starts the update
  the gait drops below 4. Confirmed (code) at `0x00224f28`; confirmed (runtime).
- **The look-behind's band** at a run: the near edge `+0x32c` read 6.1 m while R1 was held (from 4.8), not the 8.5 m
  `0x0012d7a8` is given; which field holds the 8.5 m is open.
- **The circling rate** (answered): the original circles at about 190°/s at a run and 143°/s at a walk; the 122°/s
  and 127°/s first given were the rule's share ([In the street](follow.md#street)). The rule's `a` is measured from the
  camera's orientation, which faces the aim point. Measured that way, the formula matches the turn exactly.
- **The sprint zoom** (answered, [Sprint zoom](follow.md#sprint-zoom), with its gates: no enemies or the nearest within 12 m,
  and `0x0050b19c` the number of player cameras).
- **The collision step** (answered, [World collision](collision.md#collision)): the table of fractions is never read, and the side
  angle history (`+0x428`-`+0x430`) drives the centring in a gap.
- **Slow motion's clips** (answered, [Slow motion](effects.md#slow-motion)): grab and mount power moves, tandem ends, the
  bottle smash and one boss counter.
- **The combat camera's 0.4 keep-in-view** (answered): `0x0012e170(0.4)` keeps the player's brain target (brain
  `+0x124`) in view while he is in a fight stance that counts for the camera and no
  secondary target is watched ([update step 9](follow.md#update)).
- **Teleports** (answered): `Teleport` / `TeleportToFlag` do not touch the cameras. Confirmed (code): neither
  `Object_Teleport` (`0x00385bb8`) nor `Object_TeleportToFlag` (`0x00385db0`) calls the camera code. Neither do the
  human's set-transform slot (`Human_SetTransform` `0x0023d440`: pair break, ground ray, brain heading, "moved" marks
  `+0x1b3` and `+0x54` bit `0x40000000`) or slot `+0x14c` (`Human_ResetMotion` `0x0023f158`: velocities and turn
  zeroed, transform re-applied), and the only reader of `+0x1b3` is `Human_GetNavPolygon`. So the follow camera sees
  only the target's new position. Its look-at point follows a move of more than 0.8 m at 20% of the gap per update
  ([update step 5](follow.md#update)), and once the held position is more than 1.3 × the maximum distance from the look-at
  point (or 30 m above or below it), step 7 puts it behind the player as `CameraReset` does, with one player camera
  only. A script that wants a cut calls `CameraReset` itself (inferred: the camera's path for a long move follows from
  those two steps; not seen at runtime).
- **The power camera's events and shots** (answered, [Power-move camera](power.md#power-camera)): begin `0x39`, add shot
  `0x3b`, cut `0x3a`, end `0x3d`; eight shots at most, in the followed human's frame at the begin; the first cut
  tests them with two sphere sweeps and picks a pair from a counter; every cut and the hand-back are hard cuts.
- **The power camera, still open**: a boss's counter in one-player play places the shots around the player, not the
  boss (code); whether that looks right on screen, and a blocked shot, mode 1 and two players, are not seen at
  runtime. The begin stores the saved camera's fov (`+0x324`) and nothing in the class reads it. The behaviour
  classes that hand their events to the partner (`0x4d`-`0x4f`, `0x6a`, `0x77`, `0x78`, `0x80`, `0xa0`, `0xa1`,
  `0xb2`, `0xb3`, `0xed`) are not identified.
```

### Earlier observations retained from upstream {#upstream-earlier-observations}

These passages came from upstream 8ecd4cb8 without surviving run identifiers. Their original labels and
context are retained as earlier checking notes, not a new runtime comparison.

```text
1. `CamSetupFollow(name, target)` (`0x0011bfa8`): get the player's `Cam_Follow`, store the target (slot `+0x1bc`),
   place the camera at the target's position plus the offset at `+0x200`, call `0x00124f38(180, -1)` (a reset of its
   heading and state, inferred) and return the camera's handle. Confirmed (code).
2. `CfgFollowCamera(min, max, default, pitchDegrees, fov, near, {offset}, slowmo)` (`0x0011c0b8`, player 0's follow
   camera only): the three distance setters (each clamps against the others; the default also sets the band to
   default .. default + min(0.5, max − min); the maximum recomputes the lower pitch limit `+0x3b0`), the pitch as both
   the configured and the target pitch, the field of view as both the configured and the wanted one, then
   `0x00125888`: the view is turned to the pitch at once and the field of view eases to the new value over 1 s
   (`+0x39c` = the difference per second). Then the offset (`+0x210`), the near plane (slot `+0x1a4`) and the
   slow-motion factor `0x005148a0`. **Last, with one player camera, the band is moved to the minimum** (3.0-3.5 m) and
   the zoom step to the default, exactly as `CamSetFollowZoom(0)` ([Script calls](scripted.md#script-calls)); with
   two, as
   `CamSetFollowZoom(1)`. Confirmed (code); confirmed (runtime): every checkpoint 1 state read band 3.0 / 3.5, zoom
   step 4.8, upper pitch limit 30°.
3. `CameraMakeActive(camera, seconds, ...)` (`0x0011b770` → `0x0011ee08`): with 0 seconds the camera becomes current
   at once; otherwise the blend camera (type 5, `0x0011fac8`) runs between the old and the new one ([Blends](#blends)).
   `CameraReset` (`0x00365a10`) then calls the camera's slot `+0x13c`. Confirmed (code).
4. **Activation** (slot `+0x144`, `0x00125cc0`, whenever the follow camera becomes current directly, including at the
   end of a blend): it updates the glass tasks and lays out the views, then runs the activation step `0x00125e50`. That
   step gathers the targets and snaps the look-at point onto **where the target stands now** (no ease,
   `Cam_Follow_RefreshLookAt` `0x00127d48`), so a camera made current right after a teleport looks at the new place;
   sets the **field of view to the wanted one at once** (slot `+0x194`, `ICamera_SetFov`, with `+0x394`); clears the
   wanted near edge `+0x34c` (−1); and keeps the camera's direction from the look-at point but puts it at its distance
   clamped to the band. Then, with fewer than two player cameras (`0x0050b19c`), a camera whose player index (slot
   `+0x27c`, inferred) is not below their count is reset (`0x00124d00`) and only then gets its hard band
   `+0x33c`/`+0x340` from `+0x334`/`+0x338`, so player 1's camera with one player camera keeps its hard band; with two
   player cameras the lost test (one target and more than 1.3 × the maximum distance across the ground, or more than
   30 m in height) resets it. Last, the recovered distance `+0x380` is set to the band's near edge. Confirmed (code) at
   `0x00125cc0` and `0x00125e50`.

So the intro hands back to the follow camera with a **cut hidden by a 0.5 s fade-in**, the follow camera having been
reset and run for 0.17 s. Confirmed (runtime): after the intro the stack index `0x0050b180` is −1, its slot 0 still
holds the popped follow camera, and the follow camera is the current and previous camera of player 1.
```
