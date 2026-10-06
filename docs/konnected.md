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
- **Auto-detects Security+ 1.0 vs 2.0.** A select entity can force one.
- **Obstruction comes from the protocol,** so the ratgdo's obstruction wire input
  (GPIO6) is unused.
- **No dry-contact inputs or status outputs.**
- **Re-sync button:** picks a new client ID, resets the rolling code and reboots. Use
  it if the opener won't respond after a switch.
- **Entity names match esphome-ratgdo's,** so switching an existing device keeps its
  Home Assistant entity IDs.

## Known issues

- **gdolib v1.1.2 doesn't build on ESP-IDF 6.x.** Until the fix is merged upstream,
  the config builds gdolib from
  [`davidcoulson/gdolib@idf6-driver-split`](https://github.com/davidcoulson/gdolib/tree/idf6-driver-split)
  ([konnected-io/gdolib#41](https://github.com/konnected-io/gdolib/pull/41)).
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
camera). It was then **rolled back to esphome-ratgdo** for the reason below.

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
on [`davidcoulson/gdolib@ratgdo-c5`](https://github.com/davidcoulson/gdolib/tree/ratgdo-c5).
A hardware retest on the Security+ 1.0 board is pending. Until it passes, use the ratgdo
config on Security+ 1.0 openers.

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
  [`davidcoulson/gdolib@close-after-stop`](https://github.com/davidcoulson/gdolib/tree/close-after-stop),
  included in `ratgdo-c5`.
- **Security+ 1.0 state after a reversal** can stay on `closing`; not yet diagnosed.
