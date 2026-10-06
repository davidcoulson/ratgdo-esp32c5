# Troubleshooting

## "The ESPHome device attempted to perform a Home Assistant action"

esphome-ratgdo calls `persistent_notification.create` when it fails to sync with
the opener at startup. Home Assistant blocks that by default. To allow it:

1. Go to **Settings → Devices & services → ESPHome**.
2. Open the ratgdo device's options (the cog).
3. Tick **Allow the device to perform Home Assistant actions**.

Then deal with the cause, the failed sync:

- **One-off at power-on** (for example while installing, or when the opener is power
  cycled at the same moment): restart the ratgdo. If the door, light and state work,
  you're done.
- **Every boot:**
  1. Check the protocol: Security+ 2.0 openers have a yellow learn button. Use the
     matching config.
  2. Check the solder joints on pads 19/20 and their continuity to the opener
     interface circuit.
  3. Check boot strapping. On the WT0132C5, opener TX is on IO25, a strapping pin.
     If it fails only from a cold boot, try the WT0132C6, where neither opener wire
     is on a strapping pin.

## Brownout resets

If the Reset Reason sensor shows `brownout` when nobody is touching the board, the
3.3 V supply is dipping during Wi-Fi transmit spikes.

- **Part:** 220 µF, 10 V, low-ESR, rated 105 °C. Aluminium polymer is best, for example
  Nichicon RNE1A221MDS1PX or KEMET EA750EK227M1AAA; a low-ESR electrolytic is fine.
  Optionally add a 10 µF X5R/X7R ceramic in parallel.
- **Where:** between 3V3 (pad 8) and GND (pad 15, the opposite bottom corner), or at
  the regulator output. Keep the leads short. **+ to 3V3.**
- **Don't go much past 470 µF.** The extra inrush at power-on can make things worse.
- **If brownouts continue,** the 12 V feed or the board's regulator is the limit.

## Konnected config won't build

See [konnected.md](konnected.md#known-issues).
