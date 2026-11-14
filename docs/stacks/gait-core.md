# Gait core

[hexapod-gait-core](https://github.com/TH4R4V3N1324/hexapod-gait-core) is the walking brain shared by every control stack:
the gait, and the behaviours that react to the robot's sensors. It's plain
C++17 with no ROS, Arduino or Pico SDK dependency, so the same code runs on a
Raspberry Pi and on a microcontroller.

| Stack | Uses gait core for |
|---|---|
| [ROS 2](ros2/index.md) | The `gait_controller` node, and the battery decisions in `behaviour_manager` |
| [ESP32](esp32.md) | The whole gait and battery behaviour, on the ESP32 |
| [Servo2040](servo2040.md) | The whole gait and battery behaviour, on the Servo 2040 |

A change in gait core reaches every stack, so the robots walk and react the
same way.

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
- **Behaviours,** in a separate library, `gait_core::behaviour`. Each one
  takes sensor readings and decides what to do; the stack carries it out
  with its own hardware. The gait doesn't depend on them, so a robot can use
  the gait alone.

### Servo power

The gait engine also says whether the servos **should be powered**, for a
stack with a servo power switch, such as the leg hubs' `SERVO_EN`, to follow:

- With `power_on_startup` off, the servos stay **off until the first stand
  command**. Standing powers them, waits for them to answer, moves the legs
  from wherever they are to the seated pose, then stands up.
- **Power down** sits the robot, then turns the servos off once it's seated.
  A stand before then cancels it.
- If the servos **stop answering** for half a second while running, for
  example because the E-stop was pressed, the engine turns servo power off
  and goes back to its boot state. **Releasing the E-stop never moves the
  robot by itself**: it waits for a stand command.

With `power_on_startup` on (the default, used in simulation), the servos are
powered from the start.

### Battery behaviour

`BatteryMonitor` watches the battery through up to four power monitors. It
takes the lowest voltage, filtered over 5 seconds so the dips while walking
don't count. The defaults are for a 3S pack:

| Battery | What happens |
|---|---|
| Below 10.5 V (3.5 V per cell) | Warning |
| Below 9.9 V (3.3 V per cell) | Sits down, then turns servo power off |

Each level clears 0.2 V above its threshold. A stand command overrides a
critical sit, and the monitor never stands the robot back up by itself. If a
power monitor stops reporting for 2 seconds it's ignored, and the level
holds rather than changing. How a warning is shown, such as a controller
rumble, is up to each stack.

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
how colcon builds it for ROS 2), or a PlatformIO library. The gait is
`gait_core::gait_core` and the behaviours are `gait_core::behaviour`. See the
[README](https://github.com/TH4R4V3N1324/hexapod-gait-core#linking).
