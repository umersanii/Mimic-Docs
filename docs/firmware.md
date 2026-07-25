# Firmware & Serial Protocol

!!! info "Planned — not yet in the main repo"
    The main [Mimic](https://github.com/umersanii/Mimic) repo is simulation-only for now. This
    page documents the firmware design; the `firmware/` directory will be added to that repo once
    the physical hand is actually built and wired up.

## What it does, for non-programmers

The Arduino's only job is: listen on the USB cable for a line of text with 5 numbers on it, and
move 5 motors to match those numbers. It doesn't know anything about webcams, hands, or
MediaPipe — it just reads numbers and moves motors. That simplicity is deliberate: it makes the
firmware trivial to test on its own (type numbers into a serial terminal) independent of the vision
side.

## The protocol

One line per frame, comma-separated, newline-terminated:

```
<thumb>,<index>,<middle>,<ring>,<pinky>\n
```

Each value is a servo angle, **0-180**, sent as plain ASCII digits (not binary). Example:

```
120,45,50,48,140
```

- Baud rate: **115200** (must match `--baud` on the Python side — see [Getting Started](getting-started.md#command-line-flags)).
- No handshake, no acknowledgement — it's fire-and-forget, one-way, PC → Arduino, every frame
  (~30/sec while the vision pipeline runs). If a line is malformed or short, `hand_controller.ino`
  just uses whatever it parsed (see [Parsing details](#parsing-details) below) rather than rejecting
  the whole line.

## Pin mapping

```cpp
const int SERVO_PINS[NUM_FINGERS] = {3, 5, 6, 9, 10};  // thumb, index, middle, ring, pinky
```

All 5 pins are PWM-capable Arduino pins, required since `Servo.write()` drives motors via PWM
pulses. This must match the physical wiring — see [Hardware & BOM](hardware.md#wiring-summary).

## Startup behavior

On boot, every servo is immediately written to `180` (fully open) before the main loop starts —
so the hand always powers up in a known, fully-extended position rather than wherever it last was
left, or a random position from an uninitialized PWM signal.

## Parsing details

The sketch reads the serial buffer one character at a time, accumulating into a `String` until it
sees `\n`, then parses that whole line:

```cpp
void applyLine(const String &data) {
  int angles[NUM_FINGERS];
  int start = 0;
  for (int i = 0; i < NUM_FINGERS; i++) {
    int comma = data.indexOf(',', start);
    String token = (comma == -1) ? data.substring(start) : data.substring(start, comma);
    angles[i] = constrain(token.toInt(), 0, 180);
    if (comma == -1) break;
    start = comma + 1;
  }
  for (int i = 0; i < NUM_FINGERS; i++) {
    fingers[i].write(angles[i]);
  }
}
```

Two things worth calling out:

- **`constrain(..., 0, 180)`** clamps every value into the valid servo range, so a bug upstream
  (e.g. a bad angle calculation producing a negative or >180 number) can't send a servo a command
  outside its mechanical range.
- **A short line leaves trailing servos at garbage values.** `angles[]` is a plain local array, not
  zero-initialized — if a line has fewer than 5 comma-separated values, the loop `break`s early and
  the untouched slots hold whatever was already on the stack (not necessarily `0`). In practice the
  Python side always sends exactly 5 values, so this hasn't been an issue, but it's worth knowing if
  you're hand-typing test lines into a serial terminal and send a short one.

## Dependencies

Just the Arduino `Servo` library — no external packages, no board-specific code. Should run on any
board with at least 5 PWM pins (Nano, Uno, etc. — see [Hardware & BOM](hardware.md)).
