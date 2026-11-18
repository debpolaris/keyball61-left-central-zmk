# Building locally

Runs on Debian/Ubuntu x86_64, native or WSL2. Every block below is meant to be pasted
in order; commands that depend on the virtualenv re-activate it instead of assuming
your shell still has it.

## Version pins — do not float these

| Component | Pin | Where | What goes wrong if you drift |
| --- | --- | --- | --- |
| ZMK | `v0.3` (0.3.0) | `config/west.yml` | ZMK `main` moved to Zephyr 4.1 + HWMv2 in Dec 2025: `nice_nano_v2` ceases to exist as a board name (becomes `nice_nano`), SDK 0.17 and LVGL 9.3 are required, and half the config symbols in this repo become deprecated. Symptom: `No board named 'nice_nano_v2' found`. |
| Zephyr | `v3.5.0+zmk-fixes` | pulled in by ZMK's `app/west.yml` | ZMK v0.3 is the last release on the Zephyr 3.5 line and the first that carries the pointing subsystem this config depends on. |
| PMW3610 driver | branch `zmk-0.3` | `config/west.yml` | badjeff's `main` and `zmk-0.4` target Zephyr 4.1. On `zmk-0.3` the compatible is `pixart,pmw3610` and the Kconfig symbol is `PMW3610`; the other branches use different spellings. |
| setuptools | `< 82` | the venv | setuptools 82.0.0 (2026-02-08) removed `pkg_resources`, which the old nanopb bundled with ZMK v0.3 needs to generate the ZMK Studio protobuf. Symptom: `ModuleNotFoundError: No module named 'pkg_resources'` during the **left** build only. |
| Zephyr SDK | `0.16.5` | `~/zephyr-sdk-0.16.5` | matches what Zephyr 3.5 expects. |

## 1. System packages

```bash
sudo apt update
sudo apt install -y git cmake ninja-build gperf ccache dfu-util wget \
  python3 python3-pip python3-venv unzip device-tree-compiler build-essential
```

## 2. Workspace and sources

```bash
mkdir -p ~/KeyBall61 && cd ~/KeyBall61
git clone https://github.com/debpolaris/keyball61-left-central-zmk.git
```

## 3. Virtualenv, west, and the setuptools pin

```bash
cd ~/KeyBall61/keyball61-left-central-zmk
python3 -m venv .venv
source .venv/bin/activate
pip install west
pip install "setuptools<82"
python -c "import pkg_resources; print('pkg_resources ok')"
```

Keep the venv inside the repo (`./.venv`) so the whole tree is self-contained; it is
git-ignored.

## 4. Initialise the west workspace

```bash
cd ~/KeyBall61/keyball61-left-central-zmk
west init -l config
```

`config/west.yml` declares `self: path: config`, and `west init -l config` is run from
the repo root, so the repo root becomes the workspace and `.west/` is created inside it.

If west aborts with a *self path mismatch* / *manifest* error, it is because the
manifest repo's directory name does not match `self.path`. Initialise from one level up
instead and point west at the manifest explicitly:

```bash
cd ~/KeyBall61
west init .
west config manifest.path keyball61-left-central-zmk/config
west config manifest.file west.yml
west config -l            # shows the resolved path and file
```

## 5. Pull the sources

```bash
cd ~/KeyBall61/keyball61-left-central-zmk
source .venv/bin/activate
west update
west zephyr-export
pip install -r zephyr/scripts/requirements.txt
pip install "setuptools<82"     # requirements.txt may have pulled setuptools back up
python -c "import pkg_resources; print('pkg_resources ok')"
head -2 zephyr/VERSION          # expect VERSION_MAJOR = 3 / VERSION_MINOR = 5
```

Roughly 1 GB. `head -2 zephyr/VERSION` is the cheap check that ZMK really resolved to
the Zephyr 3.5 fork.

## 6. Zephyr SDK 0.16.5

Skip this if any other ZMK build has ever run on this machine: `./setup.sh -c` registers
the SDK in `~/.cmake/packages/Zephyr-sdk/`, which is machine-global and survives reboots
and new shells. `west build` then finds it by itself.

```bash
cd ~
wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.5/zephyr-sdk-0.16.5_linux-x86_64.tar.xz
tar xf zephyr-sdk-0.16.5_linux-x86_64.tar.xz
cd ~/zephyr-sdk-0.16.5 && ./setup.sh -t arm-zephyr-eabi -c
```

Re-run `./setup.sh -c` if you move or rename the SDK directory — the registration stores
an absolute path. If a build still cannot find the toolchain, set both explicitly in the
same shell:

```bash
export ZEPHYR_TOOLCHAIN_VARIANT=zephyr-sdk
export ZEPHYR_SDK_INSTALL_DIR=~/zephyr-sdk-0.16.5
```

## 7. Build the three images

These are the same three targets `build.yaml` gives GitHub Actions.

```bash
cd ~/KeyBall61/keyball61-left-central-zmk
source .venv/bin/activate
CFG="$PWD/config"

# left = central: USB + ZMK Studio, and the trackball input processors
west build -p auto -s zmk/app -d build/left -b nice_nano_v2 \
  -S studio-rpc-usb-uart -- \
  -DSHIELD="keyball61_left nice_view" -DZMK_CONFIG="$CFG"

# right = peripheral: carries the PMW3610
west build -p auto -s zmk/app -d build/right -b nice_nano_v2 -- \
  -DSHIELD="keyball61_right nice_view" -DZMK_CONFIG="$CFG"

# pairing wipe, flashed to both halves first after any role change
west build -p auto -s zmk/app -d build/reset -b nice_nano_v2 -- \
  -DSHIELD=settings_reset -DZMK_CONFIG="$CFG"
```

Two things about that snippet are easy to get wrong:

- **`nice_view_adapter` must not appear in any `-DSHIELD` list.** Both half overlays
  already label `spi2` as `nice_view_spi`; adding the adapter produces a duplicate-label
  error. See [TROUBLESHOOTING.md](TROUBLESHOOTING.md).
- **The snippet flag goes *before* `--`.** Anything after `--` is handed to CMake, where
  `-S` means "source directory" and gets silently misread. After `--`, the equivalent is
  `-DSNIPPET=studio-rpc-usb-uart`.

Outputs: `build/left/zephyr/zmk.uf2`, `build/right/zephyr/zmk.uf2`,
`build/reset/zephyr/zmk.uf2`.

## 8. When you must clean

| You changed | Do this |
| --- | --- |
| anything under `config/` | plain rebuild is fine (`-p auto` prunes as needed) |
| `config/west.yml`, or any ZMK/driver revision | `rm -rf zephyr modules zmk-pmw3610-driver && west update`, then `head -2 zephyr/VERSION` |
| a shield string, or you hit a stale CMake cache error | `west build -t pristine -d build/<which>`, or delete `build/<which>` |
| ZMK Studio bindings and the build looks unchanged | rebuild the **left** image; Studio lives only on the central |

Skipping the second row is how people end up with a half-updated tree that no longer
matches the manifest — west does not rewrite already-cloned modules on `west update`.

## 9. Sanity checks before flashing

```bash
# the left image must contain Studio and the listener
grep -E "CONFIG_ZMK_STUDIO=|CONFIG_ZMK_INPUT_LISTENER=|CONFIG_ZMK_INPUT_PROCESSOR_TEMP_LAYER=|CONFIG_ZMK_POINTING=" build/left/zephyr/.config

# the right image must have the relay but not the listener
grep -E "CONFIG_ZMK_INPUT_SPLIT=|CONFIG_ZMK_INPUT_LISTENER=|CONFIG_PMW3610=|CONFIG_ZMK_POINTING=" build/right/zephyr/.config
```

Expected: left shows `ZMK_STUDIO=y`, `ZMK_INPUT_LISTENER=y`, `..._TEMP_LAYER=y`,
`ZMK_POINTING=y`. Right shows `ZMK_INPUT_SPLIT=y`, `PMW3610=y`, `ZMK_POINTING=y`, and
**no** `CONFIG_ZMK_INPUT_LISTENER` line at all — that absence is the central-only gate
working, not a mistake.
