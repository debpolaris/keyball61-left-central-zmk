# Tuning

Every knob that was needed to get this build, with its file, current value and the
consequence of moving it. Devicetree changes need a rebuild of the half named in the
"Reflash" column; Kconfig changes usually do too, so when in doubt rebuild both.

## 1. Sensor feel — `config/boards/shields/keyball61/keyball61_right.overlay`

| Property | Current | Range / notes | Reflash |
| --- | --- | --- | --- |
| `cpi` | `<1200>` | 200–3200 in steps of 200 (binding default is 600). Counts per inch of ball travel; the main speed control. | right |
| `invert-x` | present | flips horizontal axis | right |
| `invert-y` | present | flips vertical axis | right |
| `swap-xy` | absent | exchanges axes — use when up/down motion moves the cursor sideways | right |
| `spi-max-frequency` | `<2000000>` | 2 MHz. Leave it unless you see dropped motion at high speed. | right |
| `force-awake` | absent | keeps the sensor out of its low-power REST modes; useful for debugging, bad for battery | right |
| `irq-gpios` | `<&gpio1 11 (GPIO_ACTIVE_LOW \| GPIO_PULL_UP)>` | the only wiring property; changing it means rewiring | right |

`invert-x` + `invert-y` together reproduce the old driver's
`CONFIG_PMW3610_ORIENTATION_180=y`. If the cursor moves but in the wrong direction,
these three booleans are the whole answer — there is no rotation setting in this driver.

## 2. Driver power management — `config/boards/shields/keyball61/keyball61_right.conf`

All at defaults on purpose. Set them here if battery life or wake latency demands it.

| Symbol | Default | Notes |
| --- | --- | --- |
| `CONFIG_PMW3610_SMART_ALGORITHM` | `y` | better tracking on textured surfaces; leave on |
| `CONFIG_PMW3610_RUN_DOWNSHIFT_TIME_MS` | `128` (range 13–3264) | how long the sensor stays in full-speed RUN after the last motion. Raise it for smoothness, lower it for battery |
| `CONFIG_PMW3610_REST1_SAMPLE_TIME_MS` | `40` | sampling period once downshifted |
| `CONFIG_PMW3610_REST2_SAMPLE_TIME_MS` | `100` | deeper idle |
| `CONFIG_PMW3610_REST3_SAMPLE_TIME_MS` | `500` | deepest idle |
| `CONFIG_PMW3610_REPORT_INTERVAL_MIN` | `0` | throttles how often the driver reports. The driver's own help text calls this out for HID queues *over the air* — the first thing to try if cursor motion stutters across the split link |
| `CONFIG_PMW3610_INIT_POWER_UP_EXTRA_DELAY_MS` | `0` | add if a sensor intermittently fails to initialise at boot |

Symbols that existed in the vendor config and **do not exist here**: `PMW3610_PM`,
`PMW3610_CPI`, `PMW3610_CPI_DIVIDOR`, `PMW3610_ORIENTATION_180`, `PMW3610_SNIPE_CPI`,
`PMW3610_SNIPE_CPI_DIVIDOR`, `PMW3610_SCROLL_TICK`, `PMW3610_INVERT_SCROLL_X`,
`PMW3610_INVERT_SCROLL_Y`, `PMW3610_AUTOMOUSE_TIMEOUT_MS`, `PMW3610_MOVEMENT_THRESHOLD`,
`PMW3610_POLLING_RATE_125_SW`. Setting any of them is a silent no-op or a build error —
they belong to the other driver.

## 3. Auto-mouse — `config/boards/shields/keyball61/keyball61_left.overlay`

```dts
zip_auto_mouse: zip_auto_mouse {
    compatible = "zmk,input-processor-temp-layer";
    #input-processor-cells = <2>;
    require-prior-idle-ms = <200>;
};

&trackball_listener {
    input-processors = <&zip_auto_mouse 4 700>;
```

| Knob | Current | Effect |
| --- | --- | --- |
| first cell (`4`) | layer MOUSE | which layer a ball movement activates. Must equal the index of `mouse_layer` in the keymap |
| second cell (`700`) | 700 ms | how long the layer stays active after the last movement. Too short and clicking feels laggy; too long and the keyboard stays in mouse mode after you meant to go back to typing |
| `require-prior-idle-ms` | `200` | ball movement within this window after the previous **keystroke** is ignored. Raise (250–300) if the cursor still jumps while typing; lower (150) if there is a noticeable delay before the ball responds after you finish a word |
| `excluded-positions` | unset | key positions that do *not* cancel the temporary layer — needed so a mouse-layer click does not drop you out of mouse mode. Values are matrix positions from `config/keyball61.json`'s transform, e.g. `excluded-positions = <40 41 42 43>;` |

Reflash: left.

## 4. Scroll — same file, `scroller` child

```dts
scroller {
    layers = <5>;
    input-processors = <&zip_xy_to_scroll_mapper>, <&zip_scroll_scaler 1 16>;
};
```

| Change | Recipe |
| --- | --- |
| Scroll speed | the two cells of `zip_scroll_scaler 1 16` (numerator/denominator). `1 8` is twice as fast, `1 32` half |
| Wheel direction is wrong | insert `<&zip_scroll_transform INPUT_TRANSFORM_X_INVERT>` between the mapper and the scaler. `INPUT_TRANSFORM_Y_INVERT` inverts vertical scrolling, `INPUT_TRANSFORM_XY_SWAP` swaps axes. Add `#include <dt-bindings/zmk/input_transform.h>` at the top of the overlay |
| Which layer scrolls | `layers = <5>` — must be the index of `scroll_layer` |

> **Open item.** The vendor driver had `CONFIG_PMW3610_INVERT_SCROLL_X=y`. Nothing in
> this config reproduces that, so horizontal scrolling may run opposite to how the
> keyboard behaved before the migration. If it feels wrong, apply the
> `INPUT_TRANSFORM_X_INVERT` recipe above — that is the intended fix, not yet confirmed
> on hardware.

Reflash: left.

## 5. Snipe — same file, `sniper` child

```dts
sniper {
    layers = <6>;
    input-processors = <&zip_xy_scaler 1 3>;
};
```

`1 3` scales motion to one third. The old driver reached the same feel with
`SNIPE_CPI=400` against `CPI=1200`, so keep the ratio in mind if you change `cpi`:
the scaler is relative, the CPI is absolute.

Reflash: left.

## 6. Keymap timing — `config/keyball61.keymap`

| Node | Current | Notes |
| --- | --- | --- |
| `&lt` | `tapping-term-ms = <240>`, `flavor = "balanced"`, `quick-tap-ms = <150>` | hold-tap on `SPACE` (MOUSE 2) and `BACKSPACE` (NUM 1) |
| `&mt` | `tapping-term-ms = <200>`, `flavor = "tap-preferred"`, `quick-tap-ms = <150>` | affects every modified key |

`flavor = "tap-preferred"` biases toward the tap when you type quickly; `"hold-happy"`
biases the other way.

## 7. Radio and power — `config/keyball61.conf`

| Symbol | Current | Why it is there |
| --- | --- | --- |
| `CONFIG_BT_PERIPHERAL_PREF_MAX_INT` | `9` | with `PREF_MIN` implied, requests a ≤11.25 ms connection interval — the main lever for cursor latency over BLE |
| `CONFIG_BT_PERIPHERAL_PREF_LATENCY` | `16` | lets the host skip peripheral events to save battery |
| `CONFIG_BT_CTLR_TX_PWR_PLUS_8` | `y` | maximum transmit power |
| `CONFIG_BT_L2CAP_TX_BUF_COUNT` | `32` | buffer depth; raise if keys drop during fast typing |
| `CONFIG_ZMK_BLE_EXPERIMENTAL_CONN` | `y` | needed for the connection-parameter tuning above to take effect |
| `CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING` / `_PROXY` | `y` | lets the left screen report the **right** half's battery |
| `CONFIG_ZMK_BEHAVIORS_QUEUE_SIZE` | `512` | headroom for bursty input |
| `CONFIG_ZMK_DISPLAY` | `y` | required for nice!view on both halves |

Sleep: `CONFIG_ZMK_SLEEP=y` and `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000` (15 min) on both
halves, `CONFIG_ZMK_EXT_POWER=y` for the trackball/nice!view rail.

## 8. Re-numbering layers

If layers are added, removed or reordered in `config/keyball61.keymap`, update the
indices in `keyball61_left.overlay` (the `4` in `zip_auto_mouse 4 700`, `layers = <5>`,
`layers = <6>`) to the new declaration order. ZMK numbers layers by position, not by
label, and a mismatch builds cleanly while behaving subtly wrong.
`docs/ARCHITECTURE.md#layer-indices` has the current table.
