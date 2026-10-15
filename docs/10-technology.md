# 10. Technology stack and glossary

[← Back to index](../README.md) · [← 9. Safety and limits](09-safety-and-limits.md)

---

## 10.1 Stack by layer

### Agent and language model

| Technology | Role |
|---|---|
| **LangGraph / LangChain agent runtime** | The ReAct-style think-act loop, the middleware stack around model and tool calls, and the graph-step budget |
| **OpenAI-compatible chat completions** | The model interface. Any endpoint speaking it will do — a model served on the local network, or a hosted provider — selected by three settings |
| **Tool calling** | The only hard model requirement; vision capability is additionally wanted for photograph understanding |
| **SQLite checkpointing** | Durable per-conversation threads that survive process restarts, rolled over by age and idleness |
| **SQLite activity log** | Every turn's model calls, tool calls, timings and token counts, recorded through LangChain callbacks |
| **Telegram Bot API (long polling)** | The chat surface, restricted to allow-listed conversations, with `/stop` and `/language` commands |
| **LangGraph Studio** | Optional live view of the real agent graph during development, loopback-only |
| **Home Assistant (WebSocket API)** | Optional home control through the hub's built-in intent matcher, limited to entities exposed to its voice assistant |

### Voice

| Technology | Role |
|---|---|
| **openWakeWord** | On-device wake-word detection |
| **Silero VAD** | Voice-activity detection and endpointing |
| **Whisper-large-v3-turbo, hosted (Groq)** | The deployed English recogniser: higher accuracy, no accelerator cost |
| **Whisper-base (on-device)** | Optional speech recognition on the neural accelerator through the runtime's native speech-to-text interface, reached over a loopback endpoint; usable as an automatic fallback |
| **Sarvam speech-to-text (code-mix mode)** | Malayalam recognition, keeping English words in Latin script |
| **Piper** | On-device streaming speech synthesis, English and Malayalam voices, chosen per language |
| **Direction-of-arrival estimation** | Camera aiming from the array's four raw capsules |
| **Web Audio / AudioWorklet** | Push-to-talk capture and reply playback in the phone's browser |
| **WebRTC (WHIP / WHEP) + Opus** | Live two-way audio calls through the camera server |
| **ALSA** | Capture, playback, cue tones and the array's mixer |

### Vision

| Technology | Role |
|---|---|
| **Hailo-10H NPU + HailoRT** | Detection, face recognition, object finding and optional on-device speech, time-shared in one process through the runtime's round-robin scheduler |
| **YOLO-family detector with tracking** | COCO-class detection with persistent per-object track IDs, always on |
| **SCRFD + ArcFace (MobileFaceNet)** | Face detection and face embeddings for recognition, on demand |
| **YOLO-World + CLIP** | Open-vocabulary object finding and colour ranking, on demand, with a full-precision CPU re-check |
| **ONNX Runtime** | The CPU re-check for object finding, and the runtime under the wake word and Piper |
| **GStreamer** | The vision pipeline: one decode split into detection, stills and faces |
| **Vision-capable language model** | The search's photograph judge, called separately from the conversation |
| **MediaMTX** | The single camera owner: WebRTC publishing plus a loopback mux for every other consumer, and the call audio paths |
| **WebRTC / WHEP** | Low-latency live video in the browser |
| **libcamera** | The platform camera stack underneath |

### Robotics

| Technology | Role |
|---|---|
| **ROS 2 Jazzy** | The robot platform's messaging, parameters, launch system and tooling |
| **Nav2** | Global planning, an MPPI controller behind a rotation shim, recovery behaviours, velocity smoothing, behaviour-tree navigation |
| **slam_toolbox** | Mapping on demand and localisation against saved maps |
| **Likelihood-field scan matching** | Scout's own localisation check and whole-map relocalisation, scoring the laser scan against the map |
| **robot_localization** | Extended Kalman filter fusing wheel odometry and IMU in planar mode |
| **TF2** | The transform tree, with documented single ownership per edge |
| **rosbridge / roslibjs** | The WebSocket interface used by both the browser and the agent |
| **micro-ROS (XRCE-DDS)** | The microcontroller's participation in the same interface space, over one USB cable |
| **Foxglove bridge** | Optional full-system observability |
| **Fast DDS** | The middleware, explicitly pinned to a UDP transport across container boundaries |

### Firmware

| Technology | Role |
|---|---|
| **ESP32-S3 + Arduino framework** | The motion controller |
| **PlatformIO** | Build, dependency and upload tooling |
| **micro-ROS client** | Publishers, subscribers, executor and the agent connection state machine |
| **Hardware PWM (LEDC / MCPWM)** | Motor drive and servo pulses on peripherals that cannot collide |
| **PID control** | Per-wheel velocity loops on top of a measured feedforward model |
| **BNO055 fusion driver** | Orientation, with persistable gyro-bias calibration |

### Platform and operations

| Technology | Role |
|---|---|
| **Raspberry Pi OS + Docker** | Host services and containerised ROS 2, one container per feature |
| **systemd** | Process supervision, dependencies, conflict rules and boot policy — one unit per feature |
| **Allowlisted privileged endpoint** | The only path from the web UI to service management, bounded by exact-string matching |
| **Single-page web app (roslibjs)** | Drive, Map, Activity and Status views over one bridge connection |
| **HTTP / HTTPS serving** | The web app, with TLS because phone microphone access needs a secure context, and an HTTPS server that forwards the camera's WebRTC signalling |
| **pytest** | The agent's test suite, including protocol-boundary fakes, virtual time, syntax-tree guards and opt-in live-model policy tests |

---

## 10.2 Design principles

Repeated patterns across all three codebases, collected in one place:

1. **Put the decision where the deadline is.** Anything with a control-loop
   deadline lives next to the hardware; anything requiring conversation lives in
   the agent.
2. **One owner per resource.** One camera opener, one publisher per velocity topic,
   one owner per transform edge, one process holding the accelerator, one place
   where two coordinate systems meet.
3. **A number that exists twice will disagree with itself.** Calibration constants
   live once and are shared, even across repository boundaries.
4. **Idle means hold, and silence means absent.** Arbiters hold position when idle;
   features publish heartbeats even when off, so "off" and "not running" stay
   distinguishable.
5. **Measure, then write the number down with its measurement.** Thresholds, gains,
   friction terms, latencies and accuracy figures in this project all carry the
   experiment that produced them.
6. **Enforce policy where it cannot be reinterpreted.** Prompt text is advice;
   middleware, robot-side checks and syntax-tree tests are enforcement.
7. **Name failures precisely.** Distinct causes get distinct words, because a
   collapsed distinction sends somebody after the wrong problem.
8. **Tell the model what actually happened**, including what is about to happen, so
   it narrates reality rather than its assumptions.
9. **Stop is sacred.** Never gated, never slow, works from every state.

---

## 10.3 Glossary

| Term | Meaning |
|---|---|
| **Agent** | Here, the language-model loop that plans and calls tools. (In micro-ROS, "agent" also means the host-side serial bridge — both senses appear in this project, and these docs say which.) |
| **Barge-in** | Interrupting the robot while it is speaking |
| **Checkpointer** | The store that persists a conversation thread so it survives a restart |
| **Code-mix** | Speech that mixes languages, here Malayalam with English words, transcribed with each word in its own script |
| **Coverage ring** | A search's record of which bearings around a viewpoint were actually seen |
| **Costmap** | The inflated obstacle map a planner and controller reason over |
| **Deadband** | A region of small commands treated as zero, to avoid buzzing or hunting |
| **Dead time** | Delay between an action and its observable effect — the thing that makes a loop hunt |
| **DoA** | Direction of arrival: which way a sound came from |
| **Endpointing** | Deciding when a spoken command has finished |
| **EKF** | Extended Kalman filter, here fusing wheel odometry with the IMU |
| **Face embedding** | A short vector describing a face, compared against the gallery to name it |
| **Hangover** | The trailing silence kept after speech so the last word transcribes correctly |
| **Likelihood field** | A map in which every cell holds how close it is to a wall, used to score how well a laser scan fits |
| **Open vocabulary** | Detection of any named object, not only a fixed class list |
| **Push-to-talk** | Recording while a button is held, so the press and release mark the start and end of speech |
| **Relocalisation** | Finding the robot on the map from scratch, rather than tracking from a known pose |
| **Middleware (agent)** | Code wrapping every model call and tool call, used here for vision, resource release and budgets |
| **Middleware (ROS)** | The transport underneath ROS 2 messaging |
| **MPPI** | A sampling-based predictive controller; it returns a cost-weighted average of sampled trajectories |
| **NPU** | Neural processing unit — the on-board AI accelerator |
| **Occupancy grid** | The map as a grid of free/occupied/unknown cells |
| **Pose** | Position plus orientation |
| **ReAct** | An agent pattern: reason, act with a tool, observe, repeat |
| **Stiction** | Static friction; the reason a tracked base can buzz without turning |
| **Superstep** | One step of the agent graph; several make up one think-act round |
| **TF / transform tree** | The robot's coordinate frames and the relationships between them |
| **Track ID** | A detector-assigned identity that persists across frames |
| **VAD** | Voice activity detection |
| **Vision judge** | A separate vision-model call that decides whether a search photograph shows what was asked for, and where in the photograph |
| **WER** | Word error rate, the standard accuracy measure for speech recognition |
| **WHEP / WHIP** | The signalling protocols used to pull and to push a WebRTC stream |

---

## 10.4 Repository map

| Repository | Layer | Documented in |
|---|---|---|
| [`scout_agent`](https://github.com/Xmindz/scout_agent) | Agent, tools, voice | [2](02-ai-agent.md), [3](03-voice.md) |
| [`scout_ws`](https://github.com/Xmindz/scout_ws) | ROS 2 platform, NPU vision services, navigation, web app | [4](04-perception.md), [5](05-navigation.md), [6](06-platform.md) |
| [`scout_loco`](https://github.com/Xmindz/scout_loco) | Motion firmware | [7](07-firmware.md) |

---

| ← Previous | Next → |
|---|---|
| [9. Safety, reliability and limits](09-safety-and-limits.md) | [Index](../README.md) |
