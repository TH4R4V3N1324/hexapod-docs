# Mapping & navigation

The robot can build a 2D map with its lidar (SLAM) and plan and follow paths
on it (Nav2).

## Quick start

```bash
# Map only
ros2 launch hexapod_bringup hexapod_robot.launch.py slam:=true

# Map and navigate
ros2 launch hexapod_bringup hexapod_robot.launch.py slam:=true nav:=true
```

Use `hexapod_simulation.launch.py` instead to try it in Gazebo first. Then
send goals with RViz's **2D Goal Pose** tool, or the `navigate_to_pose`
action.

## Odometry

The robot works out how far it has moved in two stages, which both run with
every launch:

1. **Leg odometry.** The gait controller works out the body's motion from how
   the planted feet move relative to it, and publishes it on `/odometry/legs`.
   When the robot is pushing against something (for example walking into a
   wall), its feet scrub, so only the turn is counted.
2. **EKF.** `robot_localization` combines leg odometry with the IMU's
   gyroscope and publishes `/odometry/filtered`. The feet slip most when
   turning, and the gyroscope corrects that. The IMU's absolute heading isn't
   used, since it drifts without a magnetometer.

In simulation, after driving forward, turning 143° and driving forward again,
leg odometry alone was 5.7 cm and 12° off; the combined odometry was 2.4 cm
and 0° off.

The robot's base frame for odometry, SLAM and Nav2 is `base_footprint`, the
point on the floor directly below the body.

## SLAM

`slam_toolbox` builds the map from the lidar, with the legs filtered out of
the scan. In simulation, after driving a 0.6 m square around a house, SLAM
was 1.8 cm from the true position; odometry alone was 15.5 cm off.

Settings are in `hexapod_bringup/config/slam_toolbox.yaml`.

## Nav2

Settings are in `hexapod_bringup/config/nav2_params.yaml`.

- **Size:** the robot is treated as a 0.32 m radius circle (legs plus
  stride), with 0.45 m inflation, so 1 m doorways stay passable.
- **Direction:** with `nav:=true` the robot starts in strafe mode, so it can
  move in any direction, but Nav2 prefers to face along its path so the depth
  camera sees where it's going.
- **Speed:** capped just under the gait's limits: 0.15 m/s and 0.6 rad/s.
- **Low obstacles:** the lidar passes over anything lower than its scan
  plane. The depth camera fills that gap, marking anything from 3 cm to 35 cm
  above the floor.

In simulation, Nav2 got through a room of 6–20 cm obstacles (all below the
lidar) staying at least 0.5 m from each. In a house, it reached a goal across
the room, and another through a doorway into the next room, with no
recoveries.

!!! note
    The depth camera needs a ROS 2 driver on the real robot before low
    obstacles can be detected. See
    [Known limitations](index.md#known-limitations).
