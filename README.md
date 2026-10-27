AUTONEX X1: Intelligent Robotic Inspection Platform
A graduation project in Communication and Computer Engineering. AUTONEX X1 is a two-wheeled inspection robot built around a custom ESP32 motor-controller board and a Raspberry Pi 4 that acts as the robot's central computer. A web ground station is used to drive the robot, watch telemetry and cameras, and log inspection events.
> **Project status: early development.** The ESP32 firmware and a single-file ground station exist today. The Raspberry Pi backend and the React dashboard described below are **planned, not yet implemented**. See [Status](#status).
---
Table of contents
System overview
Hardware
Architecture
Safety design
ESP32 serial protocol
Repository layout
Status
Getting started (current files)
Roadmap
Known limitations and assumptions
Author
---
System overview
Layer	Technology	Responsibility
Low-level controller	ESP32-WROOM-32D, C++ (Arduino framework)	PWM, encoder decoding, closed-loop RPM control (PID), ultrasonic ranging, hard safety timeout
High-level computer	Raspberry Pi 4, Python 3 (planned: FastAPI, PySerial, OpenCV, SQLite)	Command validation, kinematics, safety supervision, camera, storage, API
Ground station	Today: single HTML file. Planned: React, TypeScript, Vite, Tailwind CSS	Driving, telemetry, camera view, event log, reports
Hardware
Taken from the firmware source. Verify every item against your own board.
ESP32-WROOM-32D on a custom PCB
Raspberry Pi 4 (high-level computer), connected to the ESP32 over UART
2 DC motors driven by 2 MD20A motor drivers (DIR + PWM)
2 quadrature encoders
2 HC-SR04 ultrasonic sensors
2 NeoPixel status LEDs
Cameras connected to the Raspberry Pi
Wi-Fi networking (Raspberry Pi)
Pin assignments live in the `PINS` block at the top of the firmware. They are marked VERIFY in the source and must be checked against the PCB silkscreen before flashing.
Architecture
```
Browser (ground station)
        │   WebSocket + REST (planned: authenticated, TLS)
        ▼
Raspberry Pi 4  ── Robot backend (validation, limits, safety, logging)
        │   UART 115200
        ▼
ESP32  ── Motor drivers / motors / encoders / ultrasonic sensors
```
Design rule: the browser never talks to the ESP32 directly. The Raspberry Pi is the single point of control. Data flows:
Commands: Frontend → Raspberry Pi → robot controller → UART → ESP32
Telemetry: ESP32 → UART → Raspberry Pi → WebSocket → Frontend
Safety design
Planned layers (the ESP32 layer is implemented today):
ESP32 timeout: if no valid motion command (`M`, `V` or `STOP`) arrives for 500 ms, the motors are commanded to stop.
Raspberry Pi deadman (planned): the Pi streams commands at a steady rate and zeroes the setpoint itself if the browser goes quiet or disconnects.
Server-side limits (planned): maximum speed and turn rate are enforced on the Pi, and invalid or non-finite values are rejected.
Emergency stop (planned): a latched state, always reachable, requiring an explicit reset.
Authentication and command audit log (planned).
A software emergency stop is not a safety-rated device. For demonstrations, use a physical cut-off on the motor supply.
ESP32 serial protocol
115200 baud, newline-terminated ASCII.
Pi → ESP32
Command	Meaning
`M,<l>,<r>`	Open-loop PWM, -255..255
`V,<l>,<r>`	Closed-loop speed in RPM (output shaft)
`STOP`	Stop
`PID,<kp>,<ki>,<kd>`	Speed PID gains (both motors)
`FF,<kff>`	Feed-forward, PWM counts per RPM
`DB,<0..400>`	PWM deadband
`RAMP,<rpm_per_s>`	Acceleration limit in closed loop
`MAXRPM,<rpm>`	Clamp for `V` commands
`FREQ,<hz>`	PWM frequency (MD20A max 20000)
`SAVE`	Store settings in flash
`GET`	Print current settings
ESP32 → Pi
Message	Meaning
`T,us1,us2,cnt1,cnt2,rpm1,rpm2,tgt1,tgt2,pwm1,pwm2`	Telemetry every 100 ms (ultrasonic in cm, `-1` = invalid)
`CFG,...`	Current settings (reply to `GET` and after settings commands)
`OK,saved`	Reply to `SAVE`
Planned browser-to-Pi messages are JSON, for example:
```json
{ "type": "drive", "linear": 0.5, "angular": 30 }
```
Units for `linear` and `angular` are still to be defined.
Repository layout
Current layout for the files that exist today:
```
autonex-x1/
├── firmware/
│   └── autonex_esp32/
│       └── autonex_esp32.ino        # ESP32 firmware
├── ground-station/
│   └── robot-ground-station.html    # Current single-file ground station
├── docs/                            # Protocol, wiring, runbook (to be written)
└── README.md
```
Planned layout once the backend and dashboard are added:
```
autonex-x1/
├── firmware/esp32/
├── backend/        # FastAPI app: api, websocket, uart, robot, safety, telemetry,
│                   #   camera, sensors, navigation, database, security, config, tests
├── frontend/       # React + TypeScript + Vite + Tailwind
├── deploy/         # systemd units, udev rules, TLS
├── legacy/         # Original single-file ground station, kept as a parity reference
└── docs/
```
Arduino IDE requires a sketch to live in a folder with the same name as the `.ino` file, which is why the firmware sits in `firmware/autonex_esp32/`.
Status
Component	State
ESP32 firmware: open-loop control, closed-loop RPM + PID, encoders, ultrasonic, PWM ramp, safety timeout, runtime config, UART telemetry	Implemented
Ground station (single HTML file): simulator, WebSocket, camera view, keyboard driving, telemetry, range visualization, anomaly log, CSV export, HTML report	Implemented (see file)
Raspberry Pi backend (FastAPI, UART, safety supervisor)	Planned
React dashboard	Planned
Authentication, roles, TLS	Planned
Database and mission logging	Planned
Navigation, OpenCV, object detection, anomaly detection	Planned
Automated tests and deployment	Planned
Getting started (current files)
Firmware
Requirements:
Arduino IDE with the Arduino-ESP32 core 3.x (the code uses `ledcAttach` / `ledcWrite(pin, duty)`)
Library: Adafruit NeoPixel
Before uploading:
Verify every pin in the `PINS` block against your PCB.
Set `ENC_PPR` and `GEAR_RATIO` to your motor's real values (the defaults, 11 and 34, are placeholders).
Flip the `MOTORx_INVERT` / `ENCx_INVERT` flags if a wheel spins the wrong way or counts go negative when moving forward.
Lift the wheels off the ground for the first test.
Open `firmware/autonex_esp32/autonex_esp32.ino`, select your ESP32 board, and upload. At 115200 baud the board prints a `CFG,...` line at boot and `T,...` telemetry every 100 ms.
Ground station
Open `ground-station/robot-ground-station.html` in a modern browser.
Roadmap
Engineering analysis of existing code (done)
Raspberry Pi backend skeleton
UART communication
Telemetry parsing
WebSocket and REST API
React dashboard
Camera streaming
Sensors and range visualization
Database and mission logging
Navigation and mapping
OpenCV
Object detection
Anomaly detection
Security hardening
Testing, documentation, deployment
The backend will include a mock mode that simulates the ESP32, so the system can be developed and tested without the physical robot.
Known limitations and assumptions
Pin numbers, encoder constants and motor parameters are assumptions in the source and must be verified on the physical robot.
The firmware has no battery sensing; battery monitoring needs hardware that is not yet defined.
The robot has two ultrasonic sensors, not a LiDAR. Any "LiDAR-style" view is a visualization of two range readings.
Hardware-level behavior (boot-time pin states, signal levels on the HC-SR04 ECHO lines, strapping pins) has not been verified on the physical board.
Author
Khaled, graduation project in Communication and Computer Engineering.
GitHub: khaledhamdy404
License
No license has been chosen yet. Add a `LICENSE` file (for example MIT) before making the repository public if you want others to reuse the code.
