# Explorer

The **Explorer** is the full-featured hexapod: 18 ST3215 serial bus servos, a
Raspberry Pi 5, a lidar, a depth camera and an IMU, running the
[ROS 2 stack](../../stacks/ros2/index.md). It walks over uneven ground, keeps
its body level on slopes, and can map a room and navigate it.

For a simpler build, the [ESP32 stack](../../stacks/esp32.md) drives the
same servos from an ESP32 instead of the Pi. It still feels the ground, but
has no lidar or cameras, so it can't map.

!!! warning "v0.1: in development"
    The Explorer isn't ready to build yet. The parts, bill of materials and
    instructions are incomplete and may change. (v0.1 is the first public
    release of the internal V6.1 design.)

## What's in it

- **6 legs, 3 joints each,** each joint an ST3215 serial bus servo with
  position and load feedback.
- **Raspberry Pi 5** as the only computer, with two custom **leg hub** boards
  handling power and servo connections.
- **Sensors:** an RPLIDAR A2M12 lidar, an Arducam ToF depth camera, a
  Raspberry Pi Camera Module 3, an ISM330DHCX IMU and two INA226 power
  monitors.
- **Sprung tibias** with 1/10-scale RC shocks and TPU feet.
- **A 3S Li-ion pack** of 21700 cells.

## Building it

1. [Parts & tools](parts.md)
2. [Printing](printing.md)
3. [Legs](legs.md)
4. [Body](body.md)
5. [Electronics](electronics.md) and [Wiring](wiring.md)
6. [Set up the ROS 2 stack](../../stacks/ros2/setup.md)

The design files are in
[hexapod-hardware/explorer](https://github.com/TH4R4V3N1324/hexapod-hardware/tree/main/explorer).
