# Simulation

The robot can run in **Gazebo Harmonic** for trying things out and
developing gaits without the hardware.

## Running it

From inside the dev container (see
[Building it](index.md#building-it)):

```bash
ros2 launch hexapod_bringup hexapod_simulation.launch.py
```

This starts Gazebo with the robot, the gait controller, teleop, the Foxglove
bridge and RViz. In simulation the robot stands up on its own after startup.

!!! tip "NVIDIA GPUs"
    On a host with an NVIDIA GPU, start the dev container with
    `-f docker-compose.nvidia.yml` as well. This needs the NVIDIA Container
    Toolkit on the host. Without it, Gazebo's sensor rendering fails with
    `libEGL warning: ... driver (null)` and crashes.

| Argument | |
|---|---|
| `world` | Which world to load (see below) |
| `gui:=false` | Run Gazebo without its window |
| `rviz:=false` | Skip RViz |
| `bridge_vision:=true` | Bridge the main camera and full-resolution depth image out of Gazebo. Off by default because they use a lot of bandwidth |
| `front_depth_width`, `front_depth_height`, `front_depth_rate` | Depth camera resolution and rate |
| `spawn_x`, `spawn_y`, `spawn_z`, `spawn_yaw` | Where the robot starts |
| `slam:=true`, `nav:=true` | Start SLAM and Nav2 (see [Mapping & navigation](navigation.md)) |

The simulated sensors match the real ones: a 10 Hz, 1600-point lidar with a
0.2–12 m range, and the ToF camera's field of view (at 15 Hz to keep
Gazebo's load down). The robot's true position is published on
`/ground_truth/odom`.

## Worlds

| World | |
|---|---|
| `hexapod_world` | The default world |
| `empty_world` | Flat ground |
| `rough_terrain` | 2–3 cm slabs and a step |
| `tilt_platform` | A deck you can tilt, for testing tilt mode and leveling |
| `low_obstacles` | A walled room with 6–20 cm objects, all below the lidar |
| `outdoor_slopes` | Hills of 5–20°, and 10° and 15° side slopes |
| `outdoor_rocks` | Rocks mostly 1–4 cm high, a few up to 7 cm, over gentle bumps |
| `outdoor_low_friction` | Lanes of dry ground, wet grass, mud and wet tile, and a 10° mud hill |

### Tilting the platform

```bash
ros2 launch hexapod_bringup hexapod_simulation.launch.py world:=tilt_platform
# switch to tilt mode, then tilt the deck (radians)
ros2 service call /set_mode hexapod_interfaces/srv/SetMode "{mode: 2}"
gz topic -t /platform/roll -m gz.msgs.Double -p 'data: 0.15'
```

`/platform/pitch` and `/platform/yaw` work the same way.

## Scenario runner

`scenario_runner.py` walks the robot through 16 scenarios across the outdoor
worlds, starting a fresh simulation for each. It scores each run on progress,
sideways drift, odometry error, body tilt and falls.

```bash
ros2 run hexapod_bringup scenario_runner.py                     # all, headless (~10 min)
ros2 run hexapod_bringup scenario_runner.py --only hill         # names containing "hill"
ros2 run hexapod_bringup scenario_runner.py --only rocks --gui  # watch in Gazebo
ros2 run hexapod_bringup scenario_runner.py --list              # list scenarios
```

Results go to `/tmp/hexapod_scenarios/results.csv`.

## Checking directions

```bash
# Unit tests of the foot motion maths
colcon test --packages-select hexapod_gait_controller && colcon test-result --verbose

# In Gazebo: drives each direction and checks how the robot moved
ros2 launch hexapod_bringup hexapod_simulation.launch.py world:=empty_world
ros2 run hexapod_bringup sim_direction_check.py   # in a second terminal
```

## Limitations

Soft ground can't be simulated, since Gazebo's ground doesn't deform. The
depth camera and lidar don't model sunlight.
