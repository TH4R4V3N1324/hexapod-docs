# Servo2040

The Servo2040 stack runs the [Lite](../variants/lite/index.md) on a Pimoroni
[Servo 2040](https://shop.pimoroni.com/products/servo-2040), an RP2040 board
with 18 servo outputs. It drives **18 hobby PWM servos** with the same gait
as the ROS 2 stack, because both are built on [gait core](gait-core.md).

The code is in [hexapod-servo2040](https://github.com/TH4R4V3N1324/hexapod-servo2040).

## How it fits together

```
handheld ──ESP-NOW──► ESP32 receiver ──I²C──► Servo 2040 ──PWM──► 18 hobby servos
controller                           ◄─ status ──     ▲
                                                      └── ISM330DHCX IMU (same I²C bus)
```

An **ESP32 receiver** on the robot takes input from a handheld controller
over ESP-NOW, and passes the joysticks and commands to the Servo 2040 over
I²C. The Servo 2040 does everything else.

!!! warning "Controller on hold"
    The handheld controller and ESP32 receiver are a custom design in
    [hexapod-controller](https://github.com/TH4R4V3N1324/hexapod-controller), built for an earlier prototype. Work on them
    is paused, so there's currently no maintained way to drive the Lite. The
    Servo 2040's side of the I²C interface is ready for when there is. 100 times a
second, it:

1. reads the latest command packet from the ESP32;
2. reads the IMU;
3. runs gait core's gait engine and writes the 18 joint angles to the servos;
4. sends a status packet back to the ESP32.

If the ESP32 stops answering for 200 ms, the sticks are treated as centred,
so the robot stops walking.

## What's different from ROS 2

Hobby servos only take a position command: they can't report where they are
or how hard they're pushing. So the Servo2040 stack runs **open loop**:

- At boot it moves the legs into the seated pose and assumes they got there.
- With no foot forces, **load balancing** and **terrain adaptation** are off.
- **Leveling** still works, using the IMU. If the IMU is missing, the robot
  walks without leveling.
- Each leg has an input for a **contact switch**, but the gait doesn't use
  them yet.
- There's no mapping or navigation.
- There's **no servo power switch**. When gait core's
  [servo power](gait-core.md#servo-power) turns off, the firmware stops the
  PWM signals and the servos go limp. A stand command starts them again and
  seats the legs, as at boot, before standing. Since the servos never report
  anything, the check for lost servos is off.

Other gait settings keep the ROS 2 stack's defaults, except that the femur
and tibia limits are narrowed to what hobby servos can reach. See
[Gaits & modes](../concepts/gaits.md#support-by-stack).

## Wiring

### Servos

Each leg uses three servo outputs in a row: coxa, femur, then tibia.

| Leg | Coxa | Femur | Tibia |
|---|---|---|---|
| 1 (right front) | SERVO 1 | SERVO 2 | SERVO 3 |
| 2 (right middle) | SERVO 4 | SERVO 5 | SERVO 6 |
| 3 (right rear) | SERVO 7 | SERVO 8 | SERVO 9 |
| 4 (left rear) | SERVO 10 | SERVO 11 | SERVO 12 |
| 5 (left middle) | SERVO 13 | SERVO 14 | SERVO 15 |
| 6 (left front) | SERVO 16 | SERVO 17 | SERVO 18 |

See [Legs & joints](../concepts/legs.md) for the leg numbers.

### I²C

The ESP32 and the IMU share the Servo 2040's I²C bus. The Servo 2040 is the
controller on the bus, at 100 kHz.

| | |
|---|---|
| SDA | GPIO 20 |
| SCL | GPIO 21 |
| ESP32 receiver | Address `0x08` |
| ISM330DHCX IMU | Address `0x6A` |

Share a ground between the boards.

**Mount the IMU with its axes along the body's:** x forward, y left and z up
(see [Coordinate frames](../concepts/frames.md)).

## Building and flashing

You'll need the [Raspberry Pi Pico SDK](https://github.com/raspberrypi/pico-sdk)
**2.0.0 or newer**, CMake 3.16 or newer, and Ninja or Make. The easiest way to
get all of these is the Raspberry Pi Pico VS Code extension, which sets them up
from the project's `CMakeLists.txt`.

```bash
git clone https://github.com/TH4R4V3N1324/hexapod-servo2040.git
cd hexapod-servo2040
git submodule update --init
git -C external/hexapod-gait-core submodule update --init   # Eigen

cmake -S . -B build -G Ninja -DPICO_BOARD=pico
cmake --build build
```

To flash it, hold **BOOTSEL** on the Servo 2040 while plugging it in. It
appears as a drive called `RPI-RP2`. Copy `build/Hexapod.uf2` onto it, or
run `picotool load -x build/Hexapod.uf2`.

Diagnostics come over the USB serial port.

## First power-up

!!! danger
    Read [Safety](../getting-started/safety.md) first, and **support the
    robot so its legs are clear of the ground** before powering the servos.

Five seconds after boot, the firmware moves each leg in turn into the seated
pose, then stands up.

**Power down** (`CMD_POWER_DOWN` from the ESP32) sits the robot down, then
stops the servos. A stand command starts them again.

## Battery

The firmware can feed the Servo 2040's servo supply voltage and current to
gait core's [battery behaviour](gait-core.md#battery-behaviour), which warns
when it's low and sits down and stops the servos when it's critical.

It's **off by default** (`robot::BATTERY_MONITORING`): gait core's thresholds
are for a 3S pack and would sit a 6 V servo supply down at once. To use it,
set thresholds for your supply in `robot::batteryParams()` and turn it on.

## Calibration

Each joint has a calibration offset of up to ±60°, stored in the Servo 2040's
flash so it survives a power cycle. They're set with commands sent through
the ESP32 receiver:

- **Config mode** moves every servo to its centre plus its offset, so you can
  check each joint against its centre pose.
- **Set config** sets one joint's offset.
- **Request config** reads back one leg's offsets.

### Centre pose

The servo centres don't all match the [ROS 2 centre pose](ros2/setup.md#where-the-centre-is).
Hobby servos only turn ±90°, so the **tibia is centred at 90°** to reach its
working range:

| Joint | Servo at centre |
|---|---|
| Coxa | Pointing straight out from the body |
| Femur | Level |
| Tibia | At 90° to the femur |

Fit the servo horns with each servo at its centre, so the joint sits as close
to this pose as possible, then fine-tune with the offsets.

## Troubleshooting

- **Build error such as `'pio_set_gpio_base' was not declared`:** the Pico
  SDK is 1.x. Use 2.0.0 or newer, then delete the `build` folder, since CMake
  caches the SDK path.
- **`Eigen not found`:** run
  `git -C external/hexapod-gait-core submodule update --init`.
- **The robot doesn't move after boot:** if the servo outputs fail to start,
  the firmware stops after the boot wait and prints the reason over USB serial
  every 2 seconds.
- **The robot doesn't respond to the controller:** check the ESP32 receiver
  is at `0x08`, the SDA, SCL and ground wiring, and that its firmware uses the
  same packet layout as `includes/DataPacket.h`.
- **The robot doesn't level:** check the IMU is at `0x6A` and mounted with its
  axes along the body's. If it's missing, the firmware retries every second.
