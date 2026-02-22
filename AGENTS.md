# AGENTS.md — ZMK Config Repository Guide

This file provides context for AI coding agents working in this ZMK keyboard firmware
configuration repository.

---

## Repository Overview

- **Keyboard**: Anywhy Flake S — 40-key (10×4) wireless split keyboard with dongle
- **Firmware**: ZMK main (pinned Feb 18 2026, commit `9490391e`) on Zephyr 4.1
- **Hardware**:
  - Left half: `nice_nano//zmk` (nRF52840, peripheral)
  - Right half: `nice_nano//zmk` (nRF52840, peripheral)
  - Dongle (central): `xiao_ble//zmk` with Prospector OPERATOR ST7789V display
- **License**: GPL v3.0

---

## Directory Structure

```
zmk-config/
├── .mise.toml              # Dev environment + ALL build tasks (primary task runner)
├── build.yaml              # ZMK CI build matrix (5 targets)
├── config/                 # ZMK user config — keymap, Kconfig, west manifest
│   ├── west.yml            # West manifest (pins ZMK main + custom modules)
│   ├── anywhy_flake.keymap # Main keymap (DTS format, 5 layers)
│   ├── anywhy_flake.conf   # Kconfig for keyboard halves (peripherals)
│   └── anywhy_flake_dongle.conf
├── flake-zmk-module/       # Shield definitions (west module/submodule)
│   └── boards/shields/anywhy_flake/
├── prospector-zmk-module/  # Prospector dongle screen module (tku137 fork, feat/new-status-screens)
│   └── boards/shields/prospector_adapter/
├── keymap-drawer/          # SVG keymap visualization + config
├── notes/                  # Developer notes (not critical to builds)
├── artifacts/              # Built UF2 outputs (gitignored)
├── build/                  # CMake build dirs (gitignored)
├── zmk/                    # ZMK source (west-managed, gitignored)
└── zephyr/                 # Zephyr source (west-managed, gitignored)
```

---

## Build Commands

All tasks are managed via **mise** (`.mise.toml`). Requires west, CMake, Ninja, dtc, and
Zephyr SDK 0.17.0 installed at `~/zephyr-sdk-0.17.0`.

### Initial Setup

```bash
mise run setup-all       # Full dev environment setup (one-time)
mise run doctor          # Sanity check tools and SDK
```

### Building Firmware

```bash
# Individual targets
mise run build-left                # Left half (peripheral, nice_nano//zmk)
mise run build-right               # Right half (peripheral, nice_nano//zmk)
mise run build-dongle              # Dongle with Prospector screen + ZMK Studio (central, xiao_ble//zmk)
mise run build-left-peripheral     # Alias for build-left (dongle setup)
mise run build-right-peripheral    # Alias for build-right (dongle setup)
mise run build-reset-flake         # Settings reset for keyboard halves
mise run build-reset-dongle        # Settings reset for dongle

# Combined targets
mise run build-all                 # Left + right + settings-reset
mise run build-all-dongle          # All 5 dongle-setup targets
mise run dongle                    # build-all-dongle + collect artifacts + flash info
mise run release                   # build-all + artifacts + draw-keymap (full release)
```

### Raw west commands (used internally by mise)

```bash
# Left half (peripheral)
west build -p -d build/flake-left -s zmk/app -b "nice_nano//zmk" \
  -- -DSHIELD=anywhy_flake_left -DZMK_CONFIG="$ZMK_CONFIG"

# Dongle (central + ZMK Studio)
west build -p -d build/flake-dongle -s zmk/app -b "xiao_ble//zmk" \
  -S studio-rpc-usb-uart \
  -- -DSHIELD="anywhy_flake_dongle prospector_adapter" \
     -DZMK_CONFIG="$ZMK_CONFIG" \
     -DCONFIG_ZMK_STUDIO=y
```

### Other Tasks

```bash
mise run artifacts           # Collect UF2s from build/ into ./artifacts/
mise run parse-keymap        # Parse .keymap → YAML (keymap-drawer)
mise run draw-keymap         # Render SVG keymap visualization
mise run clean               # Remove build/
mise run clean-all           # Remove build/, .west/, zmk/, zephyr/, .venv/, prospector-zmk-module/
mise run update              # west update + pip deps
mise run menuconfig          # Open Kconfig menuconfig for last built target
```

### There Are No Tests

ZMK is an embedded firmware project. There is no unit test suite. "Testing" means flashing
firmware to hardware. CI (`.github/workflows/build.yml`) verifies that all 5 build targets
compile successfully via the ZMK reusable workflow.

---

## Code Style

This repo spans multiple file types. Follow the conventions from the upstream Zephyr/ZMK
projects (`zephyr/.editorconfig`, `zmk/.clang-format`).

### General (all files)

- Encoding: UTF-8
- Line endings: LF (`\n`)
- Final newline: required
- Trailing whitespace: trim
- Max line length: 100 characters

### C / Header files (`*.c`, `*.h`) — Zephyr/ZMK style

- Indentation: **tabs**, tab size = 8 (Zephyr convention, not spaces)
- Brace style: Linux kernel style (K&R variant)
- `clang-format` config (from `zmk/.clang-format`):
  ```
  BasedOnStyle: LLVM
  IndentWidth: 4
  ColumnLimit: 100
  SortIncludes: false
  ```
- Include guards: use `#pragma once` or traditional `#ifndef` guards
- Include order: system headers first, then ZMK headers, then local headers
- Error handling: use Zephyr's `LOG_ERR`, `LOG_WRN`, `LOG_INF` macros from `<zephyr/logging/log.h>`
- Naming: `snake_case` for functions, variables, structs; `ALL_CAPS` for macros and Kconfig symbols
- Structs/types: `snake_case_t` suffix for typedefs is NOT used in ZMK — use plain struct names

### Devicetree / Keymap files (`*.dts`, `*.dtsi`, `*.overlay`, `*.keymap`)

- Indentation: **tabs**, tab size = 8
- Node names: `kebab-case` (e.g., `anywhy-flake-left`)
- Property names: `kebab-case` (e.g., `tapping-term-ms`, `#binding-cells`)
- Label names for ZMK behaviors: `snake_case` (e.g., `ms`, `mo_tog`, `tdsc`, `skq`)
- Behavior node names: `snake_case` matching the label (e.g., `ms: magic_shift { ... }`)
- Bindings arrays: align columns with spaces for readability (see existing keymap layers)
- Comments: use `/* block comments */` for section headers, `//` for inline (DTS supports both)
- Each layer binding grid: keep keys aligned in rows matching the physical layout

### Kconfig files (`Kconfig`, `Kconfig.defconfig`, `Kconfig.shield`)

- Indentation: **tabs**, tab size = 8
- `CONFIG_` prefix is used only when referencing symbols in `.conf` files and C code;
  Kconfig definitions themselves omit the prefix
- Boolean options: `default y` / `default n`
- `depends on` and `select` for dependencies

### YAML files (`*.yml`, `*.yaml`)

- Indentation: 2 spaces
- West manifest (`west.yml`): follow existing format with `manifest:`, `projects:`, `self:`
- `build.yaml`: follows ZMK build matrix format — `boards:` list with `board`, `shield`, `cmake-args`

### CMake (`CMakeLists.txt`)

- Indentation: 2 spaces
- Conditional compilation: use `if(CONFIG_...)` for Kconfig-gated sources
- `target_sources(app PRIVATE ...)` pattern for adding widget/driver sources

### Shell / mise tasks (`.mise.toml` `run` blocks)

- Indentation: 2 spaces (TOML)
- Shell code in `run = '''...'''` blocks: POSIX sh compatible
- Variable names: `UPPER_SNAKE_CASE` for environment variables, `lower_snake_case` for locals

---

## Key Configuration Details

### West Modules (config/west.yml)

- `zmk`: `zmkfirmware/zmk` @ `9490391e1e4010c83291d437b7f9a71ace244581` (main, Feb 18 2026)
- `flake-zmk-module`: `tku137/flake-zmk-module` @ `feat/zephyr-4.1`
- `prospector-zmk-module`: `tku137/prospector-zmk-module` @ `feat/new-status-screens`

After any `west.yml` change, run `mise run update` to sync.

### Board Qualifiers (ZMK main / HWMv2)

ZMK main adopted Zephyr HWMv2. Bare board names no longer work — always use the qualified form:
- `nice_nano//zmk` (not `nice_nano_v2`)
- `xiao_ble//zmk` (not `seeeduino_xiao_ble`)

### Split Roles

- **Left half**: peripheral (`SHIELD_ANYWHY_FLAKE_LEFT` in `Kconfig.defconfig`)
- **Right half**: peripheral (`SHIELD_ANYWHY_FLAKE_RIGHT`)
- **Dongle**: central + ZMK Studio (`SHIELD_ANYWHY_FLAKE_DONGLE` in `Kconfig.defconfig`)
- ZMK Studio must run on the **central** (dongle), not on peripherals.

### Keymap Layers (config/anywhy_flake.keymap)

5 layers: `base` (0), `nav` (1), `num` (2), `sym` (3), `fn` (4)
Conditional layer: Nav(1) + Sym(3) simultaneously activates Fn(4).

Physical layout selected: `small_layout` (4×10, 40 keys).

### Dongle Screen (prospector-zmk-module/)

Prospector OPERATOR layout. Active features are controlled by Kconfig in
`config/anywhy_flake_dongle.conf`. Key Kconfig symbols:
`CONFIG_PROSPECTOR_STATUS_SCREEN_OPERATOR`, `CONFIG_PROSPECTOR_MODIFIER_OS_MAC`,
`CONFIG_PROSPECTOR_USE_AMBIENT_LIGHT_SENSOR`.

Shield name when building: `prospector_adapter` (combined with `anywhy_flake_dongle`).

---

## CI / Automation

- **Build CI** (`.github/workflows/build.yml`): delegates to ZMK's reusable workflow;
  builds all 5 targets from `build.yaml` on every push.
- **Keymap drawing** (`.github/workflows/draw-keymaps.yml`): triggers on changes to
  `config/*.keymap` or `keymap-drawer/**`; runs `keymap-drawer` and commits the updated SVG
  back to the repo automatically.

Do not manually edit `keymap-drawer/anywhy_flake.svg` — it is auto-generated.
The canonical YAML is `keymap-drawer/anywhy_flake.yaml` (committed); local builds produce
`keymap-drawer/anywhy_flake.local.yaml` and `anywhy_flake.local.svg` (gitignored).
