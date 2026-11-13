# Choose a variant

The hexapod comes in hardware **variants**, each with one or more
**control stacks**: the software that makes it walk.

## Variants

| Variant | Servos | Brain | Status |
|---|---|---|---|
| [**Explorer**](../variants/explorer/index.md) | ST3215 serial bus servos, with position and load feedback | Raspberry Pi 5, with lidar, depth camera and IMU; or an ESP32 and IMU | v0.1, in development |
| [**Lite**](../variants/lite/index.md) | Hobby PWM servos | Pimoroni Servo 2040, with an ESP32 receiver and IMU | In development |
| [**Skeletal**](../variants/skeletal/index.md) | To be decided | To be decided | Planned |

The **Explorer** is the full-featured robot: it maps, navigates and adapts to
rough ground. With the [ESP32 stack](../stacks/esp32.md) instead of the Pi, it
still adapts to rough ground but can't map. The **Lite** is the cheap, simple one: it walks
and levels itself, but can't feel the ground or map. The
**Skeletal** aims to be a lightweight frame.

## Control stacks

| Stack | Runs on | Works with |
|---|---|---|
| [ROS 2](../stacks/ros2/index.md) | Raspberry Pi 5 | Explorer |
| [ESP32](../stacks/esp32.md) | ESP32 dev board | Explorer, without the Pi and its sensors |
| [Servo2040](../stacks/servo2040.md) | Pimoroni Servo 2040 | Lite |
| [Training](../stacks/training.md) | A PC with Isaac Lab | Simulation (experimental) |

All three robot stacks walk with the same gait, from the shared
[gait core](../stacks/gait-core.md) library.

## Which should I build?

There isn't a buildable variant yet: the Explorer is the furthest along, and
its parts list and instructions are still being completed. Follow the
Explorer pages to see where it's heading.
