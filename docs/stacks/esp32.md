# ESP32

The ESP32 stack runs the [Explorer](../variants/explorer/index.md)'s
**18 ST3215 serial bus servos** from an ESP32 dev board instead of a
Raspberry Pi. It's the middle ground between the two other stacks:

- like [ROS 2](ros2/index.md), it reads each servo's position and load, so it
  runs **closed loop**, with load balancing and terrain adaptation;
- like [Servo2040](servo2040.md), it's firmware on a microcontroller, driven
  by a handheld controller, with no lidar, cameras or mapping.

The gait comes from [gait core](gait-core.md), so it walks the same way as
the other stacks.

The code is in [hexapod-esp32](https://github.com/TH4R4V3N1324/hexapod-esp32).

## How it fits together

```
handheld ──ESP-NOW──► ESP32 ──UART──► Bus Servo Adapter ──► leg hubs ──► 18 ST3215 servos
controller ◄─ status ──  │ ▲                                  ▲          (position and load back)
                         │ └── I²C: IMU, leg hubs' INA226s ───┤
                         └──── SERVO_EN (GPIO 25) ────────────┘
```

The handheld controller talks to the ESP32 directly over ESP-NOW: there's no
separate receiver. 100 times a second, the ESP32:

1. takes the latest packet from the controller;
2. reads the IMU;
3. reads every answering servo's position and load in one go, runs gait
   core's gait engine, sets `SERVO_EN` from the engine's servo power, and
   writes the joint targets back in one go;
4. answers the controller with a status packet.

It reads the leg hubs' power monitors 10 times a second for the
[battery behaviour](#battery).

Reading and writing the servo bus takes about 4 ms of each 10 ms tick. If no
packet arrives for 200 ms, the sticks are treated as centred, so the robot
stops walking.

!!! warning "Controller on hold"
    The handheld controller is a custom design in
    [hexapod-controller](https://github.com/TH4R4V3N1324/hexapod-controller),
    built for an earlier prototype. Work on it is paused, so there's
    currently no maintained way to drive the robot with this stack. You can
    still set up and test the servos from the USB console.

## Compared with the other stacks

| | ROS 2 | ESP32 | Servo2040 |
|---|---|---|---|
| Servos | ST3215 | ST3215 | Hobby PWM |
| Brain | Raspberry Pi 5 | ESP32 | Servo 2040 |
| Feedback | Position and load | Position and load | None (open loop) |
| Load balancing, terrain adaptation | ✅ | ✅ | — |
| Leveling (IMU) | ✅ | ✅ | ✅ |
| Mapping and navigation | ✅ | — | — |
| Servo power switch | `SERVO_EN` | `SERVO_EN` | None (stops the PWM signals) |
| Battery behaviour | ✅ | ✅ | Off until set up |
| Driven by | Foxglove, Steam Deck | Handheld controller (ESP-NOW) | Handheld controller, through an ESP32 receiver |

All gait settings keep gait core's defaults, except that with a `SERVO_EN`
pin the servos stay off until the first stand command.

## Wiring

| | ESP32 pin | |
|---|---|---|
| Servo bus RX | GPIO 16 | To the bus adapter's TX. UART2 at 1 Mbaud |
| Servo bus TX | GPIO 17 | To the bus adapter's RX |
| I²C SDA | GPIO 21 | IMU and both leg hubs' INA226s, 400 kHz |
| I²C SCL | GPIO 22 | |
| SERVO_EN | GPIO 25 | Both leg hubs' J7 pin 4. `-1` in `RobotConfig.h` if there's no power switch |

- **Board:** any ESP32 dev board (`esp32dev`).
- **Bus adapter:** a Waveshare Bus Servo Adapter (A), as on the Explorer. If
  yours echoes what it sends back to RX, set `SERVO_ECHO` in
  `include/RobotConfig.h`.
- **Servo IDs:** the same as the Explorer's, **leg × 10 + joint**. See
  [Explorer wiring](../variants/explorer/wiring.md#servo-ids).
- **Leg hubs:** wire them as in [Explorer wiring](../variants/explorer/wiring.md#leg-hubs),
  with both boards' `SERVO_EN` on GPIO 25 and their I²C on GPIO 21 and 22,
  alongside the IMU. Set JP3 to 1–2 (3.3 V logic). The ESP32's own pull-ups
  are too weak for 400 kHz, so **bridge JP5 on one board**, unless something
  else on the bus, such as the IMU breakout, has pull-ups.
- **IMU:** ISM330DHCX at `0x6A`. Mount it with its axes along the body's: x
  forward, y left and z up (see [Coordinate frames](../concepts/frames.md)).
  Without it, the robot walks without leveling, and the firmware looks for it
  again once a second.

Share a ground between the ESP32, the adapter and the servo supply. Every
pin, address and setting is in `include/RobotConfig.h`.

## Building and flashing

You'll need [PlatformIO](https://platformio.org/), as the VS Code extension
or the CLI. The project pins its ESP32 platform version, so PlatformIO
fetches the right toolchain itself.

```bash
git clone --recursive https://github.com/TH4R4V3N1324/hexapod-esp32.git
cd hexapod-esp32
pio run -t upload
pio device monitor
```

`--recursive` matters: gait core is a submodule, and Eigen is a submodule
inside it. In an existing clone, run
`git submodule update --init --recursive`.

### Pair the controller

The firmware only accepts packets from the controller whose MAC address is
in `CONTROLLER_MAC` in `include/RobotConfig.h`. It prints the address of any
other sender over USB serial (`Ignored a packet from ...`), so you can copy
it in and flash again.

## Servo setup

New ST3215s all come as ID 1. Each joint's ID, centre and limits are stored
in the servo itself, and set from the firmware's USB serial console
(115200 baud, `pio device monitor`). The steps match the
[ROS 2 calibration](ros2/setup.md#3-calibrate-the-servos), and so does the
[centre pose](ros2/setup.md#where-the-centre-is): the leg laid out flat.

1. **IDs:** connect the servos **one at a time**. Run `scan all` to find each
   one, then `set-id 1 <id>`, for example `set-id 1 23` for leg 2's tibia.
2. **Check:** with all 18 connected, `scan` should list them all with
   sensible voltages.
3. **Directions:** `wiggle <id>` moves a joint +10°, back, −10° and back. If
   a joint turns the wrong way, flip its entry in `JOINT_DIRECTION` in
   `include/RobotConfig.h`.
4. **Centres and limits:** `calibrate <id>`. Move the joint through its full
   travel and press Enter, hold it in its centre pose and press Enter, then
   confirm. The limits are set 20 steps inside the ends you reached.
5. **Start:** `run` restarts the firmware into the gait. The servos stay off
   until a stand command.

`help` lists every command, including `move`, `limp`, `centre`, `offset` and
`limits`.

Servo commands work whenever the gait isn't powering the servos, such as
after boot before the first stand. They stop the gait and switch servo power
on for setup, until you type `run`. While the gait is powering the servos,
only `help`, `status`, `stop` and `run` work.

!!! danger
    `stop` turns the servo torque off, so **the legs drop**.

### Fine-tuning from the controller

The controller can also set an offset of up to ±60° per joint, stored in the
ESP32's flash and added on top of each servo's own centre. **Config mode**
moves every joint to its centre plus its offset, so you can check it against
the centre pose. On the console, `offsets` shows them and `offsets clear`
resets them.

## First power-up

!!! danger
    Read [Safety](../getting-started/safety.md) first, and **support the
    robot so its legs are clear of the ground** before powering the servos.

The servos stay **unpowered** after boot. On a stand command from the
controller, the firmware switches servo power on, holds each servo where it
is as it answers, moves the legs from wherever they are into the seated
pose, and stands up.

### Servo power

- **Losing the servos,** for example to the E-stop: with no full reading from
  the servos for half a second, the robot turns servo power off. **Releasing
  the E-stop doesn't move it.** A stand command powers the servos up and
  starts up again from wherever the legs are.
- **Power down** (`CMD_POWER_DOWN` from the controller) sits the robot down,
  then turns servo power off once it's seated.
- A servo counts as **not answering** after 3 missed replies in a row, and
  its leg is reported missing. Only one silent servo is tried each tick, so
  unpowered servos can't stall the loop. A servo that answers again is held
  where it is until it gets a new target, so it never jumps.

See [Servo power](gait-core.md#servo-power) in gait core.

### Battery

The firmware reads both leg hubs' INA226s and runs gait core's
[battery behaviour](gait-core.md#battery-behaviour): below 10.5 V it prints a
warning on the serial monitor, and below 9.9 V it sits down and turns servo
power off. A stand command overrides it. Without leg hubs, nothing happens.

The thresholds are starting points for a 3S pack: tune them in
`robot::batteryParams()` against real readings under load. `status` on the
console shows each hub's voltage and current.

## Troubleshooting

- **`fatal error: Eigen/Dense: No such file or directory`:** the nested
  submodule is missing. Run `git submodule update --init --recursive`.
- **The robot doesn't stand, and `status` shows fewer than 18 servos
  answering:** check the E-stop and kill loop, `SERVO_EN` on GPIO 25, the
  adapter's TX and RX against GPIO 16 and 17, and the IDs with `scan all`.
- **`INA226 not found` in `status`:** check the leg hubs' JP1 (left open,
  right bridged), JP3 on 1–2, and the I²C pull-ups (JP5).
- **The robot ignores the controller:** look for `Ignored a packet from ...`
  on the monitor, and put that address in `CONTROLLER_MAC`.
- **The robot doesn't level:** check the IMU is at `0x6A`, on GPIO 21 and
  22, and mounted with its axes along the body's.
