# Coordinate frames

The robot's body frame follows the usual ROS convention
([REP 103](https://www.ros.org/reps/rep-0103.html)):

- **+x is forward**, towards the front of the body (the cameras on the
  Explorer).
- **+y is left.**
- **+z is up.**
- **Positive yaw is anticlockwise**, looking down.

In the ROS 2 stack, velocity commands (`/cmd_vel`) use the same axes:
positive forward speed walks forwards,
positive sideways speed strafes left, and positive turn rate turns left.

## Body height

Body height is measured from the feet up to the hips. The default is
**12 cm** in every stack. The ROS 2 stack expresses it as the feet's
position below the hips, so it's negative: `-0.12` m.

## Leg frames

Each leg also has its own frame, rotated to point along the leg, which the
inverse kinematics work in. The stacks convert foot targets between the body
frame and each leg's frame. Because the left and right legs are mirror images,
the conversion also handles mirroring.
