# Getting Started

You don't need the physical hand built to try the software — the vision pipeline runs standalone
against your webcam, and there's also a simulated hand in Gazebo if you want to see it drive a
3D model instead of real servos.

## Prerequisites

- A webcam.
- [Conda](https://docs.conda.io/) (Miniconda is enough) — used to create an isolated Python 3.11
  environment.

!!! note "Why Python 3.11, not whatever you have installed"
    `mediapipe` (the hand-tracking library) doesn't ship a build for Python 3.13. If your system
    Python is newer than 3.11, a plain `pip install` will fail — that's why this project uses a
    dedicated conda environment rather than a venv off system Python.

## 1. Set up the environment

```fish
conda create -y -n robohand python=3.11
conda activate robohand
cd vision
pip install -r requirements.txt
```

## 2. Get the hand-tracking model file

The hand-tracking library needs a model file that isn't bundled in the pip package:

```fish
mkdir -p vision/models
curl -sL -o vision/models/hand_landmarker.task \
  "https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task"
```

## 3. Run it — vision only, no hardware

```fish
python3 hand_tracker.py --no-serial
```

A window opens showing your webcam feed with a hand skeleton overlay, a live finger count, and
FPS — this is the same pipeline that will eventually drive the real servos, just without sending
anything anywhere yet.

Press `q` to quit.

## 4. Once the hand hardware is wired up

Flash `firmware/hand_controller/hand_controller.ino` to the Arduino (Arduino IDE or `arduino-cli`),
plug it in, then:

```fish
conda activate robohand
python3 hand_tracker.py --port /dev/ttyUSB0
```

Replace `/dev/ttyUSB0` with whatever port your Arduino shows up as.

## 5. Optional: drive the simulated hand instead of (or alongside) real hardware

If you've set up the [Gazebo simulation](simulation.md), you can send the same tracked hand
movements to a simulated 3D hand model:

```fish
python3 hand_tracker.py --no-serial --gazebo
```

Add `--gazebo` to any of the above commands to also stream to the sim; drop `--no-serial` if you
want vision, real hardware, *and* the simulation all running from one webcam feed at once.

## Command-line flags

| Flag | Default | What it does |
|---|---|---|
| `--port` | `/dev/ttyUSB0` | Serial port for the Arduino |
| `--baud` | `115200` | Serial baud rate (must match the Arduino sketch) |
| `--camera` | `0` | Webcam device index |
| `--no-serial` | off | Skip sending to the Arduino entirely |
| `--gazebo` | off | Also stream to the Gazebo simulation |
| `--gazebo-container` | `robotics_gazebo_sim` | Name of the running sim Docker container |
| `--smoothing` | `0.3` | How much each new reading affects the tracked pose — see [Vision Pipeline](vision.md#smoothing) |
