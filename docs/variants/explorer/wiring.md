# Wiring

!!! tip "Files"
    The board design is in [hexapod-hardware/electronics/leg-hub](https://github.com/TH4R4V3N1324/hexapod-hardware/tree/main/electronics/leg-hub).

## Servo IDs

Legs are numbered as in [Legs & joints](../../../../concepts/legs.md). Each servo's ID is **leg × 10 + joint**, where coxa = 1, femur = 2 and
tibia = 3:

| Leg | Coxa | Femur | Tibia |
|---|---|---|---|
| 1 (right front) | 11 | 12 | 13 |
| 2 (right middle) | 21 | 22 | 23 |
| 3 (right rear) | 31 | 32 | 33 |
| 4 (left rear) | 41 | 42 | 43 |
| 5 (left middle) | 51 | 52 | 53 |
| 6 (left front) | 61 | 62 | 63 |

New ST3215s all ship with the same ID, so **set each servo's ID before
connecting it to the bus with the others**. The `test_servos` tool can do
this (see [First power-on](../../stacks/ros2/setup.md#2-test-the-hardware)).

## Leg hubs

The [leg hub README](https://github.com/TH4R4V3N1324/hexapod-hardware/tree/main/electronics/leg-hub#connectors) has the full
connector pinouts and jumper settings. In short:

| Connector | Connects to |
|---|---|
| J1 (XT30) | Battery |
| J2 (Micro-Fit, 2-pin) | 5 V out to the Pi (or peripherals) |
| J3–J5 (SPOX, 3-pin) | Legs 1–3 of that side |
| J6 BUS_IN (SPOX, 3-pin) | Bus Servo Adapter |
| J7 CTRL (JST PH, 5-pin) | Pi: SDA, SCL, INA ALERT, SERVO_EN, GND |
| J8 KILL (JST PH, 2-pin) | E-stop (normally closed) |
| J9, J10 (Qwiic) | I²C daisy-chain |
| J11 PWR_SW (JST PH, 4-pin) | Power switch and its light |

### Wiring the pair

- **Battery:** split the pack to both boards' J1 with **equal-length
  wires**.
- **Power switch:** wire one latching switch to **both boards' J11 pins 1–2
  in parallel**. For an illuminated switch, wire its light to pins 3 (+) and
  4 (−) on **one** board only.
- **E-stop:** use a button with two normally-closed contacts, and wire one
  contact across each board's J8.
- **SERVO_EN:** drive it from a Pi GPIO that's **pulled down at boot**
  (GPIO9–27), so the servos stay off while the Pi starts.
- **Never tie the two boards' 5 V outputs together.** Watch out for this
  through USB VBUS, if the Pi and a USB device are powered from different
  boards.

!!! todo
    Driving `SERVO_EN` isn't implemented in any stack yet.

### Before first power-up

- [ ] JP1 open on the **left** board (`0x40`), bridged on the **right** (`0x41`)
- [ ] JP3 on 1–2 (3.3 V logic, for the Pi and Qwiic)
- [ ] JP2, JP4, JP5 and JP6 open. JP2 and JP6 bypass the kill loop and the
  power switch, for bench use only. JP4 and JP5 aren't needed with a Pi.
- [ ] XT30 and bus adapter cable pin 1 checked against your leads
- [ ] Fuses F1–F3 fitted (5 A mini blade)

## Servo bus

All 18 servos share one serial bus at 1 Mbaud. The Pi's serial port drives
the Bus Servo Adapter, which connects to J6 (BUS_IN) on both leg hubs. Each
leg hub shares the bus out to its three legs on J3–J5.

The leg connectors use the **same pinout as the servos** (1 DATA, 2 V+,
3 GND), so a straight-through servo cable works. The leads are 3-core flat
silicone cable with Molex Mini-SPOX connectors.

!!! warning
    Don't power servos or anything else from the Bus Servo Adapter. Its
    power feed from the leg hub is unfused and only meant for the adapter's
    own chips. Servo power reaches the legs only through J3–J5.

## I²C

The IMU and both leg hubs' INA226 power monitors share the Pi's I²C bus
(`/dev/i2c-1`), running at 400 kHz. Chain them **Pi → left leg hub → right
leg hub → IMU**, through J7 or the Qwiic ports. The IMU must be at address
`0x6A`. The addresses the software expects are set in
`hexapod_hardware_interface/config/sensors.yaml`.

## Lidar and cameras

- **Lidar:** USB. The Pi setup script installs a udev rule so it always
  appears as `/dev/rplidar`.
- **Cameras:** one 22-pin to 15-pin camera cable each (300 mm; the route is
  about 200 mm) to the Pi 5's CSI ports.
