# Vision Pipeline

Code: [`vision/hand_tracker.py`](https://github.com/umersanii/Mimic/blob/main/vision/hand_tracker.py)

<video src="../assets/img/tracking-demo.mp4" autoplay loop muted playsinline style="width:100%;border-radius:8px;"></video>

## What it does, for non-programmers

The webcam sees a picture. A machine-learning model (Google's MediaPipe) has been trained on huge
numbers of hand photos and can point out where your knuckles and fingertips are in that picture,
21 points per hand, updated ~30 times a second. From those points, some geometry figures out how
bent each finger is, and that bend gets turned into a number sent to the hand.

## The MediaPipe model

This project uses MediaPipe's **Tasks API** (`HandLandmarker`), not the older `mp.solutions.hands`
API you'll see in a lot of older tutorials. `mp.solutions.hands` was removed entirely in the
`mediapipe` version this project pins. The Tasks API needs an explicit model file
(`vision/models/hand_landmarker.task`) rather than something bundled in the pip package; see
[Getting Started](getting-started.md#2-get-the-hand-tracking-model-file) for how to fetch it.

`HandLandmarker` returns 21 landmarks per detected hand. Landmark indices follow MediaPipe's
standard hand topology: index `0` is the wrist, `4` is the thumb tip, `8` the index tip, and so on.

## Turning landmarks into a finger angle

For each finger, three landmarks are picked (roughly "knuckle," "middle joint," and "tip"), and
the angle *at* the middle joint is computed:

```python
FINGER_JOINTS = {
    "thumb": (2, 3, 4),
    "index": (5, 6, 8),
    "middle": (9, 10, 12),
    "ring": (13, 14, 16),
    "pinky": (17, 18, 20),
}
```

The math is the [law of cosines](https://en.wikipedia.org/wiki/Law_of_cosines) applied to the two
line segments meeting at that middle joint (wrist→joint and joint→tip, or thumb-base→joint→tip for
the thumb, since the thumb's geometry is different from the other four fingers). The result is an
angle in degrees:

- **~180°** = finger fully straight (the two segments point in opposite directions from the joint).
- **Smaller angle** = more curled (the tip has swung around toward the wrist).

This is a genuine angle measurement, not a lookup table or a trained classifier. It works for any
hand size or shape because it's pure geometry on the tracked points.

## Smoothing

Raw landmark positions are noisy: hold your hand perfectly still and the reported angle still
wobbles a degree or two, frame to frame. That wobble would make the servos twitch. It's damped
with an [exponential moving average](https://en.wikipedia.org/wiki/Exponential_smoothing):

```python
smoothed_angles[f] = smoothing * new_angle[f] + (1 - smoothing) * smoothed_angles[f]
```

`--smoothing` (default `0.3`) controls the weight on each new reading:

- **Lower** (e.g. `0.1`) → smoother motion, but more lag between your real hand and the robot hand.
- **Higher** (e.g. `0.7`) → snappier response, but more jitter gets through.

The smoothing state is reset whenever the hand tracker loses your hand (goes out of frame, bad
lighting, etc.). Otherwise, on reacquiring your hand, it would blend the new (correct) pose with a
stale old one and lag noticeably before catching up.

## From angle to output value

Two small pure functions turn a smoothed angle into what gets sent downstream:

```python
def curl_fraction(angle, straight=170, curled=40):
    """0.0 = fully curled, 1.0 = fully straight."""
    ...

def angle_to_servo(angle, servo_open=180, servo_closed=0):
    """Map a measured joint angle to a servo command (0-180)."""
    ...
```

`curl_fraction` clamps the raw angle into the `[curled, straight]` degree range you'd actually see
from a human hand, then normalizes it to 0.0-1.0. `angle_to_servo` reuses that fraction to produce
a 0-180 servo command. The [Gazebo simulation](simulation.md) consumes the `curl_fraction` value
directly (`1.0 - curl_fraction`, since the sim's convention is 0=straight/1=curled, the opposite
of the servo's "fraction toward open"); the Arduino consumes the servo command.

## Output

Every frame, if a hand is detected:

- A line like `120,45,50,48,140\n` (servo angles for thumb,index,middle,ring,pinky) is written to
  the serial port, if connected.
- A line of comma-separated curl fractions (0.0-1.0) is streamed to the Gazebo bridge, if `--gazebo`
  is set.

Both are derived from the same smoothed-angle dict. See [Architecture](architecture.md) for why
that matters.
