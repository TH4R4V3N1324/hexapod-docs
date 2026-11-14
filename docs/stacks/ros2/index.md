# ROS 2 stack

The ROS 2 stack runs the [Explorer](../../variants/explorer/index.md) on a
Raspberry Pi 5. The code is in
[hexapod-ros2](https://github.com/TH4R4V3N1324/hexapod-ros2). It's split
into two separate ROS 2 Jazzy workspaces that only talk over the network:

- **`hexapod_ws`** runs **on the robot**, or in simulation on a dev machine.
  It has the gait controller, kinematics, hardware drivers and robot model.
- **`operator_ws`** runs **on the operator's machine** (a laptop or Steam
  Deck). It reads the gamepad and runs a Foxglove bridge.

The gamepad is read on the operator side, but turned into movement commands
on the robot.

## Data flow

```
[Gamepad] → operator: game_controller_node → /joy
                                       │  (DDS, same ROS_DOMAIN_ID)
                                       ▼
robot: controller_teleop → /cmd_vel → gait_controller
                                            │
                                            ▼  (inverse kinematics + trajectories)
                                      /joint_commands
                                            │
                                            ▼
                   joint_state_relay → hexapod_joint_controller (ros2_control)
                                            │
                                            ▼
                     ST3215System plugin → serial bus → 18 × ST3215 servos

/joint_states, /imu/data, /power/left, /power/right → foxglove_bridge → Foxglove Studio
```

## Packages

| Package | Purpose |
|---|---|
| `hexapod_interfaces` | Custom services and messages |
| `hexapod_description` | Robot model (URDF), Gazebo worlds, RViz config |
| `hexapod_hardware_interface` | Servo driver (`ros2_control` plugin), IMU and power monitor drivers, the `SERVO_EN` driver, hardware tests |
| `hexapod-gait-core` | Submodule: the shared [gait core](../gait-core.md) library, built as the `gait_core` package |
| `hexapod_gait_controller` | The `gait_controller` node, which runs gait core's gait engine |
| `hexapod_control` | `behaviour_manager`, which runs gait core's battery behaviour |
| `hexapod_teleop` | Keyboard and gamepad control |
| `hexapod_bringup` | Launch files, plus SLAM, Nav2 and EKF config |
| `operator_bringup` | Operator side: gamepad node and Foxglove bridge |

The repo also contains:

| Folder | |
|---|---|
| `foxglove_panel/` | The Hexapod Foxglove panel |
| `pi/` | Pi setup: I²C, hotspot, start at boot |
| `steamdeck/` | Steam Deck setup and launch scripts |

## Launch files

All in `hexapod_bringup`:

| Launch file | Runs |
|---|---|
| `hexapod_robot.launch.py` | The real robot: servos, sensors, gait controller, teleop, Foxglove bridge |
| `hexapod_fake.launch.py` | The same on mock hardware, for testing gait logic without servos or Gazebo |
| `hexapod_simulation.launch.py` | Gazebo, plus the gait controller, teleop, Foxglove bridge and RViz |

Both `hexapod_robot` and `hexapod_simulation` take `slam:=true` and
`nav:=true`. See [Mapping & navigation](navigation.md).

## Building it

The robot side runs in Docker. On a dev machine:

```bash
git submodule update --init --recursive   # once, for gait core
cd hexapod_ws
docker compose -f docker-compose.dev.yml up --build
# inside the container:
colcon build --symlink-install
source install/setup.bash
```

Run `colcon build` from `hexapod_ws`: it builds [gait core](../gait-core.md)
in double precision, using settings in `hexapod_ws/colcon.meta`, and the gait
controller won't compile without them.

The dev image includes Gazebo, `joy` and other dev tools. The plain
`docker-compose.yml` builds a lean runtime image for the robot, without
Gazebo, RViz or X11. It runs privileged, so the serial port and `/dev/i2c-1`
are visible.

Then, for example:

```bash
ros2 launch hexapod_bringup hexapod_fake.launch.py           # mock hardware
ros2 launch hexapod_bringup hexapod_robot.launch.py          # real robot
ros2 launch hexapod_bringup hexapod_simulation.launch.py     # Gazebo
```

## Known limitations

- **The hardware drivers are untested on real hardware.** The servo driver
  has only run against a simulated servo bus, and the sensor drivers have
  only run with no devices attached.
- **Joint directions aren't set for a real robot yet.** They default to 1 in
  `hexapod.ros2_control.xacro`. See
  [Calibrate the servos](setup.md#3-calibrate-the-servos).
- **Contact sensing and terrain adaptation are tuned in simulation only.**
  The contact, touchdown and stumble thresholds and the load balancing gain
  need checking against the real servos' load readings. The terrain map needs
  checking against the real ToF camera's noise.
- **There's no ROS 2 driver for the ToF camera yet.**
- **Leg odometry can drift** when feet slip without the robot pushing against
  anything, for example on a slippery or moving surface.
- **The robot creeps on a tilting platform in simulation.** With the deck
  rolling and pitching ±6.9° together, the levelled robot slides several cm
  and turns over 20 s. The real robot's rubber feet and sprung tibias should
  grip better than the simulation's rigid contacts.
