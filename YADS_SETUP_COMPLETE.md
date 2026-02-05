# YADS Dongle Setup for Flake S - Implementation Complete ✅

## What We Did

### Phase 1: Fork and Modify flake-zmk-module

**Repository**: `tku137/flake-zmk-module`  
**Branch**: `yads-dongle-support`  
**Base commit**: `796f7a6` (last v1 commit, nice!nano v2 support)

#### Changes Made:

1. **Created `anywhy_flake_dongle.overlay`**
   - Includes the shared `anywhy_flake.dtsi`
   - References `&small_layout` directly (same devicetree node as peripherals)
   - Uses mock kscan (dongle has no physical keys)

2. **Updated `Kconfig.shield`**
   - Added `SHIELD_ANYWHY_FLAKE_DONGLE` configuration

3. **Updated `Kconfig.defconfig`**
   - Dongle configured as central with 2 peripherals
   - Removed central role from left half
   - Left/right are now peripherals only

### Phase 2: Update zmk-config

**Repository**: `tku137/zmk-config`  
**Branch**: `yads-integrated-module`

#### Changes Made:

1. **Updated `config/west.yml`**
   - Points to `tku137/flake-zmk-module@yads-dongle-support`
   - Added `janpfischer/zmk-dongle-screen` module
   - Pinned ZMK to `v0.3.0` for stability

2. **Updated `build.yaml`**
   - Left/right halves: `nice_nano_v2` with peripheral configuration
   - Dongle: `seeeduino_xiao_ble` with `anywhy_flake_dongle dongle_screen` shields
   - Added settings_reset for all boards

3. **Created `config/anywhy_flake_dongle.conf`**
   - YADS screen configuration
   - Brightness control via F22/F23/F24
   - All widgets enabled
   - Idle timeout: 10 minutes

4. **Kept existing `config/anywhy_flake.keymap`**
   - Already has `zmk,physical-layout = &small_layout`
   - Your custom keymap will work as-is

## Key Technical Solution

The critical fix was **integrating the dongle into the same shield folder** as the keyboard halves, allowing it to:

```c
#include "anywhy_flake.dtsi"

/ {
    chosen {
        zmk,physical-layout = &small_layout;  // SAME node as peripherals!
    };
};
```

This ensures all three devices (left, right, dongle) reference the **exact same** `small_layout` and `small_transform` devicetree nodes, preventing position mapping mismatches.

## Next Steps

### 1. Trigger GitHub Actions Build

Push to the `yads-integrated-module` branch will trigger GitHub Actions to build:
- `flake-left.uf2`
- `flake-right.uf2`
- `flake-dongle.uf2`
- `reset-dongle.uf2`
- `reset-flake.uf2`

### 2. Reset All Devices

**IMPORTANT**: Only power ONE device at a time during reset!

1. Flash `reset-flake.uf2` to left half, wait for reboot
2. Flash `reset-flake.uf2` to right half, wait for reboot
3. Flash `reset-dongle.uf2` to dongle, wait for reboot

### 3. Flash New Firmware

1. Flash `flake-left.uf2` to left half
2. Flash `flake-right.uf2` to right half
3. Flash `flake-dongle.uf2` to dongle

### 4. Pair Devices

**Order matters** for battery widget display!

1. Power on the dongle (connect via USB to computer)
2. Power on the left half
3. Wait for connection (LED indicators)
4. Power on the right half
5. Wait for connection

### 5. Test All Keys

Open a text editor and test each key position:

**Left half top row**: Q W E R T  
**Left half home row**: A S D F G  
**Left half bottom row**: Z X C V B  
**Left half thumb row**: (your thumb keys)  

**Right half top row**: Y U I O P  
**Right half home row**: H J K L ;  
**Right half bottom row**: N M , . /  
**Right half thumb row**: (your thumb keys)  

If keys output correctly → Success! 🎉

### 6. Test YADS Features

- Display shows current layer, modifiers, WPM, battery, output status
- Press F22 to toggle display on/off
- Press F23/F24 to adjust brightness
- Display dims after 10 minutes of inactivity

## Troubleshooting

### Keys still wrong

1. Verify GitHub Actions built successfully
2. Check that you flashed the correct firmware to each device
3. Ensure all devices were reset before flashing
4. Try re-pairing (reset and start over)

### Dongle doesn't pair

1. Check that `ZMK_SPLIT_ROLE_CENTRAL=n` is set for peripherals
2. Verify dongle is configured as central
3. Reset all devices and try again

### Display not working

1. Check dongle firmware size (should be ~800KB+)
2. Verify `dongle_screen` shield is included in build
3. Check YADS module was pulled correctly: `cd zmk-dongle-screen && git log -1`

## Build Status

Check GitHub Actions: https://github.com/tku137/zmk-config/actions

## Files Changed

### flake-zmk-module
- `boards/shields/anywhy_flake/anywhy_flake_dongle.overlay` (NEW)
- `boards/shields/anywhy_flake/Kconfig.shield` (MODIFIED)
- `boards/shields/anywhy_flake/Kconfig.defconfig` (MODIFIED)

### zmk-config
- `config/west.yml` (MODIFIED)
- `build.yaml` (MODIFIED)
- `config/anywhy_flake_dongle.conf` (NEW)

## Resources

- YADS Module: https://github.com/janpfischer/zmk-dongle-screen
- ZMK Dongle Guide: https://zmk.dev/docs/development/hardware-integration/dongle
- Your Implementation Plan: `/Users/tku137/Projects/Keyboard/zmk-config/notes/DONGLE_IMPLEMENTATION_PLAN.md`

---

Good luck! 🚀
