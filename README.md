# SerBot: AI-Powered Autonomous Surveillance Rover

Facial recognition, intrusion detection and live tracking on a low-cost 4WD rover.
Course project for 21CSE305P (SerBot, Project Based Learning in Robotics), SRM Institute of Science and Technology, Kattankulathur.

## What it does

- Patrols and avoids obstacles using three ultrasonic sensors (front, left, right).
- Detects faces with a Raspberry Pi camera and matches them against enrolled people.
- Declares an **intrusion** when an unknown face appears in 5 consecutive processed frames.
- Raises a **silent alert**: saves a snapshot, writes an event to SQLite and POSTs to a webhook (30 s cooldown). No buzzer.
- Keeps the face centred with a pan servo (proportional control, 20 px dead-band) while the chassis holds position.
- Stops the rover if the Pi stops talking to the Arduino for 2 s (heartbeat watchdog).

## Architecture

| Unit | Role |
|---|---|
| Arduino Uno R3 | Real-time control: L298N motors, 3 ultrasonic sensors, pan servo, obstacle avoidance, watchdog |
| Raspberry Pi 5 (4 GB) + Pi camera | Vision: detection, recognition, intrusion logic, tracking, alerts, event log |
| USB serial, 115200 baud | ASCII protocol between the two |

Serial protocol (newline terminated):

| Direction | Message | Meaning |
|---|---|---|
| Pi to Arduino | `PAN,<deg>` | Set pan servo angle |
| Pi to Arduino | `MODE,PATROL` / `MODE,HOLD` | Drive or stay still |
| Pi to Arduino | `HB` | Heartbeat |
| Arduino to Pi | `DIST,<f>,<l>,<r>` | Distances in cm |

## Repository layout

```
firmware/serbot.ino        Arduino firmware
pi/serbot_pi.py            Raspberry Pi vision, tracking and alert program
pi/known_faces/<name>/*.jpg   Enrolment photos, one folder per person (you create this)
sim/serbot_sim.py          Offline simulation (avoidance, tracking, alert timing, fail-safe)
sim/test_alert.py          Runs the real alert/event code with camera and network mocked
sim/plots.py, eventlog_img.py   Figure generators for the report
docs/                      Report, poster, literature review
```

Move the files into this layout before pushing. `sim/test_alert.py` imports `serbot_pi` through `sys.path.insert(0, '../build')`; change that line to `'../pi'`.

## Hardware

Arduino Uno R3, Raspberry Pi 5 (4 GB) with Pi camera, L298N motor driver, 4WD chassis kit, 3 ultrasonic sensors (HC-SR04 type), one MG90S pan servo, 4 x AA motor pack, USB-C power bank for the Pi.

Arduino pin map (matches `serbot.ino`):

| Function | Pins |
|---|---|
| Left motors (L298N channel A) | ENA D5, IN1 D4, IN2 D7 |
| Right motors (L298N channel B) | ENB D6, IN3 D8, IN4 D9 |
| Front ultrasonic | TRIG A0, ECHO A1 |
| Left ultrasonic | TRIG A2, ECHO A3 |
| Right ultrasonic | TRIG A4, ECHO A5 |
| Pan servo signal | D10 |

Notes:
- The Pi connects to the Arduino by one USB cable (serial data and the Arduino's 5 V). The camera uses the CSI ribbon.
- All grounds (Arduino, L298N, battery pack, servo supply) must be common.
- **Servo supply is an open item.** A servo can draw current spikes that exceed what a USB-powered Arduino 5 V pin should supply and can reset the Arduino. A separate regulated 5 V supply for the servo, with ground tied to the Arduino ground, is recommended.

## Setup

### 1. Arduino

Open `firmware/serbot.ino` in the Arduino IDE (it uses the built-in `Servo` library), select Arduino Uno and upload.

### 2. Raspberry Pi

```bash
sudo apt install python3-picamera2 python3-opencv cmake
pip install --break-system-packages face_recognition pyserial numpy
```

Building `dlib` (a dependency of `face_recognition`) on the Pi takes a long time.

Enrol people by adding photos, 5 to 10 each with varied angle and lighting:

```
pi/known_faces/Alice/1.jpg
pi/known_faces/Alice/2.jpg
pi/known_faces/Bob/1.jpg
```

Optional silent alert endpoint:

```bash
export ALERT_WEBHOOK_URL="https://your-endpoint.example/alert"
```

Run:

```bash
cd pi
python3 serbot_pi.py
```

If the serial port is not `/dev/ttyACM0` (for example CH340 clones appear as `/dev/ttyUSB0`), change `PORT` at the top of `serbot_pi.py`.

## Configuration

Main values at the top of `serbot_pi.py` and `serbot.ino`:

| Setting | Value | File |
|---|---|---|
| Face match tolerance | 0.50 (library default 0.6) | `serbot_pi.py` |
| Frames to declare intrusion | 5 | `serbot_pi.py` |
| Alert cooldown | 30 s | `serbot_pi.py` |
| Resume patrol after no face | 3 s | `serbot_pi.py` |
| Pan gain / dead-band | 0.04 deg/px, 20 px | `serbot_pi.py` |
| Stop distance (front) / veer distance (sides) | 25 cm / 15 cm | `serbot.ino` |
| Link timeout | 2000 ms | `serbot.ino` |
| Drive / turn PWM | 170 | `serbot.ino` |

The tolerance and thresholds are starting values to be tuned during testing.

## Simulation and tests (no hardware needed)

```bash
pip install numpy matplotlib
python3 sim/serbot_sim.py     # writes sim_results.json
python3 sim/test_alert.py     # runs the real notify()/EventStore code with mocks
```

## Status

- Firmware and Pi program are written but **not yet run on the physical rover**.
- Results so far come from the simulation, which uses assumed physical parameters, not measurements. Example: wall ahead avoided in 199 of 200 runs; pan centring 1.22 s at 5 processed fps; fail-safe stop 1.51 to 2.13 s.
- Planned: hardware trials of recognition accuracy, Pi frame rate and field tests.

## Limitations and future work

- Three ultrasonic sensors give weak performance in narrow corridors (65% success in simulation) and on angled hard surfaces.
- HOG detection is best for frontal faces; recognition is not protected against photo or video spoofing.
- Future: tilt axis, wheel encoders with backtracking, LiDAR mapping, hardware acceleration, anti-spoofing.

## Team (Team 6)

Aarush Choudhary (RA2411026011128), Vatsal Kumar (RA2411026011111), Rennie Emmanuel (RA2411026011126), Amaira Malik (RA2411026011127), Devansh Varma (RA2411026011147).
Guide: Dr. Asha Abraham.
