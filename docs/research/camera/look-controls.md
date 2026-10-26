---
kind: topic
system: camera
owns:
  functions: ["0x00128b20", "0x00129050", "0x001298c0"]
---
# Camera look controls

Verified against: `SLUS_212.15` (NTSC-U); this page preserves the earlier camera research's findings.

## Summary

The player's right-stick camera input, centre and reverse buttons, and the idle human's head look.
The follow camera's automatic tracking stays on [Camera](follow.md#update).

## Map

| Address | Function | Subject | Evidence |
| --- | --- | --- | --- |
| `0x00129050` | `Cam_Follow_RightStick` | pad input and button gates | confirmed (code) |
| `0x001298c0` | `Cam_Follow_LookBehind` | the reverse view | confirmed (code) |
| `0x00128b20` | `Cam_Follow_HeadLook` | head-look request from camera input | confirmed (code) |

## Data

The input step reads the pad record's raw stick bytes and the per-pad camera options. Their fields, thresholds and
units are recorded with the corresponding input below; the shared camera layout is on
[the follow camera object](follow.md#the-follow-camera-object).

## Behaviour

### The right stick {#right-stick}

`0x00129050` reads the right stick's raw bytes from the pad record (`0x005dd810 + pad × 0x50`, `+0x1a` x and `+0x1b`
y, 0-255 with 128 at rest), confirmed (code). It acts only on a player's current camera, and only when the per-pad
camera options `0x0050b1b0` and `0x0050b1b8` allow it.

| Input | Raw range | Effect |
| --- | --- | --- |
| Stick left | x ≤ 64 | yaw rate `+0x358` from 150°/s at x = 0 to 60°/s at x = 64 (`2.618 − (x / 64) × π/2` rad/s) |
| Stick right | x ≥ 176 | yaw rate from −60°/s at x = 176 to −150°/s at x = 255 |
| Stick up | y ≤ 8 | pitch rate `+0x3b8` from 85°/s at y = 0 to 55°/s at y = 8 |
| Stick down | y ≥ 232 | pitch rate from −55°/s at y = 232 to −85°/s at y = 255 |
| Between | | no turn: a dead zone of ±48 horizontally and nearly the whole travel vertically |

- Inversion flags per pad swap the signs: `0x0050b1f0` for yaw (it also swaps the two side factors before they are
  read), `0x0050b1f8` for pitch. Confirmed (code).
- Any turn sets `+0x368` to 0.334 s, which holds off the framing and auto-follow rules (update step 9).
- **Side factors.** A left turn's rate is capped at `+0x438 × 150°/s` and happens only while `+0x438` > 0 (a right
  turn: `+0x434`); a turn sets the other side's factor back to 1. The [collision step](collision.md#collision) sets
  both to 1
  every update except after its wall push, which sets the side the camera is pressed against to −1, so the stick
  cannot turn the camera into that wall on the next update. Confirmed (code) in `0x00129050` and at the end of
  `0x00130990`.
- The stick also gives two **head-look angles** for [Head look](#head-look): a yaw (only tested for being non-zero)
  of 60° at full deflection to
  1.8° at the dead-zone edge (`60° − (x / 64) × 58.2°`, mirrored to the right) and, only while the target pitch is
  inside its limits, a pitch of 22.5° to 1.8° up (`22.5° − (y / 8) × 20.7°`) or −1.8° to −22.5° down.
- The stick, the centre button and the reverse button act only when the pad's options allow it (`0x0050b1b0`,
  switch 0 at `0x0050b1b8`), on a player's own camera; the stick and the centre button also need `+0x452` (set every
  update with one target) and the call's seventh argument clear. Confirmed (code) in `0x0012ae58`
  (`0x0012af5c`-`0x0012b0e0`, passed at `0x0012b4c8`):
    - **Seventh argument (right-stick strafing)**: set only when the top game mode is 1 (`GameModeStack_TopId`
      `0x0015e748`), both strafe options are on (`CfgStrafeRStick` `0x005104d0` and `CfgStrafe` `0x005104a8`) and the
      first target's control function (`+0x1bc`) is the lock-on movement `Human_FightStanceMove` (`0x00241b90`, the
      member pointer at `0x00548a00`). Both options are 0 in the image and only those two bindings write them, which
      no script calls, so in the shipped game the argument is always 0 (inferred): the stick always turns the camera.
      When set, it also skips the pitch.
    - **Eighth argument (look-behind)**: switch 11, but 0 unless the target **runs or sprints** and 0 while it is
      tagging (`0x0012b4a0`-`0x0012b4b8`). "Runs or sprints" is the update's local flag (stack `+0x1e4`, cleared at
      `0x0012aff8`) that the [sprint detection](follow.md#sprint-zoom) sets: the first target's gait `+0x1a8` is 5
      (`0x00223a98`) or 4 (`0x00223a60`) with record `+0x08` clear (no attack or held state), and with two targets
      the second as well (`0x0012b310`-`0x0012b3ac`); a target at gait 0 skips the tests; the takeover `+0x465`
      clears it (`0x0012b428`). Tagging is the first target's state `0x2000000` (`0x002238c0`) with one target, or
      both targets' (`s5`, `0x0012b2b0`-`0x0012b2d8`). Confirmed (code).
    - **Ninth argument (pitch allowed)**: the stick's pitch acts only when the views shown equal the player cameras
      (`0x0050b198` = `0x0050b19c`) or the camera has one target. Confirmed (code).
    - The whole call is skipped when the camera has no target (`+0x444` = 0).

#### Buttons {#buttons}

The three camera buttons are pad masks (bits as in [the pad record](../frontend.md#pad-record)); their defaults, read
from the data at `0x0050b22c`-`0x0050b234`, confirmed (code):

| Global | Button | Default | Evidence |
| --- | --- | --- | --- |
| `0x0050b22c` | centre | `0x200`, **L3** (the left stick pressed) | confirmed (code) |
| `0x0050b230` | reverse (look behind) | `0x8`, **R1** | confirmed (code) |
| `0x0050b234` | zoom | `0x400`, **R3** (the right stick pressed) | confirmed (code) |

`CamAssignRevCamButton` changes the reverse button; the only script call is `wchair.lua` (`StartIt`), which sets it
to 0 (none) for the wheelchair mini-game.

- **Zoom** (`0x00129050`, then `0x0012d7a8` and `0x001254f0`): a press moves the band by `+0x400 − near` and steps
  `+0x400` to the next preset (see the field table). When the reverse and zoom buttons are **the same button**, a
  hold timer (`+0x370` for the first pad) grows by `dt` while it is held: a press zooms at once (the timer has only
  just started) and resets the timer; holding it past **0.17 s** also counts as the reverse button. A press that
  follows a hold longer than 0.17 s only resets the timer, unless the look-behind switch (11) is off, when every press
  zooms. With separate buttons a zoom is any new press. Confirmed (code).
- **Centre** (`0x00129050` at `0x00129338`-`0x001293a0`): with one target, no clear heading latched (`+0x463` = 0,
  [helpers](collision.md#collision-helpers)), not in a mini-game (`0x00233c50`) and not player 1 in the [combat
  camera](follow.md#combat-camera), a new press sets the heading target `+0x350` to the player's facing (the heading
  of his
  rotation, `0x00335f48`) and, when no timer runs, `+0x40c` = **0.2 s**; the stick does nothing that update. So the
  camera swings behind the player over 0.2 s ([update step 15](follow.md#update)). Confirmed (code).
- **Reverse**: see [Looking behind](#reverse).

#### Looking behind {#reverse}

Only while the game mode on top is 1 (play, inferred), switch 11 is on and the player **runs or sprints** (gait 4 or
5 with record `+0x08` clear, [the eighth argument](#right-stick)), holding the reverse button stores the pad in
`+0x458`; letting go clears it, and so does the gate closing while it is held (`0x00129050` at
`0x001292a4`-`0x001292e8`: the look-behind ends on the update the player slows to a walk, stops or starts an attack,
as if R1 were let go). Confirmed (code). So **R1 never turns the camera round from a standstill, a walk or a fight
stance**: a player blocking (standing, gait 0) or locked on (the combat walk is gait 3,
[Combat](../combat-targets.md#fight-stance)) gets no look-behind, and R1 is free for the block. Not checked at runtime.
`0x001298c0`, confirmed (code), with the 75° and 45° cosines set by `Cam_Follow_StaticInit` (`0x00134ab0`):

1. **First held update**: the band's near edge is saved in `+0x35c`. When the angle between the player's facing and
   the camera's reversed forward is more than **75°** (the usual case: the camera behind the player), the band is
   moved out to **8.5 m** (`0x0012d7a8` with `8.5 − near`), `+0x455` set, and the camera yaws about the look-at
   point by that whole angle at once (`0x0012d688`): it stands in front of the player looking back at him. The
   wanted and held positions (`+0x250`, `+0x260`) are set to the result, so there is no lag. Within 75° nothing
   happens. Confirmed (code).
2. **Held**: nothing more while the angle stays within **45°**; if the player turns further, the look ends: the band
   goes back to the saved near edge, `+0x455` is cleared and `+0x459` set (which drops the held button at the next
   stick read).
3. **Let go**: the band goes back to the saved near edge, `+0x455` is cleared and the camera is turned back behind
   the player at once, again with no lag.

While the button is held the framing rules and auto-follow are off (update step 9) and the lag is skipped until the
view is pulled out (step 18).

Earlier observation: see the [recorded values and method](#upstream-earlier-observations).

| Update | Gait, speed | Clip, state | Camera |
| --- | --- | --- | --- |
| R1 pressed at the run (51) | 4, 7.80 m/s, unchanged for the 40 held updates | 410 (the run), state 0, record `+0x08` 0 | `+0x458` = 0 (pad 0) and `+0x455` = 1 on the first held update; the yaw jumped from 90.9° to −89.0° that update; the band's near edge (`+0x32c`) 4.8 → 6.1 m |
| R1 let go (91) | 4, 7.80 m/s | 410 | `+0x458` back to 255, `+0x455` 0, the yaw back to 91.0° at once, the near edge 4.8 m |
| The stick to rest with R1 held (141) | 0, 0 m/s that update | **606** `BLOCK_SUSTAIN`, state `0x8001`, record `+0x08` `0x80000` for 5 updates | the look ends that update (`+0x458` 255, the yaw back behind) |

### Head look {#head-look}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

With the stick turning the camera and the player idle (update step 17), `0x00128b20` turns his head toward where
the camera looks, confirmed (code):

- **Yaw**: the signed angle between the camera's horizontal forward and the player's facing; **0 beyond 150°** (so he
  does not twist to look behind). **Pitch**: the stick's head pitch (above).
- `HeadLook_AtAngles` (`0x00299f98`) stores them in the human's head-look state (`+0x284`) with weight **0.75**, flags
  `0x20 | 4`, for `(dt × 1000 & 0xff) + 100` ms (133 ms at 30 Hz), so the look lapses about 4 updates after the
  stick is let go. Confirmed (code).
- **The turn** (`Human_UpdateHeadLook` `0x00248428`, each human update): the yaw is clamped to **±55°** (±90° with
  flag `0x40`, not set here) and the pitch to **±45°**; each update the head's angles step toward them by **weight ×
  12°** (9° at 0.75; `Angle_StepToward`, last argument 30°, not traced); flag `0x20` keeps the look from relaxing when
  the yaw passes 60°. `CharacterInstance_SetHeadLook` (`0x00175c58`) then clamps the pitch to −30°..22.5° for the
  bones ([graphics](../graphics.md)). Confirmed (code).

## In Coney

The camera port notes remain on [Camera](../../engine/camera.md#coneys-implementation) during this incremental
restructure.
Their final home is the Engine section.

## Open questions

The camera system's outstanding look-behind comparisons remain in its
[open questions](index.md#open-questions); no new runtime comparison is claimed here.

## How this was checked

The earlier page recorded the following PCSX2 observation. No surviving run-record id accompanies it; this move
retains the observation as historical checking notes.

**R1 at a run looks behind and does not block.** The reverse button is R1, which also blocks
([Combat](../combat/index.md#dispatch)), but the block's fight test `Human_CanFight` (`0x00224f28`) fails while the
human
runs or sprints free: gait `+0x1a8` 4 (`0x00223a60`) or 5 (`0x00223a98`) with record `+0x08` clear. Confirmed
(code). So at a run R1 leaves the gait, the speed and the run clip alone and only turns the view; the block starts
on the first update the gait drops below 4 with R1 still held. Confirmed (runtime), PCSX2 2.9.94, scenario
`run_block` (level99's street, slot 1; stick 98 % straight up, R1 held at the run twice, then held while the stick
went back to rest):

| Update | Gait, speed | Clip, state | Camera |
| --- | --- | --- | --- |
| R1 pressed at the run (51) | 4, 7.80 m/s, unchanged for the 40 held updates | 410 (the run), state 0, record `+0x08` 0 | `+0x458` = 0 (pad 0) and `+0x455` = 1 on the first held update; the yaw jumped from 90.9° to −89.0° that update; the band's near edge (`+0x32c`) 4.8 → 6.1 m |
| R1 let go (91) | 4, 7.80 m/s | 410 | `+0x458` back to 255, `+0x455` 0, the yaw back to 91.0° at once, the near edge 4.8 m |
| The stick to rest with R1 held (141) | 0, 0 m/s that update | **606** `BLOCK_SUSTAIN`, state `0x8001`, record `+0x08` `0x80000` for 5 updates | the look ends that update (`+0x458` 255, the yaw back behind) |

### Earlier observations retained from upstream {#upstream-earlier-observations}

These passages came from upstream 8ecd4cb8 without surviving run identifiers. Their original labels and
context are retained as earlier checking notes, not a new runtime comparison.

```text
**R1 at a run looks behind and does not block.** The reverse button is R1, which also blocks
([Combat](../combat/index.md#dispatch)), but the block's fight test `Human_CanFight` (`0x00224f28`) fails while the
human
runs or sprints free: gait `+0x1a8` 4 (`0x00223a60`) or 5 (`0x00223a98`) with record `+0x08` clear. Confirmed
(code). So at a run R1 leaves the gait, the speed and the run clip alone and only turns the view; the block starts
on the first update the gait drops below 4 with R1 still held. Confirmed (runtime), PCSX2 2.9.94, scenario
`run_block` (level99's street, slot 1; stick 98 % straight up, R1 held at the run twice, then held while the stick
went back to rest):
```
