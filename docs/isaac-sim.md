# Isaac Sim Notes

!!! note "Unrelated to the hand project"
    This page documents a separate track: running **NVIDIA Isaac Sim** headlessly on the project
    maintainer's specific machine (Arch Linux, RTX 3070 Ti). The robotic hand is **not** simulated
    in Isaac Sim; that work moved to [Gazebo](simulation.md). This runbook is kept for reference
    (or for other future robotics work on the same machine), not because the hand needs it.

The full runbook lives in the main repo: `isaac-sim-setup.md`.

## Why it's documented at all

Getting Isaac Sim running reliably on this hardware/driver/kernel combination took real
troubleshooting, and the working configuration is narrow enough that it's worth writing down
rather than re-discovering:

- **Working NVIDIA driver range: R570-R580** (`nvidia-open-dkms` 580.119.02 confirmed working).
  Driver 610+ segfaults inside Isaac Sim's bundled RTX libraries due to an ABI mismatch. Driver
  565.x fails to build against kernel ≥6.12 (missing kernel symbols).
- **Isaac Sim 4.5.0 doesn't work** here (needs driver ≤565, which conflicts with the kernel
  constraint above). **5.1.0-rc.19 segfaults** (unstable release candidate). Only **5.0.0** is
  confirmed stable.
- The kernel and NVIDIA driver packages are pinned (`IgnorePkg` in `/etc/pacman.conf`) to stop
  `pacman` from silently upgrading past the working versions.
- Docker's default runtime must be `nvidia` (in `/etc/docker/daemon.json`) for
  `--runtime=nvidia` container GPU access.
- `--network=host` is required for the WebRTC streaming client to reach port 49100; the media
  stream itself uses UDP port 47998.
- Only one WebRTC streaming client can connect at a time; NVENC (present on RTX, absent on
  datacenter GPUs like the A100) is required for streaming.

## Common commands

Run headless with WebRTC streaming:

```fish
docker run --rm --runtime=nvidia --network=host -e ACCEPT_EULA=Y -e PRIVACY_CONSENT=Y \
  nvcr.io/nvidia/isaac-sim:5.0.0 ./runheadless.native.sh
```

Run a standalone script, no streaming viewer:

```fish
docker run --rm --runtime=nvidia -e ACCEPT_EULA=Y nvcr.io/nvidia/isaac-sim:5.0.0 \
  ./python.sh /isaac-sim/standalone_examples/api/omni.isaac.core/hello_world.py
```

ROS2 Humble ships bundled inside the `isaac-sim:5.0.0` image, so no host ROS2 install is needed for
this track (contrast with the [Gazebo track](simulation.md), which uses ROS2 Jazzy on the host/in
its own container).

See `isaac-sim-setup.md` in the main repo for the complete setup steps.
