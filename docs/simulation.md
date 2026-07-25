# Simulation (Gazebo)

The simulation lets the vision pipeline drive a physically accurate 3D hand model **before** the
real hardware exists — useful for testing tracking, tuning, and the control pipeline without
waiting on a 3D printer or wiring.

Stack: **Gazebo Sim (Harmonic) + ROS2 Jazzy**, run in Docker. This is a separate track from the
`isaac-sim-setup.md` runbook in the repo root — see [Isaac Sim Notes](isaac-sim.md) for why that
exists and why it's unrelated to this hand project.

## Setup

```fish
cd sim/docker
docker build -t robotics-gazebo .
./run.sh
```

`run.sh` mounts the whole `sim/` directory to `/sim` in the container (not just `worlds/`, since
mesh files live under `sim/models/`), sets `GZ_SIM_RESOURCE_PATH=/sim/models`, and names the
container `robotics_gazebo_sim` — a stable name that other tooling (like the vision bridge) targets
with `docker exec`.

## World files

- **`sim/worlds/box_world.sdf`** — a minimal learning rig (pendulum, hinge, box) for getting a feel
  for Gazebo joints and physics before touching the hand model. Uses the `dartsim` physics engine.
- **`sim/worlds/hand_world.sdf`** + **`sim/models/hand/`** — the actual hand model. Uses
  `bullet-featherstone`, not `dartsim` — see [why](#the-mimic-joint-physics-engine-gotcha) below.

## The hand model's provenance

`sim/models/hand/hand.urdf` (and its converted `.sdf`) came from the left-hand subtree
(`i01.leftHand.*` links/joints) of the whole-body URDF in
[Sentience-Robotics/inmoov_ros_sim](https://github.com/Sentience-Robotics/inmoov_ros_sim), not
written from scratch. It needed real adaptation before it was usable here, not just extraction:

1. **Rescaled ~10x.** The source meshes/joint origins were sized as if the palm were ~88cm — measured
   directly off the STL bounding boxes (~0.79×0.88×1.09 in source units). All mesh `scale` attributes
   and origin translations were multiplied by 0.1 to bring the palm to a real ~9-11cm hand size.
   Rotations were left untouched.
2. **Collapsed to 1 actuated joint per finger**, matching the real hardware's 1-servo-per-finger
   tendon design (see [Hardware & BOM](hardware.md#why-tendon-driven-not-a-servo-per-joint)). Each
   finger's proximal joint (`index_link_joint`, `majeure_link_joint`, `pinky_link_joint`,
   `ringFinger_link_joint`, `thumb1_link_joint`) is the *driver*; the other 2 joints per finger
   `<mimic>` it 1:1. The pinky/ring-finger abduction joints were converted to `type="fixed"` — no
   spread motion, since the real hardware doesn't have it either.
3. **Added collision and inertial data.** The source URDF had `<visual>` meshes only — no
   `<collision>` or `<inertial>` at all, so nothing would have had contact physics or dynamics
   without adding them. Collision geometry is box primitives sized from the STL bounding boxes;
   inertial values are box-formula inertia with hand-estimated masses (~5-90g per link).

The scripts used for this (extraction, scaling, mimic-joint insertion, collision generation) were
one-off Python/`ElementTree` scripts, not checked into the repo. If the model ever needs
regenerating — say, to also pull the right hand, or retune masses — redo it from the source repo
rather than assuming a saved script exists somewhere.

## The mimic-joint physics engine gotcha

This is the single most important thing to know if you're extending this simulation:

!!! warning "`dartsim` silently ignores `<mimic>` joints"
    `box_world.sdf` uses `dartsim`, and that's fine for a pendulum/hinge/box learning rig. But this
    Gazebo build's `dartsim` **silently ignores `<mimic>` constraints** — it logs "physics engine
    does not support mimic constraints" and just leaves the joint free/unconstrained. Nothing
    crashes; the mimic'd joints just... don't mimic.

    `bullet-featherstone` was verified to apply mimic correctly (tested: commanding one driver
    joint moved all 3 mimic-coupled finger joints together, as intended).

    **Any new world file relying on mimic joints must set `type="bullet-featherstone"`** and add

    ```xml
    <engine><filename>gz-physics-bullet-featherstone-plugin</filename></engine>
    ```

    inside the `Physics` system plugin block. Don't copy `box_world.sdf`'s `dartsim` config for
    hand/mimic-joint work without changing this — it'll "run" with no errors and just be wrong.

## The vision → Gazebo bridge

Code: [`sim/bridge/gz_hand_bridge.py`](https://github.com/umersanii/Mimic/blob/main/sim/bridge/gz_hand_bridge.py)

This lets `vision/hand_tracker.py` (running on the host, in the `robohand` conda env) drive the
simulated hand in real time — in parallel with, or instead of, the real Arduino
(`--gazebo` flag; `--no-serial` to skip real hardware entirely).

### Why a separate process, not a library call

`gz.transport13`/`gz.msgs10` — the Python bindings for talking to Gazebo — only exist inside the
`robotics-gazebo` Docker image. The host's `robohand` conda env has no path to them (confirmed via
`apt list --installed` inside the container). So `hand_tracker.py` launches the bridge with:

```python
docker exec -i robotics_gazebo_sim python3 /sim/bridge/gz_hand_bridge.py
```

and streams one CSV line of 5 curl values (`thumb,index,middle,ring,pinky`, `0.0`=straight to
`1.0`=curled — the same convention `hand_tracker.py`'s own `curls` dict uses) per frame, over that
one persistent pipe. Not a subprocess call per frame — one process, one long-lived stdin pipe, for
the whole session.

### Two gotchas worth knowing before extending the bridge

**1. Topic names use the joint's last dot-segment, not the full URDF name.**
`hand_world.sdf`'s `JointPositionController` plugins subscribe to
`/inmoov_left_hand/index_link_joint/cmd_pos` — *not*
`/inmoov_left_hand/i01.leftHand.index_link_joint/cmd_pos`. The world-generation process stripped the
`i01.leftHand.` prefix when wiring up controller topics. The bridge does the same
(`joint_name.split('.')[-1]`) — if you add a new joint and forget this, publishes go to a topic
nobody's listening on and nothing happens, with no error.

**2. A one-shot publish right after `advertise()` can be dropped.**
Gazebo Transport discovery (multicast, finding existing subscribers) takes on the order of ~1
second. Publish once immediately after advertising and tear the process down right after, and
you can race past that discovery window with nothing delivered. This isn't an issue for the real
use case (continuous per-frame streaming for a whole session — discovery settles well within the
first second, and everything after lands normally), but it'll bite you if you write a quick one-shot
test script against the bridge.

## Known issue: the hand appears to jitter (it's rendering, not physics)

If you look closely at the simulated hand, even completely at rest with zero commands being sent,
it visibly jitters/shimmers.

This was diagnosed by subscribing directly to every joint's `axis1.position` at the sim's own
~1kHz update rate for 3 seconds with nothing commanded: every joint sat flat to within ~1e-8 rad.
Physics/joint state is provably static — the jitter is a **rendering artifact**, not a control or
physics instability.

Best-fit explanation (not filed upstream as a bug — a real Gazebo/gz-sim limitation, worked around
locally rather than fixed): `hand_world.sdf`'s sun light has `<cast_shadows>false</cast_shadows>`
set deliberately, because the default directional-light shadow-map resolution/depth-bias is tuned
for room/building-scale scenes (meters). This hand is ~10cm with many small phalanx meshes packed
closely together — at that scale, the default shadow map produces shadow acne (flickering
self-shadow noise) that reads visually as jitter.

If you ever turn shadows back on (e.g. for a nicer screenshot or demo), expect the jitter to
return — either re-disable them, or hand-tune the shadow map resolution/near-far clip/bias for
cm-scale geometry.
