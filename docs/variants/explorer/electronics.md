# Electronics

This page gives an overview of the electronics in the Explorer v0.1 and what
each part does. For the full parts list, see [Parts & tools](parts.md).
For how the parts connect, see [Wiring](wiring.md).

## Overview

```
  3S battery ──XT30──┬───────────────────────┐
                     ▼                       ▼
            left leg hub (0x40)     right leg hub (0x41)
             │             │                 │
             ▼             ▼ 5 V             ▼
     legs 4–6 (9 servos)  Raspberry Pi 5   legs 1–3 (9 servos)
                             │
     UART ─► Bus Servo Adapter ─► BUS_IN on both leg hubs (servo data)
     I²C  ─► left leg hub ─► right leg hub ─► IMU
     USB  ─► lidar
     CSI  ─► ToF camera, Pi camera
```

## Compute

The **Raspberry Pi 5** runs everything on the robot: the gait controller,
the hardware drivers and a Foxglove bridge for the operator. The ROS 2 stack
runs in Docker, started at boot (see [First power-on](../../stacks/ros2/setup.md)).

## Servos

There are **18 ST3215** serial bus servos, three per leg. They're 12 V,
30 kg·cm servos using the Feetech STS protocol at **1 Mbaud**.

The Pi's serial port (`/dev/ttyAMA0` by default) drives a **Waveshare Bus
Servo Adapter (A)**, mounted on the Pi shelf. The adapter's bus goes to
`BUS_IN` on both leg hubs, which share it out to their three legs.

The ROS driver reads each servo's position, speed and load every cycle. The
load reading is used as an estimate of joint torque, which is how the robot
tells which feet are on the ground. There are **no foot contact switches**.

## Leg hubs

The [leg hub](https://github.com/TH4R4V3N1324/hexapod-hardware/tree/main/electronics/leg-hub) is the power and data board for one
side of the robot. There are **two identical boards**, one for the left legs
and one for the right. Each board:

- **feeds power and servo data to three legs**, each through its own 5 A
  blade fuse;
- **makes 5 V at 5 A** for the Pi, sensors and other peripherals;
- **measures its side's current, voltage and power** with an INA226 on I²C;
- **switches servo power** only when the Pi's `SERVO_EN` signal is high
  **and** the E-stop loop is closed, so the servos stay off while the Pi
  boots;
- **switches itself fully on and off** from a small latching switch.

The two boards differ only in the INA226 address, set by jumper JP1: left is
`0x40`, right is `0x41`.

!!! warning "v0.1"
    The leg hub is still under development and hasn't been fully verified.
    Check everything against the
    [schematic](https://github.com/TH4R4V3N1324/hexapod-hardware/tree/main/electronics/leg-hub) before powering it up.

## Sensors

| Sensor | Connection | Used for |
|---|---|---|
| Adafruit **ISM330DHCX** IMU (accelerometer + gyroscope) | I²C, address `0x6A` (STEMMA QT) | Body leveling, odometry |
| 2 × **INA226** (on the leg hubs) | I²C, `0x40` and `0x41` | Power monitoring for each side |
| Slamtec **RPLIDAR A2M12** | USB, appears as `/dev/rplidar` | SLAM, Nav2 |
| **Arducam ToF camera** (B0410) | CSI | Depth for terrain mapping and low obstacles |
| **Raspberry Pi Camera Module 3** | CSI | Front camera image |

### Lidar

The lidar sits centred on the lid, with its scan plane 73.5 mm above the hip
joints. The knees reach about 9 cm above the hips, so the legs cross the scan
plane. The software filters out everything within 0.30 m of the lidar so the
legs don't show up as obstacles.

### Cameras

Both cameras sit in the camera pod facing forward, with the ToF camera below
the Pi camera:

- **Arducam ToF:** 240×180 depth, 70° diagonal field of view (58.5°
  horizontal), up to 4 m in far mode. It's tilted 10° down to see low
  obstacles near the feet. The front legs stay outside its view.
- **Pi Camera Module 3** (standard lens): 66° horizontal field of view.

!!! note
    There's no ROS 2 driver for the ToF camera yet. On the real robot, a node
    publishing a `PointCloud2` on `/front_depth/points` is still needed.

## Power

- **Battery:** a 3S pack (9.0–12.6 V), split to both leg hubs over XT30.
- **Power switch:** one small latching switch turns both leg hubs, and so
  the whole robot, on and off. It's a hard power-off with no soft shutdown.
- **E-stop:** a normally-closed button that cuts servo power on both boards.
  The Pi stays on.
- **Fuses:** 5 A mini blade fuses, three per leg hub.

The power monitors publish an estimated battery percentage from the voltage,
which is shown in the Foxglove panel.
