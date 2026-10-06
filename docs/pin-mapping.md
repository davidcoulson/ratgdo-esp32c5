# Pin mapping

Each module's pad labels are printed on its underside. The tables below are
mirrored to the **top view**: antenna up, pad 1 at the top of the left column,
numbering counter-clockwise like the ESP-12F.

All three modules keep the ESP-12F positions for EN (pad 3), VCC (8), GND (15),
RXD (21) and TXD (22). Pads 9–14 were the ESP8266's flash pins and the ratgdo
doesn't use them.

## The table

The ratgdo functions come from esphome-ratgdo's
[`static/v25iboard.yaml`](https://github.com/ratgdo/esphome-ratgdo/blob/main/static/v25iboard.yaml).

| 12F pad | ESP8266 | D1 mini | ratgdo 2.5i function | **WT0132C5** | WT0132C6 | ESP-C5-12F |
|---|---|---|---|---|---|---|
| 1 | RST | — | reset | IO0 ⚠️ | IO0 ⚠️ | IO0 ⚠️ |
| 2 | ADC | A0 | unused | IO1 | IO1 | IO1 |
| 3 | EN | — | enable | EN | EN | EN |
| 4 | GPIO16 | D0 | `status_door` (out) | IO2 (strap) | IO2 | IO2 (strap) |
| 5 | GPIO14 | D5 | `dry_contact_open` (in) | IO4 | IO4 | IO3 |
| 6 | GPIO12 | D6 | `dry_contact_close` (in) | IO5 | IO5 | IO4 |
| 7 | GPIO13 | D7 | `input_obst` (in) | IO6 | IO6 | IO5 |
| 8 | VCC | 3V3 | 3.3 V | 3V3 | 3V3 | VCC |
| 16 | GPIO15 (board pull-down) | D8 | `status_obstruction` (out) | IO24 | IO7 | IO6 |
| 17 | GPIO2 (board pull-up) | D4 | LED / unused | IO27 (strap, wants high ✅) | IO8 (strap, wants high ✅) | IO15 |
| 18 | GPIO0 (boot) | D3 | `dry_contact_light` (in) | **IO28 = boot ✅** | **IO9 = boot ✅** | **IO28 = boot ✅** |
| 19 | GPIO4 | D2 | **opener RX** | IO26 | IO10 | IO27 (strap ⚠️) |
| 20 | GPIO5 | D1 | **opener TX** | IO25 (strap ⚠️) | IO3 | IO26 |
| 21 | RXD | RX | USB serial | GPIO12 | GPIO17 | RX0 |
| 22 | TXD | TX | USB serial | GPIO11 | GPIO16 | TX0 |
| 9–14 | flash pins | — | unused | IO7–10, 13, 14 | IO18–23 | IO7–10, 13, 14 |

⚠️ on pad 1: these modules put a GPIO where the ESP-12F has RST. See
[hardware](hardware.md#pad-1-is-not-reset).

## ESPHome substitutions

| Substitution | WT0132C5 | WT0132C6 |
|---|---|---|
| `uart_tx_pin` | GPIO25 | GPIO3 |
| `uart_rx_pin` | GPIO26 | GPIO10 |
| `input_obst_pin` | GPIO6 | GPIO6 |
| `status_door_pin` | GPIO2 | GPIO2 |
| `status_obstruction_pin` | GPIO24 | GPIO7 |
| `dry_contact_open_pin` | GPIO4 | GPIO4 |
| `dry_contact_close_pin` | GPIO5 | GPIO5 |
| `dry_contact_light_pin` | GPIO28 | GPIO9 |

## Why the WT0132 modules

- **They're designed as 12F drop-ins.** The boot pin sits on pad 18, where the
  ESP8266's GPIO0 was. The second strapping pin (C5 IO27, C6 IO8) sits on pad 17,
  which the ratgdo board already pulls high for the ESP8266's GPIO2. So the board's
  existing boot wiring keeps working.
- **ESP32-C5 strapping pins** are GPIO2, 7, 25, 27 and 28. GPIO28 low at reset means
  download mode, and GPIO27 must be high to flash reliably
  ([Espressif docs](https://docs.espressif.com/projects/esptool/en/latest/esp32c5/advanced-topics/boot-mode-selection.html)).
- **WT0132C5:** opener TX lands on IO25, a strapping pin. It has booted reliably with
  the opener connected. If a board ever won't boot while wired to the opener, this
  pin is the first suspect.
- **ESP-C5-12F:** opener RX lands on IO27, which must be high to flash. That makes it
  the worst fit of the three.
- **WT0132C6:** neither opener wire is on a strapping pin, the pulled-down pad 16
  isn't a strapping pin, and the native USB pins (GPIO12/13) aren't on any pad.
- **The light dry-contact sits on the boot pin** on every option, just as it did on
  the ESP8266 (GPIO0). If that contact is closed at power-up, the chip boots into
  download mode.
