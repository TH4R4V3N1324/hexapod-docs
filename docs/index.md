# Hexapod

An open-source family of six-legged walking robots: 3D-printed hardware in
several variants, and the software to make them walk.

<div class="grid cards" markdown>

- **[Choose a variant](getting-started/choose.md)**

    Compare the hardware variants and the control stacks they run.

- **[Explorer](variants/explorer/index.md)**

    The full-featured robot: serial bus servos, a Raspberry Pi 5, lidar and a
    depth camera, running ROS 2.

- **[Control stacks](stacks/ros2/index.md)**

    The software that makes the robots walk: ROS 2 on a Raspberry Pi, or
    firmware on an ESP32 or a Servo 2040, all built on a shared gait core.

- **[Concepts](concepts/legs.md)**

    Leg numbering, coordinate frames, gaits and modes, shared by every
    variant.

</div>

!!! warning "Project status"
    Everything here is in development. No variant is ready to build yet. The
    [Explorer](variants/explorer/index.md) is the furthest along.

## Repositories

| Repository | What's in it |
|---|---|
| [hexapod-hardware](https://github.com/TH4R4V3N1324/hexapod-hardware) | CAD (STEP and STL), bills of materials, print lists and assembly notes for each variant, and the [leg hub](https://github.com/TH4R4V3N1324/hexapod-hardware/tree/main/electronics/leg-hub) PCB. The source of truth for all hardware. |
| [hexapod-ros2](https://github.com/TH4R4V3N1324/hexapod-ros2) | The ROS 2 Jazzy stack for the Explorer, the operator workspace, the Foxglove panel, and the Pi and Steam Deck setup scripts. |
| [hexapod-gait-core](https://github.com/TH4R4V3N1324/hexapod-gait-core) | The shared gait library: kinematics, trajectories, the gait engine and leg odometry, in plain C++17. Used by every control stack. |
| [hexapod-esp32](https://github.com/TH4R4V3N1324/hexapod-esp32) | Firmware for the Explorer's servos on an ESP32: runs the gait closed loop, without the Pi. |
| [hexapod-servo2040](https://github.com/TH4R4V3N1324/hexapod-servo2040) | Firmware for the Lite: runs the gait on a Pimoroni Servo 2040, driving 18 hobby servos. |
| [hexapod-training](https://github.com/TH4R4V3N1324/hexapod-training) | Isaac Lab tasks for training the robot's skills in simulation. Experimental. |
