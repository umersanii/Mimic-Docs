# Glossary

Terms used throughout these docs, explained plainly first, with the technical detail after.

**Tendon-driven**
: A finger design where a motor pulls a string (the "tendon") to curl the finger, instead of a
motor sitting inside each joint. Mirrors how your own fingers work — the muscles are in your
forearm, tendons carry the pull down into the finger.

**MCP / PIP / DIP joint**
: Anatomical names for finger joints, used loosely in the code/docs for the tracked landmark
points. MCP = knuckle (where finger meets palm), PIP = middle joint, DIP = joint nearest the tip.
The thumb only has 2 of these (no true PIP).

**Landmark**
: One tracked (x, y, z) point on the hand, as reported by MediaPipe. 21 per hand — wrist plus 4
per finger.

**MediaPipe**
: Google's machine-learning library used here for hand tracking. Specifically the newer **Tasks
API** (`HandLandmarker`), not the older `mp.solutions.hands` API removed in recent versions — see
[Vision Pipeline](vision.md#the-mediapipe-model).

**EMA (exponential moving average)**
: A simple smoothing technique: each new value is blended with the previous smoothed value by a
weight, rather than used raw. Used here to remove frame-to-frame jitter from tracked finger angles
before sending them anywhere — see [Vision Pipeline](vision.md#smoothing).

**Serial (protocol)**
: A simple one-way stream of text over a USB cable, here carrying one line of 5 numbers per video
frame from the PC to the Arduino. No acknowledgement, no handshake — see
[Firmware & Serial Protocol](firmware.md#the-protocol).

**Servo (motor)**
: A small motor that can be commanded to hold a specific angular position (0-180°), rather than
just spinning at a speed. Used here — one per finger — to pull the tendon a controlled amount.

**Curl fraction**
: A 0.0-1.0 number representing how curled a finger is (0=straight, 1=fully curled in the
simulation's convention), derived from the tracked joint angle. See
[Vision Pipeline](vision.md#from-angle-to-output-value).

**URDF / SDF**
: Text file formats that describe a robot's physical structure — links (rigid parts), joints
(how they connect and move), visual meshes, collision shapes, and mass/inertia. URDF is ROS's
format; SDF is Gazebo's native format (this project converts URDF → SDF for the hand model).

**Mimic joint**
: A joint constraint that forces one joint to copy another joint's motion (optionally scaled),
used here so one motor's commanded position drives all 3 joints of a simulated finger together,
matching the real hardware's 1-motor-per-finger design. See
[Simulation: the mimic-joint gotcha](simulation.md#the-mimic-joint-physics-engine-gotcha).

**Physics engine (dartsim / bullet-featherstone)**
: The underlying simulator that computes how forces, joints, and collisions actually move objects
in Gazebo. Gazebo supports several interchangeable engines — this project uses two different ones
for different worlds, because one of them doesn't support a feature (mimic joints) the hand model
needs. See [Simulation](simulation.md#the-mimic-joint-physics-engine-gotcha).

**Gazebo Transport**
: Gazebo's internal publish/subscribe messaging system (topics, like ROS topics) used to send
joint commands into a running simulation from an external script.

**ROS2 (Robot Operating System 2)**
: A widely-used robotics middleware/framework. This project uses ROS2 Jazzy alongside Gazebo
Harmonic, though the hand's own control pipeline (vision → serial/Gazebo bridge) doesn't route
through ROS2 topics directly.
