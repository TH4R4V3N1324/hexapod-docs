# Setup

Setting up the ROS 2 stack on an [Explorer](../../variants/explorer/index.md),
from a fresh Pi to the first launch. Before you start, go through the leg hub
[checklist](../../variants/explorer/wiring.md#before-first-power-up).

Do these steps in order, and test each piece of hardware on its own before
running the full stack.

## 1. Set up the Pi

Start with a fresh Raspberry Pi OS install. Add a GitHub SSH key to the Pi,
clone the code to `~/hexapod-ros2`, then run the setup script once and
reboot:

```bash
git clone --recursive git@github.com:TH4R4V3N1324/hexapod-ros2.git ~/hexapod-ros2
~/hexapod-ros2/pi/setup_pi.sh
sudo reboot
```

The script:

- enables **I²C at 400 kHz** and the **servo UART** (`/dev/ttyAMA0`), and
  installs `i2c-tools` and `iw`;
- installs the udev rule that names the lidar **`/dev/rplidar`**;
- sets the Pi's hostname to **`hexapod`**;
- creates a **Wi-Fi hotspot** called `hexapod` at **192.168.50.1**, so the
  Steam Deck can connect directly without relying on a home router;
- installs **`hexapod-start.service`**, which starts the robot's software at
  every boot.

### The hotspot

The hotspot runs alongside the Pi's normal Wi-Fi connection on the same
radio. The Pi stays on your home Wi-Fi for internet and shares it with
whatever joins the hotspot.

The radio can only use one channel at a time, and the hotspot can't use 5 GHz
radar channels, so **keep the home Wi-Fi connection on 2.4 GHz**. The hotspot
then follows the home network's channel.

The hotspot password is generated on the first run. To choose or change it,
run the script with `HOTSPOT_PASSWORD`:

```bash
HOTSPOT_PASSWORD=yourpassword ~/hexapod-ros2/pi/setup_pi.sh
```

### Start at boot

At every boot, `hexapod-start.service`:

1. pulls the latest code (fast-forward only);
2. rebuilds the Docker image, picking up new ROS releases;
3. starts the container with `docker compose up -d`.

If the Pi is offline, it starts the image it already has. To see what it did:

```bash
journalctl -u hexapod-start
```

!!! warning
    The default container runs the stack on **mock hardware**
    (`hexapod_fake.launch.py`), so the servos won't move. Once the servos are
    wired up and calibrated, switch the container's command to
    `hexapod_robot.launch.py`.

## 2. Test the hardware

The code includes standalone test programs for each piece of hardware. Run
these in the container with `ros2 run hexapod_hardware_interface <test>`:

| Test | What it does |
|---|---|
| `test_servos` | Scan the bus, monitor, move or wiggle servos, and set servo IDs |
| `test_power` | Read both INA226 power monitors |
| `test_imu` | Read the IMU |
| `test_lidar` | Check the lidar |
| `test_leg` | Move one leg's foot using the real kinematics, straight on the servo bus |

`test_camera.sh` and `test_tof.py` run on the Pi itself, outside the
container.

!!! todo
    The hardware tests don't switch `SERVO_EN` on, so with the leg hubs the
    servos are unpowered while they run.

See the
[hardware tests README](https://github.com/TH4R4V3N1324/hexapod-ros2/blob/main/hexapod_ws/src/hexapod_hardware_interface/hardware_tests/README.md)
for how to use each test and a suggested bring-up order.

## 3. Calibrate the servos

Each joint needs a **direction**, a **centre** and **limits**. Calibrate one
servo at a time.

### Directions

Run `test_servos wiggle <id>` on each servo. It moves the joint +10°, back,
−10° and back, so you can see which joint it drives and which way positive
is. If a joint turns the wrong way, set its `direction` to `-1` in
`hexapod_description/urdf/hexapod.ros2_control.xacro`.

### Centre and limits

`test_servos calibrate <id>` sets a joint's centre and both of its limits in
one go:

1. With torque off, move the joint by hand through its **full travel**, then
   press Enter.
2. Hold the joint in its **centre pose** (below), then press Enter.
3. Confirm. The servo stores that pose as its centre and sets its limits 20
   steps inside the ends you reached, so it won't drive the joint into the
   body or its end stops. To change the margin, use
   `test_servos calibrate <id> <margin>`.

These settings are stored in the servo itself, so they survive a power
cycle, and the `offset` in `hexapod.ros2_control.xacro` can stay at 0.

### Where the centre is

The centre pose is the **leg laid out flat**:

| Joint | Centre |
|---|---|
| Coxa | Pointing straight out from the body |
| Femur | Level |
| Tibia | In line with the femur |

The tibia is curved, so "in line" means the **centre of the foot ball** lines
up with the femur, not the tibia's body. It's hard to judge by eye, so to
find the tibia's centre:

1. Lay the leg on a flat table, resting on its **femur joint**, with the
   **coxa pointing up**.
2. Lower the tibia until the **foot touches the table**.

That's the tibia's centre.

<!-- TODO: add pictures of each joint's centre pose from the CAD -->

### Fine-tuning

These are optional:

- `test_servos offset <id> <steps>` nudges a centre.
- `test_servos limits <id> min` or `limits <id> max` resets one limit by hand.
  You can also give it an angle in degrees.
- `test_servos centre <id>` re-centres a joint without changing its limits.
  The limits are counted from the centre, so they move with it: set them
  again afterwards.

### Check each leg

With a leg calibrated, run `test_leg limp <leg>`, then
`test_leg monitor <leg>`, and move the foot by hand:

- x should grow as you pull the foot outwards;
- y should grow as you swing it anticlockwise, seen from above;
- z should grow as you lift it.

If one goes the wrong way, flip that joint's `direction`.

Then **lift the robot so the leg hangs free** and run `test_leg test <leg>`.
It moves the foot through standing, in and out, side to side, up and down and
a full step, using the gait controller's kinematics, and shows how closely
the leg followed.

!!! note
    `test_leg` reads the servos' own encoders, so it catches a blocked joint,
    a limit or a loose horn, but **not** a joint turning the wrong way or an
    off-centre servo. Watch each move to check it goes the way it says.

## 4. First launch

With the servos calibrated, launch the full stack on the real hardware:

```bash
ros2 launch hexapod_bringup hexapod_robot.launch.py
```

| Argument | Default | |
|---|---|---|
| `hardware` | `st3215` | `st3215` for the real servos, `mock` for no hardware |
| `serial_port` | `/dev/ttyAMA0` | Servo bus port |
| `update_rate` | `100` | Control loop rate (Hz) |

On startup, the servos stay **unpowered**: the leg hubs' `SERVO_EN` is low.
When you tell the robot to **stand up**, for example with the Foxglove
panel's *Stand up* button, it:

1. switches servo power on and waits until all 18 servos answer, holding
   each one where it is as it does;
2. moves each leg in a straight line from wherever it is to the **seated**
   pose;
3. stands up.

It ignores movement commands until it's standing.

If the servos stop answering, for example because the E-stop was pressed,
the robot turns servo power off. **Releasing the E-stop doesn't move it:**
press *Stand up* again. See [Servo power](../gait-core.md#servo-power).

Servos that don't answer are logged and retried, one at a time, so they
can't stall the control loop. The IMU and power monitor drivers keep retrying
if their device is missing, so a loose connection recovers without a
restart.

### Battery

The `behaviour_manager` node watches both leg hubs' power monitors using gait
core's [battery behaviour](../gait-core.md#battery-behaviour). When the
battery is low it rumbles the gamepad; when it's critical it sits the robot
down and turns servo power off. Its settings are in
`hexapod_bringup/config/behaviour_manager.yaml`.

## 5. Check the lidar

Open RViz and put an object in front of the robot. It should appear at +x
(ahead of the robot). If the scan is rotated, adjust the yaw of `lidar_joint`
in `hexapod_description/urdf/hexapod.lidar.xacro`.
