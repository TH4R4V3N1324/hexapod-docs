# Lite

The Lite is the cheaper, simpler hexapod: **18 hobby PWM servos** driven by a
Pimoroni **Servo 2040**, rather than serial bus servos and a Raspberry Pi. It
runs the [Servo2040 stack](../../stacks/servo2040.md), which walks with the
same gait as the Explorer.

!!! info "In development"
    The Lite's hardware design hasn't been published yet. For now, the
    firmware uses the Explorer's leg geometry.

## What's in it

- **6 legs, 3 joints each,** each joint a hobby PWM servo.
- **Pimoroni Servo 2040** running the walking firmware.
- **ESP32 receiver** for the wireless controller link, talking to the Servo
  2040 over I²C. The controller is
  [on hold](../../stacks/servo2040.md#how-it-fits-together) for now.
- **ISM330DHCX IMU** for leveling.

Hobby servos can't report their position or load, so the Lite can't feel the
ground the way the Explorer does. See
[What's different from ROS 2](../../stacks/servo2040.md#whats-different-from-ros-2).
