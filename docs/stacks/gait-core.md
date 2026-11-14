# Gait core

[hexapod-gait-core](https://github.com/TH4R4V3N1324/hexapod-gait-core) is the walking brain shared by every control stack:
the gait, the behaviours that react to the robot's sensors, and what every
stack has to agree on to be driven the same way: the controller packets and
the gamepad layout. It's plain
C++17 with no ROS, Arduino or Pico SDK dependency, so the same code runs on a
Raspberry Pi and on a microcontroller.

| Stack | Uses gait core for |
|---|---|
| [ROS 2](ros2/index.md) | The `gait_controller` node, the battery decisions in `behaviour_manager`, and the gamepad layout in `controller_teleop` |
| [ESP32](esp32.md) | The whole gait and battery behaviour, and the controller packets |
| [Servo2040](servo2040.md) | The whole gait and battery behaviour, and the controller packets |
| [Controllers](../controllers.md) | The controller packets, and the receiver's gamepad layout |

A change in gait core reaches every stack, so the robots walk, react and
handle a gamepad the same way.

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
- **The protocol and the gamepad layout,** two small header-only libraries
  with no dependencies at all, not even Eigen, so controller firmware can use
  them too.

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

### Protocol

`gait_core::protocol` (`hexapod_protocol/packets.h`) defines the packets
between a controller or receiver and a microcontroller robot: ESP-NOW for
the ESP32 stack, I²C for the Servo2040 stack. It's valid C as well as C++,
since some controller firmware is C.

- **Controller to robot**, every 10 ms: the sticks, tilt yaw and height, a
  body height and a command.
- **Robot to controller**, in reply: its height, gait and mode, whether it's
  seated, whether the servos are powered, which features are on, and
  calibration offsets when asked for.

Both start with a **protocol version** (currently 2), and a robot or
controller drops packets of any other version. See
[Controllers](../controllers.md#how-it-fits-together).

### Teleop

`gait_core::teleop` (`hexapod_teleop/gamepad_teleop.hpp`) is the
[gamepad layout](../controllers.md#gamepad-controls), as code. Each stack
feeds it the gamepad's state and whether the robot is seated or in tilt
mode, and gets back velocities, a tilt pose and events: next mode or gait,
feature toggles, sit or stand, power down and height steps. ROS 2's
`controller_teleop` and the controller receiver both run it, so a gamepad
drives every stack the same way.

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
`gait_core::gait_core`, the behaviours are `gait_core::behaviour`, and the
protocol and gamepad layout are `gait_core::protocol` and `gait_core::teleop`.
A controller that only needs those two can just add `include/` to its
include path. See the
[README](https://github.com/TH4R4V3N1324/hexapod-gait-core#linking).
