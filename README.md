# ratgdo 2.5i on an ESP32-C5

This repo covers replacing the ESP-12F (ESP8266) on a **ratgdo 2.5i** with a
pin-compatible **ESP32-C5** module, and the ESPHome configs to run it. You get
dual-band Wi-Fi 6 and a current ESP-IDF stack, and the board itself needs no
other changes.

It has been done on two ratgdo 2.5i boards. One drives a Security+ 1.0 opener and
the other a Security+ 2.0 opener. Both run on 5 GHz at 98% signal from an AP in the
garage.

> **Not affiliated** with ratgdo, Konnected, Chamberlain or LiftMaster. Replacing the
> MCU means hot-air rework on your board. Do it at your own risk.

## What you need

| Part | Notes |
|---|---|
| **WT0132C5-S5** module (ESP32-C5) | Recommended. A drop-in for the ESP-12F footprint. |
| WT0132C6-S5 module (ESP32-C6) | The fallback: 2.4 GHz only, but the cleanest pin mapping. |
| Hot-air station, flux, solder wick | To remove the ESP-12F. |
| USB-serial adapter or a pogo-pin jig | For the first flash, if the board's auto-reset can't reach the new module (see below). |
| *Optional:* 220 µF 10 V low-ESR capacitor | Only if you see `brownout` resets. See [troubleshooting](docs/troubleshooting.md#brownout-resets). |

Ai-Thinker's **ESP-C5-12F** also fits but isn't recommended: the opener's RX line
lands on a strapping pin that must be high to flash. See
[pin mapping](docs/pin-mapping.md#why-the-wt0132-modules).

## C5 or C6?

- **Pick the C5** if you have a 5 GHz access point in or near the garage. The 5 GHz
  band is far less crowded, and the C5 falls back to 2.4 GHz anyway.
- **Pick the C6** if your garage only gets a usable 2.4 GHz signal, or you want the
  lowest-risk swap. Neither of its opener wires lands on a strapping pin.

A ratgdo sends tiny amounts of data, so the radio and the chip's maturity matter;
bandwidth and CPU speed don't.

## Steps

1. **Flash the module before soldering it on**, if you can. Pad 1 on these modules is
   a GPIO, not RST, so the ratgdo's USB auto-reset may not be able to reset it. After
   the first flash, every update goes over the air. See [hardware](docs/hardware.md).
2. **Swap the module.** Remove the ESP-12F with hot air and solder the C5 module on in
   the same orientation, antenna end matching. See [hardware](docs/hardware.md).
3. **Pick a firmware config** from [`firmware/`](firmware):

   | Config | Opener | Status |
   |---|---|---|
   | [`ratgdo-secplus2-wt0132c5.yaml`](firmware/ratgdo-secplus2-wt0132c5.yaml) | Security+ 2.0 (yellow learn button) | Working on hardware |
   | [`ratgdo-secplus1-wt0132c5.yaml`](firmware/ratgdo-secplus1-wt0132c5.yaml) | Security+ 1.0 | Working on hardware |
   | [`konnected-secplus-wt0132c5.yaml`](firmware/konnected-secplus-wt0132c5.yaml) | Either (auto-detect) | Working on hardware (Security+ 2.0). Needs a gdolib fix, see below |

4. **Check it in Home Assistant.** Confirm the door state, open/close, the light and
   obstruction. In the ESPHome integration's device options, turn on **Allow the
   device to perform Home Assistant actions** so a failed opener sync shows up as a
   notification. See [troubleshooting](docs/troubleshooting.md).

## Firmware options

**[esphome-ratgdo](https://github.com/ratgdo/esphome-ratgdo)** is the stock ratgdo
firmware. The configs here are the official `v25iboard` configs with the pins
remapped and an ESP-IDF `esp32:` block. Every ratgdo feature works, including the
dry-contact inputs and status outputs.

**Konnected's [`secplus_gdo`](https://github.com/konnected-io/konnected-esphome)**
is a rewrite that powers Konnected's GDO blaQ. It reads the opener through the
hardware UART instead of software serial, is event-driven, and auto-detects
Security+ 1.0 vs 2.0. It drops the dry-contact inputs and status outputs, and reads
obstruction over the protocol instead of the sensor wire. Its entity names here
match ratgdo's, so the Home Assistant entity IDs carry over when you switch.
On the C5 it needs a one-line build fix in gdolib for ESP-IDF 6, so the config
builds gdolib from a fork until that fix is merged upstream; see
[the Konnected notes](docs/konnected.md).

## Docs

- [Pin mapping](docs/pin-mapping.md): pad-by-pad table for the ESP-12F, the WT0132C5,
  the WT0132C6 and the ESP-C5-12F, plus the strapping-pin analysis.
- [Hardware swap](docs/hardware.md): removal, fitting, first flash, the pad 1 caveat.
- [Troubleshooting](docs/troubleshooting.md): sync failures, the Home Assistant
  actions permission, brownouts.
- [Konnected secplus_gdo](docs/konnected.md): switching component, known issues.

## Credits

- [ratgdo](https://github.com/ratgdo) by Paul Wieland and contributors, for the board
  and esphome-ratgdo.
- [Konnected](https://github.com/konnected-io) for `secplus_gdo` and gdolib.
- [argilo/secplus](https://github.com/argilo/secplus) for the Security+ protocol work.

The docs and configs in this repo are MIT licensed. The firmware they pull in keeps
its own license: esphome-ratgdo, Konnected's `secplus_gdo` and gdolib are GPLv3.
