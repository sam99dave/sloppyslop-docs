# openpilot Architecture Diagrams

Mermaid architecture diagrams for the openpilot codebase. View them with any
Mermaid renderer (GitHub, VS Code Mermaid extension, mermaid.live, mkdocs with
mermaid, etc.) — each file is a standalone `.mmd` diagram.

## Diagram index

| # | File | What it shows |
|---|------|---------------|
| 1 | [01-system-overview.mmd](01-system-overview.mmd) | **Start here.** The big picture: vehicle, comma device hardware, AGNOS OS, the openpilot software stack, and the comma cloud. |
| 2 | [02-process-architecture.mmd](02-process-architecture.mmd) | The manager: boot chain, process types (`PythonProcess` / `NativeProcess` / `DaemonProcess`), the full process table and when each process runs (onroad / offroad / driver view / notCar). |
| 3 | [03-messaging.mmd](03-messaging.mmd) | The nervous system: `cereal` (Cap'n Proto `Event` union + service registry), `msgq` (zero-copy shared-memory pub/sub), `visionipc` (zero-copy camera frames), the `params` store, and the ZMQ `bridge` used for PC development. |
| 4 | [04-onroad-data-flow.mmd](04-onroad-data-flow.mmd) | **Master data-flow graph.** End-to-end message flow while driving: sensors in, actuators out. Edge labels are cereal message names. |
| 5 | [05-vision-model-pipeline.mmd](05-vision-model-pipeline.mmd) | Camera frames: `camerad` → `visionipc` → `modeld` (tinygrad NNs) → `modelV2` / `cameraOdometry`, plus the driver-monitoring path and the logging path. |
| 6 | [06-controls-stack.mmd](06-controls-stack.mmd) | Planning & control internals: `selfdrived` (state machine / events / alerts), `controlsd` (lateral + longitudinal loops), `card`, `plannerd` (MPC), `radard` (radar/vision fusion). |
| 7 | [07-car-abstraction.mmd](07-car-abstraction.mmd) | How openpilot supports 300+ cars: `card` daemon, car fingerprinting (VIN → FW versions → CAN fingerprint), and the per-brand port structure in the `opendbc` submodule. |
| 8 | [08-safety-path.mmd](08-safety-path.mmd) | Safety architecture: the 5-layer defense-in-depth actuation path, `pandad` ↔ panda MCU, and the MISRA-style C safety code in `opendbc/safety` compiled into the panda firmware. |
| 9 | [09-locationd-pipeline.mmd](09-locationd-pipeline.mmd) | State estimation: the `locationd` pose EKF (rednose-generated) and its siblings (`calibrationd`, `paramsd`, `torqued`, `lagd`) plus the GNSS pipelines. |
| 10 | [10-offroad-cloud.mmd](10-offroad-cloud.mmd) | Offroad world: `hardwared` device health, the UI, `athenad` cloud link, `uploader`, `updated` OTA, and comma body / teleop (`webrtcd`, `joystickd`). |
| 11 | [11-ui.mmd](11-ui.mmd) | The raylib UI process: subscriptions, derived UI state, onroad widgets, offroad screens, and audio/feedback processes. |

## How the diagrams connect

The diagrams form a zoom-in hierarchy. 1 is the 10,000-foot view; 4 is the
complete runtime picture; 5–11 are subsystem deep-dives that each expand one
cluster of 4.

```
01 system overview
   └── 02 process architecture      (expands: "Managed processes" + manager)
       └── 03 messaging             (expands: how everything talks: msgq, visionipc, params)
           └── 04 onroad data flow  (expands: the whole onroad runtime; master graph)
               ├── 05 vision & model pipeline   (expands: camerad / modeld / dmonitoring / logging)
               ├── 06 controls stack            (expands: selfdrived / controlsd / card / plannerd / radard)
               │       ├── 07 car abstraction    (expands: card + opendbc car ports, fingerprinting)
               │       └── 08 safety path         (expands: card→pandad→panda actuation chain)
               ├── 09 locationd pipeline         (expands: live* state estimators + GNSS)
               ├── 10 offroad & cloud            (expands: hardwared, athenad, uploader, OTA, teleop)
               └── 11 UI                         (expands: the ui process)
```

Cross-references inside the diagrams use "see NN" notes (e.g. the safety
diagram points back at the controls stack).

## Key concepts needed to read the diagrams

- **Processes.** Everything except the boot scripts is a *process* supervised
  by the `manager` (see `system/manager/process_config.py` for the source of
  truth). Python processes, native binaries (`camerad`, `loggerd`, `encoderd`,
  native `pandad`), and one detached daemon (`manage_athenad`).
- **cereal messages.** Every arrow labeled with a message name
  (`carState`, `modelV2`, `livePose`, …) is a pub/sub edge on the msgq bus,
  defined in `cereal/log.capnp` and registered in `cereal/services.py`.
- **visionipc.** Camera frames bypass the normal message bus and travel
  through dedicated zero-copy buffers (`VISION_STREAM_ROAD`, `WIDE_ROAD`,
  `DRIVER`). Thick arrows (`==>`) in the diagrams indicate frame streams.
- **params.** Dashed arrows indicate the persistent key-value params store
  (`common/params.py`), used for slow / startup data such as `CarParams`,
  `ControlsReady`, and `IsOnroad` — notably the handshake that lets `pandad`
  switch the panda MCU into the car-specific safety mode.
- **onroad vs offroad.** `hardwared` publishes `deviceState.started`; the
  manager uses it to decide which processes run. Ignition comes from the panda
  (`pandaStates`).

## Rendering

```sh
# CLI render to SVG/PNG (requires @mermaid-js/mermaid-cli)
npx -y @mermaid-js/mermaid-cli -i docs/diagrams/04-onroad-data-flow.mmd -o out.svg

# Or paste the file contents into https://mermaid.live
```

## Maintenance notes

- Process names and run conditions in 02 mirror
  `system/manager/process_config.py`; message lists in 04–06 mirror the
  `SubMaster` / `PubMaster` constructors in each daemon's source.
- The edge labels are the actual cereal service names — grep
  `cereal/services.py` to check frequencies and queue sizes.
- If you add a process or a message, update 02 and 04 first; the subsystem
  diagrams (05–11) are refinements of those two.
