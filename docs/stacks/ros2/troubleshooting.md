# Troubleshooting

Problems with the Explorer and the ROS 2 stack.

## The robot doesn't respond to the gamepad

- **Is it standing?** After startup the robot sits and ignores movement
  commands until it's told to stand. Press *Stand up* in the Foxglove panel.
- **Same ROS domain?** The operator machine and the robot need the same
  `ROS_DOMAIN_ID`, or `/joy` never reaches the robot.
- **Wrong buttons?** Run `ros2 topic echo /joy` on the robot and move each
  control to check the axis and button numbers. They should follow the
  standard game controller layout. If you start plain `joy_node` instead of
  `game_controller_node`, the axes come in each controller's raw order. For
  Xbox-style controllers, including the Steam Deck, that puts the triggers on
  axes 2 and 5.

## The Steam Deck can't find the robot

- Join the robot's `hexapod` hotspot. The Deck tries 192.168.50.1 first.
- Otherwise the Deck scans its local network. Multicast must get through
  your router for ROS 2 to work, which is why the hotspot exists.

## `handshake failed` in the robot's logs

This is harmless. When the Steam Deck scans the network, it opens and closes
a connection on port 8765 at every address, and `foxglove_bridge` logs each
one as a failed handshake.

## The hotspot doesn't work, or keeps dropping

Keep the Pi's home Wi-Fi connection on **2.4 GHz**. The hotspot shares the
radio's single channel and can't use 5 GHz radar channels.

## The robot's software didn't start at boot

Check the start-at-boot service:

```bash
journalctl -u hexapod-start
```

## The servos don't move

- **Is servo power on?** The leg hubs only power the servos when `SERVO_EN` is
  high **and** the E-stop loop is closed. Check the E-stop isn't pressed and
  its wires reach J8 on both boards.
- **Is the power switch wired to both boards?** Each leg hub stays off until
  its J11 pins 1–2 are shorted.
- **Fuses:** each leg has its own 5 A fuse on its leg hub.
- The default container runs on **mock hardware**. Switch its command to
  `hexapod_robot.launch.py` (see
  [First power-on](setup.md#start-at-boot)).
- Check the bus with `test_servos` (scan). Servos that don't reply are logged
  and skipped.
- Check that every servo has a unique ID (see [Servo IDs](../../variants/explorer/wiring.md#servo-ids)).

## A leg moves the wrong way

If it moves the opposite way, flip that joint's `direction`. If it's off by
a fixed angle, re-centre the servo: check it against the
[centre pose](setup.md#where-the-centre-is), especially the tibia. See
[Calibrate the servos](setup.md#3-calibrate-the-servos).

## The IMU or power readings are missing

The sensor drivers keep retrying if their device is missing, so fix the
connection and the readings should come back without a restart. Check that
the devices appear with `i2cdetect -y 1`: the IMU at `0x6A`, and the leg hubs
at `0x40` (left) and `0x41` (right).

If both leg hubs show up at the same address, check jumper JP1: it should be
open on the left board and bridged on the right. If nothing shows up, check
that JP3 is on 1–2.

## The lidar scan is rotated in RViz

Adjust the yaw of `lidar_joint` in
`hexapod_description/urdf/hexapod.lidar.xacro`.

## Gazebo segfaults with `libEGL warning: ... driver (null)`

On a host with an NVIDIA GPU, add `-f docker-compose.nvidia.yml` when
starting the dev container. This needs the NVIDIA Container Toolkit on the
host. See [Simulation](simulation.md).
