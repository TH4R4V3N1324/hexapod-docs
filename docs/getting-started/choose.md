# Choose a variant

The hexapod comes in hardware **variants**, each with one or more
**control stacks**: the software that makes it walk.

## Variants

| Variant | Servos | Brain | Status |
|---|---|---|---|
| [**Explorer**](../variants/explorer/index.md) | ST3215 serial bus servos, with position and load feedback | Raspberry Pi 5, with lidar, depth camera and IMU | v0.1, in development |
| [**Lite**](../variants/lite/index.md) | Hobby PWM servos | Microcontroller | Planned |
| [**Skeletal**](../variants/skeletal/index.md) | To be decided | To be decided | Planned |

The **Explorer** is the full-featured robot: it maps, navigates and adapts to
rough ground. The **Lite** aims to be the cheap, simple one. The
**Skeletal** aims to be a lightweight frame.

## Control stacks

| Stack | Runs on | Works with |
|---|---|---|
| [ROS 2](../stacks/ros2/index.md) | Raspberry Pi 5 | Explorer |
| [Training](../stacks/training.md) | A PC with Isaac Lab | Simulation (experimental) |

## Which should I build?

There isn't a buildable variant yet: the Explorer is the furthest along, and
its parts list and instructions are still being completed. Follow the
Explorer pages to see where it's heading.
