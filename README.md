# Keyball61 with the left half as central, trackball still working

ZMK firmware configuration for a **Keyball61** split keyboard where the **left half is
the central (main) half** and the **PMW3610 trackball stays on the right half**, which
therefore runs as a peripheral.

The kits ship with the right half as central, because that is where the trackball, the
USB port and ZMK Studio were wired. That arrangement means the cable and the cursor
live on the same side. This configuration moves the main half to the left and rebuilds
the trackball's behaviour so it survives being on a peripheral — where there is no
keymap, and therefore no way to switch layers from a sensor driver.

![Keymap](keymap-drawer/keyball61.svg)

## Status

| | |
| --- | --- |
| Prebuilt firmware | [Releases](https://github.com/debpolaris/keyball61-left-central-zmk/releases/latest) — three UF2 files plus `SHA256SUMS.txt`; the same copies are committed in [`firmware/`](firmware/) |
| Firmware builds | GitHub Actions, three UF2 files per run (`left`, `right`, `settings_reset`) |
| Built with | ZMK `v0.3` (Zephyr `v3.5.0+zmk-fixes`), `nice_nano_v2` |
| Hardware state | The keyboard is assembled and in daily use. This tree is a reconstruction of the last known-good setup, so the tuning values in `config/` have not been re-flashed since — see [docs/PROVENANCE.md](docs/PROVENANCE.md#2-what-is-verified-and-how) |
| Provenance | Reconstructed from the 2026-07-03 handover notes, not copied out of the build tree — read [docs/PROVENANCE.md](docs/PROVENANCE.md) before changing anything version-sensitive |

## Hardware

| Item | Value |
| --- | --- |
| Keyboard | Keyball61 — 61 keys, split into two 30-key halves + 1 trackball |
| MCUs | 2 × nice_nano_v2 (nRF52840, Pro Micro footprint), BLE |
| Trackball | PMW3610 optical sensor on **SPI1 of the right half only** |
| Displays | nice!view on **both** halves, on SPI2 |
| Roles | left = central (BLE to host, USB, keymap, Studio) · right = peripheral |
| Keyboard name | `Keyball61` (`ZMK_KEYBOARD_NAME` in `Kconfig.defconfig`) |

Pin-level detail, the matrix wiring and the input data path are in
[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Build

**Easiest — grab a Release.** [Latest Release](https://github.com/debpolaris/keyball61-left-central-zmk/releases/latest)
carries the three UF2 files and `SHA256SUMS.txt`. A Release's assets are the binaries from
the Actions build of the commit its tag points at, so they match that tag's `config/`.
Checksums: `cd firmware && sha256sum -c SHA256SUMS.txt`.

**Build it yourself — GitHub Actions.** Push to this repo, or run *Build ZMK Firmware* →
*Run workflow*; the run's Artifacts archive holds the three UF2 files (artifacts expire,
Releases do not).

**Locally** — Debian/Ubuntu or WSL, about 1 GB of sources. The full runbook, including
the two version traps (`setuptools<82`, and never letting ZMK float to `main`), is
[docs/BUILD.md](docs/BUILD.md). Short form, run from the repo root:

```bash
git clone https://github.com/debpolaris/keyball61-left-central-zmk.git
cd keyball61-left-central-zmk
python3 -m venv .venv && source .venv/bin/activate
pip install west && pip install "setuptools<82"
west init -l config
west update
west zephyr-export
pip install -r zephyr/scripts/requirements.txt
pip install "setuptools<82"          # requirements.txt may have upgraded it again

CFG="$PWD/config"
west build -p auto -s zmk/app -d build/left  -b nice_nano_v2 -- \
  -DSHIELD="keyball61_left nice_view" -DZMK_CONFIG="$CFG" -DSNIPPET=studio-rpc-usb-uart
west build -p auto -s zmk/app -d build/right -b nice_nano_v2 -- \
  -DSHIELD="keyball61_right nice_view" -DZMK_CONFIG="$CFG"
west build -p auto -s zmk/app -d build/reset -b nice_nano_v2 -- \
  -DSHIELD=settings_reset -DZMK_CONFIG="$CFG"
```

Outputs are `build/{left,right,reset}/zephyr/zmk.uf2`. `nice_view_adapter` must stay
out of every `-DSHIELD` list on this board; [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md#2-duplicate-label-nice_view_spi) explains why.

## Flash

Order matters here, and skipping step 1 is the single most common way to end up with a
keyboard that types nothing.

1. `settings_reset` UF2 → **both** halves. A central/peripheral role change invalidates
   the split pairing stored in flash; without this the halves never find each other.
2. `left` UF2 → left controller (it is now the central, so this is the one you cable).
3. `right` UF2 → right controller.
4. Delete the old *Keyball61* entry from the host's Bluetooth list, then pair again.
5. Power both halves on together; they re-pair with each other automatically.

Expected result: the **left** screen shows layer, output and BLE profile (1–5); the
**right** screen shows a static mountain graphic plus battery. That mountain is not a
bug — it is what ZMK's nice!view code draws on a peripheral. Full walkthrough with
troubleshooting: [docs/FLASH.md](docs/FLASH.md).

## Keymap

Seven layers, defined in [config/keyball61.keymap](config/keyball61.keymap):

| Index | Layer | Role |
| --- | --- | --- |
| 0 | `QWRT` | default QWERTY, hold-tap on `SPACE`/`BACKSPACE` |
| 1 | `NUM` | number row + arrows |
| 2 | `SYM` | symbols, Bluetooth profile selection |
| 3 | `FUN` | F1–F12 |
| 4 | `MOUSE` | mouse buttons; entered automatically when the ball moves |
| 5 | `SCROLL` | trackball becomes a scroll wheel |
| 6 | `SNIPE` | trackball at 1/3 speed for precise aiming |

The layer **indices** are load-bearing: `keyball61_left.overlay` references 4, 5 and 6
directly. The `#define MOUSE 3` / `SCROLL 4` / `SNIPE 5` block at the top of the keymap
is stale, is used by no binding, and does not match the indices — see
[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md#layer-indices).

## Tuning

Everything you are likely to want to change, with file and line:
[docs/TUNING.md](docs/TUNING.md). The three that come up first:

- Cursor speed → `cpi = <1200>` in `keyball61_right.overlay` (200–3200, steps of 200)
- Cursor moves while typing → `require-prior-idle-ms = <200>` in `keyball61_left.overlay`
- Cursor direction is wrong → `invert-x` / `invert-y` / `swap-xy` in the same sensor node

## Documentation

| File | Answers |
| --- | --- |
| [docs/BUILD.md](docs/BUILD.md) | toolchain setup, exact build commands, clean-rebuild rules |
| [docs/FLASH.md](docs/FLASH.md) | flash order, re-pairing, what each screen should show |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | how the trackball reaches the host across the split, pin map, Kconfig gates |
| [docs/TUNING.md](docs/TUNING.md) | every knob, its file, its range, what breaks if you move it |
| [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | the six failures this config went through, symptom → cause → fix |
| [docs/PROVENANCE.md](docs/PROVENANCE.md) | what is verified, what is reconstructed, exact commit SHAs |
| [ATTRIBUTION.md](ATTRIBUTION.md) | sources and licence status of everything reused here |

## Credits

This is derived work: hardware definitions from the kit vendor's config, sensor driver
from a third-party ZMK module, firmware from ZMK. See
[ATTRIBUTION.md](ATTRIBUTION.md) — note that the vendor repo carries no licence file.
