# Training

!!! warning "Experimental"
    Training is a work in progress. The terrain adaptation task works, but
    not reliably yet, and there's no way to run a trained policy on the robot
    yet.

[hexapod-training](https://github.com/TH4R4V3N1324/hexapod-training) trains
the robot's skills with reinforcement learning in
[Isaac Lab](https://isaac-sim.github.io/IsaacLab/), rather than programming
them by hand.

## What it does now

The first task trains **terrain adaptation**: walking over uneven ground. The
robot's hand-written terrain adaptation is described in
[How it walks](ros2/walking.md#terrain-adaptation).

The policy doesn't learn to walk from scratch. The robot walks with the
tripod gait, and the policy adjusts it:

- **A foot residual:** it nudges each foot's target away from where the
  tripod gait puts it.
- **Swing lift and body height:** it sets how high each leg lifts on its
  swing, and how high the body rides.
- **What it sees:** besides the robot's own state, a height map of the
  ground around it, as time-of-flight distance sensors would measure it.
  The critic, used only in training, also gets ground truth the robot
  couldn't measure: its true body velocity and an exact height scan.

The terrain is a mix of flat ground, random roughness, slopes and stairs both
down and up, and boxes, each getting harder level by level. Robots move up a
level when they cover the ground they were commanded to, and back down when
they fall. The simulated servos use values identified from the 12 V
ST3215s.

The tripod gait here is a batched PyTorch port of the ROS 2 stack's gait,
from before [gait core](../gait-core/index.md) was split out, so it doesn't
yet follow gait core's changes.

## Where it's going

The aim is to train new skills in simulation and play them back on the real
robot. A **policy relay node** in hexapod-ros2 will run a trained policy and
send its commands to the robot. This
is still being built.

## Running it

The repo is a standard Isaac Lab task package, managed with
[uv](https://docs.astral.sh/uv/). From the repo root:

```bash
uv sync                                   # set up the environment
uv run isaaclab list_envs --show_presets  # list the tasks
uv run isaaclab train --rl_library <RL_LIBRARY> --task HexapodTraining-Locomotion-Hexapod
```

See the repo's
[README](https://github.com/TH4R4V3N1324/hexapod-training#readme) for the
physics and rendering options, and how to play back a trained policy.
