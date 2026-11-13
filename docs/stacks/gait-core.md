# Gait core

[hexapod-gait-core](https://github.com/TH4R4V3N1324/hexapod-gait-core) is the walking brain shared by every control stack.
It's a plain C++17 library with no ROS, Arduino or Pico SDK dependency, so
the same code runs on a Raspberry Pi and on a microcontroller.

| Stack | Uses gait core for |
|---|---|
| [ROS 2](ros2/index.md) | The `gait_controller` node |
| [Servo2040](servo2040.md) | The whole gait, on the Servo 2040 |

A change to the gait in gait core reaches every stack, and the robots walk
the same way.

## What's in it

- **The gait engine:** startup, walking, sitting and standing, tilt mode,
  leveling, load balancing, terrain adaptation and leg odometry. Each stack
  feeds it joint positions, IMU readings and movement commands, and gets back
  18 joint targets. See [How it walks](ros2/walking.md) for what each feature
  does.
- **Robot geometry:** link lengths, where each leg mounts, the neutral stance,
  maximum stride and joint limits. There's a preset for the Explorer.
- **Kinematics:** inverse and forward kinematics for one leg, and the foot
  force from joint torques.
- **Frames and trajectories:** converting between each leg's frame and the
  body frame, swing curves and stance lines.

## Conventions

- **SI units:** metres, radians and seconds. Each stack converts to and from
  its own units at its edges.
- **Body frame** as in [Coordinate frames](../concepts/frames.md). Each leg's
  frame has +x pointing out along the leg from its coxa joint.
- **Legs are counted from 0** in code, clockwise from the front right. Leg 0
  in code is leg 1 in [Legs & joints](../concepts/legs.md), up to leg 5 in
  code, which is leg 6.

## Versions

Releases are tagged `vMAJOR.MINOR.PATCH`. The minor version changes when the
API changes, and the patch version for fixes that keep the API.

Each stack includes gait core as a git submodule **pinned to a release tag**,
so a new release only reaches a stack once that stack is updated to it.
Clone stacks with their submodules:

```bash
git submodule update --init --recursive
```

## Building and testing

The library has its own test suite: kinematics, foot directions, gait
schedules, leg clearance, odometry and more.

```bash
git clone --recursive https://github.com/TH4R4V3N1324/hexapod-gait-core.git
cd hexapod-gait-core
cmake -S . -B build
cmake --build build --config Release
ctest --test-dir build -C Release --output-on-failure
```

By default, numbers are single-precision `float`, since the
microcontrollers have no double-precision hardware. The ROS 2 stack builds it
with `-DGAIT_CORE_USE_DOUBLE=ON` instead. CI tests both, and cross-compiles
for the RP2040's Cortex-M0+.

It can be linked as a CMake subproject, an installed CMake package (which is
how colcon builds it for ROS 2), or a PlatformIO library. See the
[README](https://github.com/TH4R4V3N1324/hexapod-gait-core#linking).
