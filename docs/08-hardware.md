# 8. Hardware

[← Back to index](../README.md) · [← 7. Firmware](07-firmware.md)

Scout is a tracked indoor rover. Everything on it is either sensing, computing,
or moving — and each component below is named with what it contributes to the
features described elsewhere in these docs.

---

## 8.1 Component summary

| Subsystem | Component | Contribution |
|---|---|---|
| **Compute** | Raspberry Pi 5, Raspberry Pi OS host plus containerised ROS 2 | Runs the agent, the voice pipeline, mapping, navigation, camera streaming and the web stack |
| **AI acceleration** | Raspberry Pi AI HAT+ 2 — Hailo-10H NPU (PCIe, 8 GB onboard memory) | Object detection **and** on-device speech recognition, time-shared in one process |
| **Camera** | Raspberry Pi Camera Module 3 | Live WebRTC view, still photographs for the agent, detection input. Opened exactly once |
| **Camera mount** | Two hobby servos (pan and tilt) | Aiming: operator, agent and tracker, by priority. Roughly ±50° pan and far more up-tilt than down-tilt in practice |
| **Audio** | Four-capsule USB microphone array with onboard speaker and hardware audio front end | Wake word, speech capture, direction-of-arrival, spoken replies, announcements, cue tones |
| **Range sensing** | Slamtec RPLIDAR C1 — 360°, 10 Hz, 0.05–16 m | Mapping, localisation, obstacle avoidance for both navigation and person following |
| **Inertial sensing** | Bosch BNO055 9-axis fusion IMU | Fused with wheel odometry; heading source for the firmware's straight-line hold |
| **Motion control** | ESP32-S3 (16 MB flash, 8 MB octal PSRAM), micro-ROS over native USB | Velocity loops, odometry, IMU and battery publishing, servo drive, failsafe |
| **Motor driver** | Cytron MDD3A dual channel | Two tracks, direction-pin PWM, coast and brake states |
| **Drivetrain** | Geared DC motors with quadrature encoders, 45 mm drive wheels, ~20 cm effective track separation | Closed-loop velocity, odometry ticks |
| **Chassis** | Tracked differential base, about 0.40 m × 0.30 m | Indoor mobility over thresholds and carpet |
| **Power** | 3S lithium pack (about 7.5 Ah), full charge 12.6 V, cutoff 8.7 V, sensed through a voltage-divider module | Battery state of charge, published and shown in the web header |

---

## 8.2 Notes worth knowing per component

**The neural accelerator is the reason two AI features coexist.** Detection
inference runs entirely on the accelerator's own silicon and memory, so it does
not compete with the host's software video encode the way a CPU-only model would
— which is what allows a lagless live camera view and continuous detection with
no mode switch. The same device also serves speech recognition, because it is
handed out as a single virtual device that only one process may hold (see
[Perception §4.2](04-perception.md#42-object-detection-on-the-npu)).

**The camera is a single-open device.** This is a platform constraint, and it
shapes the whole vision architecture — one native camera server owns it, and all
four consumers draw from that stream.

**The laser scanner sees one thin slice**, about 20 cm off the floor. It is blind
to glass, to table tops, to anything lower — and to a step down or a staircase,
which nothing on this robot can see at all. This is stated in the agent's
policy, so the robot says it rather than discovering it.

Two scanner quirks are documented rather than worked around: its stop-motor
command reports success without actually stopping the scan loop, so the correct
way to stop it is to stop its service; and its driver handles only one shutdown
signal, which is what breaks the scan loop and lets the motor stop — so the
service stops it with that signal specifically.

**The IMU driver choice was deliberate.** The breakout is one vendor's, but the
part underneath is a stock sensor at a stock address, so the choice was purely
which driver is better to live with. The selected driver can read back *and*
write the fusion's learned gyro bias — so a stationary calibration can be
persisted — and can enable the external crystal.

**The microphone array is exclusive.** Audio capture cannot be shared, so the
command-line voice tools cannot run while the agent service holds the
microphone. The array also resets its output level every time it re-enumerates
on USB, and a system rule reacts by setting it to full — a fine default, and
wrong the moment somebody has asked for quieter. So the robot's chosen volume is
**restored** after every re-enumeration rather than defended.

**The array's loudness scale is decibels, not percent.** Mapping a percentage
straight onto it would put "half volume" about eight times quieter than
expected, so the user-facing range is spread across a decibel range instead, with
zero meaning off. There are two mixer controls and both must be set, or the robot
ends up half as loud as it just said it was.

**The microcontroller's serial link resets the board.** Opening the port asserts
lines that reset the microcontroller and drop motor state mid-motion, which is
why its host-side bridge service is explicitly protected from the agent, and why
its retry loop is written to survive reset noise on connect.

**Two USB serial devices, one numbering space.** The scanner and (historically)
the motion board both enumerate as serial bridges, so device numbering can swap
across reboots. Services identify devices by stable identifiers rather than by
number — and the native-USB motion board now enumerates in a different class
entirely, which removes the ambiguity.

**Thermal state is monitored.** Throttling degrades every layer of the stack —
detection frame rate, speech latency, control-loop timing — without raising a
single error anywhere, so the host publishes its own temperature, load, memory
and disk as first-class robot telemetry.

---

## 8.3 What the hardware cannot do

Stated here, and repeated where it matters, because several of these read as
software faults if you do not know them:

- **Top speed is roughly 0.55 m/s**, against a walking pace of 1.2–1.4 m/s. The
  robot cannot keep up with a person actually walking.
- **The camera mount cannot look behind** — pan travel is roughly ±50°, so looking
  backwards means turning the body.
- **The mount tilts far up and only slightly down**, so "look down" usually
  reports hitting its stop, and somebody standing very close leaves the bottom of
  the frame.
- **Servos have no position feedback**, so the robot models where the camera was
  *told* to point; the modelled joint state lags the command during travel.
- **A person much beyond ~8 m cannot be locked onto.**
- **There is about 0.6 s of dead time** in the camera tracking loop, which is why
  it trails a fast mover and can still limit-cycle on a still subject.
- **No sensor sees drops, steps or stairs.**

The full catalogue, including software limits, is in
[Safety, reliability and limits](09-safety-and-limits.md).

---

| ← Previous | Next → |
|---|---|
| [7. Motion firmware](07-firmware.md) | [9. Safety, reliability and limits](09-safety-and-limits.md) |
