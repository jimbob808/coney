---
kind: topic
system: camera
owns:
  functions:
    - "0x001204f0"
    - "0x001205c0"
    - "0x001205e8"
    - "0x001207b0"
    - "0x00123630"
    - "0x001236b8"
    - "0x001236e0"
    - "0x00123888"
    - "0x001238a8"
    - "0x00123ba0"
    - "0x00123bf0"
    - "0x00134b90"
    - "0x00134c80"
    - "0x00134ca8"
    - "0x00134d28"
    - "0x00134d78"
    - "0x00134f08"
    - "0x00135028"
    - "0x00135138"
    - "0x00136068"
    - "0x00136160"
    - "0x00136188"
    - "0x001361a8"
    - "0x001363e8"
    - "0x00136a88"
    - "0x00136f68"
    - "0x001374d8"
    - "0x00137670"
    - "0x00137808"
    - "0x00137c68"
    - "0x00137cb0"
    - "0x00137cd0"
    - "0x00137dc0"
    - "0x00137de8"
    - "0x00137e08"
    - "0x00138078"
    - "0x00138b90"
    - "0x00139d00"
    - "0x00139d48"
    - "0x00143918"
    - "0x001439b0"
    - "0x001439d8"
    - "0x00143c58"
    - "0x00143c78"
---
# Special-purpose cameras

Verified against: `SLUS_212.15` (NTSC-U); this page preserves the earlier camera research's findings.

## Summary

The death, mini-game, mugging, hood and third-person cameras: how each enters, frames its subject and gives the
view back. Their shared manager and base state remain on [Camera](index.md#fn-manager).

## Map

| Address | Function | Subject | Evidence |
| --- | --- | --- | --- |
| `0x001205e8` | `Cam3rdPerson_Update` | third-person camera | confirmed (code) |
| `0x00123630` | `CamFailed_Construct` | death camera | confirmed (code) |
| `0x001236e0` | `CamFailed_Activate` | death camera | confirmed (code) |
| `0x001238a8` | `CamFailed_Place` | death camera | confirmed (code) |
| `0x00123bf0` | `CamFailed_Update` | death camera | confirmed (code) |
| `0x001439d8` | `CamWin_Activate` | win camera | confirmed (code) |
| `0x00143c78` | `CamWin_Update` | win camera | confirmed (code) |

### Third-person entries {#fn-3rdperson}

See the [camera function index](functions.md) for the complete entries.

### Death-camera entries {#fn-failed}

See the [camera function index](functions.md) for the complete entries.

### Hood-camera entries {#fn-hood}

See the [camera function index](functions.md) for the complete entries.

### Mini-game entries {#fn-mini}

See the [camera function index](functions.md) for the complete entries.

### Mugging-camera entries {#fn-mug}

See the [camera function index](functions.md) for the complete entries.

### Win-camera entries {#fn-win}

See the [camera function index](functions.md) for the complete entries.

## Data

The mini-game and mugging shot tables and per-camera offsets are recorded with their behaviour below. The common
camera fields are on [the base camera object](index.md#the-base-camera-object).

## Behaviour

### Death camera {#death-camera}

`CamUseDeathCamera(human, ms, ms2)` (`0x0011daa8`) makes player 1's `Cam_Failed` (type 12) current at once, keeping the
camera it replaced (`+0x1e4`), tints the screen over `ms`, starts a blur after `ms2`, hides the HUD and removes the
overhead icons. It is the game-over shot's start (below) with the two times as arguments: the game-over shot is
`(6500, 1500)`, Rumble Survival passes `(7000, 100000)`. Confirmed (code) at the cited addresses, in order:

1. Nothing happens unless `human` resolves and the failed camera can be made (`Cam_GetFailed(1)`).
2. **View flags**: `0x0050b1e8` = 1 and `0x0050b1ec` = 0, player 1's and player 2's "view may show" flags
   (`0x00121e30`): player 1's view on, player 2's off. `Cameras_DecideViews` (`0x00121888`) sets both again on its
   next run (both 1 with one player; worked out from who still counts in co-op), so the write matters only until then
   (inferred: in co-op it drops player 2's half of the screen at once).
3. Player 0's current camera goes to `+0x1e4`, and `Camera_MakeActive(0, failed, 0)` cuts to the shot. Its activation
   (`0x001236e0`) picks a target (the last target `0x005d91b8` when it is a player, else whichever player stands
   nearer the camera it replaced) and places itself. Confirmed (code).
4. It saves the screen tint (`+0x1f8`, the manager's target tint `+0x1c4`), the HUD state (`+0x1fc`), the ambience
   level (`+0x1f0`) and the music volume (`+0x1f4`).
5. **Target**: `CamFailed_SetTarget(failed, human)` (`0x00123888`) then sets `+0x1e0` = `human` and places the camera
   again (a new random heading). So `human` is the one the shot looks down on; the activation's pick is overwritten.
   Confirmed (code).
6. **Tint**: `ScreenFx_BlendTintTo(ms / 1000, 0x005fdeb8, 0xd0000014)` blends player 1's screen-effects manager
   (`0x005fdeb8`, the colour controller the death camera watches) from its current tint to `0xd0000014`, the same dark
   red at 81% as the game-over shot ([Rendering](../rendering.md#tint)), over `ms` milliseconds. Confirmed (code).
7. **Blur pulse**: `ScreenFx_StartBlurPulse(ms / 1000, 0x005fdeb8, 0, ms2)` arms the same manager's blur
   ([Graphics](../graphics.md#screen-effects)): its level stays 0 until `ms2` milliseconds have passed (`+0x204`, tested
   by `0x0018d1d0` against the start time `+0x208`), then rises to full over `ms` / 1000 s and holds. With Rumble's
   100000 the blur waits 100 s, so in practice the arena's death has the tint and no blur (inferred).
8. `HUD_Hide` with the mask of bits 6, 11, 21 and 22, then `Human_RemoveOverheadIcon` (`0x00227388`, message `0x15` to
   the icon at `+0x360`, [Crimes](../crimes.md))
   for every player and for `human`. **Nothing is stopped**: no brain, pad or velocity is touched here.
   Confirmed (code).

**What undoes it.** Nothing in the camera code returns to `+0x1e4`: activation is its only reader. The shot stays
current until a script makes another camera active (`CameraMakeActive`); its deactivation (`0x00123ba0`) then releases
it. But `Camera_MakeActive` (`0x0011ee08`) **refuses** any switch while the player's current camera is type 12 and the
game state's `+0x14c` is 1 (a failure): during a mission failure the death camera cannot be replaced, only after the
state changes (the retry sets it to 0). Confirmed (code). The HUD and tint come back only in `MissionFailed_Toggle`
(`0x00155408`) on a retry from a checkpoint, and only when player 0's current camera is still type 12: the tint is set
back at once to the saved `+0x1f8`, the blur pulse is ended (`0x0018d058` with 0 s, reverse), and the HUD is restored
(`HUD_Restore`) when `+0x1fc` says it was shown. The saved ambience and music volume are not read back there (not traced
elsewhere).

The parts of the shot itself:

- **Target** (`0x001236e0`): the last target (`0x005d91b8`) when it is a player, else whichever player stands nearer
  the camera it replaced; `CamUseDeathCamera` then overrides it with its `human` (step 5). Confirmed (code).
- **Placement** (`0x001238a8`): the look-at point is the human's position; a ray straight up from it, near plane + 5 m
  long (mask `0x200`), gives the height: the camera sits that far above the body, less the near plane, looking
  straight down (a pitch of about −88°) at a random yaw (one of 36 steps of 10° from −180°). Confirmed (code).
- **Each update** (`0x00123bf0`): it turns about the vertical at 15°/s and sinks toward the body at 0.18 m/s, no nearer
  than 2.5 m, until player 1's screen-effects manager has faded out fully ([Fade](../graphics.md#fade)); it then ends
  the blur pulse at once and stops moving. Confirmed (code).

**The game-over shot** (`Cam_GetFailed(1)` in the gameplay mode's update, `0x00158728`), confirmed (code) at the
cited addresses:

1. **When.** `W_GameState + 0x14c` is 1 (a failure: the players are out or busted,
   [Combat](../combat-reactions.md#defeat)), the level is not an Armies level, the game-state flags `+0x152` do not have
   bit 2 (with it the countdown goes to 0 at once and the menu comes with no shot), and player 0's camera is not already
   type 12. In player terms:
   - **Armies levels** (`GameState_IsArmiesLevel`, levels 60-64) never get the shot: their failure counts down
     at most 90 updates (3 s, the same cap as a level completed) with the current camera still running, then
     pushes the Armies game-over (`ANGameOver_Toggle`) instead of `MissionFailed_Toggle`.
   - **`+0x152` bit 2 means a player fell out of the world.** Its only writer besides the level reset (0,
     `GameState_ResetForLevel`) is `Human_StateUpdate` at `0x002403ec`. When player 1's or player 2's human
     (handles `+0x228`, `+0x22c`) is more than 20 m below the collision mesh's lowest vertex
     ([Characters](../characters.md#ground)), it sets the failure reason to global string `0x13`
     (`MissionFailed_SetReason`, `0x001d1fb8`), `+0x118` = 0, `+0x14c` = 1 and `+0x152` = 2. It does this only while
     game-state `+0x150` has bit 2, which `GameState_ResetForLevel` sets (7) and nothing else writes: always, in
     practice (inferred). That same update the countdown goes to 0 and the mission-failed menu is pushed: no shot,
     no tint, no wait.
   - The shot starts only while the failed camera does not yet exist (`Cam_GetFailed(0)` = 0); later updates of the
     same failure find it and skip the start. The slot `0x0050b170` is cleared again by `Cameras_ClearSlots`
     (`0x0011ea68`) and `Camera_Release` (`0x0011e7e0`), so each new level's failure gets a fresh shot (inferred).
2. **Start**, once: `Cam_GetFailed(1)` makes the camera on first use (`0x0050b170`); player 0's current camera is kept
   in `+0x1e4`, and `Camera_MakeActive(0, failed, 0)` makes the shot current with **no blend** (a cut), which runs the
   activation above (target, then placement). It saves the screen tint (`+0x1f8`), the HUD state (`+0x1fc`), the
   ambience level (`+0x1f0`) and the music volume (`+0x1f4`), starts a **6.5 s** tint blend to `0xd0000014` (bytes red
   0x14, green 0, blue 0, alpha 0xd0: a dark red at 104/128, 81%, [Rendering](../rendering.md#tint)) and a blur pulse
   (1500
   ms), hides the HUD and removes the overhead icons. Confirmed (code).
3. **Placement** (`0x001238a8`): the look-at point is the human's transform position (the feet); a ray straight up
   from it, near plane + 5 m long, gives the height (hit distance − near plane: **5 m** with nothing above); the
   camera is put that far above the feet, pitched **−88.28°** about its x axis (a half-angle of −0.7704 rad), then
   turned about its own forward axis, which points almost straight down, by a random multiple of 10° from −180°
   (`Random_Int(36)`): to the eye, a top-down shot at a random heading. Confirmed (code).
4. **Motion**, each update while rotating (`+0x1fe` = 1): it turns about its own forward (view) axis at turn rate
   (`+0x1ec`, 1) × **15°/s**, and its distance to the look-at point shrinks by **0.18 m/s** × the step, never below
   **2.5 m**; the camera stays on the view axis, so it sinks straight down toward the body while the picture turns.
   It stops (and starts the next screen pulse) once the screen effect's blend has finished (`+0x1d8` ≥ 1 and
   `+0x1dc` > 0 on `0x005fdeb8`). Confirmed (code).
5. **Length.** The level-end countdown is **180 updates** (6 s); cross with more than 10 left cuts it to 10 and
   speeds the tint blend up to 1/3 s. Each update the ambience level is lowered toward 0.7 (by 0.7 / (0.9 × the
   updates left)). At 0 the tint blend is finished, the ambience and music volumes are put back and the
   mission-failed menu (mode `0xc`) is pushed; leaving it for a retry puts the saved tint back and restores the HUD.
   Over a full 6 s the shot turns **90°** and sinks from 5 m to about **3.9 m**; the 6.5 s tint blend has not
   finished by then, so the stop in step 4 is not reached before the menu (inferred from the two lengths; the menu
   stops the gameplay update, so the shot then holds).

In Coney the death camera is `repo:src/camera/failed_camera.h`, and `Cameras::startFailed()` cuts to it once per
level. It keeps the base camera's lens (60°, near 0.3, far 60) because its constructor sets none of its own
(inferred). Other choices:

- **Coney's reading**: it stays straight above the feet as it sinks.
- **Coney choice**: the screen-effect stop is not read.
- The tint, blur, HUD, ambience and music parts of the start are not built yet.
- With one player, the target is always player 1.

### Mini-game and mugging cameras {#mini-mug}

Both are per-player cameras made on first use; both start only from a single-view follow camera, return to it with a
0.3 s blend, and turn the follow camera to the shot's direction first (`Cam_Follow_TurnToMatrix`, `0x0012ded8`) so the
hand-back does not swing. The cut in is not immediate: the start queues the new camera as the follow camera's
hand-over (`+0x47c`, blend `+0x3c0`), the follow camera turns for **0.15 s** (its collision probes off), and only
then does the new camera become current, blending over **0.08 s** for a mini-game shot (0 for kind 1) or **0.15 s**
for a mugging (`CamMini_Start` `0x001363a4`, `CamMug_Start` `0x0013804c`). On the way back the mugging camera turns
the follow camera again (0.35 s, nothing queued; `CamMug_Update` `0x00138ec8`). Confirmed (code) at the cited addresses;
the shot tables' purpose (variety) is inferred.

- **The mini-game camera** (`Cam_Mini`, type 8, `0x001361a8`) frames two humans (kind 0, the mount from
  `Player_UpdateMounting`) or a human and the stereo (kind 2, `StereoTheft_Start`); no caller passes kind 1. The shot
  is 3 m from a point on the first human (the mounting player's feet + 0.5 m, or the stereo + 0.3 m), at a yaw and
  pitch drawn at random from small tables; up to three draws are tested with a sphere sweep (`0x001374d8`, clear for
  at least 2.3 m), then a fixed shot (pitch 36°; 30° and yaw 25° for the stereo; 32° for kind 1)
  ([shot tables](#mini-mug-shots)). Each update turns it at up to 3 × its angle error per
  second to keep both in frame. The mini-game's end (`0x0022d1f8`) sets its done flag (`+0x258`), and the next update
  blends back. Confirmed (code).
- **The mugging camera** (`Cam_Mug`, type 7, `0x00137e08`, from `0x002729a8`) plays `vags/misc/mug_intro` as it cuts
  in. The shot (`0x00138078`) is 2.5 m from the pair (1.8 m when the world is in the way) at a random yaw and pitch
  from small tables. Each update (`0x00138b90`) eases the field of view to 50° over 0.33 s, turns at up to 135°/s to
  keep both humans framed and stays 3 m from other humans (`Camera_PushOutOfHumans`, `0x00122728`). When the mugger or
  the victim leaves the hold it stops the intro, plays `vags/misc/mug_outro` and blends back. Confirmed (code).

#### Shot tables {#mini-mug-shots}

The random draws use `Random_IntRange(low, high)` (`0x003353f0`, both ends included). Angles in degrees; "yaw" is a
turn about the vertical, "pitch" about the shot's side axis. Confirmed (code) at each function.

**Mount** (`CamMini_PlaceKind0`, `0x001363e8`; `+0x1e0` the mounting player), confirmed (code):

1. The point is his feet + min(max(offset z, 0), 0.5) m (+0.5 m with the usual offset), the distance 3 m
   (`+0x254`).
2. The base direction is his side axis: both sides are tested at 3 m (`CamMini_TestShot` `0x001374d8`, a sphere
   sweep from the point that must stay clear for at least **2.3 m**); the clear side wins, a frame-counter bit
   breaks a tie, and nothing is placed when neither is clear. Confirmed (code).
3. Up to three draws: pitch one of **10, 16.7, 27, 0, 22.5**, yaw one of **−22, 21, 0, 12, −14**. A draw of 0 and
   0 is placed without a test; any other is turned (pitch, then yaw) and tested. The fourth try is **pitch 36°, yaw
   0**; if that fails too the shot is not placed.

**Stereo theft** (`CamMini_PlaceKind2`, `0x00136f68`; `+0x1e0` the stereo, whose car gives the axes; confirmed (code)):

1. The point is the stereo's position + 0.3 m, the distance 3 m (tested for 2.3 m).
2. One roll of the dice per shot (`Random_Int(100)`): under 80 the yaw comes from all eight entries **10, −15, 90,
   30, 115, 60, 75, 48**, else only from the first two.
3. Each try also draws `+0x244` = 3-7° (a float range; its sign flipped on an odd frame counter; its use is not
   traced). The pitch is 0 for yaws under 21.5° or over 91.5°, else one of **17.1, 21.8, 28**.
4. Three tries, then a fixed **pitch 30°, yaw 25°**.

**Mugging** (`CamMug_PlaceShot`, `0x00138078`; `+0x1e0` the mugger, `+0x1e4` the victim; confirmed (code)):

1. The point is the mugger's position + the offset `+0x200`, whose height starts at **1.45 m**; the base
   orientation is the victim's turned 90° about the vertical (so the camera looks across the pair).
2. Pitch `+0x23c`: one of **0, 19, −5, 12.8, 7, −3** (draw 0-5); 19, 12.8 and 7 also raise the point by 0.14,
   0.11 and 0.075 m. Yaw `+0x238`: one of **−4.5, 4.5, 0, 8, −8, −13.5, 13.5** (draw 0-6), only the first five
   when the pitch is 15° or more.
3. Test: a physics sphere of the near-plane radius pushed **2.5 m** + near along the shot (object kinds 0xc-0xe,
   0x10 and 0x1c ignored) and a world ray (materials 30 and 122 skipped) must both stay clear for at least
   **1.8 m**; the camera goes to the ray's hit − the near plane, with the point raised by 2.5 × tan 5°.
4. Two tries: the drawn shot, then the plain one (victim turned 90°, no pitch or yaw); with neither the camera is
   not placed (`+0x24c` = 1).
5. A placed shot is **rolled** (a Dutch tilt), confirmed (code) at `0x00138078` and `0x00138b90`:
   - **The angle** (`+0x234`, radians): **15°** (0.2618), or **11°** (0.1920) when bit 0 of the game time in ms is
     set; then **negated** when bit 1 is set. The game time is the `GameTimer`'s `+0x48` (`0x0050b734`, read
     through its vtable `+0x34`, which returns `+0x48`), so the four cases (+15°, +11°, −15°, −11°)
     come from the shot's start time, about equally often. Confirmed (code).
   - **How it is applied**: the orientation is multiplied on the right by a rotation about the camera's own **+y**
     (its view axis; the constant `0x00511730` = (0, 1, 0, 1)), `q' = q ⊗ (0, sin(r/2), 0, cos(r/2))`, as the
     scripted roll is ([Scripted camera angles](scripted.md#scripted-angles)): a positive angle tips the top of the
     picture
     toward its right. Confirmed (code).
   - **Held for the whole shot**: each update (`CamMug_Update`) rebuilds the orientation by looking from the eye at
     the aim point `+0x210` with the world's up (`Mat_LookAt` `0x00337028`, then `Mat_ToQuat` into `+0x20`),
     keeps that un-rolled orientation in `+0x220`, and applies the same roll again; the angle never eases. The
     shot keeps the tilt until the blend back to the follow camera (0.3 s). When the follow camera is in its
     state 2 (its vtable `+0x1ec`), it is first turned (`Cam_Follow_TurnToQuat`) to the shot's base orientation
     with **no roll** (the victim's facing turned 90°, then the shot's yaw `+0x238` and pitch `+0x23c`); the tilt
     goes away through the blend. Confirmed (code).
   - To draw it, a port needs only the camera's up from the rolled quaternion (`Quat_AxisZ(q)`), as for the
     scripted rolls.

### Hood and third-person cameras {#hood-third}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

- **Hood** (`Cam_Hood`, type 11, `CamSetupHood`): each update (`0x00135138`) it sits on its vehicle at the **mount
  offset `+0x200`** in the vehicle's frame (`0x00134d78`: x along the vehicle's x axis, y along its y axis, z added to
  the height; default **(0, 0.3, 1.7)**), looks at the mean position of the players that still count raised by the
  **look offset `+0x1f0`**'s z (`0x00134f08`, which reads `+0x1f8` only; default **(0, 0, 1.53)**), and steps its
  field of view toward the wanted one `+0x214` by at most `dt` degrees per update, so **1° per second**. Confirmed
  (code). `+0x214` is 60, set by the constructor and copied by `CamHood_CopySettings`; `CamSetupHood` sets the fov
  itself, not `+0x214`, so a script's 65° drifts back to 60° over 5 s (confirmed (code); the arithmetic, not seen at
  runtime).
- **Third person** (`Cam_3rdPerson`, type 16, `CameraCreateThird`): each update (`0x001205e8`) the look-at point is the
  target's position plus the offset (default (0, 0, 1.5)); the orientation eases 10% toward the target's; the camera
  stands `distance` (default 5.1) behind along it and `height` (default 1.8) up, swung by `angle` about the look-at
  point. Confirmed (code).

## In Coney

The port notes remain on [Camera](../../engine/camera.md#coneys-implementation) during this incremental restructure
and will
move to the Engine section with the other camera implementation notes.

## Open questions

The hood field-of-view drift is a static reading rather than a runtime observation, as stated above. Other
outstanding camera comparisons remain in the system's [open questions](index.md#open-questions).

## How this was checked

### Recorded timer locator {#timer-locator}

The earlier camera notes recorded `0x004dc9a8` for the timer's vtable reader of `+0x48`. This address does not
resolve in the current function listing or image-section map. It is retained here as unresolved provenance; this
restructure does not substitute a guessed address or claim a fresh read of the executable.
