# ZMK Migration Implementation Plan
# Migrate to ZMK main (Zephyr 4.1) + Prospector OPERATOR Display

**Created**: 2026-02-22
**Status**: In Progress

---

## Objective

Migrate the Anywhy Flake S keyboard firmware from ZMK v0.3.0 (Zephyr 3.5) to ZMK main
(Zephyr 4.1), replacing the YADS dongle display module (`zmk-dongle-screen`) with the
`prospector-zmk-module` OPERATOR layout. Fix the latent bug where ZMK Studio was
incorrectly assigned to the left peripheral instead of the dongle central.

---

## Repository Map

| Repo | Local Path | Branch | Purpose |
|---|---|---|---|
| `zmk-config` | `~/Projects/Keyboard/zmk-config` | `yads-integrated-module` | Main config (keymap, build matrix, west manifest) |
| `flake-zmk-module` | `~/Projects/Keyboard/flake-zmk-module` | `feat/zephyr-4.1` (new) | Keyboard shield DTS/Kconfig (forked from upstream) |
| `prospector-zmk-module` | `~/Projects/Keyboard/prospector-zmk-module-tku137` | `feat/new-status-screens` | Dongle display module (replaces YADS) |

---

## Key Decisions

| Decision | Choice | Rationale |
|---|---|---|
| ZMK pin | `9490391e1e4010c83291d437b7f9a71ace244581` (Feb 18 2026) | Most recent stable commit; includes `//zmk` board qualifier refactor |
| Board IDs | `nice_nano//zmk`, `xiao_ble//zmk` | Required by ZMK main HWMv2 board qualifier system |
| Display module | `prospector-zmk-module` OPERATOR layout | Information-dense; caps-word via amber Shift label; adaptive battery arcs |
| Caps-word indicator | Amber Shift label in Operator modifier widget | Built into Operator; no separate widget needed |
| Theming | Hardcoded dark Operator palette for now | Tokyo Night port deferred to Phase 4 |
| Custom layer order | Natural ZMK order | Operator dot-row display; acceptable |
| ZMK Studio | Dongle (central) | Corrects the previous bug; peripherals cannot run Studio |
| Zephyr SDK | 0.17.0 (upgrade from 0.16.3) | Required for Zephyr 4.1 |
| Brightness keycodes | Deferred | No Prospector equivalent; Phase 4 item |

---

## What Changes and What Is Lost (Temporarily)

### Changes
- ZMK v0.3.0 → ZMK main (pinned commit)
- Zephyr 3.5 → Zephyr 4.1 (SDK 0.16.3 → 0.17.0)
- `zmk-dongle-screen` (YADS) → `prospector-zmk-module` (Operator layout)
- `dongle_screen` shield → `prospector_adapter` shield
- `seeeduino_xiao_ble` → `xiao_ble//zmk`
- `nice_nano_v2` → `nice_nano//zmk`
- ZMK Studio: left peripheral → dongle central (bug fix)
- `flake-zmk-module` revision: `yads-dongle-support` → `feat/zephyr-4.1`

### Feature Mapping (YADS → Prospector)

| YADS Feature | Prospector Equivalent | Status |
|---|---|---|
| Layer roller (custom order) | Layer dot row + layer name on WPM meter | Natural order, OK |
| Modifier display | Modifier row in Operator | Yes |
| Battery bars (2 peripherals) | Adaptive battery arcs (auto-detects 2 peripherals) | Yes |
| Output (USB/BLE) indicator | Output widget in Operator | Yes |
| WPM display | WPM meter (central feature of Operator) | Yes |
| Ambient light sensor | `CONFIG_PROSPECTOR_USE_AMBIENT_LIGHT_SENSOR=y` | Yes |
| Caps-word indicator (SF Symbol) | Amber Shift label in modifier row | Yes (different style) |
| Caps-word behavior override | Identical mechanism in prospector | Yes |
| Tokyo Night theme | Hardcoded dark Operator palette | Deferred |
| Keyboard brightness keycodes | No equivalent | Deferred |
| Brightness overlay on keypress | No equivalent | Deferred |

### Deferred (Phase 4)
- Tokyo Night color theming for the Operator layout
- Keyboard-driven brightness control via keycodes

---

## Phase 1 — `flake-zmk-module`

**Branch**: `feat/zephyr-4.1` (created off `yads-dongle-support`)

### 1.1 — `boards/shields/anywhy_flake/Kconfig.defconfig`

**Bug fix**: Move `ZMK_STUDIO` from the left peripheral block to the dongle central block.

```
BEFORE (incorrect — peripherals cannot run Studio):
  if SHIELD_ANYWHY_FLAKE_LEFT
    config ZMK_STUDIO
        default y
  endif

AFTER (correct):
  if SHIELD_ANYWHY_FLAKE_DONGLE
    config ZMK_STUDIO
        default y
    ...
  endif
```

### 1.2 — `build.yaml` (module-local CI)

```
BEFORE:
  - board: nice_nano_v2
    shield: anywhy_flake_left
    snippet: studio-rpc-usb-uart
  - board: nice_nano_v2
    shield: anywhy_flake_right
  - board: nice_nano_v2
    shield: settings_reset

AFTER:
  - board: nice_nano//zmk
    shield: anywhy_flake_left
  - board: nice_nano//zmk
    shield: anywhy_flake_right
  - board: nice_nano//zmk
    shield: settings_reset
```

Note: `snippet: studio-rpc-usb-uart` removed from left (Studio is on the dongle now,
which is not in this module's CI file).

---

## Phase 2 — `zmk-config`

### 2.1 — `config/west.yml`

```yaml
# BEFORE:
- name: zmk
  remote: zmkfirmware
  revision: v0.3.0
  import: app/west.yml
- name: flake-zmk-module
  remote: tku137
  revision: yads-dongle-support
- name: zmk-dongle-screen
  remote: tku137
  revision: my-setup

# AFTER:
- name: zmk
  remote: zmkfirmware
  revision: 9490391e1e4010c83291d437b7f9a71ace244581   # main, Feb 18 2026
  import: app/west.yml
- name: flake-zmk-module
  remote: tku137
  revision: feat/zephyr-4.1
- name: prospector-zmk-module
  remote: tku137
  revision: feat/new-status-screens
```

### 2.2 — `build.yaml`

```yaml
# BEFORE:
- board: nice_nano_v2,  shield: anywhy_flake_left,  cmake-args: -DCONFIG_ZMK_SPLIT=y...
- board: nice_nano_v2,  shield: anywhy_flake_right, cmake-args: -DCONFIG_ZMK_SPLIT=y...
- board: seeeduino_xiao_ble, shield: anywhy_flake_dongle dongle_screen
- board: seeeduino_xiao_ble, shield: settings_reset
- board: nice_nano_v2,  shield: settings_reset

# AFTER:
- board: nice_nano//zmk,  shield: anywhy_flake_left
- board: nice_nano//zmk,  shield: anywhy_flake_right
- board: xiao_ble//zmk,   shield: anywhy_flake_dongle prospector_adapter
                          snippet: studio-rpc-usb-uart
                          cmake-args: -DCONFIG_ZMK_STUDIO=y
- board: xiao_ble//zmk,   shield: settings_reset
- board: nice_nano//zmk,  shield: settings_reset
```

Notes:
- `cmake-args` removed from left/right (peripheral role is set by Kconfig.defconfig)
- Studio moves to dongle via `snippet` + `cmake-args`

### 2.3 — `config/anywhy_flake_dongle.conf`

Replace all `CONFIG_DONGLE_SCREEN_*` with `CONFIG_PROSPECTOR_*` equivalents:

```ini
# AFTER (full replacement):
CONFIG_PROSPECTOR_STATUS_SCREEN_OPERATOR=y
CONFIG_PROSPECTOR_MODIFIER_OS_MAC=y
CONFIG_PROSPECTOR_SHOW_MODIFIERS=y
CONFIG_PROSPECTOR_USE_AMBIENT_LIGHT_SENSOR=y
CONFIG_PROSPECTOR_LAYER_NAME_UPPERCASE=n
CONFIG_ZMK_STUDIO=y
CONFIG_ZMK_KEYBOARD_NAME="Flake Dongle"
```

### 2.4 — `config/anywhy_flake.conf`

Remove `CONFIG_ZMK_STUDIO=y` and `CONFIG_ZMK_STUDIO_LOCKING=n` — Studio is on the dongle.
Keep all power management, BLE, and naming settings.

### 2.5 — `.mise.toml`

- `sdk_version`: `"0.16.3"` → `"0.17.0"`
- All `-b nice_nano_v2` → `-b "nice_nano//zmk"`
- All `-b seeeduino_xiao_ble` → `-b "xiao_ble//zmk"`
- Dongle build: `dongle_screen` → `prospector_adapter`; add `-DSNIPPET=studio-rpc-usb-uart`
  and `-DCONFIG_ZMK_STUDIO=y`
- `build-left`: remove `-S studio-rpc-usb-uart` and `-DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=y`
  (left is peripheral only; role set by Kconfig.defconfig)
- `clean-all`: add `prospector-zmk-module/` to cleanup list

---

## Phase 3 — First Build + Error Resolution

```bash
mise run clean-all    # removes build/, .west/, zmk/, zephyr/, .venv/
mise run setup-all    # re-init with new west.yml (ZMK main + prospector)
mise run build-dongle # most complex target first
mise run build-left   # peripheral
mise run build-right  # peripheral
```

### Expected Build Issues (and resolutions)

| Issue | Expected? | Resolution |
|---|---|---|
| Kconfig symbol errors from old DONGLE_SCREEN_* | No (conf replaced) | Verify conf is clean |
| `caps_word` behavior override path changed in ZMK main | Possible | Check CMakeLists.txt in prospector for correct path |
| Missing ZMK main API calls in prospector widgets | Unlikely (branch targets main) | Fix call sites |
| Font compilation issues with Zephyr 4.1 | Unlikely | Update font declarations if needed |
| `zmk_endpoint_get_selected()` API mismatch | No (already updated in prospector fork) | Pre-verified |

---

## Phase 4 — Deferred Work

These items are out of scope for the initial migration and should be tackled separately
once the firmware compiles and runs on hardware.

### 4.1 — Tokyo Night Theming for Operator Layout

Port the `zmk,prospector-theme` DT binding approach (currently only on Radii layout) to
the Operator layout. Define a Tokyo Night palette as a theme node in the overlay and wire
it to `display_colors.h`.

Approximate effort: medium. Reference: `src/layouts/radii/display_colors.h` +
`dts/bindings/zmk,prospector-theme.yaml` in the prospector fork.

### 4.2 — Keyboard-Driven Brightness Control

Port the YADS brightness keycode system (F22/F23/F24 → toggle/down/up brightness) to the
prospector module. This requires:
1. Adding Kconfig symbols analogous to `DONGLE_SCREEN_BRIGHTNESS_KEYBOARD_CONTROL`
2. Extending `src/brightness.c` to subscribe to `zmk_keycode_state_changed` events
3. Wiring the keycode constants to PWM brightness adjustments

Approximate effort: medium-large. Reference: `zmk-dongle-screen-tku137/boards/shields/
dongle_screen/src/brightness.c`.

---

## Verification Checklist

After Phase 3 compiles successfully:

- [ ] `mise run build-dongle` succeeds
- [ ] `mise run build-left` succeeds
- [ ] `mise run build-right` succeeds
- [ ] `mise run build-reset-dongle` succeeds
- [ ] `mise run build-reset-flake` succeeds
- [ ] Flash dongle firmware → display shows Operator layout
- [ ] Flash left + right → keyboard connects to dongle over BLE
- [ ] Layer names display correctly on Operator screen
- [ ] Modifier indicators respond to keypress
- [ ] Battery arcs show both peripherals
- [ ] Caps-word activates → Shift label turns amber
- [ ] ZMK Studio connects via USB to dongle
- [ ] Ambient light sensor adjusts brightness
- [ ] GitHub Actions CI passes all 5 build targets

---

## Notes

- The `anywhy_flake_dongle.overlay` in `flake-zmk-module` remains the correct home for
  the dongle DTS config (mock kscan, small_layout chosen). No changes needed there.
- The prospector module's `boards/xiao_ble_zmk.overlay` configures the display hardware
  (SPI3, ST7789V, APDS9960, PWM backlight) — this is complementary to and does not
  conflict with the flake dongle overlay.
- The `anywhy_flake.zmk.yml` `features: [studio]` flag stays — it is correct metadata
  for ZMK Studio hardware discovery on the shield level.
- `build-left` in `.mise.toml` remains as a local-only standalone build (not in
  `build.yaml` CI). It is kept for convenience but is not the production build path.
