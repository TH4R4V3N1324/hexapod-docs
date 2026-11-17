# Protocol & teleop

Two small header-only parts of gait core are what every stack has to agree
on to be driven the same way: the **packets** between a controller and a
microcontroller robot, and the **gamepad layout**. Neither needs anything
else, not even Eigen, so controller firmware includes them too.

## Protocol

`gait_core/protocol/packets.h` (target `gait_core::protocol`) defines the
packets: over ESP-NOW for the [ESP32 stack](../stacks/esp32.md), and over I²C
through the receiver for the [Servo2040 stack](../stacks/servo2040.md). Both
are sent by the [controllers](../controllers.md). It's valid C as well as
C++, since some controller firmware is C. Every repo that sends or receives
them includes this one file, so they can't drift apart.

### Controller to robot

A `ControlPacket` every 10 ms, 29 bytes:

| Field | Type | Meaning |
|---|---|---|
| `version` | `uint8_t` | `HEXAPOD_PROTOCOL_VERSION` (4) |
| `joystick1X`, `joystick1Y` | `int16_t` | Left stick, ±128, X right, Y forward. Walking: strafe (or turn, by mode) and forward. Tilt: shift the body |
| `joystick2X`, `joystick2Y` | `int16_t` | Right stick. Walking: X turns. Tilt: roll and pitch |
| `tiltYaw`, `tiltZ` | `int16_t` | Tilt: yaw (positive turns left) and raise the body |
| `currentHeight` | `int16_t` | Body height in mm, 0 for unchanged |
| `command`, `commandArgs[3]` | | A [command](#commands) |
| `controllerRoll`, `controllerPitch`, `controllerYaw` | `int16_t` | The controller's orientation, in milliradians |
| `controllerTilt` | `uint8_t` | Non-zero while LB is held and the controller has an IMU: the orientation fields are valid |

### Robot to controller

A `HexPacket` in reply to each one, 19 bytes:

| Field | Type | Meaning |
|---|---|---|
| `version` | `uint8_t` | `HEXAPOD_PROTOCOL_VERSION` (4) |
| `legConfigs[6]` | `int16_t` | Calibration offsets for the leg asked for with `CMD_REQUEST_CONFIG`, body to foot; 0 past its last joint |
| `currentHeight` | `int16_t` | Body height in mm |
| `currentGait` | `uint8_t` | The gait it's in, or changing to |
| `currentMode` | `uint8_t` | Its mode |
| `flags` | `uint8_t` | Seated (or sitting down or standing up), servos powered, and which [features](#features) are on |
| `numGaits` | `uint8_t` | How many gaits the robot has, for cycling through them |

### Commands

A robot applies a command once, when it or its arguments change, so a
controller sends each one for a few packets and then `CMD_NONE`.

| Command | Arguments |
|---|---|
| `CMD_SET_GAIT` | `[0]` gait, 0 to `numGaits` − 1 |
| `CMD_SET_MODE` | `[0]` mode: normal, strafe, tilt, or config (every joint to its centre plus its calibration offset, handled by the firmware) |
| `CMD_SET_CONFIG` | `[0]` leg, `[1]` joint 0–5 (body to foot), `[2]` offset in degrees, clamped to ±60 |
| `CMD_HOME_STANCE` | Stand up |
| `CMD_REQUEST_CONFIG` | `[0]` leg; its offsets come back in `legConfigs` |
| `CMD_SIT` | `[0]` 1 sit down, 0 stand up (which also powers the servos) |
| `CMD_POWER_DOWN` | Sit down, then turn servo power off |
| `CMD_SET_FEATURE` | `[0]` feature, `[1]` 1 on, 0 off. A robot without the sensors refuses "on" |

### Features

The four gait features that can be switched on and off, in the order of the
gamepad's A, B, X and Y buttons: terrain adaptation, walk leveling, load
balancing and tilt leveling. `flags` reports which are on, so a controller
can toggle them without guessing.

### Gaits are the robot's

A gait is an **index into the robot's own gait list**. `HexPacket.numGaits`
says how many there are, so a controller cycles through any robot's gaits
without knowing them. The `Gait` names in the header (`GAIT_TRIPOD`,
`GAIT_RIPPLE`, `GAIT_WAVE`) are the Explorer's; another robot numbers its
own. Modes are the same for every robot.

### Versions

Both packets start with the protocol version, and **each end drops packets
of any other version or size**, saying so on its serial monitor. So when you
update one end, update the other to the same gait core release.

| Protocol | gait core | Change |
|---|---|---|
| 4 | v0.9.0 | Gaits are an index into the robot's list, with `numGaits`; `legConfigs` carries 6 joints instead of 3. `HexPacket` grows from 12 to 19 bytes |
| 3 | v0.7.0 | Controller orientation fields. `ControlPacket` grows from 22 to 29 bytes |
| 2 | v0.6.0 | First shared version |

The ESP32 and Servo2040 firmware and the controllers are all on protocol 4.
To change the layout: bump the version and the size checks at the bottom of
the header, append enum values rather than reordering them, and release gait
core so every repo moves together.

## Teleop

`gait_core/teleop/gamepad_teleop.hpp` (target `gait_core::teleop`) is the
[gamepad layout](../controllers.md#gamepad-controls) as code. ROS 2's
`controller_teleop` and the controller receiver both run it, so a gamepad
drives every stack the same way.

A stack feeds `GamepadTeleop` the gamepad's state, normalised, and whether
the robot is seated or in tilt mode. It gets back:

- **velocities and a tilt pose**, as fractions of the robot's limits, so the
  stack scales them to its own top speed and tilt range;
- **the controller's orientation**, while LB is held in tilt mode;
- **events**, at most once per press: next mode, next gait, a feature toggle,
  sit or stand, power down, and a body height step.

It's called for every gamepad report, and every 50 ms or so while a button
may be held, so the 2-second power-down hold and the D-pad's repeat fire on
time.

### Controller orientation

A gamepad with motion sensors, such as a DualSense, DualShock 4 or Switch Pro
controller, can steer the robot by tilting it. Hold **LB [L1]**:

- **Walking:** the controller's pitch drives forward and back, its roll
  strafes, and its yaw turns. The orientation when LB was pressed is
  neutral; tilting it 0.35 rad (20°) is full speed, and turning it 0.5 rad
  (29°) is full rotation. The teleop sends this as stick values, so the robot
  doesn't need to know where they came from.
- **Tilt mode:** the body follows the controller's orientation, relative to
  where it was when LB was pressed, instead of the sticks. Releasing LB, or
  losing the controller's orientation, eases the body back to neutral.

| Stack | Controller orientation |
|---|---|
| ROS 2 | ✅ The `controller_imu` node reads the gamepad's sensors through SDL2. See [Driving](../stacks/ros2/driving.md#controller-tilt) |
| ESP32, Servo2040 | The robots follow it in tilt mode, but no controller sends it yet: the receiver doesn't read gamepad motion sensors, and the OLED controller's IMU has no driver yet |
