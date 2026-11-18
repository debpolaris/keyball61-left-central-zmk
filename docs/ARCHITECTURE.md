# Architecture

How a cursor moves when you roll a ball that sits on the half that has no keymap.

## Roles

| | Left half | Right half |
| --- | --- | --- |
| ZMK role | **central** (`ZMK_SPLIT_ROLE_CENTRAL`) | peripheral |
| Owns | BLE link to the host, USB, keymap, ZMK Studio | the PMW3610 sensor |
| Cable for flashing/USB | **this one** | not used for USB |
| Keyboard name advertised | `Keyball61` | — |
| nice!view content | layer, output mode, BLE profile 1–5, battery | static `mountain` art + battery |
| Build shield string | `keyball61_left nice_view` + `studio-rpc-usb-uart` | `keyball61_right nice_view` |

Set in `config/boards/shields/keyball61/Kconfig.defconfig`, which is the only place
`ZMK_SPLIT_ROLE_CENTRAL` is decided.

## Trackball data path

```
PMW3610 (SPI1, right half)
   │  hardware registers → deltas
   ▼
badjeff pixart,pmw3610 driver          src/pmw3610.c, dts/bindings/pixart,pmw3610.yml
   │  emits INPUT_EV_REL / INPUT_REL_X, INPUT_REL_Y only
   ▼
zmk,input-split  (trackball_split)     right half: device = <&trackball>
   │  serialises input events over the split BLE link
   ▼
zmk,input-split  (same node, left half)
   │
   ▼
zmk,input-listener (trackball_listener)  left half only, status = "okay"
   │  runs the input-processors below, then feeds the pointing core
   ▼
pointing core → HID mouse report → BLE to the host
```

Two rules make this shape necessary:

1. **A peripheral half has no keymap.** `zmk_keymap_layer_activate()` and friends are
   not compiled there, so a driver that switches layers itself cannot link on a
   peripheral. The vendor's `zmk,pmw3610` driver does exactly that (its
   `automouse-layer` / `scroll-layers` / `snipe-layers` properties), which is why the
   right-half build failed before the driver swap.
2. **Layer state is only known on the central.** So the sensor stays "dumb" — raw
   relative motion — and every layer-dependent decision moves to the central half,
   where `zmk,input-listener` can consult the keymap.

## Input processors (left half, `keyball61_left.overlay`)

| Processor | Attached to | Effect |
| --- | --- | --- |
| `zip_auto_mouse 4 700` | listener default | moving the ball activates layer 4 for 700 ms; `require-prior-idle-ms = <200>` blocks activation if a key was struck within the last 200 ms |
| `zip_xy_to_scroll_mapper`, `zip_scroll_scaler 1 16` | child `scroller`, `layers = <5>` | XY deltas become wheel events at 1/16 speed |
| `zip_xy_scaler 1 3` | child `sniper`, `layers = <6>` | XY deltas scaled to 1/3 |

`zip_scroll_scaler`, `zip_xy_scaler`, `zip_xy_to_scroll_mapper` and `zip_temp_layer`
are defined by ZMK in `app/dts/input/processors/`, pulled in by
`#include <input/processors.dtsi>`. Only `zip_auto_mouse` is defined in this repo, and
it is a `zmk,input-processor-temp-layer` instance with the idle guard set — the stock
`zip_temp_layer` has no `require-prior-idle-ms`, so it would misfire while typing.

## Kconfig gating (why both halves need `CONFIG_ZMK_POINTING=y`)

From ZMK v0.3 `app/src/pointing/Kconfig`:

| Symbol | Condition | Left | Right |
| --- | --- | --- | --- |
| `ZMK_POINTING` | user setting | y | y |
| `ZMK_INPUT_SPLIT` | `DT_HAS_ZMK_INPUT_SPLIT_ENABLED && ZMK_SPLIT`, inside `if ZMK_POINTING` | y | y |
| `ZMK_INPUT_LISTENER` | `DT_HAS_ZMK_INPUT_LISTENER_ENABLED`, inside `if !ZMK_SPLIT \|\| ZMK_SPLIT_ROLE_CENTRAL` | y | not compiled |
| `ZMK_INPUT_PROCESSOR_TEMP_LAYER` | same central-only block | y | not compiled |
| `ZMK_INPUT_PROCESSOR_SCALER` / `_CODE_MAPPER` | inside `if ZMK_POINTING`, not role-gated | y | y |
| `ZMK_BEHAVIOR_MOUSE_KEY_PRESS` | `DT_HAS_... && ZMK_POINTING` (`app/CMakeLists.txt:66`) | y | y |

Consequences worth remembering:

- The `split_inputs` block in `keyball61.dtsi` must be present in **both** builds,
  because `DT_HAS_ZMK_INPUT_SPLIT_ENABLED` has to be true on both.
- `trackball_listener` is declared disabled in the shared `.dtsi` and enabled only by
  the left overlay. Leaving it enabled on the right is harmless but misleading.
- `CONFIG_ZMK_MOUSE` is a deprecated alias whose only effect is
  `default y if ZMK_MOUSE` on `ZMK_POINTING`. It is gone from both halves.

## Layer indices

`config/keyball61.keymap` declares seven layers; ZMK numbers them **by declaration
order**, starting at 0:

| Index | Node | `label` |
| --- | --- | --- |
| 0 | `default_layer` | QWRT |
| 1 | `number_layer` | NUM |
| 2 | `symbol_layer` | SYM |
| 3 | `function_layer` | FUN |
| 4 | `mouse_layer` | MOUSE |
| 5 | `scroll_layer` | SCROLL |
| 6 | `snipe_layer` | SNIPE |

The file also contains `#define MOUSE 3` / `#define SCROLL 4` / `#define SNIPE 5`,
inherited from the vendor config. Those macros are **not used by any binding** and are
each one lower than the real index. If you re-order the layers in the keymap, the
numbers `4`, `5` and `6` in `keyball61_left.overlay` must be re-derived from the new
declaration order — otherwise auto-mouse silently activates the wrong layer, and there
is no build error to warn you.

## Pin map

Derived from the overlays plus nice_nano_v2's `arduino_pro_micro_pins.dtsi`
(`pro_micro` 0→P0.8, 1→P0.6, 2→P0.17, 3→P0.20, 4→P0.22, 5→P0.24, 6→P1.0, 7→P0.11,
8→P1.4, 9→P1.6, 10→P0.9, 14→P1.11, 15→P1.13, 16→P0.10, 18→P1.15, 19→P0.2,
20→P0.29, 21→P0.31).

### Left half (central)

| Function | Bus | Pins |
| --- | --- | --- |
| Key matrix rows (5) | GPIO | pro_micro 4,5,6,7,8 |
| Key matrix cols (7) | GPIO | pro_micro 21,20,19,18,15,14,16 |
| nice!view | SPI2 | MOSI P0.17, SCK P0.20, CS P0.06 |

### Right half (peripheral)

| Function | Bus | Pins |
| --- | --- | --- |
| Key matrix rows (5) | GPIO | pro_micro 3,4,5,6,7 |
| Key matrix cols (6) | GPIO | pro_micro 9,8,18,19,20,21 |
| Trackball | SPI1 | SCK P1.13, MOSI P0.10, MISO P0.10, CS P0.09, IRQ P1.11 |
| nice!view | SPI2 | MOSI P0.17, SCK P0.08, CS P0.06 |

`col-offset = <7>` on the right half (`&default_transform`) shifts its columns into the
shared 13-column matrix transform, so one keymap covers both halves.

### Two quirks inherited from the vendor config, left in place

1. **The `oled: ssd1306@3c` node is dead weight on both halves.** Each half declares an
   SSD1306 on `&i2c0` using the *same two pins* as its nice!view SPI bus (left: P0.17 /
   P0.20, right: P0.17 / P0.08). Nothing breaks because the `nice_view` shield overrides
   the `zephyr,display` chosen node with its own, so LVGL never talks to the SSD1306.
   `Kconfig.defconfig` still defaults `CONFIG_SSD1306=y` under `if ZMK_DISPLAY`. If you
   ever want a real OLED instead of a nice!view, disable `nice_view` and free those pins
   from the SPI definitions first.
2. **`spi1` declares MOSI and MISO on the same pad (P0.10).** Unusual, but it is what
   shipped and what the assembled board was flashed with. If the trackball ever stops
   reporting motion while everything else works, start the investigation here.

## Why the right screen shows a mountain

ZMK's nice!view shield compiles different widget sources by role
(`app/boards/shields/nice_view/CMakeLists.txt`):

```cmake
if(NOT CONFIG_ZMK_SPLIT OR CONFIG_ZMK_SPLIT_ROLE_CENTRAL)
  zephyr_library_sources(widgets/status.c)      # left half
else
  zephyr_library_sources(widgets/art.c)         # right half: the mountain bitmap
  zephyr_library_sources(widgets/peripheral_status.c)
endif
```

`peripheral_status.c` declares `LV_IMG_DECLARE(mountain)`. So a static mountain on the
right screen is positive evidence that the right half is running as a peripheral and
that its display works. A mountain on the **left** screen would mean the role did not
move.
