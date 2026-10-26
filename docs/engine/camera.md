# Camera implementation

Coney's camera code lives in `src/camera/`. `Cameras` owns view selection, `FollowCamera` tracks its target,
`CameraShake` supplies shake and rumble, and `SlowMotion` supplies the character-step scale. The original's
behaviour is documented in the [camera research](../research/camera/index.md); this page records Coney's implementation
and port boundaries.

## Shake and rumble {#shake}

`CameraShake` (`repo:src/camera/camera_shake.h`) stores one level-driven shake. Its `start()` and `update()` supply
an amplitude, rumble byte and seeded view offset; the caller applies that offset. The view-offset scale remains
`kOffsetScale = 0.05F`. There is no world-amplitude/time setter in this class.

The world-shake port boundary and train behaviour are described in
[World shake](../research/camera/effects.md#world-shake). The earlier implementation notes below record the gap:

**Coney status and the fix for the implementer** (Coney as of `781d7768`): `camera::CameraShake`
(`repo:src/camera/camera_shake.h`) has one shake started at levels 1-3 only; there is no slot 0, no way to set an
amplitude and time from the world, and no `part_train_sound` behaviour (the particle systems do not know the type).
To do:

- give the shake slot 0 (or a second shake on the camera manager) with `setWorldShake(amplitude, seconds)`, the
  original's `+0x1d4` / `+0x1dc`; its current is summed with the human slots and the total capped at the largest
  level ([Slots and the view offset](../research/camera/effects.md#shake)); it never rumbles the pad;
- a `part_train_sound` behaviour: within 60 m of the player camera, every 5 updates, level = (60 − d) × 0.25 / 59
  (0.25 within 1 m) and time 0.95 s; plus its sound ([Script types](../research/script-types.md#subway-train));
- `part_pelham_subway` and `part_level2_subway` when those levels come: 0.02 for 4 s within 30 m / 50 m;
- replace Coney's stand-in offset (`kOffsetScale`) with the original's form above, now known: it decides how much the
  view moves (at the train's peak about ±2.6 cm on the aim point).

## Power-camera port boundary {#power-coney}

Coney has no power-camera class yet. Its port boundary is the
[power-camera research](../research/camera/power.md#power-camera); the required implementation pieces are:

- clip events `0x39`, `0x3b`, `0x3a`, `0x3d` with the [documented payloads](../research/camera/power.md#power-events)
  (Coney's clip loader must keep `+0x04`, `+0x06`
  and the word at `+0x08`); act only for player 0's camera, with the documented gate;
- begin: remember the current camera, the followed human's transform and the mode; shots: store them in world space
  at once;
- the first cut: test each shot with the two sphere sweeps, pick the pair from a counter kept across moves (mod 8,
  from 0), the farther in plan first, hard cut; later cuts: swap to the other shot, hard cut; fov 50°;
- each update: hold the shot, push it out of humans, let the shake turn the view; end on the event, on leaving the
  paired move, or when the human is down or dead; hand back with no blend to the saved camera, unchanged since the
  begin; forget the shots;
- tests with synthetic clips: begin, 8 shots, a cut and an end give a hard cut to the expected world pose and a hard
  cut back; two cuts alternate; blocked shots are skipped; an end before the next part's cut leaves that cut
  ignored; switch 9 off ignores all but the end.

## Door-walk troubleshooting {#door-walks}

In Coney a walk that repeats three times means `StartDoorWalk` runs three times or the walk does not end on the first
message 8; the script has one `StartTheWalk` (scheduled once, and it sets itself to nil after running), so
speculative causes are Coney sending message 8 before the turn to the flag's heading, the `env` callers firing at the
level's opening as well, or the scheduled function running more than once
([q-level95-walk](../research/camera/index.md#q-level95-walk)).

## Follow camera implementation {#coneys-implementation}

`src/camera/follow_camera.*` is the follow camera of `Cam_Follow_Update` (`0x0012ae58`), stepped after the human
by `src/human/player.*` and drawn by `--play-level` ([Running Coney](../guides/running.md#playing-a-level));
`src/camera/follow_collision.*` holds its rays against the collision mesh:

- the look-at point is the feet + 1.4 m, its move each update limited by its length `d` as in [step
  5](../research/camera/follow.md#update):
  all of it within 0.4 m, `1 − 2 × (d − 0.4)` of it to 0.8 m, 20 % beyond; in the air (a jump or a fall) it
  follows the feet directly. A climb's rise of 2.64 m is followed at 20 % for 6 updates, then in 3 (the last one a
  few millimetres);
- the view faces the **[aim point](../research/camera/follow.md#aim-point)** (`FollowCamera::aimPoint()`, the trace's
  `aim_x`-`aim_lag`), stepped
  after the world collision: the lead along the target's facing eased 35% an update (0.365 m ahead standing or walking
  away, as at runtime), the height from the body point (`Human::bodyPoint()`, the hips' bone 2 times the scale) with
  its dead band, the running bob and the sprint sway (Coney's own seeded random numbers), the 1.0 m sphere push, the
  lag from 0.06 (its across test and its 2.5 × overshoot test on the 3D velocity, the speed share on the horizontal
  speed, as in the original: [Long falls](camera.md#long-fall-coney)) and
  the pull back in plan. **Coney's reading** of the state flags: grabbing (`0xc0`) is holding a
  victim in a grab's pair; down (`0xe2c00`) is holding one with no pair (a tackle), mounted, grounded, arrested or
  knocked out; the target counts for the cameras unless dying, dead, knocked out, mounted or arrested (the upgrade
  and key exception is not built), and a long fall is airborne while not counting or grounded. The long fall's 0.6 and
  the halving apply to the update's move, not the eased lag (**Coney's reading**); the blocked-wall share uses the main
  ray's hit face. **Inferred**: the lead's world ray is a mesh ray with no type mask, skipping the see-through
  materials, and the lead's view direction is taken across the ground (runtime 0.363 m, not the 0.375 m a pitched
  view gives). Not built: the slope case, the 0.75 s timer after the aim push, the takeover, aiming, other game modes
  and two targets;
- the wanted position stays put unless its distance leaves the leash band, 4.8-5.3 m (the default distance and 0.5 m
  more, as in the street), then moves along that line to the band; the camera moves 22% of the way to it each step, held
  inside the hard band (4.56-5.65 m), which widens at once and shrinks by 1% of the difference an update;
- **auto-follow** (`0x00129c78`): the auto-centre rule ([Heading](../research/camera/follow.md#heading)) turns the
  wanted
  position toward the
  player's facing at the rule's rate of the angle `a` between the facing and the camera's view at the start of the
  update, at the stored gaits 2, 4 and 5 (walk, run, sprint; so not standing, not in the walk start at 0.76 m/s nor
  at a jog's speed, whatever clip plays). It is held off from a blocked main ray until the player stops (the `+0x45d`
  latch, set by `+0x45b` and cleared on the second update standing, as at runtime), for 0.334 s after right-stick
  input, and while the left stick points more than 157.5° from up (`+0x474`). With the
  option off (the debug menu's *Auto-centre*) the default rule (`0x0012a400`) runs instead, at the run and sprint
  gaits only. With the stick held sideways the player runs in a circle: Rembrandt in the sandbox turns about 197°/s
  at a run and 144°/s at a 35 % walk, against the original's 191°/s and 143°/s (disc test `[disc][player][sandbox]`);
- the **sprint zoom** ([Sprint zoom](../research/camera/follow.md#sprint-zoom)) with the page's fields: the first
update at
  the sprint gait arms
  and latches it (`+0x467`, `+0x466`) when the player has no enemies or the nearest is within 12 m (the brain's query
  is a hook, `Player::setNearestEnemy()`, with no enemies until Coney has brains; out of range the arm waits); the
  zoom function (`0x00128cf0`) saves the band, the zoom distance and the
  target pitch, starts the 0.5 s timer, sets the wanted near edge to the minimum distance and steps the zoom to the
  default (upper pitch limit 30°); the band's ease (`0x0012aae0`, early in the next updates) moves the near edge by
  `d × |d| / T × dt`, which gives the street's 4.569, 4.379, … 3.216, 3.0 to 0.001 m over 15 updates; the target
  pitch goes to 7° in a straight line over 14. The first update off the sprint gait sets the way back for 250 ms
  later; 8 updates on the zoom function puts back the band, the zoom (40° again) and the timer, the same curves run
  back, and on the 15th update the zoom is over. `enableSprintZoom()` is `CamEnable(5, on)` (`0x00126a30`);
- the pitch eases toward its target at 85°/s, between the lower limit (the larger of -20° and the slope of 0.4 m over
  6.6 m) and the upper one of the zoom distance (`0x001254f0`): 50° at the minimum, 40° above the default, 30° at the
  default with one player camera (`0x0050b19c` = 1);
- the **height hold** (step 12): `holdHeight()` eases the wanted position's height 30 % × `+0x398` of the way an
  update toward the height above the look-at point it held;
- the right stick turns the wanted position at the raw rates (yaw up to 150°/s outside the ±48 dead zone, pitch near
  the ends of the travel) and holds off for 0.334 s after any input;
- **collision** ([World collision](../research/camera/collision.md#collision)): the main ray from the look-at point to
  the
  camera with mask
  `0x200 | 0x800 | 1`, so it tests disabled triangles, and every ray passing through materials 30 `LOW_FENCE`, 122
  `RAILING` and 107 `CHAINLINK_NOCLIMB` (`0x00548ab0`); a hit on a
  disabled triangle is cast again without them when the look-at point is less than 0.5 m in front of its plane or the
  player's feet are not in front of it. A hit that stands sets `+0x45b` and pulls the camera to 0.2 m short of it,
  never nearer than 0.5 m. The **side probes** turn the main ray about the look-at point's vertical by 1, 2 and 3 ×
  the probe angle (7° at the band's near edge to 4° at its far edge) each way;
- the camera's forward vector turns the player's stick before the human sees it;
- **the script calls** ([Script calls](../research/camera/scripted.md#script-calls)): `src/camera/cameras.*` keeps
player 1's
  cameras (the follow
  camera, the locked ones, which is current, the blend, the scene stack, the switches, the target list, the watched
  human, the shake and slow motion), and `src/scripting/camera_bindings.*` makes `CamSetupFollow`, `CfgFollowCamera`
  (`FollowCamera::configure()`: band 3.0-3.5 m, zoom step 4.8, 30°), `CamSetFollowZoom`, `CamSetFollowAngle`
  (clamped, reached at once), `CameraReset` (behind the player at the nearest preset, pitch back to 13°),
  `CameraCreateLocked`, `CameraMakeActive`, `CamEnable` (switches 0, 5 and 6 act), `CamTarget` and `CamSetSecondary`
  (keep in view, 0.25 of the field of view, 35 % of the excess, at most 270°/s) real. The zoom step follows the band's
  near edge as it eases (3.72 m and 5.88 m);
- **blends** ([Blends](../research/camera/index.md#blends)): `CameraBlend` lerps the position and look-at point and
  slerps
  the orientation from
  the view shown when it began to the destination's live view, linear in time, the far clip never growing; at the end
  the destination becomes current directly, which runs the follow camera's activation (`FollowCamera::activate()`).
  A cut (`Cameras::cuts()` counts them) is drawn as a jump: a frame drawn between two steps takes the new camera
  whole instead of sweeping from the old one, as the original draws only whole steps.
  A **locked camera** looks along its angles at a point 3 m ahead, its far clip at most 150. While it is current it
  keeps the humans `CamLockLocked` lists inside its sides (`camera::keepInView()`, `repo:src/camera/locked_camera.h`):
  the head point 1.4 m up is pushed to 0.3 m inside the left side, or else the right, the move's part along the side
  clipped by the world 0.3 m short of a wall, the feet snapped to the ground, and after one push every later listed
  human is placed again, as the original's flag carries over. **Coney's readings**: the sides are the planes at half
  the horizontal field of view either side (left first); the move is from the human's feet at the last update; an
  airborne human is not landed (no fall damage);
- the **path camera** ([Path cameras](../research/camera/scripted.md#path-cameras), `repo:src/camera/path_camera.h`):
  `CamSetupPoizo` starts it
  from a camera's view, `CamAddPoizoPoint` / `CamAddPoizoPointCam` append up to 8 points, and while it is current it
  flies the Catmull-Rom curve (each end point's neighbour standing in for the missing one), slerping the
  orientations, and gameplay calls each point's function as it is reached and the end function at the last, where
  it stays. `CamReversePoizo` flies it back: the points reversed, each segment's time moved with it, the new end
  function, the flight started again (**Coney's reading**: each point keeps its own function). A point's angles turn
  it as a locked camera's do (`camera::scriptedOrientation()`, the same code in the original); **Coney's choice**: the
  look-at point is 3 m ahead;
- the **combat camera** ([Combat camera](../research/camera/follow.md#combat-camera)): with L1 held while the player
has a
  fight target
  (`Fighter::target()`) the band's wanted near edge goes to 2.4 m (4.8, 4.44, 4.134, ... at 4.5/s) and the target pitch
  to 15°, the enemy's point (position + half its velocity) is turned toward 27° off the view's centre (0.455 of the
  excess, at most 640°/s beyond 29°), and on release the saved band comes back while the pitch stays;
- the **shake** ([Shake](../research/camera/effects.md#shake)) at the three strengths, 0.66 in combat, its time
counted with
  the characters' step,
  and the rumble byte it drives. Player 1's camera shakes when a reaction plays to his hit (at the hit code's strength
  bits), when he reacts himself from strength 2, and at level 1 when his rage starts (`Fighter::reactionShake()`,
  `Fighter::rageStarted()`, read by `human::Player` after the step);
- **slow motion** ([Slow motion](../research/camera/effects.md#slow-motion)): the player's clip events `0x2e` / `0x2f`
set the
  characters' step
  `SlowMotion::stepSeconds()` to `CfgFollowCamera`'s factor of 1/30 s and back, and every human of the step
  (`Humans::update()`) advances by it: animation, motion, stamina, gravity, an attack's steer and a grab's alignment;
- **in play** (`--play-level`, the story): gameplay (`GameplayMode`) makes player 1's cameras before each level's
  script, gives them to the bindings (`BindingContext::cameras`, read at each call) and to the level (`ScriptedCast`),
  whose player steps them (`Player::setCameras()`); `CamSetSecondary` finds its human at its live position (the
  scripts' brains). A scene's camera is theirs too: the play mode's scene stage pushes and pops it
  (`Cameras::beginScene()`, `setSceneView()`, `endScene()`), so play blends back to the follow camera over the
  scene's `BlendCam`. The level streams round the current camera and draws through its lens, the far clip capping the
  draw distance;
- `--trace FILE` writes the camera's position, look-at point, wanted position, distance, angles, band and
  auto-follow turn after every step, with the player's state ([Running Coney](../guides/running.md#tracing)).

The world viewer keeps its own free camera with the player camera's lens
([The streamed world](../research/world.md#coneys-implementation)).

**Coney choices** where the research is silent:

- **One player camera**: `0x0050b19c`, the number of player cameras (`0x00122ed0`), is 1 (the debug menu's *One
  player camera*), as Coney has no split screen. Off is the two-player case: the sprint keeps the band and only lowers
  the pitch, and the way back goes to the maximum distance less 0.5, as the page says.
- **The zoom distance** starts at the maximum (6.6 m, upper pitch limit 40°), as read in the street, for a camera no
  script configures (the sandbox, `--play-level` without scripts).
- **Locked cameras**: their angles and path points' are turned into a view as the original does
  (`camera::scriptedOrientation()`: `q = qz(heading) × qx(pitch) × qy(roll)`, heading anticlockwise from +y, a positive
  pitch looking up, [Scripted camera angles](../research/camera/scripted.md#scripted-angles)), and the play mode draws
  through
  the camera's own up,
  so a roll shows. The line of sight test, the blend's sphere push and keep-in-view's ray (for a range above 0) are
  left out.
- **A follow camera that is not current** is not updated; it only notes where the player is, so a reset or the
  activation places it on him.
- **The shake**: one shake on the manager, applied to whichever camera is current; the view offset is a random share in
  [-1, 1] of the amplitude × 0.05 m on each axis (the original's form: [Shake](../research/camera/effects.md#shake));
  after
  its time the amplitude
  eases back to 0 at the same 65 %. Coney has one player camera, so a player's hit shakes player 1's. The animation
  event that starts a shake (type `0x20`, [Shake](../research/camera/effects.md#shake)) is not wired yet.
- **Slow motion**: the combat timers (stun, ground and game time, `nowMs`) still count whole updates of 1/30 s, and the
  cameras update by the frame's 1/30 s, as in the original ([Slow motion](../research/camera/effects.md#slow-motion)).
- **The sprint time** `+0x36c` counts only at the sprint gait and is zeroed off it, so every sprint arms the zoom (a
  run before the sprint, as in the street's runs, would otherwise keep it from arming). A sprint that starts while
  the band is still going back saves the band it was going back to, so a quick second sprint does not keep a band
  left half-way.
- **The side probes** give each side's room as the largest clear multiple of the probe angle with every smaller one
  clear; when the two differ by more than 7.5° the wanted position turns toward the roomier side by 20 % of half the
  difference (the turn that would even them up) each update. The fully blocked view's 480°/s turn, the side factors
  `+0x434` / `+0x438`, the second ray from the target, the sphere pushes and the next update's lag are not
  implemented.
- **Disabled triangles**: the main ray's rule is in place, but which triangles the game switches off in a level
  (`0x0034fba0` from a game object's box, [Collision](../research/collision.md#enable)) is not known from the data
  yet, so every
  triangle of a level stays enabled in Coney and the street's disabled panel still pulls the camera in there.
- **The height hold** is never entered by the player yet: the probe and its lower heights are now on the page
  ([Height hold](../research/camera/follow.md#height-hold)) but not modelled, nor the two special modes' 40.5 % and 48
  %.
- Beyond 157.5° the auto-centre rule's falling line is carried on for a running player (5°/s at 180°). The other
  gates of [Heading](../research/camera/follow.md#heading) (human flags, the clip's descriptor flag `0x8000`, state
  flag 4,
  the watched target)
  are not modelled.
- **The blocked-view latch** clears on the second update the player stands (gait 0, on the ground), as at runtime;
  the clearing condition is now on the page ([Heading](../research/camera/follow.md#heading)), and `+0x45c` is not
  modelled
  apart from it.
- **A fresh camera** (at the start, or after the player is put back) sits behind the player at 4.8 m and 13°. `--start`
  with a distance and a yaw places it there instead, its wanted position with it and the hard band stepped once
  (`FollowCamera::place()`), so a trace scenario starts with the camera of the original's save state.
- **The Rumble cameras** (`src/camera/win_camera.*`, [Rumble: win camera](../research/rumble.md#win-camera)):
  `CameraCreateWin`
  sets up the one win camera on the winner, which starts again when made active (the scripts teleport the winner in
  the same frame) and orbits until replaced; its stop condition is not read. `CamDelete` forgets a locked camera,
  cutting to the follow camera when it was current (**Coney choice**).
- **`CamSetFollowHeading(degrees)`** (`FollowCamera::placeBehind()`, the placement `CameraReset` uses): the camera at
  once along the player's forward turned clockwise by `degrees` (180 behind him), looking back at him at the target
  pitch, at the preset distance nearest its own (the sprint zoom's saved edge while one is held) clamped to the band;
  the band moves to that preset and the zoom step to the one after it.
- **The rail, fixed and third-person cameras** (`src/camera/rail_camera.*`, `fixed_camera.*`, `third_camera.*`):
  the rail camera follows [Rail cameras](../research/camera/scripted.md#rail) for player 1 (one target, the set-up's):
  the
  settings and their
  easing, the target and look-at points, the damping, mode 0's segment choice, foot, reach, hand-over, ceiling, fixed
  pitch and end hold, and the leading modes. **Coney choices and stand-ins**: the end hold lasts only while its
  condition does (what clears it is not traced); the tie between two nearest rail points, the 40% look-at catch-up
  (`+0x3ec`), the `+0x330` vector, the split-screen halving and extras, mode 3 (settings 5, 6, 7) and the push out of
  world collision are not built; a rail with no points stands 3 m behind and 1 m above the look-at point. The fixed
  camera always looks at the average (the original's single-entry case is not traced). `CamDelete` forgets a fixed
  camera as it does a locked one and keeps third-person cameras.

## Implementation checks {#notes}

- **`level99`'s calls** ([Script calls](../research/camera/scripted.md#script-calls)): `CfgFollowCamera` leaves the
  band at the minimum (checkpoint 1
  plays at 3.0-3.5 m); `CamSetFollowZoom(1)` from checkpoint 2 moves it to 4.8-5.3 m without moving the camera;
  `CameraReset` puts the camera behind the player and the pitch back to 13°; `CameraMakeActive(…, 1)` is a linear
  1 s blend from the locked camera's view to the live follow camera; `CamSetSecondary` swaps auto-follow for
  keep-in-view; `CamSetFollowAngle(-10)` clamps to the lower limit (−4.3°) and stays until the next reset.
- **Combat camera** ([Combat camera](../research/camera/follow.md#combat-camera)): with L1 held at a target, band to
  2.4 m (4.5/s ease), target
  pitch 15° (kept afterwards), enemy held 27° off centre; restore the saved band on release.
- **A first follow camera** that matches the numbers: look at the player's feet + 1.4 m; keep the camera where it is
  unless its distance to the look-at point leaves the 3.0-3.5 m band, then move it along that line to the band; move
  22% of the way to the wanted position each 30 Hz step; hold a 13° pitch; 65° horizontal field of view, near 0.1,
  far 115.
- **Heading**: in the street the auto-centre rule ([Heading](../research/camera/follow.md#heading)) turns the camera
  toward the facing whenever the
  player moves, with the angle measured along the camera's view at the start of the update, which faces the aim point,
  not the look-at point (about 2° more at a run's circle; [In the street](../research/camera/follow.md#street)); with
  the stick held sideways the
  player runs in a circle. At `level99`'s checkpoint 1 it was not seen. Implement it, with the leash and the right
  stick. The player's stick is turned by the camera's heading before it reaches the character
  ([Characters](../research/characters.md#input)), so the camera must ease, never snap. Gate it on the gait (walk, run
  or sprint;
  not idle, sneak speed or jog), not on which clip plays, and hold it off from a blocked main ray until the player stops
  (the `+0x45d` latch; this is why checkpoint 1 has none). With the stick held sideways the original circles at about
  190°/s at a run and 143°/s at a walk, the rule's turn plus the leash's drag.
- **Sprint zoom** ([Sprint zoom](../research/camera/follow.md#sprint-zoom)): on the first update at the sprint gait
  save the band, zoom and
  target pitch and start a 0.5 s timer; the band's near edge moves by `d × |d| / T × dt` toward the minimum distance
  (`d` what is left, `T` the timer, after it has counted down once), the target pitch in a straight line to 7°;
  250 ms after the sprint gait ends, the same back to the saved values.
- **Look-at point**: limit its move per update by the distance rule of [step 5](../research/camera/follow.md#update)
  (20% above 0.8 m, a
  linear share from 0.8 to 0.4 m, all of it below), except while jumping or falling; any trigger on "a rise" is not
  what the original does.
- **Right stick**: yaw 60-150°/s outside a ±48 raw dead zone, applied to the wanted position at once (the lag
  smooths it); pitch only near the ends of the travel; 0.334 s of no auto-follow after any input.
- **Collision**: start with a ray from the look-at point and pull in to the hit, with the main ray's mask, its
  disabled-triangle rule and the excluded materials 30, 122 and 107
  ([World collision](../research/camera/collision.md#collision)); the triangles'
  enabled bits must follow the game (doors, glass and barriers the game switches off), or a disabled panel pulls the
  camera in. The side probes and swing-away rules can come later.
- **Update order**: the cameras update at the end of the characters' 30 Hz step, after movement
  ([Characters](../research/characters.md#update)), and the device takes the lens once per frame.
- Keep the camera deterministic (no real time) so the test mode can compare frames.

In Coney, `CamCanSee` asks whether player 1's current view sees the point 1 m above a human or 0.3 m above an
object (`camera::canSeePoint()`: within the range and the far clip, inside the 4:3 view window beyond the near clip,
no collision triangle on the line, **Coney's reading**: the default ray mask), and `CamGhostDoor` sets type bit
`0x200` on a door's two triangles, which the follow camera's rays skip. `CameraSetClipping` sets a locked camera's
near and far clips and the follow camera's own far clip (at most 150; the follow camera keeps its 0.1 near clip),
`CameraGetActive` answers the current camera's handle (NilHandle for a scene's or for player 2), `CamGetPos` its
position, and `CamSetFollowPos` puts the follow camera at a point at once.

## Long-fall tracking {#long-fall-coney}

Fixed; this paragraph is how Coney differed before. Inferred, from `repo:src/camera/follow_camera.cpp`
(`stepAim()`) and `repo:src/human/player.cpp` (`followTargetOf()`): Coney built the lag from the **horizontal** speed
only. `c` was 1 while the horizontal speed was under the walk speed, and the ×2.5 needed `!stickOwned` and a
horizontal speed above walk + 18% of (run − walk), with no overshoot test. In a straight fall the horizontal speed is
about 0, so the lag stayed **0.06**, and at 35 m/s the aim point settled about 1.167 × 0.94 / 0.06 ≈ **18 m** above
the look-at point (11 m at 20 m/s) instead of under 5 m. With the camera 3.5-5 m from the look-at point the view then
faced nearly straight up: the user's "camera flies up". Coney's `longFall` (airborne while not counting or grounded)
is not the cause: the original does not set it on a plain fall either. (Coney's headless `--trace` could not show
this: with the release build of that time the trace's camera columns stayed constant even in the `walk60` scenario.)

The fix, now in `stepAim()`: it takes `c` from the 3D velocity against the view's 3D direction as the last update
left it (from the camera's place before the update to the old aim point; **Coney's reading** of the orientation
`+0x20`), with its gate on the 3D speed; keeps `s` on the horizontal speed; gates the ×2.5 on the 3D speed and
applies it when |`P` − look-at|² > 10⁻⁵ and |`P` − look-at|² < |`P` − `new`|² (`new` = the old aim point + (`P` − the
old aim point) × the lag being built, which the original keeps in `+0x418` while it builds it: written at
`0x00133be8` and `0x00133d4c`, read for the
test at `0x00133e20`; confirmed (code)), else when the stick is not the player's. **Open
question**: the slope latch `+0x460`, which also passes the first test, is not modelled (Coney has no slope latches
yet), a marked stand-in in the code. A first version took `c` against the direction to the look-at point, which in
a fall points down along the velocity (`c` near 1), and the synthetic test's lag only reached 0.141 with the aim point
8.8 m above the look-at point; read against the view toward the aim point it passes.

The camera's own height was a second difference: Coney's pitch step turned the wanted position toward its target at
most 85°/s, so in a fall the leash dragged it up its line to the falling look-at point and the camera ended almost
overhead (84-87° above the look-at point, against the original's 52-62°). The original's update step 11 turns the
wanted position the whole way to the target pitch every update (`0x0012d4e8` called at `0x0012beac`, the pitch measured
by `0x0012d908`; confirmed (code), [update](../research/camera/follow.md#update)); the 85°/s is only step 16's return
without a stick. Coney now
does the same (`FollowCamera::update()`).

After both fixes (Coney's `--trace`, the same four falls from `level99` checkpoint 1's yard with the noclip
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
