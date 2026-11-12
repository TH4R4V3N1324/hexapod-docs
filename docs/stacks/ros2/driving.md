# Driving

You drive the robot from a separate **operator** machine: a Steam Deck or a
laptop. The operator machine reads the gamepad and shows the robot's state in
Foxglove Studio. The robot does the rest.

## Connecting

=== "Steam Deck"

    The Steam Deck works as a walk-up client: launch the shortcut and it finds
    the robot and starts streaming the gamepad.

    **One-time setup.** In Konsole on the Deck, run:

    ```bash
    steamdeck/setup_distrobox.sh
    ```

    This creates a `jazzy` distrobox (Ubuntu 24.04 with ROS 2 Jazzy, the same
    as the robot) with `joy`, Foxglove Studio and the Hexapod panel. Running
    it again updates them. Then add `steamdeck/hexapod_launch.sh` to Steam as a
    non-Steam game. The `steamdeck/images/` folder has artwork for the
    shortcut.

    **Each time.** Join the robot's `hexapod` Wi-Fi hotspot (or the same
    network as the robot) and launch the shortcut. It:

    1. looks for the robot's Foxglove bridge on port 8765, first at the
       hotspot address (192.168.50.1), then on every address on the Deck's
       local network;
    2. starts reading the gamepad;
    3. opens Foxglove Studio full-screen, connected to the robot.

=== "Laptop (native)"

    You'll need Ubuntu 24.04 for ROS 2 Jazzy. On Windows, use a 24.04 WSL
    distro. From the root of the `hexapod-ros2` repo:

    ```bash
    ./init_operator.sh
    ```

    This sources ROS and any workspaces you've built, checks that
    `foxglove_bridge`, `joy`, `hexapod_interfaces` and `hexapod_description`
    are available (installing `foxglove_bridge` if it's missing), then starts
    the gamepad node, the Foxglove bridge and Foxglove Studio.

=== "Laptop (Docker)"

    ```bash
    cd operator_ws
    ./initHexOperator.sh
    ```

    This starts the operator container, waits for the Foxglove WebSocket on
    port 8765 and then opens Foxglove Studio.

!!! note
    The operator machine and the robot must use the same `ROS_DOMAIN_ID`, so
    the gamepad's `/joy` messages reach the robot.

## Gamepad controls

Buttons are named Xbox-first, with PlayStation in brackets. The Steam Deck's
controls follow the Xbox layout.

| Input | Walking (normal / strafe) | Tilt mode |
|---|---|---|
| Left stick | Forward/back, left/right | Shift the body forward/back, left/right |
| Right stick X | Turn | Roll |
| Right stick Y | — | Pitch (forward lowers the nose) |
| LT / RT [L2 / R2] | — | Yaw left / right |
| D-pad up / down | — | Raise / lower the body |
| Start [Options] | Next mode | Next mode |
| Back [Create] | Next gait | Next gait |
| A [Cross] | Terrain adaptation on/off | Terrain adaptation on/off |
| B [Circle] | Walk leveling on/off | Walk leveling on/off |
| X [Square] | Load balancing on/off | Load balancing on/off |
| Y [Triangle] | Tilt leveling on/off | Tilt leveling on/off |

Full stick is the robot's top speed for the current gait. The gamepad does
nothing until the robot's gait controller is up and has reported its limits.

To check a different controller's numbering, run `ros2 topic echo /joy` and
move each control. Axis and button numbers are parameters of
`controller_teleop` if you need to remap them.

## Keyboard control

For testing without a gamepad, `keyboard_teleop` drives the robot from a
terminal:

```bash
ros2 run hexapod_teleop keyboard_teleop
```

| Key | Action |
|---|---|
| `w` / `s` | Forward speed up / down |
| `a` / `d` | Sideways speed up / down |
| `q` / `e` | Turn speed up / down |
| `x` | Stop |
| `g` | Next gait |
| `m` | Next mode |
| `c` | Exit |

## Modes and gaits

The ROS 2 stack supports all three [modes and gaits](../../../../concepts/gaits.md).
With tripod, its top speed is about 0.17 m/s. A gait change takes effect at
the start of the next step, when every foot is planted. When entering tilt
mode, the legs first step back to their start positions; when leaving, the
body eases back to neutral before walking resumes.

## Features

Each of these can be switched on and off from the gamepad (see above), the
Foxglove panel, or a ROS service call:

| Feature | Default | What it does |
|---|---|---|
| **Terrain adaptation** | On | Each foot finds its own ground height, so the robot walks over uneven ground. |
| **Terrain map** | Off | Uses the depth camera to pick footholds ahead. Experimental: in simulation it only helped on slabs and walked worse on hills. |
| **Walk leveling** | On | Keeps the body level on slopes while walking. |
| **Tilt leveling** | On | In tilt mode, holds the body's roll and pitch steady even if the ground tilts underneath it. |
| **Tilt lock yaw** | Off | Also holds the heading in tilt mode. Off by default because the IMU's yaw drifts without a magnetometer. |
| **Load balancing** | On | While standing, spreads the weight evenly across the feet. |

For example:

```bash
ros2 service call /set_terrain_adaptation std_srvs/srv/SetBool "{data: false}"
```

See [How it walks](walking.md) for how these work.

## Body height

The body stands at 12 cm by default and can go from 6 to 16 cm. In tilt mode,
use the D-pad. Otherwise, call `/set_height` with the feet's height below the
hips, as a negative number:

```bash
ros2 service call /set_height hexapod_interfaces/srv/SetHeight "{height: -0.10}"
```

The body glides to the new height at 3 cm/s, standing or walking.

## Foxglove panel

The **Hexapod** Foxglove panel shows:

- body roll and pitch;
- which feet are on the ground, and how hard each one is pressing;
- power for each side;
- the current mode, gait and body height;
- switches for each feature.

It also has the *Stand up* button. The Steam Deck setup installs it
automatically. For a laptop, see the panel's
[README](https://github.com/TH4R4V3N1324/hexapod-ros2/tree/main/foxglove_panel)
to build and install it.

## Mapping and navigation

The robot can build a map with its lidar and navigate it. See
[Mapping & navigation](navigation.md).
