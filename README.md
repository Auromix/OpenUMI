# OpenUMI

[![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/)
[![LeRobot Compatible](https://img.shields.io/badge/LeRobot-v3.0%20Compatible-blue)](https://github.com/huggingface/lerobot)
[![Status: Design Phase](https://img.shields.io/badge/Status-Design%20Phase-yellow)]()

An open-source, wireless data collection system for robot imitation learning. Capture bimanual manipulation demonstrations with finger-mounted devices and a head-mounted camera, all streaming wirelessly to your phone.

## Overview

OpenUMI is a portable, low-cost data collection toolkit designed for collecting human demonstration data to train robot manipulation policies. Inspired by [UMI (Universal Manipulation Interface)](https://umi-gripper.github.io/), OpenUMI takes a different approach: instead of a handheld gripper, it uses **finger-mounted devices** that are lighter, more ergonomic, and capture data from the human's natural grasping perspective.

### Key Features

- **Wireless & portable** — No cables, no external tracking systems, collect data anywhere
- **Sub-millisecond sync** — Hardware timer + UDP broadcast synchronization across all devices (~100-500 us)
- **LeRobot v3.0 native** — Data pipeline outputs UMI-compatible zarr, directly convertible to LeRobot datasets
- **Low cost** — ~$30-50 per device using off-the-shelf components
- **Simple mechanics** — Scissor-style 1-DOF gripper with a single magnetic encoder
- **Configurable capture** — 640x480 or 320x240, 15 or 30 fps, adjustable JPEG quality
- **AI-driven development** — Entire project built with Claude Code + MCP tools (FreeCAD, KiCad, ESP-IDF, Xcode)

## Architecture

```
 Left Hand          Right Hand          Head
 ┌──────────┐      ┌──────────┐      ┌──────────┐
 │ESP32-S3  │      │ESP32-S3  │      │ESP32-S3  │
 │OV2640    │      │OV2640    │      │OV2640    │
 │BMI270    │      │BMI270    │      │BMI270    │
 │AS5600    │      │AS5600    │      │(no enc.) │
 │LiPo Batt │      │LiPo Batt │      │LiPo Batt │
 └────┬─────┘      └────┬─────┘      └────┬─────┘
      │ WiFi             │ WiFi             │ WiFi
      └──────────────────┼──────────────────┘
                         │
                  ┌──────▼──────┐
                  │  iPhone App │         ┌────────────┐
                  │  (SwiftUI)  │────────→│ PC Offline │
                  └─────────────┘         │ VIO + Conv.│
                   Video preview          └─────┬──────┘
                   Sensor status                │
                   Recording ctrl          LeRobot v3.0
                   Local storage              Dataset
```

The system consists of three wireless devices and a phone app:

- **Left Hand Device** — Worn on thumb and index finger, captures gripper open/close angle, 6-axis IMU, and egocentric video
- **Right Hand Device** — Same as left hand
- **Head Device** — Mounted on the head, captures a third-person view with IMU

All devices stream data in real-time over WiFi to an iOS app. After collection, a PC-side pipeline runs VIO (ORB-SLAM3) to recover 6-DoF poses and converts the data to LeRobot v3.0 format.

## Hardware

### Components per Device

| Component | Model | Interface | Purpose |
|-----------|-------|-----------|---------|
| MCU | ESP32-S3-WROOM-1-N16R8 | — | Main controller, WiFi, 8MB PSRAM |
| Camera | OV2640 | DVP | 640x480 JPEG @ 30fps (configurable) |
| IMU | BMI270 | I2C | 6-axis, 200Hz, interrupt-driven |
| Encoder | AS5600 | I2C | 12-bit magnetic, absolute angle |
| Battery | 301230 LiPo | JST-PH | ~110mAh, 12-15 min runtime |
| Charger | TP4056 | Type-C | Li-ion charge management |

> The head device uses the same PCB with AS5600 left unpopulated.

### Mechanical Design

The finger device uses a scissor mechanism: two triangular finger pieces share a single rotation axis. An AS5600 magnetic encoder reads the open/close angle. The rectangular body (~30x25x12mm) houses the PCB, battery, and Type-C port, with the camera module mounted externally.

## Software

### Firmware (ESP-IDF)

Single firmware for all three devices, role configured via NVS:
- Sensor sampling at 200Hz (interrupt-driven, hardware-timestamped)
- JPEG video streaming over TCP
- IMU/encoder data over UDP at 200Hz
- Clock synchronization protocol (<500 us accuracy)
- Configurable camera resolution, framerate, and JPEG quality

### iOS App (SwiftUI)

- Auto-discovers devices via Bonjour (mDNS)
- Three-camera live JPEG preview
- Recording control with 3-second countdown and timed stop
- Streams all data to CSV + JPEG files in app Documents
- Export to PC via Finder / Files app

### Offline Processing (Python)

- VIO pose estimation from JPEG sequences + IMU data (ORB-SLAM3)
- Encoder angle → gripper width mapping
- Converts to UMI-compatible zarr → LeRobot v3.0 dataset format
- Supports `push_to_hub()` to Hugging Face

## Data Pipeline

```
Phone (raw)                    PC (processed)                  Training
─────────────                  ──────────────                  ────────
JPEG frames ─┐                 ┌─ ORB-SLAM3 ──→ 6-DoF pose
IMU CSV ─────┤ ──export──→     ├─ Encoder    ──→ gripper width ──→ LeRobot v3.0
Encoder CSV ─┤                 └─ Assemble   ──→ UMI zarr     ──→ Diffusion Policy
metadata.json┘                                                     ACT, etc.
```

### Raw Data Format

```
session_YYYYMMDD_HHMMSS/
├── metadata.json             # Session config, clock offsets, task description
├── left_hand/
│   ├── camera/               # JPEG frames (frame_000000.jpg, ...)
│   ├── timestamps.csv        # Per-frame timestamps (us + seconds)
│   ├── imu.csv               # 200Hz accelerometer + gyroscope
│   └── encoder.csv           # 200Hz absolute encoder angle (rad)
├── right_hand/
│   └── ...                   # Same structure as left_hand
└── head/
    ├── camera/
    ├── timestamps.csv
    └── imu.csv               # No encoder for head device
```

### LeRobot Output Format

| LeRobot Field | Shape | Source |
|---|---|---|
| `observation.state` | [14] float32 | VIO pose (6) + gripper (1), per hand |
| `observation.images.left_wrist` | [480, 640, 3] video | Left hand camera → MP4 |
| `observation.images.right_wrist` | [480, 640, 3] video | Right hand camera → MP4 |
| `observation.images.head` | [480, 640, 3] video | Head camera → MP4 |
| `action` | [14] float32 | `observation.state[t+1]` |

## Development Toolchain

This project is developed using an AI-driven workflow with Claude Code and MCP integrations:

| Domain | Tool | MCP Server |
|--------|------|------------|
| Mechanical | FreeCAD | neka-nat/freecad-mcp |
| PCB | KiCad 8 | mixelpixx/KiCAD-MCP-Server |
| Parts | LCSC/JLCPCB | Averyy/pcbparts-mcp |
| Firmware | ESP-IDF v5.x | Built-in `idf.py mcp-server` |
| iOS App | Xcode / SwiftUI | getsentry/XcodeBuildMCP |
| Manufacturing | JLCPCB | kicad-jlcpcb-tools |

## Project Status

This project is in the **design phase**. See [`docs/design/system-design.md`](docs/design/system-design.md) for the full design specification.

### Roadmap

- [x] System design & specification
- [ ] Phase 1: Core pipeline validation (1 device + app prototype on dev board)
- [ ] Phase 2: Custom hardware (PCB + 3D printed enclosure via JLCPCB)
- [ ] Phase 3: Three-device integration & field testing

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## References

- [UMI: Universal Manipulation Interface](https://umi-gripper.github.io/) — Design inspiration, SLAM pipeline reference
- [Fast-UMI](https://github.com/zxzm-zak/FastUMI_Data) — Hardware pose tracking variant
- [LeRobot](https://github.com/huggingface/lerobot) — Dataset format and training framework
- [GELLO](https://wuphilipp.github.io/gello_site/) — Encoder precision reference
- [ALOHA 2](https://aloha-2.github.io/) — Bimanual teleoperation reference

## License

This project is licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).

- **Academic and non-commercial use**: Free under the terms of CC BY-NC-SA 4.0
- **Commercial use**: Requires a separate license. Contact [hermanye@auromix.org](mailto:hermanye@auromix.org) for inquiries.

See [LICENSE](LICENSE) for full details.
