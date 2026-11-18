# Flashing

Both controllers are nice_nano_v2, so flashing is UF2 drag-and-drop: double-tap the
reset button, the board enumerates as a `BOOTLOADER` drive, drop a `.uf2` on it, the
drive disappears and the board runs the new firmware.

Identify which physical half is which **before** you start — after this flash the left
half is the one that talks to the host over USB and BLE.

## Order

Skipping step 1 is the most common cause of "every key is dead". Changing the
central/peripheral role invalidates the split pairing stored in each half's flash, and
two halves holding mismatched pairing data simply never connect — no error, no keys.

1. **`settings_reset` to both halves.**
   Drop `zmk-settings_reset` (or `build/reset/zephyr/zmk.uf2`) on the left, then on the
   right. Each board reboots afterwards.

2. **Left half ← `left` image** (the central — it is the one you will cable to the PC).

3. **Right half ← `right` image** (the peripheral).

4. **Forget the old pairing on the host.** Delete *Keyball61* from your OS Bluetooth
   devices list. The central's identity changed role but not name, and a stale entry
   will fail to reconnect.

5. **Power both halves on together** and wait ~10 s. They re-pair with each other over
   split BLE automatically.

6. Connect the left half to the host — over USB (it now carries the USB stack and ZMK
   Studio) or by pairing *Keyball61* again.

## Verification checklist

Work down the list; each line tells you which subsystem is alive.

| # | Check | Expected | If not |
| --- | --- | --- | --- |
| 1 | Left screen | layer name, output icon, BLE profile 1–5, battery | role did not move, or `nice_view_spi` pinning — see [TROUBLESHOOTING.md](TROUBLESHOOTING.md) |
| 2 | Right screen | the static **mountain** graphic + battery | correct behaviour for a peripheral; a full status screen here means the role did not move |
| 3 | Typing on the left half alone, USB | keys register | left image or bootloader mismatch |
| 4 | Typing with both halves wireless | keys from **both** halves register | step 1 was skipped — re-flash `settings_reset` to both |
| 5 | Roll the trackball | cursor moves | check `CONFIG_PMW3610=y` and the sensor node status in the right build |
| 6 | Roll the trackball | the MOUSE layer (4) lights up on the left screen within ~700 ms, then drops | listener/processor config on the left |
| 7 | Hold the SCROLL layer (5) and roll | the page scrolls at a slow, controllable rate | scroller child node in `keyball61_left.overlay` |
| 8 | Hold the SNIPE layer (6) and roll | cursor moves at roughly 1/3 speed | sniper child node |
| 9 | Type fast, palm brushing the ball | cursor does **not** jump | raise `require-prior-idle-ms` — [TUNING.md](TUNING.md) |
| 10 | ZMK Studio | open at `http://localhost:8000/studio` (or the desktop app) **with the left half over USB** | Studio only runs on the central |

## Left half by itself

The central half works with no right half present — useful as a standalone macro pad or
for testing. Expect the left screen to show the keyboard as disconnected from its
partner, and the trackball to do nothing (the sensor lives on the missing half).

## Re-pairing the two halves manually

If they stop finding each other after a firmware update that was not a role change:
hold both reset buttons for ~10 s to power them off, power both on together, and wait.
If that fails, redo step 1 (`settings_reset` on both) and start over.

## Notes

- `settings_reset` wipes *all* ZMK settings, including saved host pairings on both
  halves. Re-pair after every use of it.
- The right half has no USB role in practice for Studio; if you cable it, you get plain
  USB HID from the peripheral, which is not what you want. Cable the left.
- Idle sleep is set to 15 minutes on both halves
  (`CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000`).
