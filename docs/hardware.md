# Hardware & BOM

!!! info "Planned: not yet in the main repo"
    The main [Mimic](https://github.com/umersanii/Mimic) repo is simulation-only for now. This
    page documents the hardware plan; the `hardware/` directory (BOM, wiring notes) will be added
    to that repo once the physical hand is actually built.

Status: **not yet built**. The parts list below is settled, but a few decisions are still open
(marked below).

## Parts list

| Part | Notes |
|---|---|
| 3D-printed hand + forearm shell | Tendon-driven fingers (fishing line / braided string through printed channels), articulated finger segments |
| 5x micro servos (e.g. SG90 / MG90S) | One per finger (thumb, index, middle, ring, pinky) for tendon pull |
| Arduino Nano / Uno | Runs `firmware/hand_controller`, receives angles over USB serial from the PC |
| USB cable (to PC) | Serial link carrying finger-angle commands from `vision/hand_tracker.py` |
| Elastic cord or small return springs | One per finger, for passive finger extension (servo pulls tendon to curl; elastic returns it to open) |
| Fishing line / braided tendon string | Servo horn → fingertip, routed through printed guide channels |
| 5V power supply (separate from USB) | Servos under load can brown out the Arduino if powered from USB alone |
| Screws / bearings (as needed per STL) | For joint pivots depending on the chosen hand model |

## Why tendon-driven, not a servo per joint

A biologically-accurate hand would need 3 actuators per finger (one per joint). This build uses
**1 servo per finger** instead. The tendon runs through all the joints of a finger at once, so
pulling it curls the whole finger together, the same simplification most beginner-friendly
tendon-hand builds make. It trades individual joint control (you can't curl just the fingertip) for
a much simpler build: 5 motors total instead of 15. The [simulation model](simulation.md) mirrors
this exact tradeoff. See "collapsed to 1 actuated DOF per finger" there.

## Why a separate 5V supply, not USB power

Servos draw current in spikes when moving under load, especially several at once. USB ports are
current-limited (typically ~500mA-1A), and drawing more than that can brown out the Arduino
(voltage sags enough to reset or glitch the microcontroller) mid-motion. A dedicated 5V supply with
its ground tied to the Arduino's ground avoids this; see [Wiring summary](#wiring-summary).

## Open decisions

These aren't blockers, just choices that depend on your specific build:

- **Which hand STL model to 3D print.** Not chosen yet for the physical build. (The
  [simulation](simulation.md) uses an InMoov-derived model as a stand-in for physics testing; that
  doesn't commit you to printing that same model.) Options people commonly use: the
  [InMoov](http://www.inmoov.fr/) hand/forearm, or a fully custom design.
- **Confirm actual servo current draw under load.** If servos brown out on USB power alone even with
  light loads, that's the signal to add the dedicated 5V/2A+ supply. Don't assume you need it
  without testing your specific servos/load first.
- **Wrist rotation.** The reference build (and this project so far) only covers finger flexion. A
  6th servo/joint could be added later for wrist rotation if wanted; no design work has started on
  this.

## Wiring summary

- Servo signal pins → Arduino pins **3, 5, 6, 9, 10** (PWM-capable); must match `SERVO_PINS` in
  [`hand_controller.ino`](firmware.md#pin-mapping).
- All servo grounds tied together **and** to Arduino GND (a shared ground reference is required for
  the PWM signal to be read correctly, even though power comes from a separate supply).
- Servo V+ comes from the dedicated 5V supply, **not** the Arduino's own 5V pin. The Arduino's
  onboard regulator isn't rated for 5 servos' worth of current.
