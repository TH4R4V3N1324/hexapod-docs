# Stability & body sway

A walking robot lifts some feet and stands on the rest. If its weight isn't
over the feet that are down, it tips. Six legs hide this: the Explorer's
tripod always leaves a wide triangle under the body. Four legs don't, so from
v0.9.0 gait core checks it, refuses gaits that would tip the robot over, and
sways the body over its feet for gaits that need it.

All of this is **static** stability: whether the robot would stay up if it
froze at that moment. It's judged in the body frame from the commanded foot
positions, so it needs no IMU and works the same on every stack.

## The support polygon

Seen from above, the planted feet outline a convex polygon: the **support
polygon**. The robot stays up while its centre of mass is above it. The
**stability margin** is how far the centre of mass is inside it: the distance
to the nearest edge, which is how far it could be pushed before it tips.

```
      ●───────────●         ● planted foot
     /             \        ○ lifted foot
    /       ×       \       × centre of mass
   ●    ←margin→     ●      margin: from × to the nearest edge
    \               /
     ●─────────────○
```

| Margin | Meaning |
|---|---|
| Positive | Inside: stable, with that much to spare |
| Zero | On an edge: about to tip |
| Negative | Outside: it tips |

Two feet, or any number in a line, have no area to stand in, so their margin
is never positive. That's why a quadruped's trot, which stands on two
diagonal feet, can't be statically stable however it holds its body.

`gait_core/locomotion/support_polygon.hpp` has the maths, for any stack that
wants it:

| Function | Gives |
|---|---|
| `stabilityMargin(feet, count, point)` | How far a point is inside the feet's polygon |
| `deepestSupportPoint(feet, count, centre)` | The most stable point: the centre of the largest circle inside the polygon |
| `nearestStablePoint(feet, count, point, margin, nearest)` | The nearest point to another that's at least a margin inside |

## Where the engine uses it

### Checking each gait

When the engine is built, it works out each gait's **worst margin**: for every
phase, the margin of the feet that stay down, at their neutral stance, less
`maxStride` for how far they sweep while walking. If any gait's worst margin
is under `min_stability_margin` (10 mm by default), the robot is
[unsupported](robots.md#what-the-engine-checks): it never moves, and logs
which gait and by how much:

```
Gait 0 (trot) can leave the centre of mass -60 mm from the edge of its feet,
under min_stability_margin (10 mm): it won't move. Set allow_unstable_gaits
to run it anyway
```

`gaitStabilityMargin(gait)` reports the margin, and `unstableGait()` the
first gait that falls short. The Explorer's tripod, ripple and wave all keep
several centimetres, so it's never refused and never sways.

### Tilt mode

In tilt mode the body shifts and leans over planted feet. The engine stops a
body pose where the centre of mass would come within `min_stability_margin`
of the feet's edge, the same way it stops one a leg can't reach.

### While walking

If the planted feet ever leave the centre of mass outside them while walking,
the engine warns, at most every 5 seconds. That means feet planted somewhere
unexpected, or a gait allowed to be unstable.

## Body sway

Some gaits are stable only if the body moves. A quadruped crawl lifts one leg
at a time and stands on the other three, but the triangle they make often
doesn't reach under the body's centre. Moving the body over that triangle
first, before the leg lifts, makes it stable. That's **body sway**.

The engine decides once, when it's built, whether each gait sways: only if
it falls short without sway, and sway would rescue it. Gaits that are stable
without it, which includes all of the Explorer's, never sway. `gaitSways(gait)`
reports the decision.

A gait that sways walks like this:

1. **At the start of each phase**, with every foot still down, the body moves
   towards the nearest point `sway_margin` (20 mm) inside the feet that will
   stay down, at up to `sway_speed` (0.05 m/s).
2. **The next leg lifts** once the centre of mass is `min_stability_margin`
   inside them.
3. **During the swing** the sway keeps tracking the planted feet, since they
   slide back as the body walks forward.
4. **When the robot stops**, the sway eases back to centre with every foot
   down.

The phase waits for as long as the sway is still getting there. If it
arrives and still isn't stable, or after 5 seconds regardless, the engine
lifts the next leg anyway and warns how far short it was:

```
Body couldn't sway over the planted feet in 5.0 s (8 mm short): lifting the
next leg anyway
```

Sitting moves the sway back to centre first, while every foot is down, so the
robot isn't lowered onto feet spread around an off-centre body. The sway is
part of the walking pose the feet are commanded through, so leg odometry
counts it as the real body movement it is. `bodySway()` gives the current
sway.

## Settings

All are in `GaitParams`. The ones marked live can be changed while running.

| Setting | Default | |
|---|---|---|
| `min_stability_margin` | 0.01 m | How far inside the planted feet the centre of mass must stay, in every gait's phases and in tilt mode. Live |
| `allow_unstable_gaits` | `false` | Run gaits that fall short anyway, e.g. a quadruped trot. When off, a robot with one won't move |
| `body_sway` | `true` | Sway gaits that need it. When off, they count as unstable. Read at startup |
| `sway_speed` | 0.05 m/s | Fastest the body sways. Live |
| `sway_margin` | 0.02 m | How far inside the feet swaying aims for, at least `min_stability_margin`. Live |

The margins and sway are judged from the robot's `centreOfMass`, so set it
if the robot's weight isn't over its body origin.

## Limits

- **Static only.** Nothing here models momentum, so a trot, a fast walk or a
  push can still tip a robot whose margin looks fine, and a dynamic gait that
  relies on momentum is refused unless `allow_unstable_gaits` is set.
- **Flat ground.** The polygon is judged in the body frame, from where the
  feet are commanded, not where they landed, and doesn't account for slopes.
  The IMU-based [leveling](../stacks/ros2/walking.md) keeps the body level on
  a slope, which helps.
- **A guessed centre of mass** is only as good as the guess.
