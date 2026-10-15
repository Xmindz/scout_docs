# 5. Navigation and autonomy

[← Back to index](../README.md) · [← 4. Perception](04-perception.md)

Scout navigates a house by room name. "Go to the kitchen and come back" is a
single sentence that becomes a multi-leg trip across a live map, with progress
reported while it happens and a plain-English account of how it ended.

---

## 5.1 The estimation and navigation stack

```mermaid
flowchart TB
    ENC["Wheel odometry\n(firmware)"] --> EKF["Extended Kalman filter\nfuses odometry + IMU"]
    IMU["IMU\n(firmware)"] --> EKF
    EKF -->|owns odom → robot transform| TF["Transform tree"]
    SCAN["Laser scan"] --> SLAM["slam_toolbox\nmapping or localisation"]
    TF --> SLAM
    SLAM -->|map| NAV2["Nav2\nplanner · controller · behaviours · behaviour tree"]
    SLAM -->|owns map transform| TF
    NAV2 --> SMOOTH["Velocity smoother"]
    SMOOTH --> MUX["Velocity arbitration\nteleoperation wins"]
    MUX --> FW["Firmware velocity loops"]
    BRIDGE["Navigation bridge\nmaps · areas · goals · status"] --> NAV2
    BRIDGE --> VIEW["Rendered map + live status"]
```

| Component | Role |
|---|---|
| **Extended Kalman filter** | Fuses the firmware's wheel odometry with the IMU at 30 Hz in planar mode, and is the sole owner of the odometry-to-robot transform. Which IMU fields are accepted and which are rejected is documented in its configuration. |
| **slam_toolbox** | Builds maps on demand in asynchronous mode; in localisation mode it localises against a saved map. Either way it owns the map transform — which is why the two modes are mutually exclusive. |
| **Nav2** | Global planner, an MPPI controller behind a rotation shim, recovery behaviours, a velocity smoother and a behaviour-tree navigator. Also supplies the measured drive/turn/spin behaviours the agent's relative-movement tools use. |
| **Navigation bridge** | The single service surface for goals, maps and named areas, plus a rendered map view and a live status stream at 10 Hz. Everything above it — the web app's Map view and the agent alike — speaks to this. |

Mapping and navigating are **one service each and cannot coexist**: both would
publish the same map transform, and two disagreeing estimates flipping many times
a second reads as a localisation bug rather than a configuration mistake. The
service manager enforces it, so starting either stops the other.

---

## 5.2 Rooms instead of coordinates

Scout's map carries **named, oriented rectangular areas** — "Kitchen", "Hall".
That one modelling choice is what makes natural-language navigation work.

- **Names are free-form and not unique**, and area identifiers are opaque. So what
  a person says is resolved case- and whitespace-insensitively, falling back to a
  substring match, and answering with a **question** when several rooms match or
  with **the list of real names** when none does — never with a guess. An
  identifier is accepted directly too, which is how a person answers "which
  bedroom did you mean?".
- **An area goal completes when the robot is inside the rectangle**, not when the
  planner reaches the single point it aimed at.
- **"Which room am I in" is asked of the robot**, which already owns that
  containment test. No geometry is duplicated in the agent, so the two cannot
  drift apart.
- **Areas know their own angle**, which is also what lets the map be drawn levelled
  to the building's walls rather than to the map frame's arbitrary axes.

Named areas can be created from the phone (drag out a box on the Map view) or by
asking the robot to name where it is standing.

---

## 5.3 Knowing where it is

slam_toolbox in localisation mode tracks the robot well from a good starting
guess, but it cannot *find* the robot, and nothing it publishes says whether
its answer is right. The map-to-odometry transform exists whether the pose is
accurate or two rooms away, so a "pose valid" flag only ever meant "a transform
exists". A robot that has been picked up and carried, or that started from a
stale guess, would plan from the wrong room with complete confidence.

So the navigation bridge answers the two questions that used to be left to the
operator's eyes:

| Question | How | Cost |
|---|---|---|
| **Check** — does the live scan fit the map at the pose we believe? | Score the scan at the current pose | A few milliseconds |
| **Search** — where on the map does this scan fit? | Score candidate poses across the whole map, or one named room | About a second |

Both score a scan the same way, against a **likelihood field**. Every map cell
holds a value that falls off with distance to the nearest wall, and a pose's
score is the mean value under the scan's endpoints. 1.0 is every laser return
landing on a wall; a scan in the wrong room lands mostly on open floor and
scores low. Unknown cells count half, so a scan that reaches past the mapped
area is not scored as if it had hit something. Measured on this robot with a
single scan: **0.98** at the true pose, **0.88** for the best wrong spot on the
right map, and **0.73** for the best spot anywhere on the wrong map. The
thresholds come from that, and are kept as parameters: one scan in one place
is a start, not a calibration.

How it is used:

- **Continuously.** Once a second the live scan is scored at the current pose, and
  the verdict — **ok**, **doubtful** or **lost** — is part of the navigation status.
  The Map view shows it as a chip, and "where am I" mentions a doubt.
- **Before every goal.** Every navigation goal, a tap on the Map view included,
  runs the check first. A wrong pose is corrected before planning, and the goal is
  refused if the robot is lost and cannot find itself.
- **On request.** The `localize` tool and the agent's response to "you're lost" or
  "I moved you" run the search. They can search one room when the person says
  where the robot is, and they re-seed localisation **only on a confident,
  unambiguous match**. Two places that fit equally well is an answer of "I'm not
  sure", not a coin toss.
- **Across restarts.** A pose that checks out is saved with the map, and navigation
  starts localisation from it next time, so a robot that was switched off where
  it stood comes up already knowing where it is.

The Map view's "Set position" gesture remains for seeding localisation by hand.

---

## 5.4 Multi-leg missions

The robot's goal interface is one-shot. Trips are made out of it by a mission
runner in the agent:

```mermaid
flowchart LR
    START["Snapshot start pose"] --> L1["Leg 1\ndispatch goal"]
    L1 --> P1["Poll status ~2 Hz\nemit progress every few seconds"]
    P1 --> T1{"Terminal state?"}
    T1 -- succeeded --> L2["Next leg"]
    T1 -- failed or cancelled --> STOP["Stop the trip\nreport honestly"]
    T1 -- timeout --> CANCEL["Cancel the goal\nthen stop"]
    L2 --> HOME["Optional return to start pose"]
```

The behaviours that matter:

- **Progress, not silence.** A progress line every few seconds; the chat surface
  posts one message and edits it in place, so a two-minute drive shows a shrinking
  distance-to-go.
- **A timeout cancels the goal** rather than merely giving up on waiting — the
  alternative is a returned error and a robot still driving.
- **The trip stops at the first leg that does not succeed**, because after a failed
  leg the robot is not where the rest of the trip assumed it would be.
- **"Come back" is synthesised.** There is no return-to-start command on the robot,
  so the start pose is captured before the first leg and the trip finishes with a
  goal back to it — landing within the planner's tolerance (about 0.12 m), not
  exactly on the spot, which is what the reply says.
- **Navigation is started when needed.** If a goal is requested while navigation is
  not running, the agent starts it, waits up to a minute, and reports the wait. If
  that stops an in-progress mapping session and discards an unsaved map, it says
  so in the conversation, so the loss is visible rather than silent.

### What the agent will not do

- **Send a goal when the robot does not know where it is.** An invalid pose means
  it would drive somewhere arbitrary.
- **Send a goal on a map that is drawable but not navigable.**
- **Touch the protected services** — the serial bridge to the microcontroller
  (opening its port resets the board and drops motor state mid-motion) or the
  web/bridge stack it is itself talking over.

### Overrides

`stop_robot` cancels any active goal *and* zeroes the velocity, and is never gated
behind a confirmation. Stopping a follow is the same: it takes no arguments, works
from every state including when nothing is running, and is never held back for a
confirmation. Manual driving is an override in its own right — any non-zero
teleoperation command cancels an active goal on the robot side.

---

## 5.5 Relative movement, done properly

"Move forward one metre" used to mean publishing a fixed velocity for a number of
seconds. Seconds are not metres on a tracked base: the tracks slip, the smoother
ramps, a busy host publishes late — and nothing on that path was looking at the
laser scanner, so "forward" drove into whatever was in front.

Relative moves are now **navigation goals** running the planner's own measured
drive, back-up and spin behaviours. That means they are:

- measured on odometry rather than timed,
- **stopped short of obstacles** on the local costmap,
- cancellable and reportable like any other goal,
- bounded (5 cm to 3 m; 5° to 360°), with a request past those limits answered in
  words rather than as a bare refusal.

A useful acceptance check: asking for one metre forward towards a wall 1.5 m away
should stop about half a metre short and say why.

---

## 5.6 Directions, remembered spots and the map picture

| Capability | Behaviour |
|---|---|
| `distance_and_direction` | Straight-line distance from the current pose to a room's centre or a remembered spot, a **body-relative** direction ("ahead and to my left"), and the turn that faces it. Never a compass point, because the map frame's axes are arbitrary. It is not a route length, and says so. |
| `remember_this_spot` | Labels the current pose for this session. Tied to the map it was recorded on, and refused on any other. |
| `name_this_place` | The permanent version — writes a named area to the map. |
| `go_back` | Drives to a labelled spot, or with no label to wherever the last trip started. Small nudges and "go back" itself do not overwrite that departure point, so "go back" after edging forward still means the room it came from. |
| `show_map` | Fetches the latched rendered map the web page already uses, draws rooms, the robot and remembered spots on it, levels it to the walls, and delivers it as a picture. Subscribed on demand rather than held open, since mapping republishes a full image every few seconds. |

The robot's pose is drawn **only on a live view**. A map read off disk is labelled
as saved and drawn without a robot, because a picture the robot is not localised
in should not show a confident position.

---

## 5.7 Mapping from a phone

The web app's Map view runs the whole workflow without a terminal: **new map → drive
around → save map → navigate**. Those three actions switch the mapping and
navigation services through the same allowlisted path the Status view uses. The
page also offers tap-to-navigate, tap-a-room-label to drive into it, drag-out to
create a room, pan and pinch zoom, and a "set position" gesture — press where the
robot really is and drag the way it faces — for seeding localisation.

With nothing running it draws the saved map off disk and says so, because a
picture the robot is not using should be labelled as one.

<p align="center">
  <img src="images/web-map.png" alt="The Map view: chips reading READY, Ground and Localised; the house's occupancy grid with labelled rooms including Kitchen, Living, Dining, Bedroom, Guest Bedroom and Dock; the robot drawn as a small green marker; Stop, Set position, + Area and Maps buttons" width="300">
  <br>
  <sub>The Map view. The chips show navigation state, the active map and the localisation verdict. Named rooms are drawn as rectangles levelled to the walls, and the robot is the small marker in the hallway. Tap a room label to drive into it.</sub>
</p>

---

## 5.8 Tuning a tracked base: what the robot taught

Tracked chassis are not differential-drive textbook robots, and several
navigation settings on Scout exist because of measured behaviour. These are
documented here because they are the interesting part of making autonomy work on
real hardware.

- **In-place rotation has to break stiction.** With a narrow track separation, a
  spin puts very little linear speed at each track, and below a threshold the
  tracks cannot break static friction — the robot buzzes without turning. Worse,
  the sampling controller returns a cost-weighted *average* of its candidate
  trajectories, and the average of "turn left" and "turn right" is near zero, so
  it is structurally unable to choose a tank turn. The fix is a **rotation shim**
  ahead of the main controller, with a rotation speed above the stall point, an
  acceleration limit high enough to cross that point within a tick or two, and an
  open-loop ramp — because ramping from *measured* speed while stalled clamps
  itself below breakaway and deadlocks.
- **A velocity deadband cannot substitute for that.** Two deadbands (one stopping
  the controller from *proposing* a creep, one zeroing whatever still arrives)
  keep a small angular term on purpose: stiction is a standstill problem owned by
  the rotation shim, and once the tracks are already sliding, a gentle correction
  executes fine. Raise the angular deadband to stiction levels and the robot can
  only swerve or go straight — so it weaves, or circles a goal directly ahead.
- **One acceleration limit written in three places means the smallest wins**, since
  each stage re-limits the one before it. A launch-time check warns when they
  drift apart, because nothing else logs it: the controller publishes exactly what
  it intended and a downstream node quietly flattens it.
- **Arriving "facing anywhere" is a decision.** The map frame sits at an arbitrary
  angle to the building, so a goal with no requested heading and a tight heading
  tolerance makes the robot pivot to an arbitrary direction after arrival — with no
  linear error left, the correction comes out rotation-only and below breakaway.
  Accepting any final heading, and disabling the critic that pushes for one, are
  **one decision** to be revisited together if the interface ever grows a way to
  say "arrive facing this way".
- **Gaps between critic thresholds strand the robot.** Between two mismatched
  "close enough to consider" distances, neither critic drives the robot forward:
  the goal is accepted, the robot sits still, and it fails in recovery behaviours.
- **One publisher per velocity topic, always.** Two publishers interleave at many
  hertz and last writer wins, which presents as stuttering rather than as a
  configuration error.

---

| ← Previous | Next → |
|---|---|
| [4. Perception and vision](04-perception.md) | [6. Robot platform](06-platform.md) |
