# Master prompt: the Mesquite Bridge, its Max package and its sensor test bench

*For Danyal's collaborator. Make a new empty folder (or clone the new repository), open Claude Code there, and paste
everything below the line as the first message. Claude first saves it as `docs/brief/BRIDGE_BRIEF.md`.*

---

You are working with a collaborator of **Danyal Khorami**, who leads **Mesquite**, a low-cost inertial motion-capture
suit. You build a new, separate project. Everything you need about Mesquite is in this brief. You do not need the
Mesquite source code, and you must not copy any of it into this project. The public repository
`github.com/Danyal-0-1/mesquite-mocap` may be *read* to cross-check a detail; never copy from it.

Where this brief is silent, make a choice, write it down in `docs/DECISIONS.md` and test it. Where only a person can act
(plugging in hardware, opening Max, wearing the suit), stop and give the collaborator exact steps.

## The task in one paragraph

Build **Mesquite Bridge**, a small local service that owns the Mesquite hub's USB port. It turns the hub's byte
stream into clean, timestamped orientation data for any application. It also:
- logs every frame with its arrival time;
- replays logs and simulates a suit, so everything can be developed and tested without hardware;
- applies adjustable filters that work correctly on rotations;
- publishes the data over **OSC** and **WebSocket**.

On top of it, build two clients:
1. **a Max package**, so that anyone who builds a 3D world in Max/Jitter can drive its objects with the pods, with
   per-object sensitivity and filtering;
2. **a sensor test bench**, which measures the suit's drift, restarts, freshness and loss, and writes reports.

## This session: the foundation

This brief describes the whole project, but **this first session builds only the foundation**:
1. the repository layout of §1.2, with every folder and module of the pipeline in place;
2. stages S0–S4 of the pipeline (§1.1), complete and tested: conventions, sources, recorder, parser and pod state;
3. a **walking skeleton**: data flowing from end to end, from the simulator, through a pass-through S5–S7 (map to the
   app frame, no filters), to OSC and WebSocket. A test client receives it;
4. the serial source, if a hub is on hand;
5. `docs/PIPELINE.md`, `docs/PROTOCOL.md`, `docs/DECISIONS.md`, `docs/WORKLOG.md` and `docs/NEXT_SESSION.md`.

That is milestones M0–M2 (§5). Stop there.
- Later stages get their folder, their typed interface (§1.4) and a `TODO` pointing to the section of this brief that
  defines them, but no implementation yet.
- Later sessions continue from `docs/NEXT_SESSION.md`.

## 0. Rules

1. **First commit.** Save this brief unchanged as `docs/brief/BRIDGE_BRIEF.md`. Write `CLAUDE.md` with these rules,
   so later sessions follow them. Start `docs/WORKLOG.md`, with one entry per milestone: what you did, what you tested,
   what is still open.
2. **Ask once, at the start**, then work. Ask:
   - the operating system, and the Max version (8.6+ or 9);
   - whether a Mesquite hub and pods are on hand. If not, you work from the simulator and from recordings Danyal sends;
   - the repository's name, owner and visibility, and its licence. Suggest GPL-3.0, which matches Mesquite;
   - whether Node.js 20 LTS or newer is installed.
3. **Never touch the firmware.** You never flash a device.
   - You may send exactly two commands to the hub, both defined in §2.3: *ping*, and *reboot all pods*.
   - **Send reboot only on an explicit human action**, with a confirmation step. Never send it automatically, on
     connect, or in a test against real hardware. Rebooting erases every pod's learned sensor calibration and its
     heading.
4. **One owner of the port.** Only one program can open a serial port. The bridge owns it, and every consumer reads
   from the bridge.
5. **Every number comes from this brief, a measurement or a test.** Label everything else `assumption` in the docs.
6. **Work in milestones** (§5). Each one ends with a branch, a pull request, passing tests and a WORKLOG entry.
7. **Determinism.** Replaying a log must reproduce the published output byte for byte, given the same configuration.
8. **No secrets.** The bridge uses only USB, so it needs no network credentials. Never log any.

## 1. The picture

Close your eyes. Fifteen watches are strapped to a person, one per body segment.
- Inside each watch, a sensor chip fuses its gyroscope and accelerometer into an orientation, about 55 times a second.
- The watch radios that orientation to a USB stick, the **hub**, about 32 times a second.
- The hub pours every pod's packets into one byte stream on the computer's USB port.

Your bridge sits on that port. It cuts the stream into frames, notes when each arrived, writes it to disk, cleans it,
and hands it to whoever asks: Max, a browser, Python, TouchDesigner. When there is no suit, a recording or a simulator
pours the same bytes in.

```text
 pods ──radio──► hub ──USB bytes──►┌──────────────── Mesquite Bridge ────────────────┐
                                   │ source: serial | replay | simulator             │
                                   │   └─► parser ─► per-pod state (fresh/held/lost) │
                                   │         ├─► raw log (every byte + arrival time) │
                                   │         ├─► orientation tools (map, zero, align)│
                                   │         ├─► filters (on rotations, per channel) │
                                   │         └─► test bench (drift, restarts, …)     │
                                   └──► OSC (UDP) ──► Max package ──► Jitter objects
                                    └──► WebSocket ──► browser, Python, anything
```

### 1.1 The pipeline, stage by stage

Every frame passes through the same stages in the same order. Each stage:
- is one module;
- takes one typed input and gives one typed output;
- is tested on its own before the next stage is built on it.

| Stage | What it does | Module | In | Out | Session 1 |
|---|---|---|---|---|---|
| **S0** core | monotonic clock, quaternion math, config, shared types | `src/core/` | — | used by every stage | complete |
| **S1** source | delivers bytes, each chunk stamped with its arrival time | `src/sources/` (`serial`, `replay`, `simulator`) | the port, a log, or a seed | `Chunk` | complete; serial only if a hub is on hand |
| **S2** recorder | taps every chunk to disk before anything can change it | `src/log/` | `Chunk` | `raw.bin`, `index.csv`, `meta.json` | complete |
| **S3** parser | cuts the stream into records by the rules of §2.2 | `src/protocol/` | `Chunk` | `PodFrame`, `StatusLine`, `PhoneFrame`, drop counters | complete |
| **S4** pod state | freshness, held frames, loss, restarts, health; writes `frames.csv` | `src/state/` | records | `PodSample`, `Health` | complete |
| **S5** orientation | the app-frame map, zero, heading registration | `src/orient/` | `PodSample` | `Oriented` | map only (pass-through) |
| **S6** channels | filters and mapping per object (§3.5–3.7) | `src/channels/` | `Oriented` | `ChannelOut` | interface + `none` filter |
| **S7** outputs and control | OSC, WebSocket, control commands, hub commands | `src/outputs/`, `src/control/` | `ChannelOut`, `Health`, records | network messages | pod `quat`/`raw`/`state` over OSC and WS; ping only |
| **S8** consumers | the Max package, the test bench, the dashboard | `max/`, `src/bench/`, `web/` | OSC, WS, sessions | patches, reports, a page | folders and README stubs only |

`src/pipeline.ts` wires the stages together and nothing else. A stage never reaches past its neighbours.

### 1.2 Repository layout

```text
mesquite-bridge/
├── CLAUDE.md                  rules of §0
├── README.md                  picture, install, first run with the simulator
├── package.json  tsconfig.json  .github/workflows/ci.yml
├── config/default.yaml        every default value of this brief, in one place
├── src/
│   ├── cli.ts                 serve | list-ports | record | replay | simulate | bench | report
│   ├── pipeline.ts            wires S1 → S7
│   ├── core/                  S0: time.ts quat.ts types.ts config.ts
│   ├── sources/               S1: source.ts serial.ts replay.ts simulator.ts
│   ├── log/                   S2: recorder.ts session.ts frames_csv.ts
│   ├── protocol/              S3: constants.ts parser.ts encode.ts
│   ├── state/                 S4: pod_state.ts health.ts restart.ts
│   ├── orient/                S5: frames.ts zero.ts register.ts
│   ├── channels/              S6: channel.ts mapping.ts filters/{none,slerp,oneeuro,kalman}.ts svd_tools.ts
│   ├── outputs/               S7: osc.ts ws.ts
│   ├── control/               S7: commands.ts hub_commands.ts (ping, guarded reboot)
│   └── bench/                 S8: recipes and reports (a later session)
├── test/                      mirrors src/; fixtures/golden.hex; simulated sessions
├── max/Mesquite/              S8: the Max package (a later session)
├── web/                       S8: the dashboard (a later session)
└── docs/
    ├── brief/BRIDGE_BRIEF.md  this brief, unchanged
    ├── PIPELINE.md  PROTOCOL.md  DECISIONS.md  WORKLOG.md  NEXT_SESSION.md
    └── (later) FILTERS.md  MATH.md  MAX.md  MAX_CHECKLIST.md  TEST_BENCH.md  REFERENCES.md  REPORT.md
```

### 1.3 Build from the ground up

Build in this order. A stage starts only when the stages below it pass their tests.

1. **S0 and the golden vectors.** The quaternion conventions of §4, the clock, the config loader, and the encoder of
   §2.2, which turns the golden vectors into tests.
2. **S3 the parser,** tested on hand-made byte strings, before any source exists.
3. **S1 the simulator**, which uses the encoder from step 1. Then **replay**.
4. **S2 the recorder.** The proof: simulate → record → replay → parse gives identical records.
5. **S4 pod state**, tested on simulated streams with planted held frames, loss and restarts.
6. **S5–S7 pass-through:**
   - map to the app frame;
   - the `none` channel;
   - OSC and WebSocket out;
   - a Node test client that subscribes and checks every message.

   This is the walking skeleton.
7. **S1 serial**, if a hub is on hand. A 10-minute live session is recorded, replayed and parsed identically.

Write `docs/PIPELINE.md` as you go. Open it with the picture of §1. Then give one section per stage: its job, its
types, its tests, and its state at the end of this session (complete, pass-through, or interface only).

### 1.4 Interfaces to fix now

These shapes let later sessions plug into this one without rewriting it. Refine the names if you must, record the
change in `docs/DECISIONS.md`, and keep the meaning.

```ts
type Quat = { x: number; y: number; z: number; w: number };           // Hamilton, unit, §4
interface Chunk { bytes: Uint8Array; tArrivalNs: bigint; tWallMs: number }
interface Source {                                                    // S1
  start(onChunk: (c: Chunk) => void): Promise<void>;
  stop(): Promise<void>;
  write?(bytes: Uint8Array): Promise<void>;                           // serial only; used by S7 hub commands
}
type Rec =                                                            // S3
  | { kind: 'pod'; id: number; name: string; batt: number; q: Quat; count: number; ms: number;
      legacy?: boolean; tArrivalNs: bigint }                          // legacy = a JSON pod line (§2.9)
  | { kind: 'status'; text: string; tArrivalNs: bigint }
  | { kind: 'phone'; q: Quat; pos?: [number, number, number]; tArrivalNs: bigint };
interface PodSample {                                                 // S4
  rec: Extract<Rec, { kind: 'pod' }>;
  held: boolean; gap: number; restart: boolean;
  status: 'fresh' | 'held' | 'stale' | 'lost';
}
interface Oriented { name: string; qApp: Quat; delta?: Quat; tArrivalNs: bigint }   // S5
interface Filter { reset(): void; step(q: Quat, dtS: number): Quat }                // S6
interface ChannelOut { channel: string; q: Quat; value?: number; tArrivalNs: bigint } // S6 → S7
```

## 2. The Mesquite facts you need

### 2.1 Pods and their ids

| id | name | id | name | id | name |
|---:|---|---:|---|---:|---|
| 0 | Head | 6 | RightArm | 12 | RightUpLeg |
| 1 | Spine | 7 | RightForeArm | 13 | RightLeg |
| 2 | HipsAlt (the pelvis pod) | 8 | RightHand | 14 | RightFoot |
| 3 | LeftArm | 9 | LeftUpLeg | 15 | LeftShoulder |
| 4 | LeftForeArm | 10 | LeftLeg | 16 | RightShoulder |
| 5 | LeftHand | 11 | LeftFoot | | |

The ids are 0–16; a full suit usually uses 0–14. There is also an optional phone, worn on the hips, which sends its own
pose under the name `Hips`.

### 2.2 The byte stream from the hub to the computer

Three kinds of record are interleaved on one stream. Two of them begin with the sync bytes `0xAA 0x55`.

**Pod pose, 16 bytes, little-endian:**

| Offset | Size | Field | Meaning |
|---:|---:|---|---|
| 0 | 1 | sync0 | 0xAA |
| 1 | 1 | sync1 | 0x55 |
| 2 | 1 | id | 0..16 (table 2.1) |
| 3 | 1 | batt | battery, 0..100 percent |
| 4 | 2 | qx | int16 = trunc(x × 32767) |
| 6 | 2 | qy | int16 |
| 8 | 2 | qz | int16 |
| 10 | 2 | qw | int16 |
| 12 | 2 | count | uint16, incremented once per *transmitted* packet, wraps at 65535 |
| 14 | 2 | ms | uint16, low 16 bits of the pod's own millisecond clock when the *sensor sample* was decoded |

How to read the fields:
- **The encoding truncates toward zero.** It does not round: −0.5 encodes as −16383, not −16384. Decode by dividing by
  32767.
- **Two consecutive packets from one pod with the same `ms` carry the same sensor sample.** That is a *held* frame: the
  pod had nothing new to send.
- **A gap in `count` means packets lost in the radio.**
- **`ms` wraps every 65.536 s and shares no epoch with any other pod.** The only clock shared across pods is the
  computer's arrival time.

**Hub status:** `0xAA 0x55 0xFE len` followed by `len` ASCII bytes. Some hub builds send two lines a second, for
example `I1 rx=…` and `I2 rssi=…`; others send none. Log them verbatim and show them. Parse them only if the format is
obvious.

**Phone pose:** one ASCII JSON line ending in `\n`, about 100 bytes, for example:

```json
{"bone":"Hips","x":0.01,"y":0.70,"z":0.02,"w":0.71,"px":0.12,"py":1.02,"pz":-0.30}
```

`px`, `py` and `pz` are in metres and may be missing when the phone tracks orientation only. Valid JSON here is 7-bit
ASCII, so the bytes 0xAA 0x55 can never occur inside a well-formed line.

**Parser rules.** These match the existing browser page; implement exactly these:
1. **The sync word.**
   - `0xAA` not followed by `0x55`: count a *false sync* and skip one byte.
   - `0xAA 0x55 0xFE`: a status record. Wait until all `4 + len` bytes are present.
   - `0xAA 0x55` followed by an id of 17 or more (other than 0xFE): count a false sync and skip **one** byte. Do not
     consume 16.
   - Otherwise: a 16-byte pose. Wait until all 16 bytes are present.
2. **`{` starts a JSON line**, which ends at `\n`.
   - If `0xAA 0x55` appears before the newline, abandon the line and resume parsing at that sync word.
   - Abandon a line longer than 512 bytes, or one left unterminated for more than 2 s. Then step past the `{` and
     rescan.
   - Try `JSON.parse`. Count failures. Drop lines whose x, y, z or w is not finite.
3. **Any other byte** is junk: count it and skip it.
4. **The buffer never holds more than 4096 bytes.** Shed the oldest bytes, and count them.
5. **The bytes 0xFF and 0xFC in the id position** are reserved for control traffic and never appear in this direction.

**Golden vectors.** Pin these in tests.

| Bytes (hex) | Decodes to |
|---|---|
| `AA 55 05 57 00 00 00 00 00 00 FF 7F 02 01 34 12` | LeftHand, battery 0.87, q = (0, 0, 0, 1), count 258, ms 4660 |
| `AA 55 0A 64 01 C0 FF 3F 01 C0 FF 3F FF FF 00 00` | LeftLeg, battery 1.00, q = (−16383, 16383, −16383, 16383) / 32767, count 65535, ms 0; this is the encoding of (−0.5, 0.5, −0.5, 0.5) |
| `AA 55 FE 05 68 65 6C 6C 6F` | status "hello" |
| `AA 55 FC 01 00 01` | (computer → hub) reboot all pods |

### 2.3 Commands from the computer to the hub

The frame is `0xAA 0x55 0xFC cmd len payload… xor`, where:
- `xor` is the XOR of `cmd`, `len` and every payload byte;
- `len` is 0–16.

The hub acts only on a complete frame with a correct checksum.

| cmd | Meaning | Rule |
|---|---|---|
| 0x01 | reboot all pods | **human action only**, with a confirmation (§0.3) |
| 0x02 | ping | safe |

### 2.4 The serial port

- **The hub is an ESP32-S3 on native USB.** The baud setting is ignored, but open the port at 115200 by default and
  make it configurable.
- **Port names:**

  | OS | Typical name |
  |---|---|
  | macOS | `/dev/cu.usbmodem*` |
  | Linux | `/dev/ttyACM*` |
  | Windows | `COM*` |

- **List the ports, and remember the chosen one.**
- **Reconnect automatically after an unplug.** A reconnect must never send any command.

### 2.5 Timing

- Each pod transmits on its own grid of about 32 Hz. Its sensor produces about 55 samples per second, so some packets
  are held frames.
- Pods are not synchronised with each other.
- So stamp every record with the computer's monotonic clock at arrival, plus wall-clock time.
- Keep the pod's `ms` too. It shows sample freshness, not when the sample was taken.

### 2.6 What a pod quaternion means

- **It is the sensor chip's 6-axis "game rotation vector".** The orientation is fused from the gyroscope and the
  accelerometer, with no magnetometer.
- **Its world frame is right-handed with +Z up**, opposite gravity.
- **X and Y are horizontal, but their direction is arbitrary.** Each pod fixes its own heading when it boots, so two
  pods' headings agree only if they booted lying in the same direction.
- **The quaternion is taken to rotate sensor coordinates into world coordinates.** That is an `assumption` (it is
  documented for Android sensors, not for this chip), so provide a per-pod `conjugate` switch.
- **The pod rebuilds w ≥ 0.** q and −q are the same rotation, so every interpolation must take the shortest arc:
  negate one side when the dot product is negative.
- **The resolution is 1/32767 per component.**

### 2.7 From the pods' frame to Y-up applications

Max/Jitter and three.js are right-handed with **+Y up**. The existing browser page converts each pod quaternion with
the same rule for every pod:

```text
(x, y, z, w)_pod  →  (y, z, x, w)_app        a proper rotation that takes pod +Z (up) to app +Y (up)
```

Make it the default for all pods, with a per-pod override. Any override must have determinant +1; reject mirrors.

The phone's pose is already in a +Y-up, right-handed frame (ARKit). The page additionally turns it half a turn about +Y,
mapping (x, y, z) to (−x, y, −z). Offer that turn as an option.

### 2.8 How the real suit behaves (use it in the simulator and the test bench)

| Behaviour | Value | Status |
|---|---|---|
| a pod that restarts (power, USB plug, reboot command) | its orientation jumps to identity, then its tilt is pulled back to gravity with a time constant of about 4.7 s, within 2° after about 16 s; its heading is re-zeroed to wherever it faced | measured |
| a still pod, after its sensor has calibrated | heading drift below about 0.1°/min for most pods | measured, 17 min, 12 of 13 pods |
| a still pod soon after power-on | drift up to about 2°/min, slowing over minutes, as the sensor learns its gyro bias | measured on one pod |
| a worn suit | heading drift of tens of degrees over a few minutes | measured |
| gyro scale tolerance | up to about 1.5 % of every turn (5.4° per full turn) | datasheet |

### 2.9 The legacy JSON pods (lower priority)

An older firmware sends JSON lines through a different hub, each with a `bone` name and x, y, z, w rounded to two
decimals. Accept a JSON line with a known `bone` and finite x, y, z, w as a pose, but mark it `legacy` (resolution about
1°). Build the binary path first.

## 3. What to build

### 3.1 The bridge core

- **Language:** Node.js 20+ with TypeScript.
  - Libraries: `serialport` for the port, `ws` for WebSocket, and an OSC library such as `osc`.
  - Tests: `node:test` or `vitest`.
  - A CLI, `mesquite-bridge`, with sub-commands `serve`, `list-ports`, `record`, `replay`, `simulate`, `bench` and
    `report`.
- **Sources**, behind one interface:
  - `serial`, the live port;
  - `replay`, a log at 1×, faster or stepped, with the same output as live;
  - `simulate`, synthetic pods (§3.9).
- **Per-pod state:**
  - last frame and arrival time;
  - packets per second and unique samples per second (distinct `ms` values);
  - held fraction and loss from `count` gaps;
  - battery;
  - **restart detection:** `count` or `ms` jumping backwards outside its wrap window, and an orientation snap to near
    identity;
  - **status:** `fresh` (a new sample in the last 100 ms), `held`, `stale` (no packet for 250 ms) or `lost` (2 s).
- **A configuration file** in JSON or YAML, plus overrides on the command line. Write the version and the
  configuration's hash into every log and report.

### 3.2 The raw log, and replay

- **Write one session folder per run:**
  - `meta.json`: bridge version, configuration, port, start time;
  - `raw.bin`: the exact bytes as received;
  - `index.csv`: the byte offset and arrival time of each chunk received;
  - `frames.csv`: one decoded row per record, with arrival time and all fields.
- **Append only.** A crash may lose at most the last second.
- **Replay** reads `raw.bin` and `index.csv` and feeds the bytes back with their original timing. That gives the
  byte-for-byte determinism of rule 0.7.

### 3.3 Outputs

**OSC over UDP.** The default is port 9000 to localhost, configurable, with any number of targets.

| Address | Arguments |
|---|---|
| `/mesquite/pod/<name>/quat` | x y z w (floats, the app frame of §2.7, after the pod's channel processing) |
| `/mesquite/pod/<name>/raw` | x y z w (as decoded, pod frame) |
| `/mesquite/pod/<name>/state` | status string, battery 0–1, packets/s, held fraction |
| `/mesquite/pod/<name>/restart` | sent once, when a restart is detected |
| `/mesquite/phone/quat`, `/mesquite/phone/pos` | x y z w; px py pz |
| `/mesquite/channel/<channel>/quat` and `/…/value` | a processed channel (§3.6) |
| `/mesquite/hub/status` | the status line, verbatim |

**Control over OSC.** The bridge listens on 9001 by default.
- Examples: `/mesquite/channel/create name pod`, `/mesquite/channel/<name>/filter oneeuro mincutoff beta`,
  `/mesquite/pod/<name>/zero`, `/mesquite/register box`.
- Write the complete list in `docs/PROTOCOL.md`.

**WebSocket.**
- `ws://localhost:8765/frames`: JSON messages with the same content as OSC, plus arrival time.
- `ws://localhost:8765/raw`: the raw byte stream, unmodified, so the browser page can later read from the bridge.
- A control channel with the same commands as OSC control.

**Output timing.**
- `on-arrival` (the default): one message per received frame.
- `fixed N Hz`: interpolate between samples with a configurable delay, or predict ahead with the Kalman filter's
  angular velocity.

### 3.4 Orientation tools for each pod

1. **Map** to the app frame (§2.7).
2. **Zero.**
   - Capture the pod's orientation now as its reference q_ref.
   - Output the world delta D = q ⊗ q_ref⁻¹, which is the rotation since the zero, expressed in the world frame.
   - Applied to an object's rest orientation, the object turns exactly as the pod turns.
3. **Heading registration.**
   - At a "box" moment, when all pods lie with the same heading, compute each pod's turn about up relative to a
     reference pod: K_i = twist_up(q_ref,box ⊗ q_i,box⁻¹).
   - Then use K_i ⊗ q_i, so all pods share one heading.
4. **Absolute or delta output**, per pod.

### 3.5 Filters

All filters work on rotations, never on the four numbers separately. All are time-based, so their behaviour does not
depend on the packet rate. Each can be switched on and off live and tuned per channel.

| Filter | Idea | Parameters | Defaults |
|---|---|---|---|
| `none` | pass-through | — | — |
| `slerp` | exponential smoothing: q̂ ← slerp(q̂, q, α), with α = 1 − exp(−Δt/τ) | τ (s) | 0.05 |
| `oneeuro` | the One-Euro filter on rotations: angular speed from the log map of q ⊗ q̂⁻¹ divided by Δt, smoothed; cutoff f_c = f_min + β·speed; α = 1 / (1 + 1/(2π f_c Δt)) | f_min (Hz), β, d_cutoff (Hz) | 1.0, 0.02, 1.0 |
| `kalman` | constant-angular-velocity Kalman filter on the rotation-vector error (multiplicative, error-state); measurement = the quaternion; outputs q̂ and ω̂; optional prediction ahead by a set time | process noise (rad/s²)², measurement noise (rad²), predict (s) | tune in §6 |
| `deadband` | ignore changes smaller than an angle | angle (°) | 0.5 |
| `hold-guard` | ignore held frames when estimating speed | — | on |

Report each filter's added latency, measured as the lag of a step response, in the dashboard and in `docs/FILTERS.md`.

### 3.6 Channels: per-object sensitivity

A **channel** is the processing chain for one object:

```text
pod → (map → zero / registration) → filter → mapping
```

Many channels may read the same pod with different settings. The bridge computes every channel, so the math lives in
one tested place. Max only routes the results.

| Mapping | What it does |
|---|---|
| `gain g` | scales the rotation angle: D^g = exp(g · log D). g = 0.5 halves every motion; g = 2 doubles it |
| `axis-lock axis` | keeps only the twist about one axis (swing–twist decomposition) |
| `swing-only` | removes the twist about an axis |
| `invert` | uses D⁻¹ |
| `clamp` | limits the angle to a maximum |
| `offset` | a fixed rotation applied after everything else |
| `value` | turns the channel into numbers for sound or controls: twist angle about an axis, tilt angle, angular speed, or "energy", each scaled to 0–1 or 0–127 |

### 3.7 Tools that use the SVD

| Tool | What it does | Use |
|---|---|---|
| quaternion average | the principal eigenvector of Σ qᵢqᵢᵀ over a still window (Markley et al. 2007) | a clean zero or box reference from noisy samples |
| pod-to-pod alignment | Wahba/Kabsch: the SVD of the cross-covariance of paired directions, with the determinant fix, gives the constant rotation between two pods on one rigid board | checking alignment in the box; estimating a mounting rotation |
| principal motion axis | the SVD of a window of rotation vectors (log of D); the first singular vector is the dominant rotation axis, and its singular value is how one-dimensional the motion is | mapping a hinge-like motion, such as elbow bending, to one smooth control value |

### 3.8 The test bench

This is for Danyal's sensor work. It measures what the sensor chip's fusion does, using only the stream. Each recipe
does three things:
- it prints instructions for the operator;
- it records a session (§3.2);
- it writes a report in Markdown and CSV, with plots as inline SVG or PNG.

All heading numbers use the twist about up: ψ = 2·atan2(q_z, q_w) in the pod frame (Z up), unwrapped over time.
Tilt is the angle of the remaining swing.

| Recipe | Setup | Metrics | Proposed pass |
|---|---|---|---|
| `static` | pods still for N minutes | per pod: heading slope (°/min, robust fit with 95 % interval), maximum excursion, tilt change, noise | < 0.1°/min after the sensor has calibrated |
| `box` | all pods booted lying with the same heading | each pod's heading relative to the reference pod, against time; the spread at 45 s and at 3 min | spread < 2° at 45 s |
| `restart` | a pod is restarted while still | detection; tilt settling time constant (fit); time to within 2°; heading jump | detection 100 %; τ reported |
| `turns` | a pod on a flat board is turned N full turns and returned to a stop | error per turn = Δψ / N, for each direction | < 1.8° per turn (0.5 %) |
| `health` | any session | per pod: packets/s, unique samples/s, held %, loss %, arrival jitter, battery; hub status lines | reported |

**Optional second input:** a single pod's own USB serial port, during bench work.
- Log its text lines on the same clock.
- Parse two line formats into columns: `[INSTR] key=value …`, and the proposed
  `[DMPCAL] t_ms id score_g score_a bias_x bias_y bias_z temp_C q_w q_x q_y q_z incomplete`.
- That second format may not exist yet. Make the parser tolerant, and document both formats.

### 3.9 The simulator

It produces the exact byte stream of §2.2, using the same encoder the tests pin.

- **Configurable, with a fixed seed:**
  - the number of pods, with ids and names;
  - motion per pod: still, a swing about an axis, continuous turning, or replayed rotations from a CSV;
  - the sensor rate (55 Hz) and the transmit rate (32 Hz), giving held frames naturally;
  - heading drift (°/min), noise (degrees RMS) and packet loss (%);
  - restarts at given times, with the behaviour of §2.8;
  - battery decay;
  - hub status lines;
  - a phone line at 60 Hz.
- **It doubles as the test fixture.** The test bench must recover the drift rates, losses and restart settling times the
  simulator was given (§6).

### 3.10 The Max package

- **Name** `Mesquite`. It targets Max 8.6+ and Max 9. Installation: copy the folder into the Max `Packages` directory.
- **Transport.** Max receives OSC with `[udpreceive 9000]` and sends control with `[udpsend 127.0.0.1 9001]`. That
  needs no npm inside Max. A `node.script` client for the WebSocket is optional.
- **Abstractions**, each with a help patcher:

  | Abstraction | What it does |
  |---|---|
  | `[mesquite.connect]` | ports, connection status, a ping button, and a guarded "reboot pods" button with a confirmation |
  | `[mesquite.pod <name>]` | outlets: `quat x y z w`, status, battery, a restart bang |
  | `[mesquite.object <name> <pod>]` | creates a channel in the bridge and outputs `quat x y z w`, ready for a Jitter object's or `jit.anim.node`'s `quat` attribute; inlets set filter, gain, axis-lock, zero and invert |
  | `[mesquite.value <name> <pod> <kind>]` | one number from a channel, for sound or controls |
  | `[mesquite.calibrate]` | buttons for zero-all and box registration |

- **Examples:**
  1. one pod drives one cube in a `jit.world`;
  2. fifteen pods drive fifteen shapes laid out like a body;
  3. a filter comparison: one pod, three channels side by side (none, slerp, one-euro);
  4. an elbow-bend `value` channel driving a sound parameter.
- **Jitter is right-handed, with +Y up**, so the app frame of §2.7 applies directly. Check every attribute name against
  the current Cycling '74 reference, and record what you checked in `docs/MAX.md`.
- **You cannot run Max.**
  - Keep the patchers small. Generate them from a script, or write them carefully.
  - Validate the JSON: check that every object class is one you verified in the reference, and that every connection
    index is valid.
  - Write `docs/MAX_CHECKLIST.md`: exactly what the collaborator opens and clicks, and what they should see, step by
    step.
  - Fix what they report.

### 3.11 A small local dashboard (last)

A web page served by the bridge on localhost. It shows:
- every pod as a small 3D axes gizmo, with its status, rate and battery;
- every channel's filter, with live controls;
- the test bench: start a recipe and read its report.

Use a pinned three.js from a local copy, not a CDN, so it works offline.

## 4. The mathematics

One module owns the conventions, and tests pin them:

- **Quaternions** are Hamilton, stored and sent as (x, y, z, w), and normalised on input.
- **Composition:** a ⊗ b applies b first, then a. The inverse of a unit quaternion is its conjugate.
- **Shortest arc:** if q₁·q₂ < 0, negate q₂ before you slerp or average.
- **Log and exp maps:** log(q) = θ·u is a rotation vector; exp is its inverse. Use series expansions near θ = 0.
- **Twist about a unit axis u:** normalise (w, (v·u)u). Its angle is 2·atan2(v·u, w). The swing is q ⊗ twist⁻¹.
- **The basis change of §2.7** is a conjugation by a proper rotation. For a signed permutation it reduces to moving the
  components of v.
- **The time constant:** α = 1 − exp(−Δt/τ). The One-Euro filter is from Casiez et al. (2012).
- **The Kalman filter** is error-state, on the rotation vector (Markley 2003; Solà 2017). The state is the attitude
  error and the angular velocity.
- **Averaging** is Markley et al. (2007). **Wahba by SVD** is Markley (1988), with the reflection fixed through det(U)·det(V).

## 5. Milestones

| Milestone | Session | Content | Gate |
|---|---|---|---|
| M0 | **1** | the layout of §1.2, CI (lint, type-check, tests), S0, the encoder and golden vectors, `PIPELINE.md` begun | CI green; the golden vectors pass |
| M1 | **1** | S3 parser, S1 simulator and replay, S2 recorder, S4 pod state; no hardware | every parser and state case of §6 passes; simulate → record → replay is byte-identical |
| M2 | **1** | the walking skeleton (S5–S7 pass-through), the serial source, health | a test client receives every pod's `quat`, `raw` and `state` from the simulator; with hardware, a 10-minute live session is recorded and replayed identically |
| M3 | 2 | S5 zero and registration, S6 channels and mapping, S7 control messages | the control round-trip works; the mapping tests of §6 |
| M4 | 2 | filters, SVD tools, latency measurement, fixed-rate output | the filter and SVD tests of §6 |
| M5 | 3 | the Max package, examples, help, checklist | the collaborator completes `MAX_CHECKLIST.md` in Max |
| M6 | 3 | the test bench and its reports | it recovers the simulator's planted values; one real report, if hardware exists |
| M7 | 4 | the dashboard, the remaining documentation, the final report | §7 and §8 complete |

**Session 1 ends after M2.**
- Push, and write `docs/NEXT_SESSION.md`. It holds the exact state of every stage, what is next (M3), and the first
  command to run.
- Write the report of §8 for what exists so far.

## 6. Checks that must pass

- **Parser:**
  - the golden vectors;
  - an encode→decode round trip for 10,000 random unit quaternions, with an error of at most 1/32767 per component;
  - truncation toward zero (−0.5 → −16383);
  - a false sync skips exactly one byte;
  - a bad id does not consume 16 bytes;
  - an interleaved JSON line is abandoned at the sync;
  - a 512-byte line overflow;
  - the 2 s stall;
  - the 4096-byte buffer bound;
  - records split across chunks at every possible byte boundary decode identically.
- **State:**
  - held frames, loss and restarts are detected on simulated streams with known truth;
  - wraps of `count` and `ms` are not taken for restarts.
- **Mapping:** every map has determinant +1, and pod +Z maps to app +Y.
- **Filters:**
  - identity in gives identity out;
  - a constant rotation gives itself;
  - no output jumps when the input's sign flips (q → −q);
  - the same result at 30, 32 and 60 Hz input for the same τ, within tolerance;
  - the step-response time constants match their parameters;
  - the Kalman filter's ω̂ converges on a constant spin.
- **SVD tools:**
  - Wahba recovers a known rotation from noiseless pairs within 1e−9 rad, and within the expected error from noisy ones;
  - the average of a symmetric cloud of samples is its centre;
  - the principal axis of a pure hinge motion is the hinge.
- **Test bench:** on simulated sessions it recovers the planted drift (±0.02°/min), loss (±1 %), restart τ (±10 %) and
  error per turn (±0.1°).
- **Performance:** the median added processing per frame is under 1 ms, measured, with 17 pods at 32 Hz plus the
  phone at 60 Hz.

## 7. Documentation

Write for a reader who knows some programming but is new to rotations and to Max. Open each document with a short
"close your eyes" picture of what happens, then the steps, then the details.

| File | Contents |
|---|---|
| `README.md` | what it is, install, first run with the simulator, then with the suit |
| `docs/PIPELINE.md` | the picture of §1; one section per stage (job, types, tests, state); how a frame travels from the USB port to Max |
| `docs/PROTOCOL.md` | §2 and §3.3, complete |
| `docs/FILTERS.md` | each filter: the idea, the math, its parameters, when to use it, its latency, a plot |
| `docs/MATH.md` | the conventions of §4, with small worked examples |
| `docs/MAX.md` and `docs/MAX_CHECKLIST.md` | the package, and the step-by-step test |
| `docs/TEST_BENCH.md` | each recipe: setup, steps, metrics, pass marks, how to read the report |
| `docs/DECISIONS.md` and `docs/WORKLOG.md` | |
| `docs/REFERENCES.md` and `references.bib` | with DOIs you have verified |

References to verify and cite:
- G. Casiez, N. Roussel, D. Vogel, "1 € Filter: A Simple Speed-based Low-pass Filter for Noisy Input in Interactive
  Systems," CHI 2012;
- F. L. Markley, "Attitude Error Representations for Kalman Filtering," J. Guidance, Control, and Dynamics, 2003;
- J. Solà, "Quaternion kinematics for the error-state Kalman filter," arXiv:1711.02508, 2017;
- F. L. Markley, Y. Cheng, J. L. Crassidis, Y. Oshman, "Averaging Quaternions," J. Guidance, Control, and Dynamics,
  2007;
- F. L. Markley, "Attitude Determination Using Vector Observations and the Singular Value Decomposition," J. Astronautical
  Sciences, 1988;
- G. Wahba, "A Least Squares Estimate of Satellite Attitude," SIAM Review, 1965;
- W. Kabsch, "A solution for the best rotation to relate two sets of vectors," Acta Crystallographica A, 1976;
- K. Shoemake, "Animating Rotation with Quaternion Curves," SIGGRAPH 1985;
- the Cycling '74 references for `udpreceive`, `udpsend`, `node.script`, `jit.world`, `jit.anim.node` and the Jitter
  OB3D `quat` attribute.

## 8. When you stop: the report

Write `docs/REPORT.md`. It covers:
- what works, with the evidence: tests, logs and screenshots the collaborator took;
- what is untested on real hardware;
- every `assumption` still open;
- the measured latency and performance;
- what Danyal should run first with the suit. Suggested order: `health` → `static` → `box` → `restart` → `turns`.
