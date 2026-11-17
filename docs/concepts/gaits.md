# Gaits & modes

## Stance and swing

Each leg alternates between two phases:

- **Stance:** the foot is on the ground and pushes the body along.
- **Swing:** the foot lifts and moves to its next foothold.

A **gait** decides which legs swing together. A **mode** decides how stick
input becomes movement.

## Gaits

Each gait swings a group of legs at a time, while the others hold the body
up. Fewer legs swinging at once is slower but more stable.

| Gait | |
|---|---|
| **Tripod** | Alternating triangles of three legs, such as legs 1, 3 and 5, then 2, 4 and 6. Fastest: three legs are always on the ground. |
| **Ripple** | Legs move in overlapping waves, two at a time. |
| **Wave** | One leg at a time. Slowest and most stable. |

See [Legs & joints](legs.md) for the leg numbers. A gait change is applied
only at a safe point, rather than mid-step.

These are the Explorer's and the Lite's gaits. In gait core, each robot lists
its own gaits as data: how long each foot is down (the **duty factor**) and
when in the cycle each leg lifts. So the controllers cycle through whatever
gaits the robot has. Every gait is checked before the robot moves, so the
feet that stay down always hold the body up; all three of these do with
centimetres to spare. See [Describing a robot](../gait-core/robots.md#gaits)
and [Stability & body sway](../gait-core/stability.md).

## Modes

| Mode | What it does |
|---|---|
| **Normal** | Walks forwards and backwards and turns. |
| **Strafe** | Walks in any direction and turns. |
| **Tilt** | Stands still with the feet planted and moves the body: shift, roll, pitch, yaw and height. |

## Support by stack

Every stack runs the same gait from [gait core](../gait-core/index.md), so
what differs is mostly down to the hardware.

| | ROS 2 (Explorer) | ESP32 (Explorer) | Servo2040 (Lite) |
|---|---|---|---|
| Tripod, ripple, wave | ✅ | ✅ | ✅ |
| Normal, strafe, tilt | ✅ | ✅ | ✅ |
| Steering by tilting the controller (LB) | ✅ | Not yet: no controller sends it | Not yet: no controller sends it |
| Leveling (IMU) | ✅ | ✅ | ✅ |
| Ground contact | From servo load | From servo load | Not yet |
| Load balancing, terrain adaptation | ✅ | ✅ | — (no servo feedback) |
| Mapping and navigation | ✅ | — | — |
| Servo power off until standing, and on E-stop | ✅ | ✅ | Power down only (stops the PWM) |
| Battery warning and auto sit | ✅ | ✅ | Off until set up |
