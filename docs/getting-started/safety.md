# Safety

!!! info "Work in progress"
    This page will cover battery handling, emergency stops and first
    power-up in detail.

A walking robot can move suddenly and pinch fingers, and its battery can
deliver a lot of current. Until this page is complete:

- **Support the robot** so its legs can't touch anything the first time you
  power it, and whenever you flash new firmware.
- **Test one thing at a time:** one servo, then one leg, then one mode.
- **Fit an E-stop** that cuts servo power, and check that it works before
  the robot walks. The Explorer's leg hubs have a kill loop for this.
- **Fuse the battery** and use the correct charger for the pack. Never leave
  a lithium pack charging unattended.
- **Calibrate every joint's centre and limits** before letting the robot
  take its own weight, so no joint can be driven into the body or its end
  stops.
