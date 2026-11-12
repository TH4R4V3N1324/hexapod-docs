# How it walks

The `gait_controller` node turns velocity commands into foot movements, and
foot positions into joint angles. It runs at 100 Hz.

## Steps

See [Gaits & modes](../../../concepts/gaits.md) for stance, swing and the
gaits themselves, and [Coordinate frames](../../../concepts/frames.md) for the
axes. In the ROS 2 stack:

- Stance feet move back at the body's speed.
- Swing feet lift first, move across high (peaking about 5 cm up), then come
  down almost vertically.
- Each foot stays within 6 cm of its neutral position. At higher speeds, steps
  get shorter in time rather than longer in distance, down to 0.7 s per phase.
  That sets each gait's top speed (about 0.17 m/s for tripod).
- Commands take effect straight away, not just at the end of a step. A
  forward-to-backward reversal starts in about 0.06 s.
- Joint speeds are capped at 4 rad/s, and the gait slows down to keep within
  that.
- After 2 s without commands, the legs return to their start positions.

In **normal** mode, sideways stick input turns the robot rather than
strafing. In **strafe** mode, `linear.y` strafes and `angular.z` turns.

## Feeling the ground

There are no contact switches. Instead, each leg's three servo load readings
are turned into the force on its foot. A foot counts as on the ground above
2 N and off it below 1 N. These forces are published on
`/foot_force/leg1` to `/foot_force/leg6` and shown in the Foxglove panel.

!!! note
    The ST3215's load reading is an estimate from motor current, not a true
    torque sensor, and these thresholds were tuned in simulation.

## Terrain adaptation

With terrain adaptation on, each foot remembers its own ground height, stays
there through its stance and aims for it on its next swing:

- A swinging foot that hits ground early stops there.
- A foot that finishes its swing without touching anything reaches down until
  it finds the ground.
- A foot that hits something from the side mid-swing lifts higher and carries
  on over it.

Ground heights are limited to 4 cm up or down. The body rises or falls so the
feet's average ground height stays at nominal.

On a test course of 2–3 cm slabs in simulation, terrain adaptation cut
sideways drift from 9 cm to almost zero and halved the body's roll and pitch.

### Terrain map

With the **terrain map** on, the depth camera also builds a height map ahead
of the robot. Each swing then takes its landing height from the map, shifts
its foothold up to 3 cm to avoid landing on an edge, and lifts high enough to
clear what's in the way.

It's off by default: in simulation it only helped on slabs, and walked worse
on hills than the feet alone.

## Leveling

**Walk leveling** uses the IMU to keep the body level on slopes while
walking. It responds slowly, so it follows the slope rather than reacting to
each step. It tilts the body by up to about 8.6° (0.15 rad). In simulation,
on a 5.7° slope, the body stays within 0.1° of level.

**Tilt leveling** does the same in tilt mode, but faster. It holds the body
at the requested roll and pitch even if the surface underneath tilts. In
simulation, with the deck rolling ±6.9°, the body stays within 1.5°.

## Load balancing

With stiff legs on hard ground, tiny height differences decide which feet
carry the weight. Without help, the robot often ends up standing on three
legs with the other three carrying nothing.

Load balancing, while standing with four or more feet down, raises feet
carrying more than their share and lowers feet carrying less, by up to 2 cm.

## Tilt mode limits

| | Default |
|---|---|
| Largest roll, pitch or yaw | 0.3 rad (about 17°) |
| Largest shift on each axis | 4 cm |
| Body height | 6–16 cm |

Poses the legs can't reach safely are cut back: yaw first, then the whole
move.

## Tuning

Everything above is set by `gait_controller` parameters. Tilt parameters can
be changed while running with `ros2 param set`. For the full parameter list,
see the
[hexapod_gait_controller](https://github.com/TH4R4V3N1324/hexapod-ros2/tree/main/hexapod_ws/src/hexapod_gait_controller)
package.
