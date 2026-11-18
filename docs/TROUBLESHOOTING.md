# Troubleshooting

Symptom → cause → fix. Entries 1–6 are failures this configuration actually hit during
the original session; 7 onward are the checks to run when behaviour is wrong but the
build passed. Verification commands assume a west workspace built as in
[docs/BUILD.md](BUILD.md) (paths relative to the workspace root).

## Build failures

### 1. `parse error: undefined node label 'nice_view_spi'` (left)

The nice!view widget tree resolves `zephyr,display`/`&nice_view_spi` at devicetree
preprocessing. The shield is missing from the build, not the overlay.

Fix: pass the display shield explicitly — `-DSHIELD="keyball61_left nice_view"`. Local
`west build` never reads `build.yaml`, so the shield list has to be typed every time.

### 2. Duplicate label `nice_view_spi`

```
Label 'nice_view_spi' appears on &spi2 and on &{/...}
```

Both `config/boards/shields/keyball61/keyball61_left.overlay` and
`keyball61_right.overlay` already declare `nice_view_spi: &spi2`, because this board's
display connector is not on the pins `nice_view_adapter` expects. Adding
`nice_view_adapter` to either shield list defines the same label twice.

Fix: use `nice_view` **without** `nice_view_adapter` on both halves. That is what
`build.yaml` and every command in [docs/BUILD.md](BUILD.md) do; the upstream
handover notes still show the adapter in one place — they predate this repo and are
wrong for it (see [docs/PROVENANCE.md](PROVENANCE.md#3-places-where-this-repo-deliberately-disagrees-with-the-notes)).

### 3. `-S studio-rpc-usb-uart` has no effect

```
CMake Error: The source directory ".../studio-rpc-usb-uart" does not appear to contain CMakeLists.txt.
```

or, worse, no error and the snippet simply did not apply. Anything after `--` goes to
CMake, where `-S` already means "source directory".

Fix: put the west flag **before** `--` (`west build -S studio-rpc-usb-uart ... --`) or
use the CMake variable after it (`-DSNIPPET=studio-rpc-usb-uart`). Both forms appear in
this repo; `-DSNIPPET=` is the one in the short-form commands.

Verify it applied:

```bash
grep -R "SNIPPET" build/left/zephyr/.config | head
grep CONFIG_ZMK_STUDIO_OVER_USB build/left/zephyr/.config   # =y
```

### 4. `undefined reference to zmk_keymap_layer_activate` (right half)

The vendor `zmk,pmw3610` driver implements automouse/scroll/snipe **inside the driver**,
which means it calls into `zmk_keymap` and `zmk_pointing`. Those subsystems are compiled
out of a peripheral image, so the link fails.

Fix: use badjeff's driver (`pixart,pmw3610`), which only reports raw XY deltas, and move
the behaviour to the central half with `zmk,input-split` + `zmk,input-listener`. That is
what this repo does — see [docs/ARCHITECTURE.md](ARCHITECTURE.md#trackball-data-path).
If you see this error again, someone put `compatible = "zmk,pmw3610"` back into
`keyball61_right.overlay` or dropped the `zmk-pmw3610-driver` entry from
`config/west.yml`.

### 5. `No board named 'nice_nano_v2' found`

`config/west.yml` drifted off `v0.3`. ZMK `main` moved to Zephyr 4.1 and HWMv2 board
naming (`xiao_nrf52840`, `nice_nano/nice_nano_v2/...`), which breaks this board, the
LVGL version and the nice!view shield.

Fix: pin back and rebuild the tree from scratch (partial updates leave the old Zephyr
behind):

```bash
cd config && git diff west.yml          # revision must read v0.3, not main
rm -rf ../zephyr ../modules             # workspace root, NOT inside this repo
west update
head -2 zephyr/VERSION                  # major 3, patch 5
```

The same drift shows up as the driver branch disappearing: `zmk-pmw3610-driver` must be
on `zmk-0.3`, and `west update` failing on that revision means the remote branch was
moved — see [docs/PROVENANCE.md](PROVENANCE.md#4-known-drift-risks).

### 6. `ModuleNotFoundError: No module named 'pkg_resources'`

`setuptools` 82.0.0 removed `pkg_resources`, which ZMK's nanopb/Studio code generation
still imports. It also gets reinstated by any later `pip install -r ...requirements.txt`.

Fix, then rebuild:

```bash
pip install "setuptools<82"
python -c "import pkg_resources; print('ok')"
```

## Runtime behaviour

### 7. The halves never connect to each other

Almost always stale pairing data: the role swapped (right → left central), but each half
still holds the old bond.

Fix: flash `settings_reset-nice_nano_v2-zmk.uf2` to **both** halves, then the two
firmware images, then re-pair. Full order and rationale in
[docs/FLASH.md](FLASH.md#order). If it still fails, confirm the roles really are
what this repo intends:

```bash
grep -E "CONFIG_ZMK_SPLIT=|CONFIG_ZMK_SPLIT_ROLE_CENTRAL" build/left/zephyr/.config
grep -E "CONFIG_ZMK_SPLIT=|CONFIG_ZMK_SPLIT_ROLE_CENTRAL" build/right/zephyr/.config
# left:  ZMK_SPLIT=y, ZMK_SPLIT_ROLE_CENTRAL=y
# right: ZMK_SPLIT=y, ZMK_SPLIT_ROLE_CENTRAL unset
```

Those two lines come from the `if SHIELD_KEYBALL61_LEFT` block in
`config/boards/shields/keyball61/Kconfig.defconfig`; if the block ever reads
`SHIELD_KEYBALL61_RIGHT` again, the roles are inverted and the keyboard will not pair.

### 8. Keys work, trackball does nothing

Work through these in order:

```bash
# a) driver compiled into the peripheral at all?
grep -E "CONFIG_PMW3610=|CONFIG_INPUT=|CONFIG_ZMK_POINTING=" build/right/zephyr/.config
grep -E "CONFIG_ZMK_POINTING=|CONFIG_ZMK_INPUT_SPLIT=|CONFIG_ZMK_INPUT_LISTENER=" build/left/zephyr/.config
# right: PMW3610=y INPUT=y ZMK_POINTING=y
# left:  ZMK_POINTING=y INPUT_SPLIT=y INPUT_LISTENER=y

# b) is the sensor node actually enabled, on both ends of the relay?
grep -A3 "trackball@0" build/right/zephyr/zephyr.dts            # status = "okay"
grep -B1 -A3 "trackball_split@" build/right/zephyr/zephyr.dts   # device = <&trackball>
grep -A4 "trackball_listener" build/left/zephyr/zephyr.dts      # status = "okay", device = <&trackball_split>
```

| Result | Cause |
| --- | --- |
| `CONFIG_PMW3610` missing | `keyball61_right.conf` lost the line, or the driver module is not in `config/west.yml` |
| node `status = "disabled"` | the `&trackball` node lost `status = "okay"` |
| `device` property absent from `trackball_split` | the `&trackball_split { device = <&trackball>; };` block was dropped from the right overlay — nothing is relayed |
| listener `disabled` on the left | `&trackball_listener { status = "okay"; }` missing from the left overlay |
| everything correct | hardware: check the sensor's `irq-gpios` (P1.11) and the SPI pins documented in [docs/ARCHITECTURE.md](ARCHITECTURE.md#pin-map) |

### 9. Cursor moves the wrong way

`invert-x`, `invert-y`, `swap-xy` on the `trackball` node in `keyball61_right.overlay`
are the only levers. Reflash the **right** half. See
[docs/TUNING.md](TUNING.md#1-sensor-feel--configboardsshieldskeyball61keyball61_rightoverlay).

### 10. Horizontal scroll runs opposite to before

The old vendor config had `CONFIG_PMW3610_INVERT_SCROLL_X=y`, and nothing in this repo
reproduces it yet — the scroll chain is mapper → scaler with no transform. Add
`<&zip_scroll_transform INPUT_TRANSFORM_X_INVERT>` between them in the left overlay.
This is the one tuning item the original session left open; it has **not** been
confirmed on hardware. Recipe: [docs/TUNING.md](TUNING.md#4-scroll--same-file-scroller-child).

### 11. Auto-mouse fights the typing

Two opposite symptoms, one knob:

| Symptom | Move |
| --- | --- |
| cursor jumps when a palm brushes the ball mid-word | raise `require-prior-idle-ms` (200 → 250–300) |
| ball ignores the first moment of motion after a keystroke | lower it (200 → 150) |
| mouse mode drops out while clicking | add `excluded-positions` for the click keys |
| mouse mode lingers after touching the ball | lower the `700` in `input-processors = <&zip_auto_mouse 4 700>` |

All four live in `keyball61_left.overlay`; reflash the **left** half.
Details: [docs/TUNING.md](TUNING.md#3-auto-mouse--configboardsshieldskeyball61keyball61_leftoverlay).

### 12. Scroll or snipe does nothing on its layer

Layer indices, always. ZMK addresses layers by declaration order in
`config/keyball61.keymap`, and the `#define MOUSE 3 / SCROLL 4 / SNIPE 5` block at the
top of that file is **stale** — no binding uses it. Current truth:
`default=0 number=1 symbol=2 function=3 mouse=4 scroll=5 snipe=6`, so `layers = <5>` and
`layers = <6>` are correct. If you reordered layers and nothing else changed, scroll and
snipe are firing on the wrong layer while the cursor keeps moving.

### 13. The right screen shows a mountain instead of layer info

Not a fault. `zmk/app/src/display/CMakeLists.txt` compiles a different widget set per
role: the central gets `widgets/status.c` (layer, output, battery, profile 1–5) and the
peripheral gets `widgets/art.c` + `peripheral_status.c`, which draws the `mountain`
graphic. A peripheral half has no keymap to report, so it cannot show layers. Only the
left screen is a status display.

### 14. ZMK Studio cannot connect

Studio runs over USB on the **central** only, which is why
`studio-rpc-usb-uart` is applied to the left build and not the right. Check:

```bash
grep CONFIG_ZMK_STUDIO build/left/zephyr/.config            # =y
grep CONFIG_ZMK_STUDIO_OVER_USB build/left/zephyr/.config   # =y (needs the snippet)
```

If both are `y`, plug the left half in, use the ZMK Studio web app, and hit *Connect*
after the OS enumerates the CDC-ACM port. Studio changes live in the keyboard's settings
partition, so a `settings_reset` flash wipes them.

### 15. Battery drains fast on the right half

The trackball is the cost. Check `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT` (900000 ms here), then
the driver's downshift defaults in [docs/TUNING.md](TUNING.md#2-driver-power-management--configboardsshieldskeyball61keyball61_rightconf),
and make sure `force-awake` was not added to the sensor node while debugging — it
disables the sensor's REST modes and is easy to forget.

## Diagnostic habits

- Read the generated tree before guessing: `build/<half>/zephyr/zephyr.dts` is the
  devicetree actually compiled, `build/<half>/zephyr/.config` the options actually set.
  A value in a source overlay proves nothing if the shield name was wrong.
- `west build -p auto` wipes the build dir; without `-p` an edited shield list may not
  take effect and you will debug a stale image.
- Compare against the working baseline instead of against memory:
  `git diff` a single file, or `git show HEAD:config/boards/shields/keyball61/keyball61_left.overlay`.
- Only one variable per flash. UF2 flashing takes seconds; guessing costs hours.
