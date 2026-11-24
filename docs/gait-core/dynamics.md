# Dynamic balance

[Stability & body sway](stability.md) is about **static** stability: whether
the robot would stay up if it froze. That's enough for the Explorer, whose
gaits always leave a wide polygon of feet under it. It isn't enough for a
robot that walks fast, gets pushed, or uses a gait that's never statically
stable, like a quadruped's trot. From v0.12.0 gait core can also take the
robot's **momentum** into account.

!!! info "Everything here is off by default"
    Each feature is a `GaitParams` switch, and all of them default to off,
    so a stack that updates gait core walks as it did unless it turns them
    on. With them off, the engine still works out the balance state below,
    for `balanceState()` to report.

    They were developed and tested on simulated robots in MuJoCo, not on a
    built Explorer. On the Explorer's tripod, `dynamic_stability` changes
    nothing measurable.

## What the robot needs

Most of this needs to know where the robot's weight is and how it moves.

| Feature | Needs |
|---|---|
| [Dynamic stability](#dynamic-stability) | Nothing extra. Better with the robot's [masses](robots.md#centre-of-mass) |
| [Balance feedback](#balance-feedback) | An IMU with angular velocity and acceleration, and joint efforts or foot contact sensors |
| [Preview control](#preview-control) | A gait that [sways](stability.md#body-sway) |
| [Joint torques](#joint-torques) | The robot's masses, and actuators that take a torque as well as a position |
| [Foot placement](#foot-placement) | A gait that balances by stepping, and a good velocity estimate from outside the engine |

The IMU's angular velocity and linear acceleration come in with
`setImu`, in `ImuSample` (from v0.12.0). Both are in the body frame (+x
forward, +y left, +z up), so a stack rotates an IMU mounted another way.
Linear acceleration is what an accelerometer reads, gravity included: about
(0, 0, +9.81) m/s² with the body level and still, as `sensor_msgs/Imu`
carries it.

## The body as a pendulum

Every walking tick, `BalanceController` treats the robot as an **inverted
pendulum** over its planted feet: all its mass at the centre of mass *c*, a
height *h* above the feet. Two points summarise its motion:

| Point | Formula | Inside the feet means |
|---|---|---|
| **Capture point** ξ | *c* + *ċ* / ω | The robot can still come to rest over these feet |
| **Zero moment point** (ZMP) | *c* − *c̈* / ω² | The feet can push hard enough for this acceleration |

where ω = √(*g* / *h*). Static stability is the special case of a body
standing still, where both are just *c*.

`balanceState()` reports all of it each tick: the centre of mass, its
velocity and acceleration, *h*, ω, the capture point and ZMP, and how far
each is inside the planted feet. The centre of mass is the live one, with
the legs where they are, so it moves as they swing.

## Dynamic stability

`dynamic_stability` (from v0.12.0) puts that state to work, from what the
gait **commands**:

- The body speeds up and slows down **no harder than the planted feet can
  push for**: the ZMP stays `min_stability_margin` inside them.
- A leg **only lifts once the body's momentum fits the feet that stay down**:
  the capture point and ZMP are that far inside them. Until then the body
  slows just enough. `liftHeld()` says when a lift is waiting.

On a quadruped's crawl this keeps the ZMP within 2 mm of the feet, against
4 mm outside without it. Feet with no area between them, like a trot's
diagonal pair, have no inside to keep anything in, so these limits don't
hold such a gait back.

## Balance feedback

`balance_feedback` (from v0.12.0) reacts to the motion the robot **really
has**, not just what it was told, so it can recover from a shove:

1. **It measures the body's velocity.** `BodyVelocityEstimator` blends leg
   odometry, which doesn't drift, with the IMU, which is fast and sees the
   whole robot slide or tip, which leg odometry can't. Leg odometry is
   compared with the command it actually measured, a tick or two behind,
   so the gait's own motion is never taken for a push.
2. **It finds the feet that are pushing.** The support polygon is the feet
   the force estimates (or contact sensors) say are down, at least three,
   and their centre of pressure.
3. **It predicts the capture point** over the loop delay
   (`balance_delay`), since the error grows while the servos catch up:
   about 30% over 30 ms on the Explorer.
4. **It shifts the body** over its feet to correct the error, no harder than
   the feet can push for, up to `correction_max`.
5. **If the shift can't catch it, the robot steps**: swinging feet land
   where the capture point will be at touchdown, from standing if need be
   (`capture_stepping`).

It only acts on an error that stands out from the gait's normal wobble:
beyond `capture_deadband` plus three times the error's running RMS. For its
first 2 s after starting, it learns that wobble before acting, so soft
servos and normal rocking aren't taken for pushes. It doesn't shift or step
for a gait whose feet are ever just a line (`gaitHasSupportArea()` is false),
since those balance by their motion.

`balanceCorrection()`, `catchingStep()` and `stepOffset()` report what it's
doing.

In MuJoCo, on simulated dogs, a hexapod, a pentapod, an octopod and a
four-joint spider: statically stable gaits walk unchanged and brake 3–18%
harder after a shove, and a trot walks as it does without.

## Preview control

A swaying gait normally reacts at each phase: it waits, moves the body over
the next feet, then lifts. With `preview_control` (from v0.12.0), it
**plans the sway over the coming phases** instead, so the body is already
heading over the next feet before they're all it has.

`PreviewController` is ZMP preview control (Kajita's cart-table model). The
engine predicts where the feet will be from the gait's schedule and the
command, runs a ZMP reference through points inside both one phase's feet
and the next's, and the body follows the smoothest path that tracks it.
When consecutive phases' feet share too little, it waits with every foot
down while the body moves over. The old hold at each phase's start stays as
a safety net. Gaits that don't sway, like all the Explorer's, are untouched.

A quadruped's crawl walks 15% further in the same time, with more margin
and half the sway acceleration. ANYmal's crawl in MuJoCo, which falls
without it, completes its run.

## Joint torques

With `torque_output` (from v0.12.0), `GaitOutput::torques` gives a
**feedforward torque** for each joint, to add to position control on
actuators that take one:

- Each planted leg carries its share of the robot's weight, and of
  accelerating it along a planned path, as τ = −Jᵀ*f* at the measured joint
  angles.
- The shares put the centre of pressure under the centre of mass, or on the
  planned ZMP with preview control.
- Each foot's share ramps in and out over 50 ms as it lands and lifts.
  Swinging legs get zero.

`hasTorques` is false when there are none: off, no masses, or no foot down.
It needs the robot's [masses](robots.md#centre-of-mass). Foot contact
sensors, given in `JointFeedback::contact`, replace the contact estimated
from efforts everywhere, so only feet that really are down are loaded.

The Explorer's ST3215 and STS3250 servos take a position, not a torque, so
none of the stacks uses this.

## Foot placement

A trot, pace or bound balances by **where it steps**. With `foot_placement`
(from v0.13.0), each swinging foot lands further by the capture point of the
velocity the body has beyond what was commanded:
`foot_placement_gain` × (measured − commanded) / ω, smoothed over about a
phase and capped at a stride. `placementOffset()` reports it. Gaits that
stand on an area are left alone.

This is only as good as the velocity it's given. Leg odometry and the IMU
can't see a trot's feet slip, and with only those, it can make a trot
worse. `setBodyVelocity` takes the body's velocity from an estimator outside
the engine, such as visual-inertial odometry or motion capture. While that
is fresh (under 0.1 s old), balance feedback and foot placement use it
instead of their own estimate. On MuJoCo dogs given their true velocity, it
helps a little and never hurts. **Leave it off without an outside
estimator.**

## Settings

All are in `GaitParams`, and all can be changed while running.

| Setting | Default | |
|---|---|---|
| `dynamic_stability` | `false` | Limit acceleration and hold lifts by the ZMP and capture point |
| `com_acceleration_filter` | 0.05 s | Low-pass time constant of the centre of mass acceleration |
| `balance_feedback` | `false` | React to the measured motion: body shift and catching steps |
| `velocity_filter` | 0.2 s | How long the IMU's acceleration is trusted before leg odometry takes over |
| `balance_delay` | 0.03 s | From measuring to the feet acting on it |
| `capture_gain` | 0.5 | Share of the capture point error the body shift corrects, 0 to 1 |
| `capture_deadband` | 0.01 m | Capture point error always left alone |
| `correction_max` | 0.03 m | Furthest the body shifts over its feet to correct |
| `capture_stepping` | `true` | Step when the shift can't catch the error |
| `preview_control` | `false` | Plan a swaying gait's sway over the coming phases |
| `preview_time` | 1.0 s | How far ahead it plans, at most 200 ticks |
| `torque_output` | `false` | Fill `GaitOutput::torques` |
| `foot_placement` | `false` | Step by the capture point, for gaits that balance by stepping |
| `foot_placement_gain` | 1.0 | Share of that capture point offset the feet step by |
