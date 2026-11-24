# Gait core

[legged-gait-core](https://github.com/TH4R4V3N1324/legged-gait-core) is the
walking brain shared by every control stack: the gait, the behaviours that
react to the robot's sensors, and what every stack has to agree on to be
driven the same way: the controller packets and the gamepad layout. It's
plain C++17 with no ROS, Arduino or Pico SDK dependency, so the same code runs
on a Raspberry Pi and on a microcontroller.

!!! info "Renamed from hexapod-gait-core"
    Since v0.9.0 the engine isn't specific to hexapods, and the repository is
    now **legged-gait-core**. The old `hexapod-gait-core` URL redirects to it,
    so existing clones and submodules keep working. The stacks still check it
    out at `external/hexapod-gait-core` (or `hexapod_ws/src/hexapod-gait-core`).

## A robot is data

Up to v0.8.0, gait core was a hexapod library: six legs, three joints each,
and the Explorer's numbers built in. From **v0.9.0** a robot is described as
data, and the engine walks whatever it's given:

- **Legs** are chains of revolute joints, each with its own origin, axis and
  limits, so a leg can have anything from two to six joints.
- **The leg count** is a runtime value, up to a build-time maximum (six by
  default).
- **Gaits** belong to the robot, as a duty factor and each leg's offset in
  the cycle, so a hexapod's tripod and a quadruped's crawl are both just data.
- **Stability** is checked against the support polygon of the planted feet.
  A gait that would tip the robot is refused, and a gait that's only stable
  with the body moved over its feet (a quadruped crawl) sways the body before
  each leg lifts.
- **The core has no built-in robot.** The Explorer is one definition, in a
  separate `gait_core::robots` library.

In v0.9.0 the Explorer's numbers were unchanged, and ROS 2's recorded
"golden trace" of its joint commands still matched.

## Since v0.9.0

Releases v0.10.0 to v0.13.0 add to the API rather than change it, and the
protocol is still version 4. They do change how the Explorer walks:

- **Per-joint speed limits** (v0.10.0): each joint can have its own top
  speed. The Explorer's are its servos' real speeds on a 3S pack: 4.36 rad/s
  for the coxa and tibia, 7.28 rad/s for the femur, instead of 4 rad/s for
  every joint.
- **A faster walk** (v0.11.0): the Explorer's stride grows from 0.06 m to
  0.07 m, and the shortest phase from 0.7 s to 0.55 s, so each gait's top
  speed rises, as far as the joint speed limits allow.
- **Masses** (v0.12.0): a robot can give its body's and links' masses, and
  the engine keeps a live centre of mass as the legs move. The Explorer
  gives its CAD masses, 3.32 kg in all.
- **Richer IMU input** (v0.12.0): `ImuSample` carries angular velocity and
  linear acceleration as well as orientation.
- **Dynamic balance** (v0.12.0, v0.13.0): optional, off by default.
  Acceleration and leg lifts limited by the ZMP and capture point, feedback
  that recovers from shoves, preview-planned body sway, feedforward joint
  torques, capture-point foot placement, and an input for the body's
  velocity from an outside estimator. See [Dynamic balance](dynamics.md).

| Page | What it covers |
|---|---|
| [Describing a robot](robots.md) | `RobotDefinition`: legs, joints, solvers, stance, gaits and centre of mass, with the Explorer as the example |
| [Stability & body sway](stability.md) | How the engine judges whether a gait will tip the robot over, and how it sways the body to fix it |
| [Dynamic balance](dynamics.md) | Optional: momentum, pushes and gaits that balance by moving. Capture point, ZMP, preview control, joint torques and foot placement |
| [Beyond the hexapod](other-robots.md) | A quadruped, four-joint legs and two-joint legs: what works, what doesn't, and why |
| [Protocol & teleop](protocol.md) | The controller packets (protocol v4) and the shared gamepad layout |

## Who uses it

| Stack | Uses gait core for | Pinned to |
|---|---|---|
| [ROS 2](../stacks/ros2/index.md) | The `gait_controller` node, the battery decisions in `behaviour_manager`, and the gamepad layout in `controller_teleop` | v0.9.0 |
| [ESP32](../stacks/esp32.md) | The whole gait and battery behaviour, and the controller packets | v0.9.0 |
| [Servo2040](../stacks/servo2040.md) | The whole gait and battery behaviour, and the controller packets | v0.9.0 |
| [Controllers](../controllers.md) | The controller packets, and the receiver's gamepad layout | v0.9.0 |

A change in gait core reaches every stack once it's updated to that release,
so the robots walk, react and handle a gamepad the same way. Every stack is
on v0.9.0, each passing the engine `gait_core::explorer::robot()` (or the
Lite's version of it, with narrower joint limits). None is on v0.13.0 yet,
so the [changes since v0.9.0](#since-v090) haven't reached the robots, and
the stack pages, such as [How it walks](../stacks/ros2/walking.md), still
describe v0.9.0.

## What's in it

Everything is under one include root, `include/gait_core/`, in a folder per
area, and one namespace, `gait_core`.

| Folder | CMake target | What |
|---|---|---|
| `robot/` | `gait_core::gait_core` | What a robot is: legs, joint chains, stance, gaits and masses. Data only |
| `kinematics/` | `gait_core::gait_core` | Solving one leg: the `LegKinematics` interface, an exact analytic solver and a numerical one for any chain, and a leg's mass properties |
| `locomotion/` | `gait_core::gait_core` | Making the robot walk: the gait engine, gait timing, trajectories, frames, the support polygon, and the balance and preview controllers |
| `estimation/` | `gait_core::gait_core` | Leg odometry (body motion and climb from the planted feet), and the body's velocity from leg odometry and the IMU |
| `behaviour/` | `gait_core::behaviour` | Decisions from sensors, such as the [battery monitor](#battery-behaviour) |
| `robots/` | `gait_core::robots` | Real robots' definitions. Only the Explorer so far |
| `protocol/` | `gait_core::protocol` | Controller ↔ robot packets. Header-only, no dependencies |
| `teleop/` | `gait_core::teleop` | The shared gamepad layout. Header-only, no dependencies |

The gait doesn't depend on the behaviours or the robots, so a robot can use
the gait alone with its own definition. The protocol and teleop headers need
nothing at all, not even Eigen, so controller firmware can use them too.

### The gait engine

`GaitEngine` is the whole gait: startup, walking, sitting and standing, tilt
mode, leveling, load balancing, terrain adaptation, leg odometry and, if
turned on, [dynamic balance](dynamics.md). See
[How it walks](../stacks/ros2/walking.md) for what each feature does. A
stack builds one from its settings and its robot, feeds it what it measures,
and calls `update` at a steady rate:

```cpp
#include "gait_core/locomotion/gait_engine.hpp"
#include "gait_core/robots/explorer.hpp"

gait_core::GaitParams params;            // defaults are the ROS 2 parameter defaults
params.power_on_startup = false;         // servos off until the first stand
gait_core::GaitEngine engine(params, gait_core::explorer::robot());

// Whenever they arrive
engine.onJointStates(feedback);          // every leg's joint angles (and efforts, if any)
engine.setImu(imu);                      // optional: orientation for leveling; rates and
                                         // acceleration for balance feedback
engine.setVelocity({vx, vy, wz});        // m/s and rad/s, body frame

// Every tick, e.g. at 100 Hz
const gait_core::GaitOutput out = engine.update(dt, now);
setServoEnable(out.servoPower);
if (out.commanded) {
    for (int leg = 0; leg < engine.numLegs(); ++leg) {
        writeJoints(leg, out.joints[leg]);   // body to foot, rad
        // With torque_output on, out.torques[leg] is a feedforward torque per joint (N m)
    }
}
```

Sensors are optional. A stack that never gives joint efforts, foot contact,
an IMU, a body velocity (`setBodyVelocity`) or a controller orientation just
doesn't get the features that need them. Commands
(`setGait`, `setMode`, `setHeight`, `sit`, `powerDown`) return whether they
were accepted, with a message saying why not. Logging, a callback when the
robot sits or stands, and a foothold planner (ROS 2's height map) are
optional hooks.

Before it moves anything, the engine checks the robot it was given: it must
have legs and gaits, every gait must run on its legs, and every gait must be
stable. If not, it **never moves and keeps the servos off**, and logs why
every 5 seconds. See [Describing a robot](robots.md#what-the-engine-checks).

Nothing allocates on the heap while it runs: every per-leg and per-joint
array is a fixed size, set when gait core is built.

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
  its own units at its edges, never inside the core.
- **Body frame** as in [Coordinate frames](../concepts/frames.md): +x
  forward, +y left, +z up. Each leg's frame has +x pointing out along the leg
  from its first joint.
- **Legs are counted from 0** in the order the robot's definition lists them.
  The Explorer's run clockwise from the front right, so leg 0 in code is leg 1
  in [Legs & joints](../concepts/legs.md), up to leg 5 in code, which is
  leg 6.
- **Joints are listed body to foot.** On the Explorer that's coxa, femur,
  tibia.

## Versions

Releases are tagged `vMAJOR.MINOR.PATCH`. The minor version changes when the
API changes, and the patch version for fixes that keep the API.

| Release | What changed |
|---|---|
| v0.13.0 | Capture-point [foot placement](dynamics.md#foot-placement) for gaits that balance by stepping, and `setBodyVelocity` for a velocity from an outside estimator |
| v0.12.0 | [Dynamic balance](dynamics.md): robot masses and a live centre of mass, `BalanceController` (capture point and ZMP), balance feedback with `BodyVelocityEstimator`, preview control, joint torque output and foot contact sensors. `ImuSample` gains angular velocity and linear acceleration. The Explorer gets its CAD masses |
| v0.11.1 | A gait that doesn't sway eases off any sway left behind |
| v0.11.0 | Landing targets slow with the gait so they don't break the joint speed limits. The Explorer's stride grows to 0.07 m, and `min_phase_duration` drops to 0.55 s |
| v0.10.1 | Explorer joint speed limit tidy-up |
| v0.10.0 | Per-joint speed limits (`JointDefinition::max_velocity`), set on the Explorer to its servos' speeds |
| v0.9.0 | **Any legged robot**: `RobotDefinition` with a runtime leg count, legs as joint chains, gaits as duty factor and offsets, the support polygon and body sway. The Explorer moves to `gait_core::robots`. One include root, `gait_core/`, with the behaviour and teleop namespaces renamed to `gait_core::behaviour` and `gait_core::teleop`. Protocol version 4 |
| v0.8.0 | Driving by tilting the controller: hold LB to walk with the controller's pitch, roll and yaw |
| v0.7.0 | Controller orientation: in tilt mode, the body follows the controller while LB is held. Protocol version 3 |
| v0.6.0 | The controller protocol and the shared gamepad layout. Protocol version 2 |
| v0.5.0 | Servo power control |
| v0.4.0 | The behaviour library, with `BatteryMonitor` |
| v0.3.0 | No heap allocation while running; legs indexed from 0 |
| v0.2.0 | `GaitEngine`: the whole gait, moved out of ROS 2's `hexapod_gait_controller` |
| v0.1.0 | The ROS-free gait maths, split out of the ROS 2 stack |

!!! note "Upgrading from v0.9.0 to v0.13.0"
    No code changes are needed: the new fields and settings all have
    defaults, and the protocol is still version 4, so a controller on
    v0.9.0 still drives a robot on v0.13.0. What does change is how the
    Explorer walks: its joint speed limits, longer stride and shorter
    shortest phase make it faster, and its stability margins are judged
    from its live centre of mass. Check that the robot still walks well
    before relying on it, and pass the IMU's rates and acceleration in
    `ImuSample` if you'll use balance feedback.

!!! warning "Upgrading to v0.9.0"
    Code that uses gait core directly needs changing:

    - Include paths move under `gait_core/`, e.g.
      `hexapod_protocol/packets.h` is now `gait_core/protocol/packets.h`, and
      the gait engine is `gait_core/locomotion/gait_engine.hpp`.
    - `GaitEngine` has no default robot: pass `gait_core::explorer::robot()`
      (from `gait_core/robots/explorer.hpp`, target `gait_core::robots`).
    - The leg and gait names move to `gait_core::explorer`, e.g.
      `explorer::LEG_FRONT_RIGHT` and `explorer::GAIT_TRIPOD`.
    - Joint angles are a `JointVec` per leg rather than a fixed
      `JointAngles` struct.
    - A controller and robot must both be on protocol version 4. See
      [Protocol & teleop](protocol.md#versions).

Each stack includes gait core as a git submodule **pinned to a release tag**,
so a new release only reaches a stack once that stack is updated to it.
Clone stacks with their submodules:

```bash
git submodule update --init --recursive
```

## Building and testing

The library has its own test suite: kinematics on both solvers, foot
directions and forces, gait schedules, the support polygon, leg clearance,
odometry, mass properties, the balance and preview controllers, the body
velocity estimator, the packets, the gamepad layout, a check that nothing
allocates, and [three other robots](other-robots.md) run through everything
the Explorer does.

```bash
git clone --recursive https://github.com/TH4R4V3N1324/legged-gait-core.git
cd legged-gait-core
cmake -S . -B build
cmake --build build --config Release
ctest --test-dir build -C Release --output-on-failure
```

| CMake option | Default | Effect |
|---|---|---|
| `GAIT_CORE_USE_DOUBLE` | `OFF` | Numbers are `double` instead of `float` |
| `GAIT_CORE_MAX_LEGS` | `6` | Most legs a robot can have. Sizes every per-leg array |
| `GAIT_CORE_MAX_JOINTS` | `6` | Most joints in one leg. The protocol carries up to 6 |
| `GAIT_CORE_BUNDLED_EIGEN` | `OFF` | Always use the bundled Eigen, even if the system has one. Always on when cross-compiling |
| `GAIT_CORE_BUILD_TESTS` | `ON` when built on its own | Build the test suite |

By default, numbers are single-precision `float`, since the
microcontrollers have no double-precision hardware. The ROS 2 stack builds it
with `-DGAIT_CORE_USE_DOUBLE=ON` instead. CI tests both, and cross-compiles
for the RP2040's Cortex-M0+.

It can be linked as a CMake subproject, an installed CMake package (which is
how colcon builds it for ROS 2), or a PlatformIO library. A controller that
only needs the protocol and gamepad layout can just add `include/` to its
include path, without cloning Eigen. See the
[README](https://github.com/TH4R4V3N1324/legged-gait-core#linking).
