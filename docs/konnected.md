# Konnected secplus_gdo on the C5

[`firmware/konnected-secplus-wt0132c5.yaml`](../firmware/konnected-secplus-wt0132c5.yaml)
runs Konnected's `secplus_gdo` ESPHome component, the firmware behind Konnected's
GDO blaQ, instead of esphome-ratgdo.

## Which repo

Use the component from
[`konnected-io/konnected-esphome`](https://github.com/konnected-io/konnected-esphome)
(`components/secplus_gdo`), together with
[gdolib](https://github.com/konnected-io/gdolib), Konnected's protocol library, as an
ESP-IDF component.

Don't use the older standalone repo `konnected-io/esphome-secplus-gdo`. It was last
updated in March 2024 and ships gdolib as a precompiled Xtensa-only binary, so it
can't link for the RISC-V C5 or C6.

## Differences from esphome-ratgdo

- Reads the opener through the **hardware UART** (UART1) instead of software serial,
  and is event-driven.
- **Auto-detects Security+ 1.0 vs 2.0.** A select entity can force one; the config
  sets it from the `secplus_protocol` substitution.
- **Obstruction comes from the protocol,** so the ratgdo's obstruction wire input
  (GPIO6) is unused.
- **No dry-contact inputs or status outputs.**
- **Re-sync button:** picks a new client ID, resets the rolling code and reboots. Use
  it if the opener won't respond after a switch.
- **Entity names match esphome-ratgdo's,** so switching an existing device keeps its
  Home Assistant entity IDs.

## Known issues

- **gdolib v1.1.2 doesn't build on ESP-IDF 6.x.** Until the fix is merged upstream,
  the config builds gdolib from a fork (see [the gdolib fork](#the-gdolib-fork)); the
  fix is [konnected-io/gdolib#41](https://github.com/konnected-io/gdolib/pull/41).
  Details: Its `CMakeLists.txt` requires the
  old `driver` umbrella component, which IDF 6 split into `esp_driver_*` components,
  so `driver/uart.h` isn't found. The fix is to require `esp_driver_uart` and
  `esp_driver_gpio` instead, guarded so IDF releases before 5.3 keep `driver`. With
  that fix, the full config builds for the ESP32-C5 on both IDF 6.1.0 and IDF 5.5.5
  (ESPHome 2026.9.1's default).
- **The cover's `pre_close_warning_duration` defaults to a bare `0`,** which fails
  validation. The config sets it to `0s`. The ratgdo board has no buzzer or warning
  LED, so the warning is off.

## Tested on hardware

On 2026-10-06 a ratgdo 2.5i with a WT0132C5 was switched from esphome-ratgdo to this
config on a Security+ 2.0 opener (ESPHome 2026.9.1, ESP-IDF 6.1.0, gdolib with the
fix above):

- It synced on first boot and auto-detected Security+ 2.0.
- It read the opening count from the opener.
- Close and open from Home Assistant worked: about 15.6 s down and 14.2 s up, with
  motor and door states tracking each move.
- The Home Assistant cover entity ID carried over from the ratgdo config unchanged.

The same day a second board, on a **Security+ 1.0** opener with a smart wall panel,
was switched the same way. It synced, auto-detected "Security+ 1.0 with smart panel",
and opened and closed from Home Assistant (about 11 s up, 12.7 s down, confirmed on
camera). It was rolled back to esphome-ratgdo for the reason below, then switched
back to this config with the fixed gdolib on 2026-10-08, and has run it since.

## Security+ 1.0: reboot opens the door

Three reboots with the Konnected firmware running on the Security+ 1.0 board:

| Reboot | Started from | Door |
|---|---|---|
| OTA reflash, ratgdo → Konnected | ratgdo firmware | stayed closed |
| OTA reflash, Konnected → Konnected | Konnected firmware | **opened** |
| Restart button | Konnected firmware | **opened** |

The Security+ 2.0 board rebooted alongside each time and never moved.

What's known:
- A Security+ 1.0 opener treats its wall-control line being held low as a button
  press. The ratgdo's TX transistor shorts that line whenever its GPIO is high.
- gdolib sends no door command at startup. With a smart panel it only listens, and its
  panel-emulation poll bytes (`0x35 0x33 0x53 0x38 0x3A 0x39`) never include the door
  toggle (`0x30`).
- So the TX GPIO is going high during the reboot. Konnected drives it as an inverted
  UART1 output (idle low). Candidates: the peripheral reset at shutdown, when the
  inversion bit clears while the GPIO matrix still routes UART TX to the pin, and the
  C5 strapping pull-up on GPIO25 during reset. Not yet proven either way.
- Security+ 2.0 is immune because it ignores a shorted line.

Also seen on the Security+ 1.0 board: after a close was interrupted and the door
reversed to open, Home Assistant sat on `closing` for six minutes until a wall-button
press closed the door for real.

**Cause:** ESPHome runs `on_shutdown()` before every OTA update and restart.
`secplus_gdo`'s `on_shutdown()` calls gdolib's `gdo_deinit()`, which releases the TX pin
with ESP-IDF's `gpio_reset_pin()`. That function enables the pin's internal pull-up, which
switches on the board's TX transistor and holds the Security+ 1.0 line low (a button
press) for the whole reboot. esphome-ratgdo never resets the pin, so its TX stays low; a
Restart on the ratgdo firmware left the door closed.

**Fix:** [konnected-io/gdolib#42](https://github.com/konnected-io/gdolib/pull/42) parks TX
at the line's idle level with `gpio_hold` instead. It is combined with the IDF 6 build fix
in the fork the config pins (see [the gdolib fork](#the-gdolib-fork)).

On the Security+ 1.0 board, the OTA updates of 2026-11-07 (door open) and 2026-11-13
(door closed) left the door where it was.

Limits:
- The fix runs in `gdo_deinit()`, so it covers restarts and OTA updates. A crash,
  watchdog reset or brownout skips it.
- The C5 strapping pull-up on GPIO25 (opener TX on the WT0132C5) during reset hasn't
  been ruled out as a second cause.

## The gdolib fork

The config pins a commit on
[`davidcoulson/gdolib@ratgdo-c5-next`](https://github.com/davidcoulson/gdolib/tree/ratgdo-c5-next).
On top of upstream it has:

- the ESP-IDF 6 build fix ([#41](https://github.com/konnected-io/gdolib/pull/41));
- TX held at idle on deinit ([#42](https://github.com/konnected-io/gdolib/pull/42));
- toggle-only close after a stop ([#43](https://github.com/konnected-io/gdolib/pull/43));
- Security+ 1.0: Stop while closing stops the door instead of reversing it, and checks
  that the stop took;
- Security+ 1.0: presses held for 250 ms, and no `GET_STATUS` sent on a 1.0 bus;
- Security+ 2.0: the RX stream realigns after a framing error, and a stale pending
  receive is cleared at start (it could block the first sync);
- obstruction is polled while set, so it can't stick.

Go back to `konnected-io/gdolib` once these are merged and tagged.

## Cold-boot sync workaround

After a power-on boot, one of the two boards never synced with its opener, while any
software restart synced within seconds. The cause isn't found. The config's `on_boot`
block restarts once if a hardware reset hasn't synced after 90 s.

## Keeping it reproducible

The config pins `konnected-esphome` and gdolib to commits rather than tracking
`master`, so a rebuild months later gets the same code. Bump the pins on purpose, after
reading the upstream changes.

Two small fixes to `secplus_gdo` are also worth sending upstream:

- `cover/__init__.py` defaults `pre_close_warning_duration` to a bare `0`, which fails
  validation; it should be `"0s"`. The config here sets it explicitly to work around that.
- `secplus_gdo.cpp`'s panic handler calls `gpio_ll_func_sel` on `GPIO_NUM_1` (the blaQ's
  TX pin) rather than `GDO_UART_TX_PIN`. The following direction and pulldown calls do use
  the configured pin, so the handler still mostly works on other boards.

## Other findings from a code review

- **Force the protocol.** If auto-detection fails, gdolib falls back to transmitting the
  other protocol on the wire, and the component retries sync forever. The config sets the
  protocol select's `initial_option`.
- **Close after a stopped close (Security+ 1.0 / toggle-only).** `gdo_door_close()` lacked
  the stop-then-move sequence `gdo_door_open()` has, so "close" on a door stopped part-way
  while closing ran it up. Fixed on
  [`davidcoulson/gdolib@close-after-stop`](https://github.com/davidcoulson/gdolib/tree/close-after-stop)
  ([#43](https://github.com/konnected-io/gdolib/pull/43)), included in the pinned fork.
- **Security+ 1.0 state after a reversal** can stay on `closing` until the next
  restart; not yet diagnosed.
