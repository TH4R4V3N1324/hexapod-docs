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

    The software that makes the robot walk: ROS 2 on a Raspberry Pi, and
    training in simulation.

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
| [hexapod-training](https://github.com/TH4R4V3N1324/hexapod-training) | Isaac Lab tasks for training the robot's skills in simulation. Experimental. |
