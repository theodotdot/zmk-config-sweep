# AGENTS.md

## What this repo is

A [ZMK](https://zmk.dev) split-keyboard firmware *configuration* repo (not firmware source). It targets the **cradio** shield on a **nice_nano** controller. There is no firmware code here: only the keymap, Kconfig overrides, and CI wiring that the upstream ZMK build system consumes.

## Layout

| Path | Purpose |
|------|---------|
| `config/` | ZMK config root (declared by `west.yml` `self: path: config`). Everything the firmware build reads lives here. |
| `config/cradio.keymap` | Devicetree keymap: layers, combos, custom behaviors, conditional layers. |
| `config/cradio.conf` | Kconfig overrides (BLE, sleep, USB). |
| `config/west.yml` | West manifest. Pins ZMK to `main` via `zmkfirmware/zmk` and imports `app/west.yml`. |
| `build.yaml` | GitHub Actions build matrix (repo root is mandatory). Each `include` entry is one board+shield target. |
| `boards/shields/` | Placeholder (only `.gitkeep`) for custom shields. Not currently used. |
| `zephyr/module.yml` | Declares this repo a Zephyr module with `board_root: .`. |
| `keymap-drawer/` | **Generated output** of the `draw-keymaps` workflow (`cradio.svg`, `cradio.yaml`) plus hand-written `config.yaml` styling. |
| `.github/workflows/build.yml` | Delegates to the upstream reusable `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main`. |
| `.github/workflows/draw-keymaps.yml` | Delegates to `caksoylar/keymap-drawer` to render the SVG/YAML and auto-commit them. |

## Building and verifying

There is **no local build, test, or lint tooling** in this repo (no Makefile, `package.json`, scripts, or test suite). Verification happens entirely in GitHub Actions:

- **Firmware build**: `.github/workflows/build.yml`, triggered on push / PR / manual dispatch. Matrix defined in `build.yaml`.
- **Keymap rendering**: `.github/workflows/draw-keymaps.yml`, triggered on changes to `config/*.keymap`, `config/*.dtsi`, or the drawer config, plus manual dispatch.

Because there is no local compile step, a keymap change cannot be validated in-repo. Push and watch the Actions run, or use the [Keymap Editor](https://nickcoutsos.github.io/keymap-editor/) referenced in the README.

## Keymap conventions (`config/cradio.keymap`)

- 34-key, 4-row layout written as `10 / 10 / 10 / 4` bindings per layer. Key positions in combos and hold-triggers are **flat indices** across those rows (row 1 = 0-9, row 2 = 10-19, row 3 = 20-29, thumbs = 30-33).
- Layer indices are aliased near the top: `DEF 0`, `LWR 1`, `RSE 2`, `ADJ 3`. These `#define`s document intent but are not referenced in bindings; bindings and `conditional_layers` use the raw numbers.
- Layer nodes are **not named after the aliases**. The mapping is:
  - index 0 `default_layer` (display-name `MAIN`) = DEF
  - index 1 `nav` = LWR
  - index 2 `num_fun` = RSE
  - index 3 `bt_layer` (display-name `BT`) = ADJ. The "adjust" slot doubles as the Bluetooth layer.
- The `num_fun` and `bt_layer` nodes have no `display-name`/`label`; the drawer derives names from the node name.
- `&lt 2 TAB` / `&lt 1 SPACE` in the keymap refer to layer *indices* (num_fun, nav), not names.
- Home-row mods: `&mt` on the left half, custom `hm_r` (defined inline in the keymap using `hold-trigger-key-positions`) on the right half. Multiple commits have iterated on the right-side mods per-key; expect these two sides to evolve independently.
- The tri-layer is a `conditional_layers` entry (`if-layers = <2 1>` -> `then-layer = <3>`).
- Timing values (`quick-tap-ms`, `require-prior-idle-ms`, `flavor`, `tapping-term-ms`) are duplicated across the `&lt`/`&mt` overrides and the `hm_r` behavior. When retuning, update all relevant blocks or the halves behave differently.

## Gotchas

- **draw-keymaps path mismatch**: `draw-keymaps.yml` points `config_path` (and the `paths` trigger) at `keymap_drawer/config.yaml` with an **underscore**, but the directory on disk is `keymap-drawer` with a **hyphen**. The upstream action ignores a missing config file, so the styling in `keymap-drawer/config.yaml` may not actually be applied, and edits to it won't trigger a re-render. Fix the path in both the `paths` list and `config_path` if you want the custom mappings to take effect.
- **`keymap-drawer/cradio.svg` and `cradio.yaml` are generated** and auto-committed by CI. Don't hand-edit them; change `config/cradio.keymap` (or the drawer config) and let the workflow regenerate, or run keymap-drawer locally yourself.
- **`.conf` naming**: `config/cradio.conf` targets the `cradio` shield. ZMK strips the `_left`/`_right` suffix from split shields, so the single `cradio.conf` applies to both `cradio_left` and `cradio_right` targets.
- **`build.yaml` board qualifier**: entries use `board: nice_nano//zmk`. The `//zmk` suffix is required by the current ZMK/Zephyr toolchain (a prior commit "fix build.yaml since zephyr update" corrected this); don't drop it.
- **`west.yml` tracks ZMK `main`**, so firmware behavior can shift without any change in this repo. If a build breaks unexpectedly, suspect an upstream change first.
- The root `build.yaml` filename and location are fixed by the ZMK build workflow; moving it breaks CI.
