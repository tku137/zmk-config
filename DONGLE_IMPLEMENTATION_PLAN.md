# Flake Keyboard Dongle Implementation Plan

## Executive Summary

This document outlines the problems encountered when trying to add dongle support to the Anywhy Flake keyboard using ZMK firmware, and provides a detailed implementation plan to resolve them.

**Goal**: Make the Flake S keyboard work with a YADS (Yet Another Dongle Screen) dongle as the central device, with both keyboard halves as peripherals.

---

## Table of Contents

1. [Background](#background)
2. [Problem Analysis](#problem-analysis)
3. [Root Cause](#root-cause)
4. [Solution Overview](#solution-overview)
5. [Implementation Plan](#implementation-plan)
6. [File Changes Reference](#file-changes-reference)
7. [Testing Checklist](#testing-checklist)

---

## Background

### Current Setup

- **Keyboard**: Anywhy Flake S variant (40 keys, 4 rows × 10 columns)
- **Keyboard halves**: nice!nano v2 controllers
- **Dongle**: Seeeduino XIAO BLE with YADS screen
- **Firmware**: ZMK v0.3.0

### How the Flake Module Works

The `flake-zmk-module` defines three physical layouts for different Flake variants:
- `large_layout` / `large_transform`: 58 keys (Flake L)
- `medium_layout` / `medium_transform`: 46 keys (Flake M)
- `small_layout` / `small_transform`: 40 keys (Flake S)

The M and S variants share the same PCB - the S variant has the outer row and columns physically snapped off.

### Original Design (No Dongle)

The module was designed for a **non-dongle setup**:
- Left half = Central (runs keymap, connects to host)
- Right half = Peripheral (sends key events to left half)

```
[Right Half] --BLE--> [Left Half (Central)] --USB/BLE--> [Host Computer]
```

### Desired Design (With Dongle)

```
[Left Half]  --BLE--> [Dongle (Central)] --USB--> [Host Computer]
[Right Half] --BLE--> [Dongle (Central)]
```

---

## Problem Analysis

### Symptom

When pressing keys on the keyboard with the dongle setup:

**Left half sends wrong keys:**
- Top row (should be Q W E R T) → outputs `f g h j k`
- Second row → outputs `n m , . /`
- Third row → outputs layer keys
- Thumb row → outputs nothing

**Right half sends wrong keys:**
- Top row → outputs `l ; z x c`
- Second row → outputs modifier keys
- Bottom rows → output nothing

### What This Tells Us

The keys being output don't match their physical positions. For example:
- `f g h j k` are positions 13-17 in the keymap (home row, spanning both halves)
- But they're being triggered by the top row of the left half (positions 0-4)

This is a **position mapping mismatch** - the dongle is receiving key positions that don't match what the peripherals think they're sending.

---

## Root Cause

### Problem 1: Separate Transform Definitions

The current setup has the dongle shield in a **separate folder** (`config/boards/shields/flake_screen_dongle/`) with its own transform definition:

```
flake-zmk-module/boards/shields/anywhy_flake/
├── anywhy_flake.dtsi          ← defines small_transform
├── anywhy_flake_left.overlay  ← includes dtsi, uses small_transform
└── anywhy_flake_right.overlay ← includes dtsi, uses small_transform

config/boards/shields/flake_screen_dongle/
└── anywhy_flake_dongle.overlay ← defines its OWN small_transform_dongle (COPY!)
```

Even though `small_transform_dongle` has identical RC() values to `small_transform`, they are **different devicetree nodes**. ZMK's position mapping may handle them differently.

### Problem 2: Physical Layout Mismatch

The module's `anywhy_flake.dtsi` sets the default physical layout:

```c
chosen {
    zmk,physical-layout = &large_layout;  // DEFAULT is large, not small!
};
```

When building the left/right peripherals, they default to `large_layout` (58 keys) unless explicitly overridden. But the dongle expects `small_layout` (40 keys).

### Problem 3: Overlay Files Not Applied

We created `config/anywhy_flake_left.overlay` and `config/anywhy_flake_right.overlay` to override the layout:

```c
/ {
    chosen {
        zmk,physical-layout = &small_layout;
    };
};
```

**However**, ZMK does not automatically load overlay files from the config folder. They must be explicitly passed via `-DEXTRA_DTC_OVERLAY_FILE=...` during build.

Even after adding this flag, the problem persisted, suggesting the issue is deeper than just layout selection.

### Problem 4: The Fundamental Architecture Issue

In ZMK split keyboards:

1. **Peripheral** kscan detects physical key press → (row, col)
2. **Peripheral** applies its transform → converts to position number
3. **Peripheral** sends position to central
4. **Central** receives position and maps to keymap binding

The critical insight: **The peripheral applies the transform, not the central.**

This means:
- Left peripheral uses `small_transform` from the module's dtsi
- Right peripheral uses `small_transform` (with col-offset=6) from the module's dtsi
- Dongle has `small_transform_dongle` which is a **different node**

Even if the RC() values are identical, ZMK may be handling the position calculation differently because they're separate devicetree nodes, not references to the same node.

---

## Solution Overview

### The Fix: Integrate Dongle into the Module

The dongle shield must be **part of the same module** as the keyboard halves, in the **same shield folder**, so it can:

1. `#include "anywhy_flake.dtsi"` to get the **exact same** transform nodes
2. Reference `&small_transform` and `&small_layout` directly (not copies)
3. Share all devicetree definitions with the keyboard halves

### Why This Works

When the dongle includes the dtsi:
```c
#include "anywhy_flake.dtsi"

/ {
    chosen {
        zmk,physical-layout = &small_layout;  // Same node as peripherals!
    };
};
```

Now all three devices (left, right, dongle) reference the **exact same** `small_layout` and `small_transform` nodes. Position numbers will be consistent.

---

## Implementation Plan

### Phase 1: Fork the Flake Module

**Step 1.1**: Fork the repository
- Go to https://github.com/anywhy-io/flake-zmk-module
- Click "Fork" to create your own copy
- Clone your fork locally

**Step 1.2**: Create a dongle branch (optional but recommended)
```bash
cd /path/to/your/flake-zmk-module-fork
git checkout -b dongle-support
```

### Phase 2: Add Dongle Shield to Module

**Step 2.1**: Create the dongle overlay file

Create `boards/shields/anywhy_flake/anywhy_flake_dongle.overlay`:

```c
/*
 * Anywhy Flake Dongle Overlay
 * 
 * Sets up the dongle as central with no physical keys.
 * Uses small_transform/small_layout from the shared dtsi.
 */

#include "anywhy_flake.dtsi"

/ {
    chosen {
        zmk,kscan = &mock_kscan;
        zmk,physical-layout = &small_layout;
    };

    mock_kscan: mock_kscan_0 {
        compatible = "zmk,kscan-mock";
        columns = <0>;
        rows = <0>;
        events = <0>;
    };
};
```

**Step 2.2**: Update Kconfig.shield

Edit `boards/shields/anywhy_flake/Kconfig.shield` to add dongle:

```kconfig
config SHIELD_ANYWHY_FLAKE_LEFT
    def_bool $(shields_list_contains,anywhy_flake_left)

config SHIELD_ANYWHY_FLAKE_RIGHT
    def_bool $(shields_list_contains,anywhy_flake_right)

# ADD THIS:
config SHIELD_ANYWHY_FLAKE_DONGLE
    def_bool $(shields_list_contains,anywhy_flake_dongle)
```

**Step 2.3**: Update Kconfig.defconfig

Edit `boards/shields/anywhy_flake/Kconfig.defconfig`:

```kconfig
# DONGLE CONFIG (ADD THIS SECTION)
if SHIELD_ANYWHY_FLAKE_DONGLE

config ZMK_KEYBOARD_NAME
    default "Flake Dongle"

config ZMK_SPLIT_ROLE_CENTRAL
    default y

config ZMK_SPLIT
    default y

config ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS
    default 2

config BT_MAX_CONN
    default 7

config BT_MAX_PAIRED
    default 7

endif

# LEFT HALF CONFIG (MODIFY - remove central role)
if SHIELD_ANYWHY_FLAKE_LEFT

config ZMK_KEYBOARD_NAME
    default "Anywhy Flake"

# REMOVE these lines (dongle is now central):
# config ZMK_SPLIT_ROLE_CENTRAL
#     default y
# config ZMK_STUDIO
#     default y

endif

# BOTH HALVES
if SHIELD_ANYWHY_FLAKE_LEFT || SHIELD_ANYWHY_FLAKE_RIGHT

config ZMK_SPLIT
    default y

endif
```

**Step 2.4**: Commit and push
```bash
git add .
git commit -m "feat: add dongle support for Flake S with YADS screen"
git push origin dongle-support
```

### Phase 3: Update Your ZMK Config

**Step 3.1**: Update west.yml

Edit `config/west.yml` to point to your fork:

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: YOUR_GITHUB_USERNAME
      url-base: https://github.com/YOUR_GITHUB_USERNAME
    - name: janpfischer
      url-base: https://github.com/janpfischer
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
    - name: flake-zmk-module
      remote: YOUR_GITHUB_USERNAME
      revision: dongle-support  # Or 'main' if you merged
    - name: zmk-dongle-screen
      remote: janpfischer
      revision: main
  self:
    path: config
```

**Step 3.2**: Update build.yaml

```yaml
---
include:
  # Left peripheral
  - board: nice_nano_v2
    shield: anywhy_flake_left
    cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
    artifact-name: flake-left
  
  # Right peripheral  
  - board: nice_nano_v2
    shield: anywhy_flake_right
    cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
    artifact-name: flake-right
  
  # Dongle with YADS screen
  - board: seeeduino_xiao_ble
    shield: anywhy_flake_dongle dongle_screen
    artifact-name: flake-dongle
  
  # Settings reset
  - board: seeeduino_xiao_ble
    shield: settings_reset
    artifact-name: reset-dongle
  - board: nice_nano_v2
    shield: settings_reset
    artifact-name: reset-flake
```

**Step 3.3**: Clean up old files

Remove these files/folders that are no longer needed:
- `config/boards/shields/flake_screen_dongle/` (entire folder)
- `config/anywhy_flake_left.overlay`
- `config/anywhy_flake_right.overlay`
- `config/anywhy_flake_left.conf`
- `config/anywhy_flake_right.conf`

**Step 3.4**: Update west and rebuild

```bash
# Update dependencies to fetch your forked module
west update

# Clean old builds
rm -rf build/

# Build all firmware
west build -s zmk/app -p -b nice_nano_v2 -d build/flake-left -- \
  -DSHIELD=anywhy_flake_left \
  -DZMK_CONFIG="$(pwd)/config" \
  -DCONFIG_ZMK_SPLIT=y \
  -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n

west build -s zmk/app -p -b nice_nano_v2 -d build/flake-right -- \
  -DSHIELD=anywhy_flake_right \
  -DZMK_CONFIG="$(pwd)/config" \
  -DCONFIG_ZMK_SPLIT=y \
  -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n

west build -s zmk/app -p -b seeeduino_xiao_ble -d build/flake-dongle -- \
  -DSHIELD="anywhy_flake_dongle dongle_screen" \
  -DZMK_CONFIG="$(pwd)/config"
```

### Phase 4: Flash and Test

**Step 4.1**: Reset all devices

Flash settings_reset firmware to clear stored bonds:
1. Flash `reset-flake.uf2` to left half
2. Flash `reset-flake.uf2` to right half  
3. Flash `reset-dongle.uf2` to dongle

**Important**: Only power ONE device at a time during reset!

**Step 4.2**: Flash new firmware

1. Flash `flake-left.uf2` to left half
2. Flash `flake-right.uf2` to right half
3. Flash `flake-dongle.uf2` to dongle

**Step 4.3**: Pair devices

1. Power on the dongle (connect via USB)
2. Power on the left half
3. Wait for pairing (LED should indicate connection)
4. Power on the right half
5. Wait for pairing

**Step 4.4**: Test all keys

Verify each key outputs the correct character according to your keymap.

---

## File Changes Reference

### Files to ADD to forked flake-zmk-module

| File | Purpose |
|------|---------|
| `boards/shields/anywhy_flake/anywhy_flake_dongle.overlay` | Dongle devicetree overlay |

### Files to MODIFY in forked flake-zmk-module

| File | Changes |
|------|---------|
| `boards/shields/anywhy_flake/Kconfig.shield` | Add `SHIELD_ANYWHY_FLAKE_DONGLE` config |
| `boards/shields/anywhy_flake/Kconfig.defconfig` | Add dongle config, remove central role from left |

### Files to MODIFY in zmk-config

| File | Changes |
|------|---------|
| `config/west.yml` | Point to your forked module |
| `build.yaml` | Update build matrix for dongle |

### Files to DELETE from zmk-config

| File | Reason |
|------|--------|
| `config/boards/shields/flake_screen_dongle/*` | Replaced by module-integrated dongle |
| `config/anywhy_flake_left.overlay` | No longer needed |
| `config/anywhy_flake_right.overlay` | No longer needed |
| `config/anywhy_flake_left.conf` | No longer needed |
| `config/anywhy_flake_right.conf` | No longer needed |

---

## Testing Checklist

### Build Verification

- [ ] Left half builds successfully
- [ ] Right half builds successfully
- [ ] Dongle builds successfully
- [ ] Dongle firmware size is ~800KB+ (indicates full features compiled)

### Devicetree Verification

After building, check that the correct layout is selected:

```bash
grep "zmk,physical-layout" build/flake-left/zephyr/zephyr.dts
# Should show: zmk,physical-layout = &small_layout;

grep "zmk,physical-layout" build/flake-dongle/zephyr/zephyr.dts
# Should show: zmk,physical-layout = &small_layout;
```

### Functional Testing

- [ ] Dongle powers on and shows display
- [ ] Left half pairs with dongle
- [ ] Right half pairs with dongle
- [ ] Left half top row outputs: Q W E R T
- [ ] Left half home row outputs: A S D F G
- [ ] Left half bottom row outputs: Z X C V B
- [ ] Left half thumb row outputs: GUI ALT CTRL SPACE NAV
- [ ] Right half top row outputs: Y U I O P
- [ ] Right half home row outputs: H J K L ;
- [ ] Right half bottom row outputs: N M , . /
- [ ] Right half thumb row outputs: SYM SHIFT NUM RALT HYPER
- [ ] Layer switching works correctly
- [ ] Combos work correctly

---

## Troubleshooting

### Keys still wrong after implementing fix

1. Verify west pulled the correct module version:
   ```bash
   cd flake-zmk-module
   git log -1  # Should show your dongle commit
   ```

2. Clean and rebuild everything:
   ```bash
   rm -rf build/
   west update
   # Rebuild all
   ```

3. Make sure all devices were reset before flashing new firmware

### Dongle not pairing

1. Ensure both halves are configured as peripherals (not central)
2. Check `ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2` in dongle config
3. Reset all devices and re-pair

### Only one half works

1. Check that both halves have `ZMK_SPLIT_ROLE_CENTRAL=n`
2. Verify the half that doesn't work was actually flashed with new firmware
3. Try resetting and re-pairing

---

## Alternative Approaches (If Above Fails)

### Option A: Use ZMK Studio to Select Layout

If the module supports ZMK Studio, you might be able to:
1. Build without layout overrides
2. Connect via ZMK Studio
3. Select "Flake S Layout" in the Studio interface

This would set the layout at runtime rather than compile time.

### Option B: Modify the Module's Default Layout

Instead of keeping `large_layout` as default, change the module's `anywhy_flake.dtsi`:

```c
chosen {
    zmk,physical-layout = &small_layout;  // Change default to small
};
```

This is more invasive but ensures all devices use `small_layout` by default.

### Option C: Create a Completely Separate Module

Create a new module specifically for Flake S + Dongle that:
1. Only includes `small_layout` (removes L and M)
2. Includes the dongle shield
3. Is purpose-built for this configuration

This is the most work but gives you full control.

---

## Appendix: Understanding ZMK Split Architecture

### How Key Events Flow

```
Physical Key Press
       │
       ▼
┌─────────────────┐
│   Peripheral    │
│    (L or R)     │
│                 │
│  kscan: (r,c)   │──── Raw row/column from GPIO matrix
│       │         │
│       ▼         │
│  transform:     │──── Converts (r,c) to position number
│  RC(r,c) → pos  │
│       │         │
│       ▼         │
│  Send position  │
│  to central     │
└────────┬────────┘
         │ BLE
         ▼
┌─────────────────┐
│    Central      │
│    (Dongle)     │
│                 │
│  Receive pos    │
│       │         │
│       ▼         │
│  keymap[pos]    │──── Look up behavior for this position
│       │         │
│       ▼         │
│  Execute        │
│  behavior       │
│       │         │
│       ▼         │
│  Send HID       │
│  to host        │
└─────────────────┘
```

### Why Transform Sharing Matters

The position number is just an index (0, 1, 2, ... 39 for a 40-key keyboard).

The transform converts (row, column) → position:
```
small_transform map:
RC(1,1) RC(1,2) RC(1,3) RC(1,4) RC(1,5)  RC(1,6) RC(1,7) RC(1,8) RC(1,9) RC(1,10)
  ↓       ↓       ↓       ↓       ↓        ↓       ↓       ↓       ↓       ↓
  0       1       2       3       4        5       6       7       8       9
```

If the peripheral and central have different transforms (even with same RC values), the position numbers might be calculated differently, causing the mismatch we observed.

By sharing the **exact same devicetree node** via `#include`, we guarantee consistency.
