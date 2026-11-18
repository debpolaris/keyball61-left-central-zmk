# Provenance

What each part of this repository is, where it came from, and how much of it has
actually been checked. Read this before trusting a version number, a SHA or a checksum.

## 1. Commit structure

Two commits carry the history deliberately:

| Commit | Role |
| --- | --- |
| `52bf9ffb8345157f0540d97ce06102d08208a3b9` — *Import upstream Keyball61 shield definitions* | squashed base: `tangbonze/zmk-config-Keyball61`, branch `niceview`, commit `ce1e93274d8abfd353c770501edcf93844c96647` (2026-04-21). Upstream's 127 commits were folded into one so the delta of this project is one `git diff` wide |
| `4db3d5e0644f2bb317f1d5ddb6a364b03dcf7a9a` — *Make the left half central and keep the trackball working from the peripheral* | everything this project changed |

The import commit was produced by copying upstream's tree, normalising line endings, and
then proving the result byte-for-byte against `git show ce1e932:<path>` for every tracked
file. The only files in it that upstream does not have are `.gitattributes` and
`.gitignore`.

```bash
git show 52bf9ff --stat          # the import
git diff 52bf9ff..HEAD --stat    # every change this project made
```

## 2. What is verified, and how

| Claim | Status | How to re-check |
| --- | --- | --- |
| All three targets compile from this tree | **verified** — Actions run `44243250619` (2026-11-18), commit `4db3d5e`: left + Studio snippet, right, `settings_reset`, all green | `gh run view 44243250619` |
| Shield lists are `keyball61_left nice_view` / `keyball61_right nice_view` with **no** `nice_view_adapter` | **verified** by that same build; adding the adapter is a duplicate-label error | `cat build.yaml` |
| `revision: v0.3` is an annotated tag, not a branch | **verified** — tag object `3cf9006a9377f21c99e944e29e2a507d8c3e9069`, message "Release v0.3", peels to commit `edf5c0814fd3ea202e43aad2d68fd32e882a518c` | `gh api repos/zmkfirmware/zmk/git/tags/3cf9006a9377f21c99e944e29e2a507d8c3e9069` |
| ZMK version at that commit is 0.3.0 | **verified** — `app/VERSION` there reads 0 / 3 / 0 | `gh api "repos/zmkfirmware/zmk/contents/app/VERSION?ref=edf5c081…"` |
| Zephyr is pinned `v3.5.0+zmk-fixes`, SDK 0.16.5 | pinned in the manifests / tooling; **not** re-derived in this repo | `head -2 zephyr/VERSION` after `west update` |
| `badjeff/zmk-pmw3610-driver` branch `zmk-0.3` = `5c5af40de4d8cdf55dc63c4c5907af1e52da6a95` | **verified** (2026-11-18) | `gh api repos/badjeff/zmk-pmw3610-driver/commits/zmk-0.3 --jq .sha` |
| `tangbonze` branch `niceview` = `ce1e932…` | **not verified** — that branch has moved; `52bf9ff` is pinned to `ce1e932` and the drift is expected | see §4 |
| The QWERTY keymap in `config/keyball61.keymap` is the one in daily use | **confirmed by the owner** (2026-11-18), not derived from build output | — |
| Per-pin wiring, Kconfig gating, driver binding properties, `pro_micro` pin numbers | **verified against source** — each claim in [docs/ARCHITECTURE.md](ARCHITECTURE.md) cites the file it came from | read the cited files |
| Trackball behaviour on hardware (auto-mouse feel, scroll and snipe directions, cursor direction) | **unverified since reconstruction** — the flashed keyboard predates the `cpi`/invert/processor values chosen here | flash and try; tune per [docs/TUNING.md](TUNING.md) |
| Left screen = status (layer, output, battery, profile 1–5), right screen = mountain art | owner's recollection, and it matches the widget source that ZMK compiles per role | `app/src/display/CMakeLists.txt`, `app/src/display/widgets/status.c`, `app/src/display/widgets/art.c` |

## 3. Places where this repo deliberately disagrees with the notes

The 2026-07-03 handover document and the 2026-06-12 build log are the origin of this
work, but they are not authoritative on these points:

1. **`nice_view_adapter` in the shield list.** The handover's §4-8 and §5-3 both show it;
   `build.yaml` here does not. Both half overlays already label `&spi2` as
   `nice_view_spi`, so the adapter produces
   `Label 'nice_view_spi' appears on ...` and the build stops. Verified by the green
   build above.
2. **`zip_trackball_scroll_scaler`.** The June config hand-defined this processor.
   Upstream `zip_scroll_scaler` has the same cells and semantics, so the custom node is
   gone.
3. **Stale layer `#define`s.** `config/keyball61.keymap` still declares
   `MOUSE 3 / SCROLL 4 / SNIPE 5`; the layers actually sit at 4/5/6. No binding uses
   those macros, so they are harmless — but they are wrong, and reading them is how
   someone ends up with `layers = <4>` on the scroller. Left in place to keep the
   diff against upstream minimal.
4. **Legacy `CONFIG_PMW3610_*` symbols.** The old driver's tuning symbols are not carried
   over; the equivalents are devicetree properties on the sensor node. Mapping in
   [docs/TUNING.md](TUNING.md#1-sensor-feel--configboardsshieldskeyball61keyball61_rightoverlay).
5. **The squashed history.** Neither source document has this repo's shape; the two-commit
   structure is a presentation choice, not upstream's.

## 4. Known drift risks

Things that can break this repo without any commit here changing.

| Risk | Consequence | Countermeasure |
| --- | --- | --- |
| `badjeff/zmk-pmw3610-driver` `zmk-0.3` branch moves | build silently changes driver code | `config/west.yml` records the branch; pin the SHA in `revision:` if you need an exact lock |
| Upstream deletes that branch or the `v0.3` tag | `west update` / Actions fail outright | fork `zmk` + the driver and repoint `config/west.yml`; keep the UF2s in the Release as the last known-good build |
| ZMK `main` (Zephyr 4.x, HWMv2) is picked up by an unpinned manifest | `No board named 'nice_nano_v2' found` | every manifest revision in this repo is a tag or named branch; never switch to `main` casually — see [docs/TROUBLESHOOTING.md](TROUBLESHOOTING.md#5-no-board-named-nice_nano_v2-found) |
| `setuptools` ≥ 82 reinstalls itself during `pip install -r requirements.txt` | `ModuleNotFoundError: No module named 'pkg_resources'` | re-run `pip install "setuptools<82"` after any requirements install |
| `caksoylar/keymap-drawer` `main` changes rendering | diagrams churn in `[Draw]` commits | pin the workflow ref (currently `@main`) to a release tag if noise becomes a problem |
| Actions artifacts expire (default 90 days) | `gh run download` stops working | binaries are committed under `firmware/` and published as Release assets; checksums in `firmware/SHA256SUMS.txt` |

## 5. Prebuilt firmware

Built by Actions run `44243250619` from commit `4db3d5e` on 2026-11-18, then copied out of
the merged `firmware` artifact. The same three files are committed under `firmware/` and
attached to the GitHub Release.

| File | Bytes | SHA-256 |
| --- | --- | --- |
| `keyball61_left_nice_view-nice_nano_v2-zmk.uf2` | 767,488 | `534c0efb38f6baac0f7c72b0b836f5fc8a45db9a79fb54f84b917d07b379d0d7` |
| `keyball61_right_nice_view-nice_nano_v2-zmk.uf2` | 547,840 | `fcc13a215247be3b47d8cf8f516d02b6a9c243a2ebe931b12e9868e0d2bb50ba` |
| `settings_reset-nice_nano_v2-zmk.uf2` | 92,672 | `4031c0b2f3482cd7070ea5c57b95f3713785c4ca61441bdd986d81fe7de16957` |

Verify after download:

```bash
cd firmware && sha256sum -c SHA256SUMS.txt
# Windows PowerShell:  Get-FileHash -Algorithm SHA256 *.uf2
```

Filenames were changed from the artifact names (`keyball61_left nice_view-…uf2`, with a
space) to underscores, so the checksum list stays parseable and the names survive
shells and URLs. Content is unchanged.

Because these are compiled works, redistributing them carries the licence conditions
described in [ATTRIBUTION.md](../ATTRIBUTION.md) — the Keyball61 hardware lineage is
GPL-3.0 and the two vendor repos publish no licence file of their own.

## 6. Keymap diagrams

`keymap-drawer/keyball61.svg` and `keymap-drawer/keyball61.yaml` were generated by the
`Draw ZMK keymaps` workflow (`caksoylar/keymap-drawer/.github/workflows/draw-zmk.yml@main`)
in run `44243250823`, from `config/keyball61.keymap` at commit `4db3d5e`. The `.yaml` is
keymap-drawer's parsed intermediate — it is what makes a later diagram diff readable.
The workflow writes to `output_folder: keymap-drawer`, so regenerate by dispatching it
rather than editing these files.

Seven layers are drawn, matching the keymap's declaration order:
`default=0 number=1 symbol=2 function=3 mouse=4 scroll=5 snipe=6`.
