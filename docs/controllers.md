# Controllers

How you drive a hexapod depends on its control stack:

| Stack | Drive it with |
|---|---|
| [ROS 2](stacks/ros2/index.md) | A Steam Deck or laptop running Foxglove, or a gamepad plugged into or paired with the Pi. See [Driving](stacks/ros2/driving.md). |
| [ESP32](stacks/esp32.md) | The OLED controller, straight to the robot; or a Bluetooth gamepad through a receiver |
| [Servo2040](stacks/servo2040.md) | The OLED controller or a Bluetooth gamepad, through a receiver |

Every stack uses the same [gamepad controls](#gamepad-controls). The rest of
this page covers the two microcontroller stacks. Their firmware is in [hexapod-controller](https://github.com/TH4R4V3N1324/hexapod-controller),
and the controller PCBs are in
[hexapod-hardware/controllers](https://github.com/TH4R4V3N1324/hexapod-hardware/tree/main/controllers).

## Gamepad controls

A gamepad drives **every stack the same way**: the layout is code in
[gait core](stacks/gait-core.md#teleop), run by ROS 2's `controller_teleop`
and by the receiver. Buttons are named Xbox-first, with PlayStation in
brackets, and go by position, so a Switch or 8BitDo gamepad uses the same
places. The Steam Deck follows the Xbox layout.

| Input | Walking (normal / strafe) | Tilt mode |
|---|---|---|
| Left stick | Forward/back, left/right | Shift the body forward/back, left/right |
| Right stick X | Turn | Roll |
| Right stick Y | — | Pitch (forward lowers the nose) |
| LT / RT [L2 / R2] | — | Yaw left / right |
| D-pad up / down | Body height up / down 1 cm (repeats while held) | Raise / lower the body |
| RB [R1] | Sit down / stand up. Hold 2 s to power down | The same |
| Start [Options] | Next mode: normal, strafe, tilt | The same |
| Back [Create] | Next gait: tripod, ripple, wave | The same |
| A [Cross] | Terrain adaptation on/off | The same |
| B [Circle] | Walk leveling on/off | The same |
| X [Square] | Load balancing on/off | The same |
| Y [Triangle] | Tilt leveling on/off | The same |

- **RB**, on release, sits the robot down if it's standing, and stands it up
  otherwise. It's how you stand the robot up after boot. **Held for 2
  seconds**, it powers down instead: the robot sits, then turns servo power
  off. The next press stands it up and powers the servos again.
- **The D-pad** moves the body 1 cm per press, and every 0.3 s while held,
  within 6–16 cm. It does nothing while the robot is seated.
- **A, B, X and Y** follow what the robot reports is on, so they never undo
  a change made somewhere else. The Servo2040 stack has no servo feedback,
  so it refuses to turn on terrain adaptation and load balancing.
- **Full stick** is the robot's top speed for the current gait.

Config mode and calibration on the microcontroller stacks are left to the
OLED controller.

!!! warning "Untested"
    The OLED controller was built for an earlier prototype. It now uses the
    current packet layout, but it hasn't been tested with either robot stack
    yet. The receiver's gamepad support builds, but hasn't been tested on
    hardware.

## How it fits together

```
ESP32 stack, handheld:   OLED controller ──ESP-NOW─────────────────────────► robot's ESP32
ESP32 stack, gamepad:    gamepad ──Bluetooth──► receiver ──ESP-NOW relay──► robot's ESP32
Servo2040 stack:         OLED controller ──ESP-NOW──┐
                         gamepad ──Bluetooth────────┴► receiver ──I²C──► Servo 2040
```

A controller sends the robot its sticks and a command every 10 ms, and the
robot answers each packet with its status: its gait, mode and height,
whether it's seated, whether the servos are powered, and which features are
on. The receiver turns a gamepad into the same packets, so the robot can't
tell the difference.

The packets are defined once, in gait core's
[protocol](stacks/gait-core.md#protocol), which every controller and robot
includes. Each packet starts with a protocol version, and each end drops
packets of another version or size. So when you update one end, update the
other to the same gait core version.

- **One input at a time.** Whichever of the OLED controller and the gamepad
  is driving keeps control while it's connected. The other only takes over
  when it's lost. A second gamepad is refused.
- **Failsafe.** If the OLED controller goes quiet for 250 ms, or the gamepad
  disconnects, the receiver centres the sticks, so the robot stops. It keeps
  the body height, so the robot doesn't drop. The robots also stop by
  themselves after 200 ms without a packet.

## OLED controller

A handheld controller with two thumbsticks, a rotary encoder, four shoulder
switches and a 128×64 OLED, built around an ESP32-S2. It runs from a single
Li-ion cell, charged over USB-C.

- **PCB:** [OLED controller](https://github.com/TH4R4V3N1324/hexapod-hardware/tree/main/controllers/oled-controller),
  rev v3. There's also a larger alternative layout with no IMU or current
  sensor.
- **Firmware:** `oled-controller/` in hexapod-controller, built with
  [PlatformIO](https://platformio.org/):

    ```bash
    git clone --recursive https://github.com/TH4R4V3N1324/hexapod-controller.git
    cd hexapod-controller/oled-controller
    pio run -t upload
    ```

It walks the robot, changes gait and mode, and sets the body height with the
encoder. It's also how you fine-tune the joint centres on the microcontroller
stacks: its config page puts the robot in config mode and sets each joint's
offset.

## Receiver

A stock **classic ESP32** dev board, such as an ESP32 DevKit, with no custom
hardware. It needs a classic ESP32: the S2 has no Bluetooth, and the S3 and
C3 only have Bluetooth Low Energy, which leaves out PlayStation and Switch
Pro controllers.

- **Servo2040 stack:** connect the receiver's GPIO 21 (SDA) and GPIO 22 (SCL)
  to the Servo 2040's GPIO 20 and 21, and share a ground. See
  [Servo2040 wiring](stacks/servo2040.md#i2c).
- **ESP32 stack:** the receiver only needs power. It relays packets to the
  robot's ESP32 over ESP-NOW.

The firmware is `receiver/` in hexapod-controller, built with
[ESP-IDF](https://docs.espressif.com/projects/esp-idf/) **5.3**:

```bash
cd hexapod-controller/receiver
idf.py set-target esp32       # first time only
idf.py -p PORT build flash monitor
```

## Pairing

### ESP-NOW

Each ESP-NOW device only talks to the MAC addresses built into its firmware,
so set these and flash each end:

| Firmware | Setting | Set it to |
|---|---|---|
| OLED controller | `receiverMAC` in `include/DataPacket.h` | The robot's ESP32 (ESP32 stack), or the receiver |
| Receiver | `CONTROLLER_MAC` in `main/Config.h` | The OLED controller, or all zeros for none |
| Receiver | `ROBOT_MAC` in `main/Config.h` | ESP32 stack: the robot's ESP32. Servo2040 stack: all zeros |
| hexapod-esp32 | `CONTROLLER_MAC` in `include/RobotConfig.h` | The OLED controller, or the receiver when it's relaying |

The receiver prints its own MAC address at boot. The receiver and the ESP32
stack both print the address of any sender they ignore
(`Ignored a packet from ...`), so you can copy addresses from the serial
monitor.

### Gamepads

The receiver pairs most Bluetooth gamepads: DualShock 4, DualSense, Xbox
Wireless (on controller firmware 5.x or later), Switch Pro, Joy-Con, 8BitDo
and others. See Bluepad32's
[supported controllers](https://bluepad32.readthedocs.io/en/latest/supported_gamepads/).

1. Press the receiver's **BOOT** button. The blue LED blinks quickly for 60
   seconds while it's ready to pair. A receiver that has never paired starts
   ready by itself.
2. Put the gamepad in pairing mode:
    - **DualShock 4 / DualSense:** hold Share (Create) and PS until the light
      flashes.
    - **Xbox:** hold the pair button on top until the Xbox button flashes
      quickly.
    - **Switch Pro:** hold the sync button on top.
    - **8BitDo:** see its manual.
3. It connects, rumbles and is remembered. From then on it reconnects when
   you turn it on.

Only remembered gamepads can connect outside the pairing window, so nobody
nearby can take over the robot. **Hold BOOT for 5 seconds** to forget every
paired gamepad.

## Troubleshooting

- **The robot ignores the controller:** check the MAC addresses in the
  pairing table, and look for `Ignored a packet from ...` on the robot's or
  receiver's serial monitor.
- **`Ignored a ... packet` with a size or version mismatch:** the two ends
  are on different gait core versions. Update gait core in both and rebuild.
- **The robot stops every few seconds:** the receiver's failsafe is tripping
  (`Failsafe: ...` on its monitor). Move the controller closer.
- **A gamepad won't pair:** check the LED is blinking quickly (press BOOT),
  and that the gamepad is in pairing mode, not just on. If it was paired with
  something else, it may connect to that first.
- **A paired gamepad won't reconnect:** hold BOOT for 5 seconds to forget
  every gamepad, then pair it again.

See the
[hexapod-controller README](https://github.com/TH4R4V3N1324/hexapod-controller#readme)
for building the firmware and the touch controller, an unfinished redesign.
