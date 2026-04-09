# OpenUMI System Design Specification

**Version:** 1.0  
**Date:** 2026-04-09  
**Status:** Approved for implementation

## 1. Overview

### 1.1 Purpose

OpenUMI is a wireless data collection system for robot imitation learning. It captures bimanual human manipulation demonstrations — including hand pose, gripper aperture, egocentric video, and head-mounted video — for training robot control policies that target parallel-jaw grippers.

### 1.2 Design Goals

- **Portable**: No wires, no external tracking systems, collect demonstrations anywhere
- **Simple mechanics**: 1-DOF scissor gripper per hand, single encoder
- **Sub-millisecond sync**: All sensors and devices time-aligned to <500 us
- **Real-time streaming**: All data streamed to phone over WiFi, no on-device storage
- **Low cost**: Target <$50 per device using commodity components
- **Open source**: Full hardware designs, firmware, and app source code (CC BY-NC-SA 4.0)

### 1.3 System Composition

| Device | Quantity | Sensors | Role |
|--------|----------|---------|------|
| Hand device (L) | 1 | Camera + IMU + Encoder | Captures left-hand manipulation |
| Hand device (R) | 1 | Camera + IMU + Encoder | Captures right-hand manipulation |
| Head device | 1 | Camera + IMU | Captures third-person view |
| iPhone App | 1 | — | Control, preview, storage |

---

## 2. Hardware Design

### 2.1 Component Selection

#### 2.1.1 MCU: ESP32-S3-WROOM-1-N16R8

- Dual-core Xtensa LX7 @ 240MHz
- 16MB Flash + 8MB PSRAM
- WiFi 802.11 b/g/n (2.4GHz)
- BLE 5.0
- DVP camera interface (native)
- I2C, SPI, UART peripherals
- Module size: 18 x 25.5 mm

**Why this variant**: The N16R8 variant provides 8MB PSRAM, essential for buffering JPEG frames during WiFi transmission. The DVP interface natively supports the OV2640 camera without USB Host complications.

#### 2.1.2 Camera: OV2640

- 2MP CMOS sensor
- Built-in JPEG encoder (ISP on-chip)
- DVP parallel interface (8-bit data + PCLK + VSYNC + HREF)
- Default output: 640x480 JPEG @ 30fps (configurable)
- Supported configurations: 640x480 or 320x240, 15fps or 30fps, JPEG quality 50-90
- Typical JPEG frame size: 15-30 KB at quality 70 (640x480)
- Module size: ~8 x 8 mm (with lens)
- FPC 24-pin connector

**Why OV2640 over USB UVC H.264**: ESP32-S3 only supports USB Full Speed (12 Mbps), and the USB Host UVC driver only supports MJPEG format — H.264 UVC is not feasible on S3. OV2640 via DVP has mature ESP-IDF driver support and is the standard camera for ESP32-S3 projects.

**Bandwidth analysis (default 30fps, 640x480)**:
- Per frame: 15-30 KB JPEG
- Per device: 3.6-7.2 Mbps
- Three devices total: 10.8-21.6 Mbps
- iPhone hotspot throughput (2.4GHz): ~15-25 Mbps (practical, multi-device)
- Margin: tight at default settings; fallback configurations available

**Configurable profiles** (set via App before recording):

| Profile | Resolution | FPS | JPEG Quality | Per-Device BW | 3-Device BW | Stability |
|---------|-----------|-----|-------------|---------------|-------------|-----------|
| Default | 640x480 | 30 | 70 | ~5.4 Mbps | ~16.2 Mbps | Good in clean WiFi |
| Safe | 320x240 | 30 | 70 | ~1.9 Mbps | ~5.7 Mbps | Very stable |
| Hi-Res | 640x480 | 15 | 80 | ~3.0 Mbps | ~9.0 Mbps | Very stable |

#### 2.1.3 IMU: BMI270

- 6-axis (3-axis accelerometer + 3-axis gyroscope)
- Max sample rate: 1600Hz (accel) / 6400Hz (gyro)
- Operating sample rate: 200Hz
- Interface: I2C (up to 1MHz) or SPI
- Package: LGA 2.5 x 3.0 x 0.83 mm
- Power consumption: ~0.9 mA
- DATA_READY interrupt pin for precise timing

**Why BMI270**: Low power, high accuracy, widely used in VIO pipelines. The DATA_READY interrupt enables precise timestamping at the hardware level.

#### 2.1.4 Encoder: AS5600

- 12-bit contactless magnetic rotary encoder
- Resolution: 0.088 degrees (4096 positions/revolution)
- Interface: I2C (same bus as BMI270)
- Package: SOIC-8
- Requires diametrically magnetized magnet on rotation axis
- No mechanical wear

**Why AS5600**: Compact, contactless, proven in robotics teleoperation (similar precision level to GELLO). I2C interface allows sharing a bus with BMI270 for synchronized reads.

#### 2.1.5 Power

- **Battery**: 301230 LiPo, ~110mAh, 30 x 12 x 3 mm
- **Charge IC**: TP4056 with DW01 protection
- **Charging**: USB Type-C connector (shared with firmware flashing)
- **LDO**: ME6211 3.3V (low dropout, low quiescent current)
- **Runtime estimate**: ~12-15 minutes at ~400-500mA continuous draw (WiFi + camera + sensors)
- **Connector**: 2-pin JST-PH for battery

#### 2.1.6 Indicators

- LED 1 (green): Power / charging status
- LED 2 (red/blue): Device status (connecting / ready / recording)

### 2.2 Mechanical Structure

#### 2.2.1 Hand Device

The hand device consists of:

1. **Rectangular body** (~30 x 25 x 12 mm): Houses PCB, battery, Type-C port
2. **Two triangular finger pieces**: Worn on thumb and index finger
3. **Shared rotation axis**: Single shaft connecting both finger pieces to the body via a hinge
4. **AS5600 + magnet**: Encoder at one end of the shaft, diametric magnet fixed to the shaft

**Scissor mechanism**: The two finger pieces are riveted to the same axis, forming a scissor-like linkage. When the user opens or closes their fingers, both pieces rotate symmetrically. The AS5600 reads the angular displacement, which maps to parallel-jaw gripper aperture.

**Camera mount**: OV2640 module attached externally on the body via a small bracket, facing forward (egocentric view).

#### 2.2.2 Head Device

- Same rectangular body (without encoder cutout)
- Head strap mounting points (two slots for elastic band)
- Camera module on front-facing bracket
- No encoder, no finger pieces

#### 2.2.3 Manufacturing

- **Body and finger pieces**: JLCPCB 3D printing (SLA resin or MJF nylon)
- **Rotation axis**: Metal pin (stainless steel, press-fit)
- **Design tool**: FreeCAD with MCP automation

### 2.3 PCB Design

#### 2.3.1 Board Specifications

- **Dimensions**: ~25 x 30 mm (2-layer or 4-layer)
- **Universal design**: Same PCB for all three devices; AS5600 left unpopulated for head device
- **Manufacturer**: JLCPCB standard process

#### 2.3.2 Key Layout Considerations

- WiFi antenna keepout zone (no copper under/near ESP32-S3 antenna)
- BMI270 placed near board center for minimal mechanical vibration coupling
- OV2640 FPC connector on board edge
- Type-C connector on board edge (accessible for charging and flashing)
- Battery JST-PH connector
- I2C bus: BMI270 + AS5600 on shared SDA/SCL with pull-ups
- Two LED footprints with current-limiting resistors
- Programming/debug header (optional, for development)

#### 2.3.3 Schematic Blocks

```mermaid
graph LR
    USB["USB Type-C"] --> TP["TP4056<br/>Charge IC"]
    TP --> BAT["LiPo<br/>Battery"]
    BAT --> LDO["ME6211<br/>3.3V LDO"]
    LDO --> ESP["ESP32-S3"]
    ESP -- "I2C" --> IMU["BMI270"]
    ESP -- "I2C" --> ENC["AS5600"]
    ESP -- "DVP" --> CAM["OV2640<br/>(FPC)"]
    ESP -- "GPIO" --> LED["LED × 2"]
    ESP -- "ADC" --> VDIV["Battery<br/>Voltage Divider"]
```

#### 2.3.4 Design Tool

- **KiCad** with MCP server (mixelpixx/KiCAD-MCP-Server)
- **Parts sourcing**: pcbparts-mcp for LCSC availability
- **Manufacturing export**: kicad-jlcpcb-tools plugin (Gerber + BOM + CPL)

---

## 3. Firmware Design

### 3.1 Framework

- **SDK**: ESP-IDF v5.x (official Espressif SDK)
- **RTOS**: FreeRTOS (built into ESP-IDF)
- **Language**: C
- **Build**: CMake + idf.py
- **Development tool**: ESP-IDF built-in MCP server (`idf.py mcp-server`)

### 3.2 Single Firmware, Multi-Role

All three devices run the same firmware binary. Role is configured via NVS (Non-Volatile Storage):

```
NVS Configuration:
  device_role    = LEFT | RIGHT | HEAD
  wifi_ssid      = "<phone hotspot SSID>"
  wifi_pass       = "<phone hotspot password>"
  mdns_name      = "left.openumi" | "right.openumi" | "head.openumi"
  camera_res     = VGA | QVGA          (640x480 | 320x240, default VGA)
  camera_fps     = 30 | 15             (default 30)
  jpeg_quality   = 50-90               (default 70)
```

HEAD role skips AS5600 initialization and fills `encoder_angle = 0` in sensor packets.

Camera configuration can be updated at runtime via App control commands (no reflash needed). NVS stores defaults used on boot before App connects.

Initial NVS configuration is written via USB serial during first setup.

### 3.3 Task Architecture (FreeRTOS)

```mermaid
graph TB
    subgraph Core0["Core 0"]
        WIFI["WiFi Protocol Stack<br/>(system, pinned)"]
        SENSOR["sensor_task<br/>⚡ HIGHEST priority<br/>─────────────<br/>BMI270 DATA_READY ISR<br/>Read IMU via I2C<br/>Read AS5600 via I2C<br/>Hardware timestamp (μs)<br/>Write to send queue<br/>200Hz cycle"]
        NET["net_ctrl_task<br/>MEDIUM priority<br/>─────────────<br/>UDP sensor data send<br/>UDP control cmd receive<br/>Clock sync response"]
    end

    subgraph Core1["Core 1"]
        CAM["camera_task<br/>HIGH priority<br/>─────────────<br/>DVP frame capture<br/>JPEG buffer mgmt<br/>Write to send queue"]
        VID["video_send_task<br/>MEDIUM priority<br/>─────────────<br/>Dequeue JPEG frames<br/>TCP send to phone<br/>Flow control / reconnect"]
    end

    SENSOR --> NET
    CAM --> VID
```

### 3.4 Boot Sequence

1. Read NVS configuration (role, WiFi credentials)
2. Initialize hardware:
   - I2C bus → BMI270 + AS5600 (HEAD skips AS5600)
   - DVP interface → OV2640
   - Hardware timer (microsecond precision)
   - ADC → battery voltage
3. Connect to WiFi hotspot (retry with LED blink)
4. Start mDNS broadcast on connect
5. Send UDP "device online" heartbeat every 1 second
6. Wait for phone TCP connection (video channel)
7. Ready → LED solid

### 3.5 Device State Machine

```mermaid
stateDiagram-v2
    [*] --> IDLE : Power on + WiFi connected
    IDLE --> SYNCING : SYNC_REQ received
    SYNCING --> RECORDING : START received

    state RECORDING {
        [*] --> Sampling
        Sampling : Sensor 200Hz + Camera capture
        Sampling : TCP/UDP streaming
        Sampling : Respond to SYNC (drift correction)
        Sampling : LED blink red
    }

    RECORDING --> FLUSHING : STOP received

    state FLUSHING {
        [*] --> Draining
        Draining : Drain send queues
        Draining : Send END_OF_STREAM marker
    }

    FLUSHING --> IDLE : Flush complete (LED solid)
```

### 3.6 Firmware Components

| Component | Description | Interface |
|-----------|-------------|-----------|
| `sensor_driver` | BMI270 + AS5600 I2C driver, interrupt-driven | I2C, GPIO |
| `camera_driver` | OV2640 DVP capture, JPEG output | DVP |
| `net_manager` | WiFi STA connection, mDNS, reconnect | WiFi |
| `data_streamer` | TCP video send + UDP sensor send | Socket |
| `sync_protocol` | Clock sync request/response handler | UDP |
| `power_manager` | Battery ADC, charge detection, low-battery alert | ADC, GPIO |
| `config_manager` | NVS read/write for device configuration | NVS |
| `led_indicator` | LED status patterns (connecting/ready/recording) | GPIO |

---

## 4. Communication Protocol

### 4.1 Network Topology

```mermaid
graph TD
    PHONE["📱 iPhone Personal Hotspot<br/>172.20.10.1"]
    LEFT["🤚 Left Hand ESP32<br/>172.20.10.x<br/>WiFi STA"]
    RIGHT["✋ Right Hand ESP32<br/>172.20.10.x<br/>WiFi STA"]
    HEAD["👤 Head ESP32<br/>172.20.10.x<br/>WiFi STA"]

    LEFT -- "TCP :19801 (video)<br/>UDP :19802 (sensor)" --> PHONE
    RIGHT -- "TCP :19801 (video)<br/>UDP :19802 (sensor)" --> PHONE
    HEAD -- "TCP :19801 (video)<br/>UDP :19802 (sensor)" --> PHONE
    PHONE -- "UDP :19803 broadcast<br/>(START/STOP/SYNC)" --> LEFT
    PHONE -- "UDP :19803 broadcast" --> RIGHT
    PHONE -- "UDP :19803 broadcast" --> HEAD
```

- Phone acts as WiFi hotspot
- All three ESP32 devices connect as WiFi stations
- Phone app communicates with devices on the local subnet

### 4.2 Device Discovery

**Primary**: mDNS (Bonjour on iOS)
- `left.openumi.local`
- `right.openumi.local`
- `head.openumi.local`

**Fallback**: Each device broadcasts a UDP "heartbeat" packet every 1 second to port 19800 (broadcast address). Packet contains device_id, role, IP address, firmware version. App listens and discovers devices this way if mDNS is unreliable.

### 4.3 Transport Channels

#### 4.3.1 Video Channel: TCP (port 19801)

One TCP connection per device. Carries JPEG frames.

**Frame format (variable length):**

| Field | Size | Type | Description |
|-------|------|------|-------------|
| magic | 4 bytes | uint32 | `0x554D4956` ("UMIV") |
| frame_len | 4 bytes | uint32 | Length of JPEG data |
| timestamp | 8 bytes | uint64 | Device hardware timer, microseconds |
| frame_seq | 4 bytes | uint32 | Frame sequence number |
| reserved | 4 bytes | - | Reserved for future use |
| jpeg_data | frame_len bytes | bytes | JPEG frame data |

**Why TCP**: Video frames cannot be dropped — missing frames break VIO trajectory and create gaps in the dataset. TCP guarantees delivery.

#### 4.3.2 Sensor Channel: UDP (port 19802)

All three devices send to the phone's UDP port 19802. Differentiated by `device_id` field.

**Packet format (48 bytes, fixed):**

| Field | Size | Type | Description |
|-------|------|------|-------------|
| magic | 4 bytes | uint32 | `0x554D4953` ("UMIS") |
| device_id | 1 byte | uint8 | 0=left, 1=right, 2=head |
| seq_num | 3 bytes | uint24 | Sequence number (loss detection) |
| timestamp | 8 bytes | uint64 | Device hardware timer, microseconds |
| accel_x | 4 bytes | float32 | Accelerometer X (m/s^2) |
| accel_y | 4 bytes | float32 | Accelerometer Y (m/s^2) |
| accel_z | 4 bytes | float32 | Accelerometer Z (m/s^2) |
| gyro_x | 4 bytes | float32 | Gyroscope X (rad/s) |
| gyro_y | 4 bytes | float32 | Gyroscope Y (rad/s) |
| gyro_z | 4 bytes | float32 | Gyroscope Z (rad/s) |
| encoder_angle | 4 bytes | float32 | Gripper angle (rad), 0 for head |
| battery_pct | 1 byte | uint8 | Battery percentage |
| status_flags | 1 byte | uint8 | bit0=recording, bit1=low_battery, bit2=sync_locked |
| reserved | 2 bytes | - | Reserved |

**Why UDP**: Sensor data at 200Hz tolerates occasional packet loss. Interpolation can fill gaps. UDP avoids head-of-line blocking that could delay real-time sensor delivery.

**Bandwidth**: 48 bytes x 200 Hz x 3 devices = 28,800 bytes/s = ~230 Kbps (negligible).

#### 4.3.3 Control Channel: UDP (port 19803)

Phone → devices broadcast. Devices → phone unicast responses.

**Packet format (16 bytes, fixed):**

| Field | Size | Type | Description |
|-------|------|------|-------------|
| magic | 4 bytes | uint32 | `0x554D4943` ("UMIC") |
| cmd_type | 1 byte | uint8 | Command type (see below) |
| device_id | 1 byte | uint8 | Source device (for responses) |
| reserved | 2 bytes | - | Reserved |
| timestamp | 8 bytes | uint64 | Sender's hardware timer, microseconds |

**Command types:**

| Code | Name | Direction | Description |
|------|------|-----------|-------------|
| 0x01 | SYNC_REQ | Phone → Device | Clock sync request |
| 0x02 | SYNC_RESP | Device → Phone | Clock sync response |
| 0x03 | START | Phone → Device (broadcast) | Begin recording |
| 0x04 | STOP | Phone → Device (broadcast) | Stop recording |
| 0x05 | HEARTBEAT | Device → Phone | Device alive + status |

### 4.4 Time Synchronization

Three-phase protocol achieving <500 us accuracy across all devices:

```mermaid
sequenceDiagram
    participant Phone
    participant Device

    Note over Phone,Device: Phase 1: Startup Calibration (×10, take median)
    Phone->>Device: SYNC_REQ (phone_time_t1)
    Device->>Phone: SYNC_RESP (device_time_t2)
    Note over Phone: RTT = t3 - t1<br/>offset = t2 - (t1 + RTT/2)

    Note over Phone,Device: Phase 2: Synchronized Start
    Phone->>Device: START (broadcast, absolute timestamp)
    Note over Device: Begin recording<br/>Local hardware timer (μs) free-running

    Note over Phone,Device: Phase 3: Drift Correction (every 10s)
    Phone->>Device: SYNC_REQ
    Device->>Phone: SYNC_RESP
    Note over Phone: Update offset<br/>ESP32 drift ~20ppm → corrected <100μs
```

### 4.5 Intra-Device Sensor Synchronization

Within each ESP32, all sensors share a single hardware timer for timestamping:

1. **BMI270 DATA_READY** fires GPIO interrupt at 200Hz
2. In the ISR, read hardware timer → `timestamp`
3. In the ISR task, read BMI270 via I2C → accel + gyro
4. Immediately read AS5600 via I2C (same bus) → encoder angle
5. Package {timestamp, accel, gyro, encoder} as one sensor sample

IMU and encoder are thus strictly synchronized (read in the same interrupt cycle).

Video frames are timestamped when the DVP VSYNC interrupt fires, then correlated with the nearest sensor timestamp during offline processing.

---

## 5. iOS App Design

### 5.1 Technical Stack

- **UI**: SwiftUI (iOS 16.0+)
- **Networking**: Network.framework (NWConnection, NWBrowser)
- **Video preview**: UIImage from JPEG data (native, no decode library needed)
- **Storage**: FileManager + FileHandle (streaming write to Documents)
- **Development**: Xcode + XcodeBuildMCP + apple-doc-mcp

### 5.2 App Architecture

```mermaid
graph TD
    subgraph UI["SwiftUI View Layer"]
        VL["VideoView<br/>(left)"]
        VR["VideoView<br/>(right)"]
        VH["VideoView<br/>(head)"]
        RV["RecordingView<br/>Controls + Sensor Status"]
    end

    RS["RecordingSession<br/>ObservableObject<br/>State Machine"]

    subgraph DM["Device Managers"]
        DML["DeviceManager<br/>(left)"]
        DMR["DeviceManager<br/>(right)"]
        DMH["DeviceManager<br/>(head)"]
    end

    VL --> RS
    VR --> RS
    VH --> RS
    RV --> RS
    RS --> DML
    RS --> DMR
    RS --> DMH
    DML -- "TCP + UDP" --> ESP1["ESP32 Left"]
    DMR -- "TCP + UDP" --> ESP2["ESP32 Right"]
    DMH -- "TCP + UDP" --> ESP3["ESP32 Head"]
```

### 5.3 Modules

| Module | Responsibility |
|--------|---------------|
| `DeviceManager` | Bonjour discovery (NWBrowser), TCP/UDP connections (NWConnection), device lifecycle |
| `VideoPreview` | Receive JPEG frames, display via UIImage on SwiftUI Canvas |
| `SensorReceiver` | Parse UDP sensor packets, update UI bindings |
| `RecordingSession` | State machine: idle → countdown → calibrating → recording → saving → complete |
| `SyncEngine` | NTP-like clock calibration, drift correction every 10s |
| `DataWriter` | Streaming file writes: JPEG frames to camera/ dir, IMU/encoder to CSV, metadata JSON |

### 5.4 User Flow

1. User enables iPhone Personal Hotspot (preconfigured SSID/password)
2. Power on three devices (auto-connect to hotspot)
3. Open OpenUMI app
4. App discovers devices via Bonjour, shows connection status
5. Three video previews appear automatically
6. User sets recording duration (picker wheel)
7. Tap "Start" → 3-2-1 countdown → clock calibration (~200ms) → recording begins
8. During recording: live preview, sensor readout, countdown timer
9. Timer expires or user taps "Stop" → recording stops
10. Data saved in app Documents directory
11. Export to PC via Finder/iTunes file sharing

### 5.5 Background Execution

iOS suspends apps when backgrounded. Mitigations:

- Request `beginBackgroundTask` on recording start (~30s buffer)
- Use `audio` background mode with silent audio session to keep app alive
- Display recording indicator in status bar
- Show warning: "Do not switch apps during recording"

### 5.6 Data Storage

Data format is designed to align with UMI/Fast-UMI conventions for easy conversion to LeRobot v3.0.

```
Documents/
└── openumi_data/
    └── session_20260409_143052/
        │
        ├── metadata.json
        │
        ├── left_hand/
        │   ├── camera/
        │   │   ├── frame_000000.jpg
        │   │   ├── frame_000001.jpg
        │   │   └── ...
        │   ├── timestamps.csv
        │   ├── imu.csv
        │   └── encoder.csv
        │
        ├── right_hand/
        │   ├── camera/
        │   │   └── ...
        │   ├── timestamps.csv
        │   ├── imu.csv
        │   └── encoder.csv
        │
        └── head/
            ├── camera/
            │   └── ...
            ├── timestamps.csv
            └── imu.csv              # no encoder.csv for head
```

**timestamps.csv** — one row per video frame:
```csv
frame_index,timestamp_us,timestamp_s
0,1712678432000000,0.000
1,1712678432033333,0.033
2,1712678432066667,0.067
```

**imu.csv** — 200Hz sensor data:
```csv
timestamp_us,accel_x,accel_y,accel_z,gyro_x,gyro_y,gyro_z
1712678432000000,0.12,-9.78,0.34,0.001,-0.002,0.001
```

**encoder.csv** — 200Hz encoder data (hand devices only):
```csv
timestamp_us,angle_rad
1712678432000000,0.45
```

**Why CSV over binary**: Human-readable, directly loadable with `pandas.read_csv()`, easy to inspect and debug. The small overhead (~2x vs binary) is negligible given the dominant cost is JPEG frames.

**metadata.json** contains:
```json
{
  "session_id": "20260409_143052",
  "start_time_utc": "2026-04-09T14:30:52Z",
  "end_time_utc": "2026-04-09T14:35:52Z",
  "fps": 30,
  "camera_resolution": [640, 480],
  "jpeg_quality": 70,
  "imu_sample_rate_hz": 200,
  "devices": ["left_hand", "right_hand", "head"],
  "clock_offsets_us": {
    "left_hand": 125.3,
    "right_hand": -89.7,
    "head": 42.1
  },
  "encoder_zero_offset_rad": {
    "left_hand": 0.15,
    "right_hand": 0.12
  },
  "camera_intrinsics": {
    "left_hand": {"fx": 320.0, "fy": 320.0, "cx": 320.0, "cy": 240.0, "dist": [0,0,0,0,0]},
    "right_hand": {"fx": 320.0, "fy": 320.0, "cx": 320.0, "cy": 240.0, "dist": [0,0,0,0,0]},
    "head": {"fx": 320.0, "fy": 320.0, "cx": 320.0, "cy": 240.0, "dist": [0,0,0,0,0]}
  },
  "task_description": "pick up the cup",
  "firmware_version": "0.1.0",
  "app_version": "0.1.0"
}
```

### 5.7 File Sharing

Enable in Info.plist:
- `UIFileSharingEnabled = YES`
- `LSSupportsOpeningDocumentsInPlace = YES`

Users can access session directories via Finder (macOS) or Files app (iOS).

---

## 6. Offline Data Processing & LeRobot Integration

### 6.1 Three-Stage Conversion Pipeline

The pipeline converts raw phone data to LeRobot v3.0 format via an intermediate UMI-compatible zarr, reusing the existing LeRobot UMI conversion script.

```mermaid
graph TD
    subgraph S1["Stage 1: Raw Session (from phone)"]
        RAW["openumi_data/session_xxx/<br/>├── {device}/camera/*.jpg<br/>├── {device}/timestamps.csv<br/>├── {device}/imu.csv<br/>├── {device}/encoder.csv<br/>└── metadata.json"]
    end

    PROC["openumi_process.py<br/>(Steps 1-5)"]

    subgraph S2["Stage 2: UMI-Compatible Zarr"]
        ZARR["session_xxx.zarr/<br/>├── data/robot0_eef_pos (T,3)<br/>├── data/robot0_eef_rot_axis_angle (T,3)<br/>├── data/robot0_gripper_width (T,1)<br/>├── data/robot1_* (right hand)<br/>├── data/camera0/1/2_rgb (T,H,W,3)<br/>└── meta/episode_ends (N,)"]
    end

    CONV["openumi_to_lerobot.py<br/>(adapted umi_zarr_format.py)"]

    subgraph S3["Stage 3: LeRobot v3.0 Dataset"]
        LR["openumi_dataset/<br/>├── meta/ (info.json, stats, tasks, episodes)<br/>├── data/ (chunk Parquet files)<br/>└── videos/<br/>    ├── observation.images.left_wrist/*.mp4<br/>    ├── observation.images.right_wrist/*.mp4<br/>    └── observation.images.head/*.mp4"]
    end

    RAW --> PROC --> ZARR --> CONV --> LR
```

### 6.2 Processing Steps (openumi_process.py)

| Step | Input | Output | Description |
|------|-------|--------|-------------|
| 1 | timestamps.csv + metadata.json | Aligned timestamps | Apply clock offsets, convert to seconds from episode start |
| 2 | camera/*.jpg + imu.csv (per hand) | camera_trajectory.csv | ORB-SLAM3 visual-inertial odometry → 6-DoF camera poses |
| 3 | camera_trajectory.csv + calibration | eef_pos + eef_rot | Hand-eye calibration: camera pose → TCP (tool center point) pose |
| 4 | encoder.csv + mechanical params | gripper_width (meters) | Encoder angle → gripper width via mechanical geometry |
| 5 | All above | session_xxx.zarr | Resample all streams to video framerate (30fps), assemble UMI zarr |

### 6.3 LeRobot Feature Mapping

```json
{
  "codebase_version": "v3.0",
  "robot_type": "openumi",
  "fps": 30,
  "features": {
    "observation.state": {
      "dtype": "float32",
      "shape": [14],
      "names": {
        "motors": [
          "left_x", "left_y", "left_z",
          "left_rx", "left_ry", "left_rz",
          "left_gripper",
          "right_x", "right_y", "right_z",
          "right_rx", "right_ry", "right_rz",
          "right_gripper"
        ]
      }
    },
    "observation.images.left_wrist": {
      "dtype": "video", "shape": [480, 640, 3]
    },
    "observation.images.right_wrist": {
      "dtype": "video", "shape": [480, 640, 3]
    },
    "observation.images.head": {
      "dtype": "video", "shape": [480, 640, 3]
    },
    "action": {
      "dtype": "float32",
      "shape": [14],
      "names": {"motors": ["...same as observation.state..."]}
    }
  }
}
```

**observation.state** = `[left_pos(3) + left_rot_axis_angle(3) + left_gripper_width(1) + right_pos(3) + right_rot_axis_angle(3) + right_gripper_width(1)]` = 14D

**action** = `observation.state[t+1]` (next-state-as-action, standard in UMI/Diffusion Policy)

### 6.4 VIO Algorithm

- **Initial**: Offline ORB-SLAM3 (visual-inertial mode)
  - Uses JPEG sequence + IMU data
  - IMU integration provides metric scale (absolute meters)
  - First session builds a 3D map; subsequent sessions relocalize to shared map
  - Expected accuracy: ~6mm position, ~3.5 degrees rotation (matching UMI results)
- **Future**: Real-time VIO on iPhone (ARKit or custom), upgrading from Phase C to Phase B

### 6.5 Tools & Dependencies

- **Python 3.10+** for all processing scripts
- **ORB-SLAM3** for visual-inertial odometry
- **OpenCV** for JPEG decoding and ArUco/calibration
- **NumPy / Pandas** for data manipulation
- **zarr** for intermediate UMI format
- **lerobot** Python package for final conversion and push_to_hub
- **ffmpeg** for JPEG sequence → MP4 transcoding (called by lerobot)

---

## 7. AI-Driven Development Toolchain

### 7.1 MCP Server Configuration

```json
{
  "mcpServers": {
    "freecad": {
      "command": "python",
      "args": ["<path>/freecad-mcp/server.py"]
    },
    "kicad": {
      "command": "node",
      "args": ["<path>/KiCAD-MCP-Server/dist/index.js"]
    },
    "pcbparts": {
      "type": "http",
      "url": "https://pcbparts.dev/mcp"
    },
    "esp-idf": {
      "command": "idf.py",
      "args": ["mcp-server"]
    },
    "xcodebuild": {
      "command": "node",
      "args": ["<path>/XcodeBuildMCP/dist/index.js"]
    },
    "apple-docs": {
      "command": "node",
      "args": ["<path>/apple-doc-mcp/dist/index.js"]
    }
  }
}
```

### 7.2 Tool-Domain Mapping

| Development Phase | Primary Tool | MCP/Plugin | Manufacturing |
|-------------------|-------------|------------|---------------|
| Mechanical design | FreeCAD | neka-nat/freecad-mcp | JLCPCB 3D Print |
| PCB design | KiCad 8 | mixelpixx/KiCAD-MCP-Server | JLCPCB PCB+SMT |
| Parts sourcing | LCSC | Averyy/pcbparts-mcp | JLCPCB/LCSC |
| Firmware | ESP-IDF v5.x | idf.py mcp-server | USB flash |
| iOS App | Xcode / SwiftUI | getsentry/XcodeBuildMCP | TestFlight |
| Documentation | Apple docs | MightyDillah/apple-doc-mcp | — |
| Offline processing | Python | Claude Code (direct) | — |

---

## 8. Implementation Phases

```mermaid
gantt
    title OpenUMI Development Roadmap
    dateFormat YYYY-MM-DD
    axisFormat %b

    section Design
    Phase 1: System Design & Spec           :done, p1, 2026-04-01, 2026-04-09

    section Software Validation
    Phase 2: Dev Board Prototype            :p2, after p1, 14d
    Phase 3: iOS App Single-Device          :p3, after p2, 14d
    Phase 4: Offline Pipeline (VIO+LeRobot) :p4, after p2, 21d

    section Custom Hardware
    Phase 5: PCB Design & Fabrication       :p5, after p4, 21d
    Phase 6: Mechanical Design & 3D Print   :p6, after p5, 14d

    section Integration
    Phase 7: Single-Device Integration      :p7, after p6, 7d
    Phase 8: Three-Device Joint Validation  :p8, after p7, 14d

    section Ecosystem
    Phase 9: Data Visualization Tools       :p9, after p8, 21d
    Phase 10: Model Training & Deployment   :p10, after p8, 28d
```

### Phase 1: System Design & Specification ✅

**Goal**: Complete system architecture and design specification.

- System architecture, component selection, protocol design
- Data format aligned with UMI/LeRobot ecosystem
- AI-driven development toolchain selection
- **Validation criteria**: Design reviewed and approved

### Phase 2: Dev Board Prototype Validation

**Goal**: Validate core electronics and firmware on off-the-shelf dev boards before committing to custom PCB.

1. Assemble dev kit: ESP32-S3 dev board + OV2640 camera module + BMI270 breakout + AS5600 breakout
2. Firmware prototype: WiFi JPEG streaming (640x480 @ 30fps) + IMU 200Hz + encoder reading
3. Validate single-device WiFi throughput and stability
4. **Validation criteria**: Stable 30fps JPEG + 200Hz IMU streaming over WiFi for 15+ minutes

### Phase 3: iOS App Single-Device Validation

**Goal**: Complete iOS app connected to one dev board.

1. Bonjour device discovery + TCP/UDP connection
2. Live JPEG video preview
3. Recording control (countdown, timed stop)
4. Data storage in CSV + JPEG format (matching design spec)
5. **Validation criteria**: Record a 5-minute session, data files match format spec, export to PC works

### Phase 4: Offline Data Pipeline Validation

**Goal**: End-to-end from raw capture to LeRobot dataset.

1. Run ORB-SLAM3 on captured JPEG + IMU data → 6-DoF trajectory
2. Encoder angle → gripper width mapping
3. Assemble UMI-compatible zarr
4. Convert zarr → LeRobot v3.0 format
5. Load dataset with `lerobot` Python package, verify structure
6. **Validation criteria**: Valid LeRobot v3.0 dataset loadable for policy training

### Phase 5: Custom PCB Design & Fabrication

**Goal**: Miniaturized PCB matching dev board functionality.

1. Schematic design in KiCad (ESP32-S3 + BMI270 + AS5600 + TP4056 + OV2640 FPC)
2. PCB layout (~25x30mm, antenna keepout, I2C routing)
3. DRC + manufacturing rule check for JLCPCB
4. Export Gerber + BOM + CPL, order from JLCPCB (PCB + SMT assembly)
5. **Validation criteria**: Assembled PCB boots, all sensors respond, WiFi connects

### Phase 6: Mechanical Design & 3D Printing

**Goal**: Enclosure and finger mechanism designed around actual PCB.

1. Measure final PCB dimensions
2. Design in FreeCAD: body shell, scissor finger pieces, camera mount, head strap mount
3. Export STL, order from JLCPCB 3D printing (SLA resin or MJF nylon)
4. Test fit, iterate if needed
5. **Validation criteria**: All components fit, finger mechanism moves freely, comfortable to wear

### Phase 7: Single-Device Integration Test

**Goal**: One fully assembled device (custom PCB + enclosure + battery) performs as well as dev board.

1. Assemble: PCB + battery + camera + encoder magnet + 3D printed enclosure
2. Flash firmware, configure NVS
3. Run same test as Phase 2-3 with assembled device
4. Compare data quality (IMU noise, JPEG quality, WiFi stability) against dev board baseline
5. **Validation criteria**: No degradation vs dev board; battery lasts 12+ minutes

### Phase 8: Three-Device Joint Validation

**Goal**: Full system with synchronized three-device capture.

1. Assemble three devices (2x hand + 1x head)
2. Three-device simultaneous WiFi streaming to iPhone
3. Clock synchronization protocol validation (<500 us)
4. iOS app: three-camera preview + full recording workflow
5. Field testing: collect real bimanual manipulation demonstrations
6. Process through full pipeline → LeRobot dataset
7. **Validation criteria**: Synchronized three-view dataset, all timestamps aligned, suitable for policy training

### Phase 9: Data Visualization & Annotation Tools

**Goal**: Tools for inspecting, browsing, and quality-checking collected data.

1. Web-based session browser (list sessions, preview video, plot IMU/encoder)
2. 3D trajectory visualization (VIO output)
3. Data quality metrics (dropped frames, sync drift, coverage)
4. Episode annotation / tagging interface
5. **Validation criteria**: Can quickly identify and filter bad episodes from a collection session

### Phase 10: Model Training & Robot Deployment Verification

**Goal**: Close the loop — train a policy on collected data and deploy on a real robot.

1. Train Diffusion Policy / ACT on OpenUMI LeRobot dataset
2. Deploy policy on robot with parallel-jaw gripper (e.g., UR5e, Franka)
3. Evaluate task success rate
4. Compare with UMI-collected data on same task
5. **Validation criteria**: Trained policy achieves reasonable success rate on target manipulation task

---

## 9. Risk Register

| Risk | Severity | Likelihood | Mitigation |
|------|----------|------------|------------|
| WiFi bandwidth insufficient for 3x JPEG streams | High | Medium | Reduce JPEG quality / framerate; test early in Phase 1 |
| OV2640 JPEG quality too low for VIO | Medium | Low | Test with ORB-SLAM3 early; can upgrade to OV5640 if needed |
| iOS background suspension during recording | Medium | High | Silent audio session + beginBackgroundTask; user warning |
| mDNS unreliable on iPhone hotspot | Low | Medium | UDP heartbeat fallback discovery mechanism |
| ESP32-S3 dual-core task contention | Medium | Low | Priority-based scheduling; sensor task at highest priority |
| Battery life too short (<10 min) | Medium | Medium | Optimize WiFi power; reduce camera framerate during idle |
| 3D-printed enclosure too fragile | Low | Low | Test with MJF nylon; iterate design |

---

## 10. References

- [UMI: Universal Manipulation Interface](https://umi-gripper.github.io/) — Design inspiration, SLAM pipeline
- [Fast-UMI](https://github.com/zxzm-zak/FastUMI_Data) — Hardware pose tracking variant, data format reference
- [LeRobot](https://github.com/huggingface/lerobot) — Dataset format v3.0, training framework
- [GELLO](https://wuphilipp.github.io/gello_site/) — Encoder precision reference
- [ALOHA 2](https://aloha-2.github.io/) — Bimanual teleoperation reference
- [ESP-IDF Programming Guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/)
- [OV2640 Datasheet](https://www.uctronics.com/download/cam_module/OV2640DS.pdf)
- [BMI270 Datasheet](https://www.bosch-sensortec.com/products/motion-sensors/imus/bmi270/)
- [AS5600 Datasheet](https://ams.com/en/as5600)
- [Network.framework Documentation](https://developer.apple.com/documentation/network)
- [VideoToolbox Documentation](https://developer.apple.com/documentation/videotoolbox)
