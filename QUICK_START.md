# YADS Dongle Quick Start Guide

## ⚡ Fast Track

### 1. Download Firmware
Go to: https://github.com/tku137/zmk-config/actions

1. Click on the latest workflow run for `yads-integrated-module` branch
2. Download the firmware artifacts (zip file)
3. Extract the zip - you should have:
   - `flake-left.uf2`
   - `flake-right.uf2`
   - `flake-dongle.uf2`
   - `reset-flake.uf2`
   - `reset-dongle.uf2`

### 2. Reset Everything (IMPORTANT!)

**Power only ONE device at a time!**

1. **Left half**: Press reset twice quickly → appears as USB drive → drag `reset-flake.uf2` → wait for reboot
2. **Right half**: Press reset twice quickly → appears as USB drive → drag `reset-flake.uf2` → wait for reboot
3. **Dongle**: Press reset twice quickly → appears as USB drive → drag `reset-dongle.uf2` → wait for reboot

### 3. Flash New Firmware

1. **Left half**: Reset twice → drag `flake-left.uf2`
2. **Right half**: Reset twice → drag `flake-right.uf2`
3. **Dongle**: Reset twice → drag `flake-dongle.uf2`

### 4. Pair (Order Matters!)

1. Connect dongle to computer via USB
2. Turn on left half (or press reset)
3. Wait ~10 seconds for LED to indicate pairing
4. Turn on right half (or press reset)
5. Wait ~10 seconds for LED to indicate pairing

### 5. Test

Open a text editor and type on each key to verify correct output.

**Expected layout:**
```
Left:               Right:
Q W E R T           Y U I O P
A S D F G           H J K L ;
Z X C V B           N M , . /
(thumb keys)        (thumb keys)
```

If you get `F G H J K` on the top row instead of `Q W E R T`, something went wrong.

## 🎛️ YADS Display Controls

| Key | Function |
|-----|----------|
| F22 | Toggle display on/off |
| F23 | Decrease brightness |
| F24 | Increase brightness |

You'll need to add these to your keymap if you want to use them.

## 🐛 Quick Troubleshooting

### Wrong keys output?
→ Reset all devices again, make sure to power one at a time

### Dongle doesn't connect?
→ Check USB cable, try different USB port

### Only one half works?
→ Check batteries, make sure you flashed the correct firmware to each half

### Display is blank?
→ Check that dongle firmware was flashed successfully, try adjusting brightness with F23/F24

## 📝 Configuration

### Change Display Brightness
Edit `config/anywhy_flake_dongle.conf`:
```conf
CONFIG_DONGLE_SCREEN_DEFAULT_BRIGHTNESS=50  # Change this (10-80)
```

### Change Idle Timeout
Edit `config/anywhy_flake_dongle.conf`:
```conf
CONFIG_DONGLE_SCREEN_IDLE_TIMEOUT_S=600  # Change this (seconds)
```

### Enable Ambient Light Sensor
If you have an APDS9960 sensor installed:
```conf
CONFIG_DONGLE_SCREEN_AMBIENT_LIGHT=y
```

### System Icon (macOS/Linux/Windows)
```conf
CONFIG_DONGLE_SCREEN_SYSTEM_ICON=0  # 0=macOS, 1=Linux, 2=Windows
```

After changing config, commit and push to GitHub to trigger a new build.

## 🔄 Update Workflow

When you want to update your keymap or configuration:

1. Edit files in `config/` directory
2. Commit and push to `yads-integrated-module` branch
3. Wait for GitHub Actions to build
4. Download new firmware
5. Flash to devices (no need to reset if you're not changing pairing)

## 📚 More Info

See `YADS_SETUP_COMPLETE.md` for detailed explanation of what was changed and why.

---

**Repositories:**
- Your fork: https://github.com/tku137/flake-zmk-module/tree/yads-dongle-support
- Your config: https://github.com/tku137/zmk-config/tree/yads-integrated-module
- YADS module: https://github.com/janpfischer/zmk-dongle-screen
