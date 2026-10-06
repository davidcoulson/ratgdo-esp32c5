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

- **gdolib v1.1.2 doesn't build on ESP-IDF 6.x.** Its `CMakeLists.txt` requires the
  old `driver` umbrella component, which IDF 6 split into `esp_driver_*` components,
  so `driver/uart.h` isn't found. The fix is to require `esp_driver_uart` and
  `esp_driver_gpio` instead, guarded so IDF releases before 5.3 keep `driver`. With
  that fix, the full config builds for the ESP32-C5 on both IDF 6.1.0 and IDF 5.5.5
  (ESPHome 2026.9.1's default).
- **The cover's `pre_close_warning_duration` defaults to a bare `0`,** which fails
  validation. The config sets it to `0s`. The ratgdo board has no buzzer or warning
  LED, so the warning is off.
