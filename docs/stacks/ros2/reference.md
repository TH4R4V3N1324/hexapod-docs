# ROS reference

!!! note
    This page may lag behind the code. If something doesn't match, the
    [hexapod-ros2](https://github.com/TH4R4V3N1324/hexapod-ros2) repo is
    the source of truth.

## Topics

| Topic | Type | From | To |
|---|---|---|---|
| `/cmd_vel` | `geometry_msgs/Twist` | `keyboard_teleop`, `controller_teleop`, Nav2 | `gait_controller` |
| `/body_pose` | `geometry_msgs/Pose` | `controller_teleop` | `gait_controller` (tilt mode) |
| `/joy` | `sensor_msgs/Joy` | `game_controller_node` (operator) | `controller_teleop` |
| `/joint_states` | `sensor_msgs/JointState` | `joint_state_broadcaster` | `gait_controller`, `robot_state_publisher` |
| `/joint_commands` | `sensor_msgs/JointState` | `gait_controller` | `joint_state_relay` |
| `/hexapod_joint_controller/commands` | `std_msgs/Float64MultiArray` | `joint_state_relay` | `hexapod_joint_controller` |
| `/imu/data_raw` | `sensor_msgs/Imu` (no orientation) | `ism330dhcx_node` | `imu_filter_madgwick` |
| `/imu/data` | `sensor_msgs/Imu` | `imu_filter_madgwick` (robot), Gazebo (sim) | |
| `/power/left`, `/power/right` | `sensor_msgs/BatteryState` | `ina226_node` | |
| `/foot_force/leg1` … `leg6` | `geometry_msgs/WrenchStamped` | `gait_controller` | |
| `/seated` | | `gait_controller` | |
| `/odometry/legs` | `nav_msgs/Odometry` | `gait_controller` | `ekf_filter_node` |
| `/odometry/filtered` | `nav_msgs/Odometry` | `ekf_filter_node` | |
| `/lidar/scan` | `sensor_msgs/LaserScan` | `rplidar_ros` (robot), Gazebo (sim) | `laser_filters` |
| `/lidar/scan_filtered` | `sensor_msgs/LaserScan` (legs removed) | `laser_filters` | SLAM, Nav2 |
| `/front_depth/points` | `sensor_msgs/PointCloud2` | Gazebo (sim) | `gait_controller`, Nav2 |
| `/camera/image` | `sensor_msgs/Image` | Gazebo (sim) | |
| `/terrain_map` | `sensor_msgs/PointCloud2` | `gait_controller` | |

## Services

| Service | Type | |
|---|---|---|
| `/set_gait` | `hexapod_interfaces/SetGait` | `0` tripod, `1` ripple, `2` wave |
| `/set_mode` | `hexapod_interfaces/SetMode` | `0` normal, `1` strafe, `2` tilt |
| `/set_height` | `hexapod_interfaces/SetHeight` | Feet's height below the hips, negative: `-0.12` for 12 cm. Range 0.06–0.16 m |
| `/get_capabilities` | `hexapod_interfaces/GetCapabilities` | Available modes and gaits, current speed and tilt limits, current mode and gait |
| `/set_terrain_adaptation` | `std_srvs/SetBool` | |
| `/set_terrain_map` | `std_srvs/SetBool` | |
| `/set_walk_leveling` | `std_srvs/SetBool` | |
| `/set_tilt_leveling` | `std_srvs/SetBool` | |
| `/set_tilt_lock_yaw` | `std_srvs/SetBool` | |
| `/set_load_balancing` | `std_srvs/SetBool` | |
| `/ik_solver`, `/fk_solver` | `hexapod_interfaces/IKSolver`, `FKSolver` | Standalone `kinematic_solver` node, not started by the launch files |

## Frames

- `map` → `odom`: published by SLAM.
- `odom` → `base_footprint`: published by the EKF.
- `base_footprint` → `base_link`: published by the gait controller. It's the
  body's height above the floor, plus its roll and pitch from the IMU.

`base_link` follows REP 103: +x forward, +y left, +z up.
