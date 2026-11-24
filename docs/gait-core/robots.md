# Describing a robot

Gait core has no robot built in. A `GaitEngine` walks whatever
`RobotDefinition` it's given: the legs, how each one is put together, where
the feet stand, which gaits it can walk in, and where its weight is. This page
goes through each part, using the Explorer's definition
(`gait_core::explorer::robot()` in `src/robots/explorer.cpp`) as the example.

```
RobotDefinition
├── numLegs, legs[]          LegDefinition per leg
│   ├── mount                where the leg's first joint sits on the body
│   ├── numJoints, joints[]  JointDefinition per joint, body to foot
│   ├── foot                 last joint to the foot
│   ├── solver               which kinematics to use
│   └── stanceOffset         this leg's foot relative to the shared stance
├── standReach, standHeight  the neutral stance
├── maxStride                how far a foot moves from neutral
├── centreOfMass, bodyMass   where the weight is
└── numGaits, gaits[]        GaitPattern per gait
```

Everything is in SI units, in the [body frame](../concepts/frames.md) (+x
forward, +y left, +z up) unless it says it's in a leg's frame.

## Legs

### Where a leg mounts

A leg's `mount` places its frame on the body: a **yaw** about the body's z
axis, and the **position** of its first joint. The leg's frame has +x pointing
out along the leg, so a leg straight out to the right has a yaw of −90°.

The Explorer's middle legs are 0.118 m either side of the centre; its corner
legs are at 45°:

| Leg | Yaw | Position (m) |
|---|---|---|
| 0, front right | −45° | (0.119, −0.090, 0) |
| 1, middle right | −90° | (0, −0.118, 0) |
| 2, rear right | −135° | (−0.119, −0.090, 0) |
| 3, rear left | 135° | (−0.119, 0.090, 0) |
| 4, middle left | 90° | (0, 0.118, 0) |
| 5, front left | 45° | (0.119, 0.090, 0) |

The legs can be in any order, but the order is their number everywhere else:
in the gaits, the joint feedback and the engine's output.

### Joints

Each leg is a chain of up to `GAIT_CORE_MAX_JOINTS` (6 by default) revolute
joints, listed body to foot. Each `JointDefinition` has:

| Field | Meaning |
|---|---|
| `origin` | Where the joint is, from the previous joint (or the leg frame's origin, for the first), with every joint at zero (m) |
| `axis` | The axis it turns about, in the leg frame with every joint at zero |
| `lower`, `upper` | Its limits (rad) |
| `max_velocity` | The fastest its actuator may turn (rad/s). Unset, `GaitParams::max_joint_speed` applies. From v0.10.0 |
| `mass`, `centreOfMass` | The link this joint turns: its mass (kg) and centre of mass, in this joint's frame, as a URDF link's `<inertial>` gives them. Zero mass leaves it out. From v0.12.0 |

`foot` is the offset from the last joint to the foot, again with every joint
at zero. The Explorer's leg is a coxa, then a femur and a tibia:

```cpp
constexpr Scalar ST3215_SPEED  = 4.36;   // rad/s, coxa and tibia
constexpr Scalar STS3250_SPEED = 7.28;   // rad/s, femur
const JointDefinition coxa  = revolute(v(0,     0, 0), v(0, 0, 1), -1.4,  1.4,  ST3215_SPEED);
const JointDefinition femur = revolute(v(0.063, 0, 0), v(0, 1, 0), -1.6,  1.4,  STS3250_SPEED);  // + lowers the foot
const JointDefinition tibia = revolute(v(0.100, 0, 0), v(0, 1, 0), -0.78, 2.63, ST3215_SPEED);   // + folds the knee
const Vec3 foot = v(0.186, 0, 0);
```

So with every joint at zero the leg sticks straight out: the femur joint
0.063 m from the coxa, the tibia joint 0.100 m further, and the foot 0.186 m
beyond that. The femur and tibia limits match the URDF in hexapod-ros2.

The speed limits are each servo's no-load top speed on a 3S pack
(11.1 V). The URDF's figures, 4.7 and 7.9 rad/s, are for 12 V, which the
battery can't reach. The gait slows down so no joint goes faster than its
own limit, and if one still would, that joint is held to it and the foot
lags its target for that tick. Before v0.10.0, one limit,
`max_joint_speed` (4 rad/s), applied to every joint.

The angle limits matter too: a foot target that would need a joint outside them is
treated as out of reach, and the engine refuses or cuts short a move (a body
tilt, a sit, a height change) that would need one. A stack whose servos
can't reach the full range narrows them; the Servo2040 stack does this for
the Lite's hobby servos.

### Solvers

Each leg needs **kinematics**: the foot's position from the joint angles
(forward), the joint angles for a foot position (inverse), the Jacobian, and
the force on the foot from the joint torques. `LegDefinition::solver` says
which to use:

| Solver | For | How |
|---|---|---|
| `LegSolver::YawPitchPitch` | A yaw joint then two pitch joints in a line: a coxa, femur and tibia, as on the Explorer and most hexapods | Exact, closed-form. Fast |
| `LegSolver::Chain` (default) | Any chain | Exact forward kinematics and Jacobian; numerical inverse kinematics (damped least squares) |

The engine checks the leg's shape before it uses the analytic solver: the
first joint about +z at the leg's origin, the next two about +y spaced out
along +x, and the foot along +x, whatever the lengths. A leg that asks for
`YawPitchPitch` without that shape gets `Chain` instead; `legSolver(leg)`
says which each leg ended up with.

`ChainKinematics` handles any leg, but it's iterative: it finds the solution
nearest the joints' current angles, to within 1 µm, in a bounded number of
steps. It adapts its damping so nearly straight legs converge, and retries
from mid-range if the current angles don't. Use the analytic solver where it
fits; it's what the Explorer uses. A stack can also hand the engine its own
solver for a leg with `setLegKinematics(leg, &solver)`, as long as it
implements `LegKinematics` for the same number of joints.

For a leg with **more than three joints**, many poses put the foot in the
same place. After each solve, `ChainKinematics` eases the spare joints 2% of
the way towards the middle of their ranges, without moving the foot, so a leg
comes back to the same pose whatever path it took, rather than drifting
towards its limits. See [Beyond the hexapod](other-robots.md#four-joints-per-leg).

## Stance and stride

The **neutral stance** is where each foot stands, in its own leg's frame:
`standReach` out from the first joint and `standHeight` relative to the hips
(negative is below). It's shared by every leg, which suits a robot whose legs
are all alike. The Explorer stands each foot 0.177 m out and 0.134 m below
its hip.

`LegDefinition::stanceOffset` moves one leg's foot from that shared stance,
in its own frame. It's zero for the Explorer, whose legs all stand alike. A
robot whose legs don't, such as a quadruped whose rear feet stand further
back, sets it per leg. The offset is kept through height and reach changes,
sitting included.

`maxStride` is how far a foot moves from its neutral position while walking:
0.07 m on the Explorer (0.06 m before v0.11.0). With the shortest phase allowed
(`GaitParams::min_phase_duration`), it sets each gait's top speed.

## Gaits

A **gait** is a `GaitPattern`: each leg's timing in one walking cycle.

| Field | Meaning |
|---|---|
| `name` | Shown in logs and controllers, up to 15 characters |
| `numLegs` | How many legs it times. It must equal the robot's `numLegs` |
| `dutyFactor` | The fraction of the cycle each foot is on the ground, 0 to 1 |
| `offset[leg]` | The point in the cycle at which each leg lifts off, 0 to 1 |

Most gaits are easiest to write as **groups of legs that take turns**, which
`GaitPattern::fromGroups` turns into a duty factor and offsets. The
Explorer's three gaits:

```cpp
g.numGaits = 3;
g.gaits[GAIT_TRIPOD] = GaitPattern::fromGroups("tripod", {
    {LEG_FRONT_RIGHT, LEG_REAR_RIGHT, LEG_MIDDLE_LEFT},
    {LEG_MIDDLE_RIGHT, LEG_REAR_LEFT, LEG_FRONT_LEFT},
});
g.gaits[GAIT_RIPPLE] = GaitPattern::fromGroups("ripple", {
    {LEG_REAR_RIGHT, LEG_FRONT_LEFT},
    {LEG_MIDDLE_RIGHT, LEG_REAR_LEFT},
    {LEG_FRONT_RIGHT, LEG_MIDDLE_LEFT},
});
g.gaits[GAIT_WAVE] = GaitPattern::fromGroups("wave", {
    {LEG_REAR_RIGHT}, {LEG_MIDDLE_RIGHT}, {LEG_FRONT_RIGHT},
    {LEG_REAR_LEFT}, {LEG_MIDDLE_LEFT}, {LEG_FRONT_LEFT},
});
```

With *n* groups, each foot is down for (*n* − 1)/*n* of the cycle, and group
*k* lifts at *k*/*n*:

| Gait | Groups | Duty factor | Feet down at once |
|---|---|---|---|
| Tripod | 2 | 1/2 | 3 |
| Ripple | 3 | 2/3 | 4 |
| Wave | 6 | 5/6 | 5 |

A robot can have up to 8 gaits (`kMaxGaits`). Their order is their number:
the gait id a controller sends and ROS 2 lists, and the order the gamepad
cycles through them.

### What the engine can run

The engine steps in **phases**, each one swing long: in every phase one group
of legs swings while the rest stand. So it can run a pattern when:

- the duty factor is (*n* − 1)/*n*, for 2 to `numLegs` phases;
- every leg's offset falls on a phase boundary (a multiple of
  1 − duty factor);
- at least one leg lifts in every phase.

That covers tripod, ripple and wave for six legs, and trot and crawl for
four. `GaitPattern::schedule` says whether a pattern fits, and turns it into
the phases the engine runs. Patterns where swings overlap partly, rather than
in whole phases, can't be run.

## Centre of mass

The engine keeps the robot's centre of mass over its planted feet: see
[Stability & body sway](stability.md) and [Dynamic balance](dynamics.md).
There are two ways to describe it.

**Without masses** (`bodyMass` is 0, the default), `centreOfMass` is the
whole robot's, in the body frame, and fixed. It defaults to the body's
origin, a fair guess for a symmetric robot. A robot with a heavy battery or
payload off-centre should set it.

**With masses** (from v0.12.0), `bodyMass` is the body's mass without its
legs, `centreOfMass` is the body's own, and each joint's `mass` and
`centreOfMass` describe the link it turns. The engine adds up the links
where the joints put them, every tick, so the centre of mass moves as the
legs swing. `centreOfMass()` on the engine gives the live value. The masses
are also what [joint torques](dynamics.md#joint-torques) need.

The Explorer gives its masses, from the CAD, as `hexapod.inertials.xacro` in
hexapod-ros2 has them: 3.32 kg in all, two thirds of it in the legs. The
printed parts are taken at 60% of solid until they're weighed.

| Part | Mass | Centre of mass (m) |
|---|---|---|
| Body, without legs | 1.108 kg | (−0.0043, 0.0001, 0.0099), body frame |
| Coxa link, each | 0.093 kg | (0.020, 0, −0.0007), coxa joint frame |
| Femur link, each | 0.172 kg | (0.050, 0, 0), femur joint frame |
| Tibia link, each | 0.103 kg | (0.078, 0, 0.0128), tibia joint frame |

## What the engine checks

When a `GaitEngine` is built, it checks its robot. If any check fails,
`robotSupported()` is false: the engine **never moves and keeps the servos
off**, and logs an error every 5 seconds saying why.

| Check | Error |
|---|---|
| At least one leg and one gait | `Robot has no legs or no gaits: it won't move` |
| Every gait fits the robot's legs ([above](#what-the-engine-can-run)) | `Gait 1 (trot) can't run on this robot's 4 legs: it won't move` |
| Every gait keeps the centre of mass at least `min_stability_margin` inside the feet, swaying if it needs to ([Stability](stability.md)) | `Gait 0 (trot) can leave the centre of mass -60 mm from the edge of its feet …` |

`numLegs` above `GAIT_CORE_MAX_LEGS` is clamped to it, so build gait core
with a bigger maximum for a robot with more than six legs. Each leg's chain
is checked too: a leg asking for the analytic solver gets it only if it has
the right shape.

The stability check can be overridden with `allow_unstable_gaits`, for
example to try a quadruped's trot on a robot that can catch itself. The
others can't.

## Feeding it joint states

The engine's inputs and outputs are sized per leg. Each leg's angles in a
`JointFeedback` and a `GaitOutput` are a `JointVec` with that leg's number of
joints, body to foot.

`JointFeedback` starts each leg with three zeros, which suits the Explorer.
For legs with another number of joints, size each one to its leg's
`numJoints`: a leg whose angles have the wrong number of entries counts as
not reported. Mark legs whose servos aren't answering as not `present`,
rather than repeating stale angles, so the engine can tell the servos have
gone.

A robot with **foot contact sensors** sets `hasContact` and fills
`contact` per leg (from v0.12.0). The engine then uses them instead of the
contact it works out from joint efforts, everywhere it needs to know which
feet are down. The Explorer has none.

## A new robot, step by step

1. **Measure each leg** with every joint at zero: where each joint is from
   the last, which way it turns, its limits and its top speed. A URDF has
   all of this, and each link's mass too.
2. **Write the definition**, as a function returning a `RobotDefinition`.
   Start from `explorer.cpp`.
3. **Pick a solver** per leg: `YawPitchPitch` if the leg is a coxa, femur and
   tibia in a line, otherwise `Chain`.
4. **Give the masses**, if you know them, or at least set `centreOfMass`.
5. **Set the stance and stride** so every foot can reach its whole stride at
   every body height you'll use.
6. **List the gaits**, with `fromGroups`, so that the feet that stay down
   always surround the centre of mass.
7. **Run it in the test harness.** gait core's tests drive a robot through
   stand, walk, turn, stop, tilt, sit and power down with perfect simulated
   servos, checking every command against its joints' limits
   (`test/test_robots.hpp`). Add yours there and watch for warnings.
8. **Check `robotSupported()`** and the log on the real robot before it
   stands.

See [Beyond the hexapod](other-robots.md) for three robots built this way.
