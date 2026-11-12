# Printing

The STLs (and STEP files for editing) are in
[hexapod-hardware/explorer](https://github.com/TH4R4V3N1324/hexapod-hardware/tree/main/explorer). The
[print list](https://github.com/TH4R4V3N1324/hexapod-hardware/blob/main/explorer/print-list.md) is the source of truth for which
parts to print and how many.

!!! warning "Draft"
    Print settings (layer height, walls, infill and orientation) are still
    to be confirmed.

## Materials

**Any rigid filament works for the structural parts** (PETG, ASA, PLA+). The
reference build uses PA12-CF for the legs for extra stiffness. There are two
exceptions:

- **Feet: TPU (required).** They need to flex for grip.
- **Tibia cartridges: PETG (required).** Their bearing fit was tuned in PETG.
  Print the **shanks** in PETG as well, since the rod fits are tuned as a pair.

Things to know:

- **Fits were tuned on one printer with the reference materials.** The same
  CAD fits differently in PETG and PA12-CF. If bearings or rods bind or are
  loose, adjust the clearance in the STEP file, or reprint in the reference
  material.
- **PLA is fine for testing, but use PETG or better for a robot you'll run
  for a long time.** PLA softens at around 55 °C. The servos get warm under
  load and sit right next to the coxa and femur.
- **The femurs take the most stress.** Print them in your stiffest material.

## What you'll print

- **Body:** body, lid and Pi shelf.
- **Camera pod:** shell and plate.
- **Legs**, six of each: leg mount and cap, coxa (two halves and a panel),
  femur (two halves and a panel), tibia, tibia cartridge, shank and foot,
  plus five joint caps per leg.
