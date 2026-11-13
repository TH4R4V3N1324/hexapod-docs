# Legs & joints

Every variant has six legs with three joints each, giving 18 degrees of
freedom.

## Joints

From the body outwards:

| Joint | Moves |
|---|---|
| **Coxa** | The hip. Swings the leg forwards and backwards. |
| **Femur** | Lifts and lowers the leg. |
| **Tibia** | The knee. Reaches the foot in and out. |

## Leg numbering

Looking down on the robot with the front at the top:

```
                 front
         leg 6 ●           ● leg 1
                 \       /
     leg 5 ●──── [ body ] ────● leg 2
                 /       \
         leg 4 ●           ● leg 3
                  rear
```

- **Legs 1–3 are on the right** (front, middle, rear).
- **Legs 4–6 are on the left** (rear, middle, front).

In code, [gait core](../stacks/gait-core.md) counts the legs from 0, in the
same clockwise order: leg 0 in code is leg 1 here.

In the ROS 2 stack, the joints are named `leg<N>_coxa_joint`,
`leg<N>_femur_joint` and `leg<N>_tibia_joint`.

How each joint maps to a servo depends on the variant:

- **Explorer:** a servo ID on the bus, set from its leg and joint. See
  [Explorer wiring](../variants/explorer/wiring.md#servo-ids).
- **Lite:** a Servo 2040 output, three per leg in order. See
  [Servo2040 wiring](../stacks/servo2040.md#servos).
