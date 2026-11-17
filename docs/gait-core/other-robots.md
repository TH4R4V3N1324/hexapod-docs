# Beyond the hexapod

The Explorer is the only robot gait core ships, but the engine isn't specific
to it. To prove that, the test suite builds three robots that each differ
from the Explorer in one way, and runs each through the same life: power up,
stand, walk, turn, stop, tilt, sit, power down and stand again. Every joint
command is checked against its leg's joint count and limits as it goes, and
the Explorer runs the same life as the control.

| Robot | Differs by | Result |
|---|---|---|
| [Quadruped](#quadruped) | Four legs, crawling | Does everything, without a warning |
| [Four joints per leg](#four-joints-per-leg) | An ankle | Does everything, without a warning |
| [Two joints per leg](#two-joints-per-leg) | No knee | Walks and turns; refuses, with reasons, what needs a foot at a chosen height |

None of them allocates while running. They live in `test/test_robots.hpp`
and `test/robot/test_other_robots.cpp`, and are the best place to start a
new robot from.

!!! note
    These robots exist only in simulation, inside the tests, with perfect
    servos. None has been built. They show the engine handles their shape,
    not that a real one would walk well.

## Quadruped

The Explorer's legs on the corners of a 0.24 m square body, crawling one leg
at a time:

```cpp
RobotDefinition quadruped() {
    RobotDefinition robot = gait_core::explorer::robot();
    const LegDefinition leg = robot.legs[0];
    const Scalar c = 0.12f;
    // Front right, rear right, rear left, front left
    const Scalar yaws[4] = {-kPi / 4, -3 * kPi / 4, 3 * kPi / 4, kPi / 4};
    const Scalar xs[4] = {c, -c, -c, c};
    const Scalar ys[4] = {-c, -c, c, c};
    robot.numLegs = 4;
    for (int i = 0; i < 4; ++i) {
        robot.legs[i] = leg;
        robot.legs[i].mount = {yaws[i], Vec3(xs[i], ys[i], 0)};
    }
    robot.numGaits = 1;
    // Rear right, front right, rear left, front left
    robot.gaits[0] = GaitPattern::fromGroups("crawl", {{1}, {0}, {2}, {3}});
    return robot;
}
```

A crawl stands on three feet at a time, and the triangle they make doesn't
reach under the body's centre, so it's only stable with the body moved over
them. The engine works that out when it's built and
[sways](stability.md#body-sway) the body over the three planted feet before
each leg lifts. The lift order puts each lift next to the last one's side,
so the body sways a short way each time.

A **trot** (two diagonal pairs) is refused unless `allow_unstable_gaits` is
set: two feet can't hold the body up, and no sway helps.

## Four joints per leg

The Explorer with an ankle: its tibia split into a 0.100 m shank and an
0.086 m foot link by a pitch joint, so each leg has four joints for three
dimensions. Same stance and reach.

```cpp
for (int i = 0; i < robot.numLegs; ++i) {
    LegDefinition& leg = robot.legs[i];
    leg.numJoints = 4;
    leg.joints[3] = revolute(v(0.100, 0, 0), v(0, 1, 0), -1.2, 1.2);   // ankle
    leg.foot = v(0.086, 0, 0);
    leg.solver = LegSolver::Chain;
}
```

It isn't a coxa, femur and tibia any more, so it runs on `ChainKinematics`,
the numerical solver.

A leg with more joints than it needs reaches each foot position in many
poses. Solving from wherever the leg was would keep whatever pose the path
left it in: after a walk, a leg came back to the same stance with joints up
to 0.19 rad (11°) apart, free to drift towards their limits. So after each
solve the spare joints are eased 2% of the way towards the middle of their
ranges, in the directions that don't move the foot. A stance now comes back
in the same pose whatever path the legs walked. Legs with three joints or
fewer have no spare directions and aren't affected.

## Two joints per leg

The Explorer with no knee: a coxa, then one pitch joint and a straight
0.176 m leg to the foot.

```cpp
for (int i = 0; i < robot.numLegs; ++i) {
    LegDefinition& leg = robot.legs[i];
    leg.numJoints = 2;
    leg.joints[0] = revolute(v(0, 0, 0), v(0, 0, 1), -1.4, 1.4);      // coxa
    leg.joints[1] = revolute(v(0.063, 0, 0), v(0, 1, 0), -1.6, 1.4);  // hip
    leg.foot = v(std::hypot(0.177 - 0.063, 0.134), 0, 0);
    leg.solver = LegSolver::Chain;
}
```

With two joints, the foot can't be put anywhere: it moves on a surface, where
each distance out from the hip has one height. The engine copes with what
that allows and refuses the rest:

| | |
|---|---|
| Walking and turning | ✅. Planted feet differ in height by up to about 10 mm as their reach changes through a stride |
| Startup | Stands it straight up, with a warning, since it can't take the seated pose |
| Sitting and power down | Refused, with the reason: sitting steps the feet out before lowering, and a foot can't step out at standing height |
| Tilt mode | Only small tilts, limited by reach, with a warning |
| Foot forces | Not estimated: two joints can't resolve a 3D force, so load balancing and terrain adaptation have nothing to work from |

## What it takes

Between them, these robots exercise every part of
[Describing a robot](robots.md):

- **A runtime leg count**, with every per-leg array sized at build time by
  `GAIT_CORE_MAX_LEGS`.
- **Any joint count**, sized by `GAIT_CORE_MAX_JOINTS`, with a `JointVec` per
  leg in and out of the engine.
- **The numerical solver** wherever the analytic one doesn't fit.
- **Gaits from the robot**, not the engine.
- **The support polygon and body sway** for a robot whose gaits aren't
  stable as they stand.

Running these robots also found bugs the Explorer never hit, all fixed in
v0.9.0: a quadruped lowered onto feet spread around a swayed body pushed a
femur past its limit, a fixed wait for the sway was too short after turning,
and the trajectories kept a stale copy of the stance after a height change.
