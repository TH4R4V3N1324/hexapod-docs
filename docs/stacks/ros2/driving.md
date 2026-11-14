# Driving

You drive the robot with a gamepad, either:

- on a separate **operator** machine, a Steam Deck or a laptop, which reads
  the gamepad and shows the robot's state in Foxglove Studio; or
- [plugged into or paired with the Pi](#gamepad-on-the-robot), with no
  operator machine at all.

The robot does the rest.

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

## Gamepad on the robot

You don't need a Steam Deck or laptop: the robot reads a gamepad connected to
the Pi itself, with the same [controls](../../controllers.md#gamepad-controls). Any controller SDL knows
works, including Xbox One and Series, DualShock 4, DualSense, Switch Pro and
8BitDo.

- **USB:** plug it in, even while the robot is running.
- **Bluetooth:** put the controller in pairing mode, then run this on the Pi
  (not in the container):

    ```bash
    ~/hexapod-ros2/pi/pair_controller.sh
    ```

    The script lists how to enter pairing mode on common controllers. The Pi
    trusts the controller, so from then on turning it on reconnects it. If
    its name isn't recognised, pass its address:
    `pair_controller.sh AA:BB:CC:DD:EE:FF`.

`ros2 topic echo /joy` shows whether it's being read. Launch with
`joy:=false` to turn it off.

!!! warning
    Don't drive with a gamepad on the Pi and the Steam Deck at the same time.
    Both publish `/joy`, and the robot follows whichever spoke last.


## Gamepad controls

The gamepad layout is the same on every stack: see
[Gamepad controls](../../controllers.md#gamepad-controls). In short: the
sticks walk and turn, **RB [R1]** sits and stands (hold 2 s to power down),
Start and Back change the mode and gait, the D-pad sets the body height, and
A, B, X and Y switch the features below.

The gamepad does nothing until the robot's gait controller is up and has
reported its limits.

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

The ROS 2 stack supports all three [modes and gaits](../../concepts/gaits.md).
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

The body stands at 12 cm by default and can go from 6 to 16 cm. Use the
D-pad, or call `/set_height` with the feet's height below the hips, as a
negative number:

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
