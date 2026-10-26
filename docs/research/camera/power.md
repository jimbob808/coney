---
kind: topic
system: camera
owns:
  functions:
    - "0x00139d68"
    - "0x00139df8"
    - "0x00139e20"
    - "0x00139e40"
    - "0x00139e90"
    - "0x00139ef8"
    - "0x0013a170"
    - "0x0013a3b0"
    - "0x0013a448"
    - "0x0013a548"
    - "0x0013a818"
---
# Power-move camera

Verified against: `SLUS_212.15` (NTSC-U); findings retain the earlier camera research's evidence levels.

## Summary

The camera sequence carried by power-move animation events: begin, store shots, cut between them and hand back.
The shared camera manager stays on [Camera](index.md#fn-manager).

## Map {#fn-power}

The full function descriptions are in the [camera function database](functions.md). Entry points:

| Address | Function | Subject | Evidence |
| --- | --- | --- | --- |
| `0x0013a170` | `CamPower_Begin` | sequence setup | confirmed (code) |
| `0x0013a548` | `CamPower_CutToShot` | first and subsequent cuts | confirmed (code) |
| `0x0013a818` | `CamPower_PlaceShot` | shot collision tests and selection | confirmed (code) |
| `0x0013a3b0` | `CamPower_End` | hand-back | confirmed (code) |

## Data

The event payloads, per-camera fields and stored shot arrays are described with the operation that uses them below.

## Behaviour

### Power-move camera {#power-camera}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

`Cam_Power` (type 6) films a finisher from fixed angles. The finisher's animation clip carries the whole sequence as
frame events: at its start it **begins** a sequence and **adds** up to eight camera shots placed around the attacker,
then each **cut** jumps the view to a shot, and an **end** event (or the move ending) hands the view back. The
player sees a hard cut to a still camera at a 50° field of view, a hard cut to the other angle on each extension
hit that has a cut, and a hard cut back to the follow camera where it was. Confirmed (code) at the functions in
[the table](#fn-power); historical runtime notes are retained in [At runtime](#power-runtime).

#### The events {#power-events}

`Anim_FireFrameEvents` (`0x00101dd8`) handles the four events as the clip's cursor passes their frames
([Animation format](../formats/animation.md#keyframes-chunk-0x00) for the 24-byte event and its scaling). Confirmed
(code) at the handlers:

| Type | Handler | Calls | Switch 9 | Payload | Evidence |
| --- | --- | --- | --- | --- | --- |
| `0x39` begin | `0x00102ac8` | `Cam_GetPower(0, create)`, then `CamPower_Begin(cam, mode, human)` | needed | `+0x06` s16 mode: 1 = one shot, anything else = 2 (two shots that alternate) | confirmed (code) |
| `0x3b` add shot | `0x00102fbc` | `Cam_GetPower(0)` (never creates), then `CamPower_AddShot(cam, &pos, &quat, slot)` | needed | `+0x04` s16, must be 0 (else the event is ignored); `+0x06` s16 slot 0-7; `+0x08`-`+0x0c` s16 position x / 1023, y / 1023, z / 2047 m; `+0x0e`-`+0x12` s16 quaternion x, y, z × 2⁻¹⁵, w = √(1 − x² − y² − z²) | confirmed (code) |
| `0x3a` cut | `0x00102ea0` | `Cam_GetPower(0)`, then `CamPower_CutToShot(fov, cam)` | needed | `+0x08` read as one 32-bit float (`lwc1 f12,0x8(s1)` at `0x00102fa8`): the field of view, used when above 0 | confirmed (code) |
| `0x3d` end | `0x001032a8` | `CamPower_End` on player 0's current camera when it is a power camera | not read | none | confirmed (code) |

**The shot's frame.** The position is in the frame of the human the camera follows, at the moment of the begin:
x to his right, y forward, z up from his feet (the transform's position in `0x00714b00`, not the head). The
quaternion turns the camera's own axes (+y the view, +z up, +x right, [Axes](index.md#the-base-camera-object)) into that
frame, so in the world the shot's orientation is `q_h ⊗ q_e` and its position `p_h + q_h · p_e`, with `q_h`, `p_h`
the human's rotation and position at the begin (`CamPower_AddShot`'s quaternion products, `0x0013a470`-`0x0013a530`).
Confirmed (code), historical observation ([checking notes](#upstream-earlier-observations)). On the disc (the survey
below) shots stand 0.3 to 6.4 m from the human in plan
(median 2.8 m), from 0.2 m below his feet to 4.4 m above them (median 0.9 m), pitched from straight down to 36° up
(median 1° down), and look within 54° of a point 1 m above his feet (median 12°): low, close and aimed at the body.
Every cut on the disc holds −1.0 (bits `0xbf800000`), so **every power shot is at 50°**; no event carries a near or
far plane.

**Who counts** (the handlers' common gate, confirmed (code)). Let *p* be the player number (`+0x1b0`) of the clip's
human. A human with no player number takes his partner's (handle `+0xc4`) when his behaviour class (`+0xcc`) is one
of `0x4d`-`0x4f`, `0x6a`, `0x77`, `0x78`, `0x80`, `0xa0`, `0xa1`, `0xb2`, `0xb3`, `0xed` (the shake event's list; the
classes are not identified, and the clips that need it are bosses' counters, [Which clips](#power-clips)); any other
human with no player number is dropped, so an AI ally's half of a tandem does nothing. The begin is also dropped when
the
human plays anim 654, 656, 658 or 660 (the special attacks' reactions) against a partner of class `0xb2` or `0xb3`.
In hide-and-seek (`0x0050b1dc`, set by `HideAndSeekGoal_Start`) anim 654 or 656 instead counts as player 0,
following the clip's own human.

**The begin's own conditions** (`0x00102ca4`-`0x00102e78`). Player 0's current camera must be the follow (type 2),
fixed (0), power (6) or rail (9) camera, a rail camera not in its mode 3 (`+0x3ed`), and switch 9 on. With one
player camera (`0x0050b19c` = 1) it passes no human, so `CamPower_Begin` follows **the saved camera's first target**
(the follow camera's `+0x314`, the rail camera's `+0x340`) when that is a player: the player, even when the clip that
fired is his partner's. With two player cameras it goes ahead only on one shared view (`0x0050b198` = 1) and when
the clip's player and his partner's are both players, following player *p*'s human; on a split screen there is no
power camera. The add-shot event needs the same camera types and an existing power camera. Confirmed (code).

#### Fields {#power-fields}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

The class's own fields, confirmed (code) at the constructor and the event functions:

| Offset | Meaning | Constructor |
| --- | --- | --- |
| `+0x180` | look-at point (base field): set at the cut, 0.305 m beyond the human along the view | |
| `+0x1e0 + 16·i` | shot *i*'s position, world space, *i* < 8 | |
| `+0x260` | the human's position at the begin | |
| `+0x270` | the position of the shot cut to (the update returns to it) | |
| `+0x280 + 16·i` | shot *i*'s orientation quaternion, world space | |
| `+0x300` | the human's orientation at the begin | |
| `+0x310` | the orientation of the shot cut to | identity |
| `+0x320` | the human followed | 0 |
| `+0x324` | the saved camera's field of view (written by the begin, read by none of the class's functions) | −1 |
| `+0x328` | distance from the shot to the human + 0.305 m (the look-at distance) | 0 |
| `+0x32c` | u16 mask of the shots that passed the tests | 0 |
| `+0x32e` | unplaced: set by the begin, cleared by the first cut; shots are added only while it is set | 1 |
| `+0x32f` | in a move: set by the begin, cleared by the end | 0 |
| `+0x330` | mode, 1 or 2 | 2 |
| `+0x331` | s8 shot count; −1 after the end, −99 before any begin | −99 |
| `+0x332` / `+0x333` | s8 current and alternate shot, −1 for none | |
| `+0x334` | the saved camera, the one handed back to | 0 |

#### Begin {#power-begin}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

`CamPower_Begin` (`0x0013a170`), confirmed (code): clears the two humans the shot test ignores (`0x0050b27c`,
`0x0050b280`); saves player 0's current camera in `+0x334` (when that is already the power camera, the camera it
saved stays, so a second begin in the same move does not nest); copies that camera's near plane and far clip
(vtable `+0x204` → `+0x1a4`, `+0x20c` → `+0x1ac`) and stores its fov in `+0x324`; takes the human (above; with none
the count stays below 0 and every later event is ignored); copies his position and rotation from the transform table
into `+0x260` / `+0x300`; sets unplaced and in-a-move, count 0, and the mode. The camera is not made current: the
view changes only at the first cut.

#### Shots {#power-shots}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

`CamPower_AddShot` (`0x0013a448`) stores the shot in world space at once (above), so the angles are fixed to where
the human stood at the begin, not to where he is later, and counts it. It ignores a slot of 8 or more and any shot
after the first cut. The count is a count, not the highest slot: every clip on the disc adds slots 0 to *n* − 1.
Confirmed (code).

#### Choosing the shots {#power-choice}

On the first cut, `CamPower_PlaceShot` (`0x0013a818`) tests each stored shot and picks two. Confirmed (code):

1. **The head point**: the human's live position (now, not at the begin) raised by **1.45 m**, or by **0.75 m**
   while he plays one of the mount power strikes (anims 225, 227, 229, 231, 233, 235).
2. **The test sphere**: radius **max(the human's capsule radius, 0.1)** (the capsule radius is 0.35 × his scale,
   [Physics](../physics.md#human-body); 0.1 is the power camera's own radius, vtable `+0x5c`); in hide-and-seek the
   smaller of the two. It is swept with `IPhysics_CollideShape` against the level mesh and the bodies (query options
   `0x12813`), the human himself being the owner, and a filter callback skips his partner (handle `+0xc4`) and,
   when the human and his partner are both in a paired move, the partner's partner (`0x0050b27c` / `0x0050b280`).
   Confirmed (code).
3. **Two sweeps per shot**. First from a point beyond the human, on the shot's line of view at the shot's distance
   from the head point plus the capsule radius (raised by the radius − 0.1 when the shot is not above the head),
   back to the shot; then from the human's feet to the shot. A shot is good when neither sweep hits (no contact, or
   one at a fraction of 1 or more), and bit *i* of `+0x32c` is set. A blocked shot is dropped, not moved closer.
4. **None good**: no cut. The power camera never becomes current and the follow camera stays for the whole move.
5. **The pick** starts from the counter `0x0050b2a0` (0-7, 0 at boot): `c = counter mod n`. In mode 2 it looks for a
   good pair `(c, c + 1 mod n)`, moving `c` on by one up to *n* times; a pair found gives the current and the
   alternate shot. With no good pair, or in mode 1, the current shot is the first good one from `c` on, and there is
   no alternate. Confirmed (code).
6. **Order**: in mode 2, of the pair, the one **farther in plan** (x, y) from the saved camera's position becomes the
   current shot, so the first cut is the bigger change of angle.
7. The counter moves on by one (mod 8) whenever at least one shot was good, so successive power moves start from
   different shots: the same move done twice in a row is filmed from different angles.

#### The cut {#power-cut}

`CamPower_CutToShot` (`0x0013a548`), confirmed (code):

- nothing when the count is 0 or below (no begin, or after the end); with no human it ends the camera;
- on the first cut it chooses the shots (above);
- with a current shot: the camera's position and orientation become that shot's and are kept in `+0x270` / `+0x310`;
  the look-at distance `+0x328` = the distance from the shot to the human's live position + 0.305 m, and the
  look-at point `+0x180` is that far along the view;
- on the first cut, unless it already is, the camera becomes player 0's current camera with **no blend**
  (`Camera_MakeActive(0 s, power, player 0)` after one 1/60 s `ICamera_FinishUpdate`): a hard cut;
- then current and alternate swap (when there is an alternate), so the next cut goes to the other shot of the pair:
  in mode 2 the cuts alternate A, B, A ...; in mode 1 every cut returns to the one shot;
- the field of view becomes the event's when it is above 0 (also gated by `0x0050b29c`, data, always 1), else
  **50°**, at once (`ICamera_SetFov`), with no ease (confirmed (code));
- unplaced is cleared: later cuts do not test the shots again, even if the human has moved.

Earlier observation: see the [recorded values and method](#upstream-earlier-observations).

#### Holding a shot {#power-hold}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

`CamPower_Update` (`0x00139ef8`), each camera update while it is current, confirmed (code):

1. **Ends** (`CamPower_End`) when the human is gone, down or dead (`Human_IsDownOrDead`), or, outside hide-and-seek,
   when he is in neither side of a paired move (state `0x1000`, the attacker, or `0x2000`, the victim,
   [Combat](../combat/index.md#state-flags)), or he is a victim whose partner has gone or left `0x1000`.
2. Otherwise the camera is put back at the cut's position and orientation (`+0x270`, `+0x310`), the position is
   pushed out of nearby humans (`Camera_PushOutOfHumans` with 3.0, none skipped), and the look point is set 3 m
   along the view.
3. The **shake** (`Cam_UpdateShake`, [Shake](effects.md#shake)) moves that look point; only when it did is the view
re-aimed
   at it (`Mat_LookAtUp` with the shot's own up axis). The camera does not track the human: the view stays fixed for
   the whole shot. The earlier position/orientation observation is retained in [checking notes](#power-runtime).

#### The end and the hand-back {#power-end}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

`CamPower_End` (`0x0013a3b0`) runs on the clip's `0x3d` event (whatever switch 9 says) or from the update's checks.
When the power camera is a player's current camera it makes the saved camera current with **no blend**. That camera
was not updated while the power camera showed, so it comes back exactly where it was before the move and goes on
from there (the follow camera's activation, [Setting up](index.md#setting-up), step 4). The power camera is
released as
soon as it stops being current (`CamPower_Deactivate` → `Camera_Release`), and the next begin makes a new one with
no shots. Confirmed (code). See the [earlier runtime notes](#power-runtime).

So an **extended power move** keeps the power camera only if the next part starts before the current clip reaches
its end event; the part's own cut (`0x3a`) then swaps to the alternate shot. If the end comes first, the view is
back on the follow camera and the next part's cut is dropped (no power camera exists); a part with a begin of its
own (the `~2` variants below) starts a new sequence. Some parts have no cut and keep the shot. Inferred from the
code; the kept-shot and swap observations are retained in [checking notes](#power-runtime).

#### Slow motion and shake {#power-slowmo}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

- **Slow motion** (`0x2e` / `0x2f`, [Slow motion](effects.md#slow-motion)) slows only the characters' step. The
  cameras
  still update at 1/30 s and the power camera is still, so slow motion changes nothing in it, except that the clip's
  events come later in updates (the clip advances 1/150 s an update). Inferred from the code at the cited addresses.
- **Shake**: event `0x20` in a power move's clip (131 of the 164 clips below have one) shakes player 0's current
  camera, the power camera, through its slot `+0x15c` (`CamPower_StartShake`), which starts it only for a human on
  the camera target list. The shake turns the view through the look point; the camera does not move. Its time
  counts down by `dt × min(1, 54 × step)`, so a shake in slow motion lasts longer ([Shake](effects.md#shake)).

#### Which clips {#power-clips}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

A survey of the disc's animation clips (the `coney-tools extract` output; events counted, no clip data copied):
**164** clip files have power-camera events. **94** begin a sequence (mode 2: 80, mode 1: 14), **53** only cut and
end (later parts of a move), **17** only end. Shots per begin: 8 (73 clips), 4 (19), 1 (2). Event `+0x04` is 0 and
the cut's word −1.0 in every one. Families:

| Family | Begin | Shots | Notes |
| --- | --- | --- | --- |
| grab power strikes `gen_grab_front_power{1-24}_p1` (anims 57, 63) | mode 2 | 8 | as every begin on the disc: at frame 0-6, all its shots on one later frame (the next in 61 of 94), the first cut on the shots' frame or after (the next in 56, up to 22 later) |
| their extensions `_p2`, `_p3` (anims 59, 61, 65, 67) | none; the `~2` variants of 3 and 4 have one | | one cut each or none (power 1, 10, 11, 15 `_p2`), then the end |
| mount power strikes `gen_mount_power{1-20}_p1`, `_p2`, `_p3` (anims 225-236) | mode 2 on `_p1` | 8 | as the grab's |
| weapon grabs `gen_grab_front_bottle`, `_knife`, `gen_grab_back_bottle`, `_knife` | mode 1 | 8 | |
| weapon mounts `gen_beerbottle_mounting`, `gen_knife_mounting` | mode 2 | 8 | |
| wall grabs `gen_grab_front_wall_back`, `_front`, `_left`, `_right` | mode 1 | 4 | |
| stealth kills `player_stealthkill_barehands`, `_baton`, `_knife`, `_weak` | mode 1 | 8 | |
| `bottle_smash_attacker` | mode 1 | 8 | |
| tandem starts `gen_tandem1`, `gen_tandem2` (`_attacker_begin`, `_grabber_begin`), `gen_tandem_knife_*_beg*` | mode 2 | 8 | no end: their `_end` clips end it |
| tandem ends `gen_tandem3`, `gen_tandem_bat` (`_attacker_end`, `_grabber_end`), `gen_tandem5_*_end` | mode 2 | 8; 4 for tandem 5 | |
| bosses: `bopp_bo_grab_counter_victim`, `bopp_bo_tackle_counter_react`, `hiha_bo_*`, `samo_bo_tandem*`, `ghost_grab_front_power_react`, `dest_bo_special_nut_kck_react`, `virgil_uppercut_react` | mode 2 (Virgil 1) | 4; 8 for Hi-Hat's grab counter; 1 for Dest and Virgil | some with two cuts or no end |

No rage clip has its own events: a raging power strike plays another grab power clip (63, 65 ...). The full list:

??? note "The 164 clips: frames, begin (frame, mode), shots, cut frames, end frame"

    | Clip | Frames | Begin (frame, mode) | Shots | Cuts at frames | End at frame |
    | --- | --- | --- | --- | --- | --- |
    | `bopp_bo_grab_counter_victim` | 53 | 3, 2 | 4 | 9 | 32 |
    | `bopp_bo_grab_counter_victim~2` | 53 | 3, 2 | 4 | 9 | 32 |
    | `bopp_bo_tackle_counter_react` | 60 | 3, 2 | 4 | 17, 34 | 50 |
    | `bopp_bo_tackle_counter_react~2` | 60 | 3, 2 | 4 | 17, 34 | 50 |
    | `bottle_smash_attacker` | 34 | 3, 1 | 8 | 15 | 29 |
    | `dest_bo_special_nut_kck_react` | 99 | 0, 2 | 1 | 16 | - |
    | `gen_beerbottle_mounting` | 48 | 0, 2 | 8 | 8 | 28 |
    | `gen_grab_back_bottle` | 41 | 6, 1 | 8 | 10 | 29 |
    | `gen_grab_back_knife` | 57 | 6, 1 | 8 | 12 | 47 |
    | `gen_grab_front_bottle` | 32 | 0, 1 | 8 | 11 | 23 |
    | `gen_grab_front_knife` | 62 | 0, 1 | 8 | 11 | 52 |
    | `gen_grab_front_power10_p1` | 33 | 3, 2 | 8 | 4 | 28 |
    | `gen_grab_front_power10_p2` | 70 | - | - | - | 70 |
    | `gen_grab_front_power11_p1` | 39 | 3, 2 | 8 | 8 | 37 |
    | `gen_grab_front_power11_p2` | 40 | - | - | - | 38 |
    | `gen_grab_front_power11_p3` | 32 | - | - | 3 | 31 |
    | `gen_grab_front_power12_p1` | 37 | 0, 2 | 8 | 2 | 30 |
    | `gen_grab_front_power12_p2` | 50 | - | - | 1 | 43 |
    | `gen_grab_front_power12_p3` | 50 | - | - | 1 | 46 |
    | `gen_grab_front_power14_p1~2` | 41 | 0, 2 | 8 | 2 | 28 |
    | `gen_grab_front_power14_p2~2` | 45 | - | - | 1 | 24 |
    | `gen_grab_front_power15_p1` | 35 | 0, 2 | 8 | 2 | 25 |
    | `gen_grab_front_power15_p2` | 40 | - | - | - | 30 |
    | `gen_grab_front_power15_p3` | 35 | - | - | 2 | 25 |
    | `gen_grab_front_power16_p1` | 40 | 0, 2 | 8 | 2 | 36 |
    | `gen_grab_front_power16_p2` | 49 | - | - | 3 | 43 |
    | `gen_grab_front_power16_p3` | 33 | - | - | 2 | 30 |
    | `gen_grab_front_power17_p1` | 31 | 0, 2 | 8 | 2 | 24 |
    | `gen_grab_front_power17_p2` | 31 | - | - | 1 | 25 |
    | `gen_grab_front_power17_p3` | 31 | - | - | 1 | 23 |
    | `gen_grab_front_power18_p1` | 55 | 2, 2 | 8 | 4 | 49 |
    | `gen_grab_front_power18_p2` | 28 | - | - | 2 | 26 |
    | `gen_grab_front_power19_p1` | 45 | 1, 2 | 8 | 3 | 39 |
    | `gen_grab_front_power19_p2` | 51 | - | - | 1 | 45 |
    | `gen_grab_front_power19_p3` | 20 | - | - | 1 | 19 |
    | `gen_grab_front_power1_p1` | 22 | 0, 2 | 8 | 5 | 20 |
    | `gen_grab_front_power1_p2` | 30 | - | - | - | 25 |
    | `gen_grab_front_power1_p3` | 25 | - | - | 5 | 22 |
    | `gen_grab_front_power20_p1` | 45 | 0, 2 | 8 | 2 | 40 |
    | `gen_grab_front_power20_p2` | 36 | - | - | 0 | 35 |
    | `gen_grab_front_power21_p1` | 45 | 0, 2 | 8 | 2 | 44 |
    | `gen_grab_front_power21_p2` | 37 | - | - | 0 | 36 |
    | `gen_grab_front_power22_p1` | 45 | 0, 2 | 8 | 2 | 41 |
    | `gen_grab_front_power22_p2` | 30 | - | - | 1 | 27 |
    | `gen_grab_front_power23_p1` | 30 | 1, 2 | 8 | 3 | 24 |
    | `gen_grab_front_power23_p2` | 34 | - | - | 2 | 26 |
    | `gen_grab_front_power2_p1` | 40 | 0, 2 | 8 | 2 | 36 |
    | `gen_grab_front_power2_p2` | 42 | - | - | 16 | 38 |
    | `gen_grab_front_power2_p3` | 24 | - | - | 7 | 21 |
    | `gen_grab_front_power3_p1` | 37 | 0, 2 | 8 | 14 | 33 |
    | `gen_grab_front_power3_p1~2` | 37 | 3, 2 | 8 | 14 | 33 |
    | `gen_grab_front_power3_p2` | 24 | 0, 2 | 8 | 7 | 21 |
    | `gen_grab_front_power3_p2~2` | 24 | - | - | 7 | 21 |
    | `gen_grab_front_power4_p1` | 33 | 3, 2 | 8 | 6 | 28 |
    | `gen_grab_front_power4_p1~2` | 33 | 0, 2 | 8 | 6 | 28 |
    | `gen_grab_front_power4_p2` | 23 | - | - | 4 | 19 |
    | `gen_grab_front_power4_p2~2` | 23 | 0, 2 | 8 | 4 | 19 |
    | `gen_grab_front_power5_p1` | 55 | 3, 2 | 8 | 5 | 45 |
    | `gen_grab_front_power5_p2` | 28 | - | - | 1 | 27 |
    | `gen_grab_front_power6_p1` | 29 | 1, 2 | 8 | 3 | 25 |
    | `gen_grab_front_power6_p2` | 42 | - | - | 15 | 35 |
    | `gen_grab_front_power6_p3` | 22 | - | - | 7 | 16 |
    | `gen_grab_front_power8_p1` | 35 | 6, 2 | 8 | 10 | 27 |
    | `gen_grab_front_power8_p2` | 26 | - | - | 11 | 21 |
    | `gen_grab_front_power9_p1` | 39 | 3, 2 | 8 | 6 | 32 |
    | `gen_grab_front_power9_p2` | 45 | - | - | 10 | 37 |
    | `gen_grab_front_power9_p3` | 26 | - | - | 6 | 19 |
    | `gen_grab_front_power9_p3~2` | 26 | - | - | 6 | 19 |
    | `gen_grab_front_wall_back` | 34 | 3, 1 | 4 | 8 | 26 |
    | `gen_grab_front_wall_front` | 54 | 3, 1 | 4 | 5 | 50 |
    | `gen_grab_front_wall_left` | 29 | 0, 1 | 4 | 11 | 24 |
    | `gen_grab_front_wall_right` | 28 | 0, 1 | 4 | 9 | 24 |
    | `gen_knife_mounting` | 60 | 0, 2 | 8 | 8 | 49 |
    | `gen_mount_power10_p1` | 60 | 0, 2 | 8 | 3 | 40 |
    | `gen_mount_power10_p2` | 45 | - | - | - | 29 |
    | `gen_mount_power11_p1` | 45 | 0, 2 | 8 | 4 | 37 |
    | `gen_mount_power11_p2` | 46 | - | - | 1 | 35 |
    | `gen_mount_power11_p3` | 36 | - | - | 0 | 25 |
    | `gen_mount_power12_p1` | 42 | 0, 2 | 8 | 2 | 36 |
    | `gen_mount_power12_p3` | 36 | - | - | 5 | 31 |
    | `gen_mount_power13_p1` | 38 | 0, 2 | 8 | 2 | 25 |
    | `gen_mount_power13_p2` | 49 | - | - | 4 | - |
    | `gen_mount_power14_p1` | 40 | 0, 2 | 8 | 4 | 30 |
    | `gen_mount_power14_p2` | 55 | - | - | 6 | 40 |
    | `gen_mount_power14_p3` | 45 | - | - | 3 | 34 |
    | `gen_mount_power15_p1` | 38 | 0, 2 | 8 | 4 | 31 |
    | `gen_mount_power15_p2` | 38 | - | - | 2 | 28 |
    | `gen_mount_power15_p3` | 39 | - | - | 2 | 26 |
    | `gen_mount_power16_p1` | 50 | 0, 2 | 8 | 2 | 45 |
    | `gen_mount_power16_p2` | 28 | - | - | 2 | 23 |
    | `gen_mount_power17_p1` | 51 | 0, 2 | 8 | 3 | 40 |
    | `gen_mount_power17_p2` | 50 | - | - | 4 | 40 |
    | `gen_mount_power18_p1` | 52 | 0, 2 | 8 | 2 | 47 |
    | `gen_mount_power18_p2` | 50 | - | - | - | 45 |
    | `gen_mount_power18_p3` | 46 | - | - | 2 | 42 |
    | `gen_mount_power19_p1` | 40 | 0, 2 | 8 | 2 | - |
    | `gen_mount_power19_p2` | 29 | - | - | 0 | 24 |
    | `gen_mount_power1_p1` | 45 | 4, 2 | 8 | 11 | 37 |
    | `gen_mount_power1_p2` | 32 | - | - | 6 | 24 |
    | `gen_mount_power1_p2~2` | 32 | 0, 2 | 8 | 6 | 24 |
    | `gen_mount_power20_p1` | 37 | 2, 2 | 8 | 5 | 29 |
    | `gen_mount_power20_p2` | 39 | - | - | - | 26 |
    | `gen_mount_power2_p1` | 43 | 0, 2 | 8 | 5 | 31 |
    | `gen_mount_power2_p1~2` | 43 | 0, 2 | 8 | 9 | 31 |
    | `gen_mount_power2_p2` | 25 | - | - | 4 | 22 |
    | `gen_mount_power2_p2~2` | 25 | - | - | 8 | 22 |
    | `gen_mount_power3_p1` | 51 | 3, 2 | 8 | 23 | 45 |
    | `gen_mount_power3_p1~2` | 51 | 0, 2 | 8 | 23 | 45 |
    | `gen_mount_power3_p2` | 30 | - | - | 10 | 26 |
    | `gen_mount_power3_p2~2` | 30 | 0, 2 | 8 | 10 | 26 |
    | `gen_mount_power4_p1` | 39 | 3, 2 | 8 | 7 | 33 |
    | `gen_mount_power4_p2` | 30 | - | - | - | 26 |
    | `gen_mount_power4_p3` | 26 | - | - | 4 | 22 |
    | `gen_mount_power5_p1` | 56 | 4, 2 | 8 | 9 | 40 |
    | `gen_mount_power5_p2` | 30 | - | - | 8 | 23 |
    | `gen_mount_power6_p1` | 37 | 6, 2 | 8 | 11 | 31 |
    | `gen_mount_power6_p2` | 34 | - | - | 6 | 28 |
    | `gen_mount_power6_p3` | 30 | - | - | 9 | 24 |
    | `gen_mount_power7_p1` | 53 | 3, 2 | 8 | 17 | 48 |
    | `gen_mount_power7_p1~2` | 53 | 0, 2 | 8 | 17 | 48 |
    | `gen_mount_power7_p2` | 42 | - | - | 7 | 36 |
    | `gen_mount_power7_p2~2` | 42 | 0, 2 | 8 | 7 | 36 |
    | `gen_mount_power7_p3` | 35 | - | - | 7 | 22 |
    | `gen_mount_power8_p1` | 45 | 3, 2 | 8 | 5 | 28 |
    | `gen_mount_power8_p2` | 47 | - | - | 9 | 32 |
    | `gen_mount_power8_p3` | 41 | - | - | 8 | 23 |
    | `gen_mount_power9_p1` | 47 | 0, 2 | 8 | 3 | 33 |
    | `gen_mount_power9_p2` | 53 | - | - | - | 42 |
    | `gen_mount_power9_p3` | 41 | - | - | 2 | 27 |
    | `gen_tandem1_attacker_begin` | 27 | 3, 2 | 8 | 14 | - |
    | `gen_tandem1_attacker_end` | 27 | - | - | - | 20 |
    | `gen_tandem1_grabber_begin` | 27 | 3, 2 | 8 | 14 | - |
    | `gen_tandem1_grabber_end` | 20 | - | - | - | 19 |
    | `gen_tandem2_attacker_begin` | 35 | 3, 2 | 8 | 26 | - |
    | `gen_tandem2_attacker_end` | 35 | - | - | - | 18 |
    | `gen_tandem2_grabber_begin` | 35 | 3, 2 | 8 | 26 | - |
    | `gen_tandem2_grabber_end` | 31 | - | - | - | 17 |
    | `gen_tandem3_attacker_end` | 30 | 0, 2 | 8 | 2 | 28 |
    | `gen_tandem3_grabber_end` | 32 | 0, 2 | 8 | 2 | 30 |
    | `gen_tandem5_attacker_end` | 35 | 2, 2 | 4 | 4 | 34 |
    | `gen_tandem5_attacker_end~2` | 35 | 2, 2 | 4 | 4 | 34 |
    | `gen_tandem5_victim_end` | 43 | 2, 2 | 4 | 9 | 25 |
    | `gen_tandem_bat_attacker_end` | 50 | 0, 2 | 8 | 10 | 35 |
    | `gen_tandem_bat_grabber_end` | 50 | 0, 2 | 8 | 10 | 35 |
    | `gen_tandem_knife_attacker_beg` | 51 | 0, 2 | 8 | 2 | - |
    | `gen_tandem_knife_attacker_end` | 16 | - | - | - | 15 |
    | `gen_tandem_knife_grabber_begi` | 51 | 0, 2 | 8 | 2 | - |
    | `gen_tandem_knife_grabber_end` | 13 | - | - | - | 12 |
    | `ghost_grab_front_power_react` | 47 | 4, 2 | 4 | 6, 26 | 40 |
    | `hiha_bo_grab_counter_victim` | 39 | 5, 2 | 8 | 7 | - |
    | `hiha_bo_tackle_counter_react` | 55 | 0, 2 | 4 | 10 | 54 |
    | `player_stealthkill_barehands` | 41 | 3, 1 | 8 | 6 | 34 |
    | `player_stealthkill_baton` | 67 | 3, 1 | 8 | 8 | 55 |
    | `player_stealthkill_knife` | 40 | 3, 1 | 8 | 6 | 34 |
    | `player_stealthkill_weak` | 39 | 4, 1 | 8 | 6 | 38 |
    | `samo_bo_tandem3_attacker_end` | 35 | 2, 2 | 4 | 4 | 34 |
    | `samo_bo_tandem3_victim_end` | 45 | 2, 2 | 4 | 9 | 25 |
    | `samo_bo_tandem_1_attacker_beg` | 34 | 3, 2 | 4 | 6, 26 | - |
    | `samo_bo_tandem_1_attacker_end` | 19 | - | - | - | 15 |
    | `samo_bo_tandem_1_victim_begin` | 34 | 3, 2 | 4 | 5 | - |
    | `samo_bo_tandem_1_victim_end` | 19 | - | - | - | 16 |
    | `samo_bo_tandem_2_attacker_end` | 46 | 3, 2 | 4 | 5 | 41 |
    | `samo_bo_tandem_2_victim_end` | 43 | 3, 2 | 4 | 8 | 39 |
    | `virgil_uppercut_react` | 44 | 0, 1 | 1 | 21 | - |

#### Two players and other humans {#power-players}

Confirmed (code) at the cited addresses unless a finding states another evidence level.

Confirmed (code) at the gate and `CamPower_Begin`, except where marked:

- **One power camera**, player 0's: every event asks `Cam_GetPower(0)`, and the end looks at player 0's current
  camera. With one player camera the shots are placed around the **player** (the follow camera's target), even when
  the clip that fired is a boss's counter; the boss clip's shots then land in the player's frame, not the boss's
  (inferred from the code; not seen at runtime).
- **Two players on one view**: the shots follow the human of the player whose clip fired; the power camera is player
  0's and replaces the shared view.
- **Split screen**: no begin, so no power camera (outside hide-and-seek).
- **The player as victim**: an AI's clip counts only for the boss classes above, through its partner, the player.
  An ordinary enemy's finisher on the player does not move the camera.

## In Coney

### Port status {#power-coney}

The implementation boundary and synthetic-test requirements are on
[Camera implementation](../../engine/camera.md#power-coney).

## Open questions

The inherited runtime notes below identify unobserved cases: blocked shots, mode 1, two players and a boss counter.
The system's [open questions](index.md#open-questions) retain the broader camera comparisons.

## How this was checked

The following observation predates run-record ids. It is retained as historical checking notes, not a new runtime
comparison established by this restructure.

### At runtime {#power-runtime}

Confirmed (runtime) for every reading below, made in PCSX2 as this subsection says.

Confirmed (runtime), PCSX2 2.9.94, a copy of quick-save slot 6 (`level99`, PoizoCiv placed 1.0 m in front and made a
puppet), scenarios `power_camera`, `power_camera_ext` and `power_camera_rage` in
`repo:research/traces/scenarios/` (circle to grab, cross held and square for the power strike, cross taps for the
extension; L1 + R1 with the rage meter written full for the raging one), the player's current camera (`0x005d9150`)
and the power camera's fields read every update:

- **The power strike 57** (`gen_grab_front_power10_p1`) started on update 67, the begin came on update 67 (mode 2,
  count 0, `0x005d9188` set), the shots and the cut on update 68: count 8, mask `0xff` (every shot clear in the open
  street), the pair (4, 5), and the view cut in one update from the follow camera at (36.24, 43.00, 2.76), 65°, to
  shot 5 at (33.73, 34.75, 1.45), 50°, near 0.1, far 115 (copied from the follow camera). The eight stored shots equal
  the clip's events turned by the player's heading at the begin (160.7°) and added to his position, to the
  millimetre. The camera stayed still until update 100, the end event at frame 28 of 33, when the follow camera came
  back at (36.24, 43.00, 2.76), where it had been; `0x005d9188` read 0 from then on. The counter `0x0050b2a0` went
  from 4 to 5.
- **With the extension 59** (`gen_grab_front_power10_p2`, a part with no cut) the shot held from update 68 through
  59, and the follow camera came back on update 181 (the end at the part's last frame).
- **Raging, 63 then 65** (`gen_grab_front_power14_p1~2`, `_p2~2`): the begin on update 134, the cut on 135 to shot 4
  at (31.33, 39.27, 3.48), looking down 53°, with 5 the alternate; the extension started on update 157 and its cut on
  158 jumped to shot 5 at (35.06, 40.31, 1.93) in one update; the follow camera came back on 189 at its position from
  before the move. Both shots' orientations equal `q_h ⊗ q_e` from the clip.
- Slow motion did not come in (`0x005102cc` stayed 1/30 s): neither `gen_grab_front_power10` nor `power14~2` has the
  pair.

Not seen at runtime: a blocked shot, mode 1, two players, a boss's counter.

### Earlier inline observations {#power-inline-observations}

The previous page also attached runtime evidence to the following passages. They are retained here with their
original labels; the checking method and unobserved cases are recorded above. No new run record is asserted.

**The shot's frame.** The position is in the frame of the human the camera follows, at the moment of the begin:
x to his right, y forward, z up from his feet (the transform's position in `0x00714b00`, not the head). The
quaternion turns the camera's own axes (+y the view, +z up, +x right, [Axes](index.md#the-base-camera-object))
into that
frame, so in the world the shot's orientation is `q_h ⊗ q_e` and its position `p_h + q_h · p_e`, with `q_h`, `p_h`
the human's rotation and position at the begin (`CamPower_AddShot`'s quaternion products, `0x0013a470`-`0x0013a530`).
Confirmed (code), confirmed (runtime). On the disc (the survey below) shots stand 0.3 to 6.4 m from the human in plan
(median 2.8 m), from 0.2 m below his feet to 4.4 m above them (median 0.9 m), pitched from straight down to 36° up
(median 1° down), and look within 54° of a point 1 m above his feet (median 12°): low, close and aimed at the body.
Every cut on the disc holds −1.0 (bits `0xbf800000`), so **every power shot is at 50°**; no event carries a near or
far plane.

Every cut is a hard cut: the camera jumps in one update, and no blend camera (type 5) is made. Confirmed (runtime).

1. **Ends** (`CamPower_End`) when the human is gone, down or dead (`Human_IsDownOrDead`), or, outside hide-and-seek,
   when he is in neither side of a paired move (state `0x1000`, the attacker, or `0x2000`, the victim,
   [Combat](../combat/index.md#state-flags)), or he is a victim whose partner has gone or left `0x1000`.
2. Otherwise the camera is put back at the cut's position and orientation (`+0x270`, `+0x310`), the position is
   pushed out of nearby humans (`Camera_PushOutOfHumans` with 3.0, none skipped), and the look point is set 3 m
   along the view.
3. The **shake** (`Cam_UpdateShake`, [Shake](effects.md#shake)) moves that look point; only when it did is the view
re-aimed
   at it (`Mat_LookAtUp` with the shot's own up axis). The camera does not track the human: the view stays fixed for
   the whole shot. Confirmed (runtime): position and orientation unchanged to the fourth decimal across a shot.

`CamPower_End` (`0x0013a3b0`) runs on the clip's `0x3d` event (whatever switch 9 says) or from the update's checks.
When the power camera is a player's current camera it makes the saved camera current with **no blend**. That camera
was not updated while the power camera showed, so it comes back exactly where it was before the move and goes on
from there (the follow camera's activation, [Setting up](index.md#setting-up), step 4). The power camera is
released as
soon as it stops being current (`CamPower_Deactivate` → `Camera_Release`), and the next begin makes a new one with
no shots. Confirmed (code); confirmed (runtime).

So an **extended power move** keeps the power camera only if the next part starts before the current clip reaches
its end event; the part's own cut (`0x3a`) then swaps to the alternate shot. If the end comes first, the view is
back on the follow camera and the next part's cut is dropped (no power camera exists); a part with a begin of its
own (the `~2` variants below) starts a new sequence. Some parts have no cut and keep the shot. Inferred from the
code; the kept shot and the swap confirmed (runtime).

### Recorded filter locator {#power-filter-locator}

The earlier notes identify the partner-skipping filter as `0x00136040`. This locator does not resolve in the current
function listing or image-section map. It remains unresolved provenance here, not a newly verified entry.

### Earlier observations retained from upstream {#upstream-earlier-observations}

These passages came from upstream 8ecd4cb8 without surviving run identifiers. Their original labels and
context are retained as earlier checking notes, not a new runtime comparison.

```text
**The shot's frame.** The position is in the frame of the human the camera follows, at the moment of the begin:
x to his right, y forward, z up from his feet (the transform's position in `0x00714b00`, not the head). The
quaternion turns the camera's own axes (+y the view, +z up, +x right, [Axes](index.md#the-base-camera-object)) into that
frame, so in the world the shot's orientation is `q_h ⊗ q_e` and its position `p_h + q_h · p_e`, with `q_h`, `p_h`
the human's rotation and position at the begin (`CamPower_AddShot`'s quaternion products, `0x0013a470`-`0x0013a530`).
Confirmed (code), confirmed (runtime). On the disc (the survey below) shots stand 0.3 to 6.4 m from the human in plan
(median 2.8 m), from 0.2 m below his feet to 4.4 m above them (median 0.9 m), pitched from straight down to 36° up
(median 1° down), and look within 54° of a point 1 m above his feet (median 12°): low, close and aimed at the body.
Every cut on the disc holds −1.0 (bits `0xbf800000`), so **every power shot is at 50°**; no event carries a near or
far plane.

Every cut is a hard cut: the camera jumps in one update, and no blend camera (type 5) is made. Confirmed (runtime).
```
