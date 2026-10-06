# Hardware swap

## Before you start

- **Check your module's pads against [pin mapping](pin-mapping.md)** with a
  multimeter. A different revision could move pads.
- **Flash the new module before soldering it on** if you have a jig or can tack
  wires to it: 3V3, GND, TXD, RXD, plus IO28 (C5) or IO9 (C6) held to GND for
  download mode, and EN to reset. After that, every update is over the air.

## Removing the ESP-12F

1. Unplug the ratgdo from the opener and USB.
2. Flux all the castellated pads generously.
3. Use hot air (about 350–380 °C, medium airflow) and move it evenly around the
   module until all the pads reflow together, then lift the module off with tweezers.
   Shield nearby parts with Kapton tape if they're close.
4. Wick the pads flat and clean the flux off.

## Fitting the C5 / C6 module

1. Line the module up in the same orientation as the ESP-12F: antenna end over the
   board edge or keep-out, pad 1 at the same corner.
2. Tack two opposite corner pads, check the alignment, then drag-solder or iron each
   castellated pad.
3. Check for bridges, especially between neighbouring pads, and check continuity from
   pads 19/20 to the board's opener interface circuit.
4. Before connecting the opener, power from USB and confirm 3.3 V between pads 8
   and 15.

## Pad 1 is not reset

On the ESP-12F, pad 1 is RST. On the WT0132 and ESP-C5-12F modules it's a GPIO (IO0).
If the board's USB-serial auto-reset drives pad 1 instead of EN, it can't reset the
new module.

- **Simplest fix:** flash the module before fitting it, as above, then use OTA.
- **Otherwise:** reset it by hand. Briefly short EN (pad 3) to GND while holding the
  boot pin (pad 18) to GND, then release the boot pin.

## Power

The C5 draws short current spikes when it transmits on 5 GHz. If you see `brownout`
in the Reset Reason sensor, add a bulk capacitor. See
[troubleshooting](troubleshooting.md#brownout-resets).
