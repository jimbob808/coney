---
kind: topic
system: camera
owns:
  functions: ["0x00130990"]
---
# Follow camera: world collision

Verified against: `SLUS_212.15` (NTSC-U); findings below retain the earlier camera research's evidence levels.

## Summary

How the follow camera moves its wanted position around world geometry: ray probes, swing, pull-in and recovery.
The follow update and its heading rules are on [Camera](follow.md#update).

## Map

| Address | Function | Subject | Evidence |
| --- | --- | --- | --- |
| `0x00130990` | `Cam_Follow_Collide` | world probes and the resulting camera position | confirmed (code) |

## Data

The collision step uses the follow camera's position, aim point, distance band and collision history fields.
Their shared layout stays on [the follow camera object](follow.md#the-follow-camera-object); individual fields
and constants are cited with the step that reads or writes them below.

## Behaviour

### World collision {#collision}

`0x00130990` (about 3,000 lines decompiled) runs after the camera's wanted position is known. Confirmed (code) for the
calls and constants; the overall reading is inferred:

- **Materials it ignores**: at `0x00130a5c` it copies a list from `0x00548ab0` to `sp + 0x40`: **30 `LOW_FENCE`,
  122 `RAILING`, 107 `CHAINLINK_NOCLIMB`**, ended by 1. Every `CollisionMesh_RayCast` of the step passes it as its
  fourth argument, the materials to skip (`0x001311a8`, `0x001312e0`, `0x00131730`, ...). So the camera sees
  through low fences, railings and unclimbable chain-link. Confirmed (code).
- **A height ray** (`0x00130c28`) straight down from the look-at point (`0x00511770`, (0, 0, −1)), as long as the
  look-at offset plus 0.5 m (1.9 m), mask `0x200`. Only a hit on a face whose normal's `z` lies from 0.6
  (`0x005d91cc`) up to cos 15° (a slope between 15° and 53.13°, tested at `0x00130c44`-`0x00130c64`) calls
  `Cam_Follow_SlopePitch` (`0x0012f3e0`); flat ground and steep faces leave the pitch offset (`+0x3a4`) as it is.
  `0x0012f3e0` has two cases, picked by the dot product of the view direction (look-at point to the wanted position)
  with the slope's normal: below −cos 87° (the camera on the uphill side) and above cos 87° (the downhill side);
  within 3° of across the slope it does nothing. Each case casts its own `0x200` ray along the reversed normal, at
  least 6 m long, from a height above the look-at point (the up case's from the caller's height, the down case's
  1.45 m less the look-at offset `+0x208`), and acts only when that ray hits a face within 0.5° of the first normal:
  it then sets `+0x460` and eases `+0x3a4` 10% per update toward an angle from the slope (the up case toward the
  slope's angle, the down case toward half that scaled by the dot product, opposite in sign), or sets it to 0 when
  a second direction the caller passes (its `t0` argument) lies on the wrong side of the face. Confirmed (code) at
  `0x0012f3e0` for the cases, rays and constants; which way each case tilts the view is inferred.
- **The main ray** (the cast at `0x001311b4`, the recast at `0x001312e4`): from the look-at point toward the wanted
  position, its full length, with mask **`0x200 | 0x800 | 1`** (`| 1` only with one target). Through the [ray cast's
  rules](../collision.md#ray-cast) that skips triangles with type bit 9 (`0x200`), and **tests disabled triangles too**
  (mask bit 0); one-sided faces are hit only from their front. When the hit is a disabled triangle (flag bit 0 clear,
  and not `0x800`) and either the camera of the last update (`+0x1e0`, the position copied at the end of each update) is
  not in front of its plane (by less than 0.001 × (1 + the plane normal · the direction from the last look-at point
  `+0x270` to it)) or the look-at point is less than **0.5 m** in front of it, the ray is cast again without mask bit 0,
  so the disabled triangle is ignored; a hit sets `+0x45b` (which latches auto-follow off until the player stops,
  `+0x45d`, [Heading](follow.md#heading)) and its distance becomes the limit the rest of the step works from. When
  that ray hits
  something that is not a ceiling (normal `z` above cos 150°), a second ray (`0x001314d0`) from the camera of the last
  update to the look-at point checks whether the obstacle is between them; if so (and the latch `+0x479` is not −1), the
  main ray is cast again from a point moved along the view by `0.8 / tan(3 × probe angle)` and that distance is added to
  its hit. Confirmed (code) for the masks, the 0.5 m and the recast; the meaning of the second ray inferred.
- **Side probes** (casts from `0x00131744`): the main ray turned about the vertical through the look-at point by **3, 2
  and 1 × the probe angle** to each side (the loop counts down from 2), each as long as the main ray, with the same mask
  and the same disabled-triangle recast. The probe angle is 7° at the near edge of the distance band down to 4° at the
  far one (`7° − 3° × t`, `t` the position in the band). The free angle found on each side is limited to 3 × the probe
  angle. A table of fractions 1.0, 0.7, 0.5, 0.3 and 0.15 (`0x00548b70`) is copied to the stack beside them
  (`0x0013106c`) but never read: no other instruction in `0x00130990` touches that copy (confirmed (code); a leftover).
  After the swing the step keeps the two free angles in `+0x42c` and `+0x430` (0 for a side with nothing in the way,
  `0x00132be4`-`0x00132c04`), for the next update's centring test below.
- **Swinging away**: when one side is clearly freer (the two differ by more than 7.5°), the camera yaws toward it by
  20% of the needed angle per update. When the view is fully blocked it turns toward the target direction by
  the angle between them, at most **dt × 480°/s** (`8.377581` rad/s); the latch `+0x479` records the first
  direction (1 for a positive turn, 2 for a negative one) and, when the needed turn reverses, becomes −1: the turn is
  given up (0), the player's position is kept in `+0x2c0`, and the camera pulls in to the hit instead. `+0x479` is
  cleared when the view is not fully blocked (inferred). Confirmed (code) before `0x001325cc`.
- **Centring in a gap** (`0x001320fc`-`0x00132198`, confirmed (code)): when both sides are blocked and the probe hits
  are uneven (the farthest hit lies beyond the nearest and short of the full ray) the 7.5° rule is not used. Instead,
  while the innermost probe on one side or the other hits short of the full ray and **both** free angles have changed
  by more than 0.1° since the last update (`+0x42c`, `+0x430`), the swing for this update is
  `+0x428 × 0.75 + (a − b) / 2 × 0.25` (`a` the `+0x430` side's free angle, `b` the `+0x42c` side's), stored back
  in `+0x428`: an eased turn toward the middle of the opening (inferred: a player walking down an alley). When only one
  side changed, or neither, it does not turn. `+0x428` is written only here and zeroed by the constructor.
- **While a hand-over is queued** (`+0x47c` set, [update step 21](follow.md#update)) the main ray, the side probes and
  the
  swing away are skipped (`0x001313b0` jumps to `0x00132060`), so the short turn before a mini-game or mugging shot is
  not held back by walls. Confirmed (code).
- **What moves the camera, in order** (confirmed (code) at the addresses cited; the step works on a copy of the
  wanted position `C`, from the look-at point `L` along the unit direction `u` = (`C` − `L`) / |`C` − `L`|):
    1. The swing away (above) turns `C` about `L`.
    2. **Pull-in** (`0x001327f0`-`0x00132a20`), when the main ray or the side probes are blocked: `C` = `L` + `u` ×
       the **nearest hit distance among the main ray and the six side probes** (the probes are as long as the main
       ray, so a face met at a slant by a probe can be nearer than the main ray's hit). With only side probes
       blocked and `+0x471` clear, the distance is instead the farthest probe hit, or, when that is nearer than last
       update's allowed distance, last update's distance moved toward it by 2 × `+0x1a0` (the time step, inferred) of
       the gap. `C` is then moved off the hit's plane, and off a second
       probe plane if there is one, until it is `m` in front of it, with `m` = min(near plane × (1 − `n` · (−`u`)),
       **0.01 m**): almost nothing for a face met head-on. No margin like Coney's 0.2 m.
    3. **Creep out** (`0x00132f5c`-`0x00133250`, [Heading](follow.md#heading)): when `C` is now farther from `L` than
      the
       allowed distance of the last update, it is put at that distance plus 4% of the gap (blocked) or 3% (clearing).
       A pull-in nearer than last update's distance stands at once. Confirmed (code).
    4. `Camera_PushOutOfHumans` (`0x00122728`) when `C` is more than min(`+0x300`, `+0x32c`) or 1 m from `L`.
       Confirmed (code).
    5. The aim point `P` work, including both radius-1.0 sphere pushes ([The aim point](follow.md#aim-point) step 3,
      and again
       in `0x0012f858` after it raises `P`): those pushes move `P` (or `+0x240` with two targets), **never `C`**.
       Confirmed (code).
    6. **Raise when blocked** (helpers, below) lifts `C`'s height only; its plan distance is unchanged.
    7. The **wall push** (below), only while `+0x464` is set: the one sphere push of the camera itself, radius the
       near plane + 10⁻⁵.

  So the distance a blocked view leaves is the nearest of the seven hits, less up to 1 cm; the near plane is not kept
  clear of the face unless the wall push runs. In the street run above the camera came to rest 0.25 m in front of the
  disabled panel; a slanted side probe meeting the panel nearer than the main ray fits that (inferred). (The sway
  `+0x3f8` is the sprint's, at most 0.028 m: [The aim point](follow.md#aim-point) step 2.)
  The [aim point](follow.md#aim-point) gets its own 1.0 m push. **Wall push** (only while `+0x464` is set, set by
  [choose a clear
  heading](#collision-helpers); one view per player; the final position's distance in front of a near-vertical face,
  `|n.z|` < cos 10°, under 1.05 × the near-plane radius, or the main ray blocked): a sphere push of the near-plane
  radius, then the camera is moved off the plane to that radius, the wanted and held positions are reset to the band's
  near edge on the line from the look-at point, `+0x45b` is cleared, the side factor on the side the camera is turning
  toward (inferred) is set to −1 and the other to 1 ([right stick](look-controls.md#right-stick)), and the lag becomes
  max(lag, 1): the
  next update takes the position at once. Confirmed (code) from `0x0013464c` to `0x00134a18`.
- **The lag it chooses for the next update** (`+0x388`), in this order, confirmed (code) from `0x00132a24`:
    1. **0.27** when the main ray was blocked (`+0x45b`), or when the full-block turn was given up (`+0x479` = −1)
       and `+0x471` is clear; **0.27 − 0.05 × s** when `+0x471` is set (set when a given-up turn finds one side
       open), `s` being the nearer side probe's free distance as a share of the probe length (0 to 1); else
       **0.22**.
    2. Back to **0.22** when the main hit is a ceiling (normal `z` below cos 150°) and a slope pitch `+0x3a4` is set.
    3. When neither `+0x45b` nor `+0x471` is set, the new value is reached only by **0.5% per update** from the old
       one (`old + (new − old) × 0.005`), so after a block the lag drifts back from 0.27 to 0.22 over a few seconds.
       Otherwise, when the camera moved and no timed heading target runs, it is scaled by
       `min(1, d × 0.2222) × 0.7 + 0.3`, `d` the free distance in front (so 30% of it at 0 m, all of it from 4.5 m).
- **Side factors** `+0x434` / `+0x438` are set to 1 at the end of every collision step except after the wall push
  above. They are not eased.
- **Recovered distance** `+0x380` eases toward the distance the probes allow by 10% per update; nothing reads it
  (inferred from the field's readers), so it can be left out.
- **No final line-of-sight test**: the position the step returns is used as it is (update step 20).

#### The step's helpers {#collision-helpers}

Confirmed (code) at each function; where the reading of a direction or a gate is not certain it says so.

- **Slope pitch** (`0x0012f3e0`, called at `0x00130c8c`). The height ray (down from the look-at point, the offset +
  0.5 m long) must hit a slope between **15° and 53.13°** (normal `z` from cos 15° down to 0.6, `0x005d91cc`). With
  `h` the horizontal direction from the look-at point to the camera, `d` the 3D one, and `n` the slope's normal:
    - **Camera uphill** (`h · n` < −cos 87°): a ray from the look-at point's x, y at feet + max(offset / 4, 0.25 m),
      along −`n`, as long as max(the camera's distance in plan, 6 m), mask `0x200`. If it hits the same plane (normals
      within 0.5°), `+0x460` = 1 and, when `d · n` < 0 (the camera below the slope's plane), the target is
      `asin(−d · n)`; `+0x3a4` moves 10% of the way toward it but never past it (`min`); otherwise `+0x3a4` = 0.
    - **Camera downhill** (`h · n` > cos 87°): the same ray from feet + 1.45 m; on the same plane, `+0x460` = 1 and,
      when `d · n` > 0, the target is `−asin(d · n) × (d · n) / 2`, eased 10% (`max`); otherwise 0.
    - **Use** (`0x00130c94`-`0x00130ef4`): while `|+0x3a4|` > 10⁻⁵ the wanted, held and last-wanted positions are
      turned about the look-at point by `+0x3a4` round the camera's horizontal side axis (a pitch), and the turned
      height is kept in `+0x3a0`; when `+0x460` was not set this update the offset first decays 10%. So on a slope
      the camera is raised off the uphill ground and lowered a little toward downhill ground (the sign inferred from
      the two cases). `+0x460` also ends the height hold, bounds the stick's target pitch (update step 16) and picks
      the 0.22 lag under a ceiling.
- **Tilt over camera obstacles** (`0x0012ec50`, before the collision step). The objects listed at `0x00715280`
  are the **moving vehicles**: `WorldObject_Init` (`0x00391adc`) adds an object only when its type's kind byte `+0x86`
  is 29 (`TYPE_MOVINGVEHICLE`) and player 1's follow camera already exists (`Cam_GetFollow(0, 0)`); no other flag is
  tested. On the disc that is six object types (`dyn_coney_car_a`, `dyn_hearse`, `dyn_ridecart`, `dyn_rogues_car`,
  `dyn_s_train_kill_p`, `dyn_turnbul_bus`, [Objects](../../references/objects.md)). `WorldObject_Delete` removes it, the
  follow camera's constructor clears the list, and a 32nd vehicle is not listed.
  They are boxes: their type's size (`+0x78`, `+0x7c`, `+0x80`) halved,
  at their position and rotation. A sphere of radius max(the near plane, the player's capsule radius) is swept from
  the look-at point to the wanted position against each (`Sweep_SphereOrientedBox`); the first hit `t` < 1 is used:
    - when `t` = 0 or the view runs within 10° of the object's −y axis (`cos 170°`), the camera **tilts over** it:
      with `D` the camera's distance, `h` its height over the look-at point and `top` the box top + 1.0 m over the
      look-at point, if `h` < `top` the target is `asin(h / D) − asin(top / D)` and `+0x3a8` eases 90% toward it
      (`max`), `+0x461` set;
    - otherwise the camera is **pulled in** to `t × D − near` along the line, and if that moves it away from the last
      camera position `+0x1e0`, only 30% of the way from there.
    - While `|+0x3a8|` > 10⁻⁵ the wanted position is turned by it about the look-at point round the view's side axis;
      without a tilt this update it decays 10% per update.
- **Raise when blocked** (`0x0012f858`, when the step pulls the camera in). `s` = clamp((m − dist) / (m − 1), 0, 1),
  `m` = max(minimum distance, 0.6 × near edge), `dist` the pulled-in distance: 0 at `m`, 1 at 1 m. For a wall hit
  (normal `|z|` < 0.6), not aiming, the target a player: the camera's height moves `s` of the way up toward feet +
  2.0 + 0.085 m (feet + 0.5 m or the look-at height in states `0xe0c00` / `0x2000`, feet + 0.145 × scale with the
  last argument set; inferred: crouching), never down. Then with one target, the main ray blocked, no clear heading
  latched (`+0x463`) and no timed move (`+0x475`), **choose a clear heading** (below); then the height moves `s` of
  the way to that target − 0.05 m once more, applied to the aim point `P` (one target; `+0x240` with two), and `P`
  is sphere-pushed again with radius half of 2.0 (1.0 m; 0.5 m on one sway path), the same push as
  [The aim point](follow.md#aim-point) step 3. Its result goes to the step's push flag (`+0x45a`). The camera is
  not pushed
  here.
- **Choose a clear heading** (`0x0012fd20`). Only when the player is less than **1.8 m** in front of the blocking
  face, not aiming, not holding someone in front, and the allowed distance is under 9 m (or he is in a fight stance
  that counts, or has one of state flags `0x3800f0c00`). It sets `+0x462`, `+0x463` and `+0x464` (the wall push), and
  casts 11 rays (mask `0x201`) from the look-at point: along the face's normal, and turned about the vertical by ±30°,
  ±45°, ±60°, ±77° and ±82°, each `L / cos(angle)` long (`L` the allowed distance); each scores its free length, at
  most `L`. The best average of three neighbouring rays wins (ties go to the straight one), and `+0x43c` = that ray's
  heading − 180°: update step 9 then turns the camera there over 0.1 s. The three latches clear when neither the
  camera nor the look-at point is within **1.65 m** in front of the blocking face (end of `0x00130990`).
- **Sphere test** (`0x001303c8`, switch 1, off by default). A physics sphere of the near-plane radius at the camera
  is collided with the physics world; hits on object kinds 0xc-0xe, 0x10 and 0x1c are ignored (what they are is not
  traced). Any other hit puts the camera at the contact point plus the near plane along the contact normal, except
  on a box body, where the camera is moved along the view line over a timer `+0x410` (0.15 s when unset) to stay
  two near planes off the box's nearer face (inferred).

## In Coney

The implementation status is described with the other camera port notes on
[Camera](../../engine/camera.md#coneys-implementation). Those notes will move to the Engine section with this
restructure.

## Open questions

The inherited reading distinguishes confirmed constants and calls from inferred geometric meaning below.
The camera system's [open questions](index.md#open-questions) retain the outstanding comparisons.

## How this was checked

The earlier camera page recorded the following runtime observation in PCSX2. It has no surviving run-record id;
this restructure retains the observation as historical checking notes and does not claim a fresh comparison.

**At runtime** (stick magnitude 1.0, `level99` checkpoint 1), after running for 1.2 s (a run needs a stick magnitude of
at least 0.95, [method](../../guides/research-workflow.md#driving-pcsx2)): the camera was 3.16 m from the look-at point
(3.08 m horizontally) and 0.70 m above it, a pitch of about −12.8°, which is the 3.0-3.5 m leash band and the 13° target
pitch. Confirmed (runtime).
