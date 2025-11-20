# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains the ZMK firmware configuration for the Kinesis Advantage 360 Pro split ergonomic keyboard. It uses a custom fork of ZMK maintained by Kinesis at https://github.com/ReFil/zmk (branch: adv360-z3.5-2) with keyboard-specific features like RGB indicator LEDs and layer colors.

## Architecture

### Repository Structure

- `config/` - Main firmware configuration directory
  - `adv360.keymap` - Primary keymap file defining layers, macros, and behaviors
  - `adv360_left.keymap` / `adv360_right.keymap` - Side-specific keymap includes (both include adv360.keymap)
  - `west.yml` - West manifest defining the ZMK fork and version
  - `version.dtsi` - Auto-generated version info (modified during builds, should be reverted after)
  - `boards/arm/adv360/` - Board definition files
    - `adv360_left_defconfig` / `adv360_right_defconfig` - Kconfig settings for each side
    - `adv360.dtsi` - Device tree hardware definition
    - `adv360-layouts.dtsi` - Physical layout definitions
    - `adv360_pinctrl.dtsi` - Pin control configuration

### Keymap Architecture

The keymap uses ZMK devicetree syntax:
- Layers are defined in the `keymap` node with `bindings` arrays
- Custom behaviors defined in the `behaviors` node (e.g., `mac_tog`, `smrtsft`, tap-dances)
- Macros defined in the `macros` node (e.g., `vim_save`, `tmx_cmd`, `jbhide`)
- Uses 14-column x 5-row matrix per side (see assets/key-positions.md for exact positions)

### Two Build Variants

GitHub Actions builds two variants:
1. **Legacy (firmware-no-clique)** - Standard ZMK build
2. **Clique (firmware-clique)** - Includes ZMK Studio support (`-DCONFIG_ZMK_STUDIO=y` and `-S studio-rpc-usb-uart`)

Both variants build left and right firmware files separately.

## Building Firmware

### Local Container Build (Preferred for Development)

Requirements: Docker or Podman, Make

```bash
# Build both sides (creates Clique variant by default)
make

# Build left side only
make left

# Clean compiled firmware only
make clean_firmware

# Clean Docker container only
make clean_image

# Clean everything
make clean
```

Firmware output: `firmware/{TIMESTAMP}-{COMMIT}-left.uf2` and `firmware/{TIMESTAMP}-{COMMIT}-right.uf2`

The Makefile automatically:
- Generates version info via `bin/get_version_local.sh`
- Builds a Docker container from the Dockerfile
- Runs the build in the container
- Reverts `config/version.dtsi` after build

### GitHub Actions Build

Triggered on push, PR, or manual workflow dispatch. Builds both legacy and Clique variants. Artifacts are timestamped with format: `{YYYYMMDDHHMM}-{7-char-commit-hash}-{left|right}.uf2`

### West Build (Advanced)

For direct west builds (requires ZMK development environment):

```bash
west init -l config
west update
west zephyr-export

# Build left side
west build -s zmk/app -d build/left -b adv360_left -- -DZMK_CONFIG="${PWD}/config"

# Build right side
west build -s zmk/app -d build/right -b adv360_right -- -DZMK_CONFIG="${PWD}/config"
```

## Flashing Firmware

1. Connect keyboard via USB
2. Enter bootloader mode: Press `Mod+macro1` (left) or `Mod+macro3` (right)
   - Physical reset buttons also available (see User Manual section 2.7)
3. Keyboard appears as USB drive
4. Copy corresponding `.uf2` file to the drive
5. Drive automatically ejects when flashing completes

Flash both sides for keymap changes. Left side must be flashed for configuration changes in `adv360_left_defconfig`.

## Configuration Options

### Key Defconfig Settings (adv360_left_defconfig)

- `CONFIG_ZMK_HID_KEYBOARD_EXTENDED_REPORT` (line 65) - Enable F13-F24 and INTL keys with NKRO
- `CONFIG_BT_BAS` (line 58) - Enable BLE battery reporting (disabled by default to prevent spurious wake)
- `CONFIG_ZMK_RGB_UNDERGLOW_MOD_COLOR` - Modifier indicator LED color (hex RGB, e.g., 0xFF0000 for red)

Changes require flashing new firmware to both sides.

## Layer System

ZMK supports 32 layers (0-31). Layer LEDs display the active layer:
- Layers 0-7: Same color on both modules
- Layers 8+: Right module cycles through colors first, then left module changes
- Layer 0 uses black/off LEDs

Default layers in keymap:
- Layer 0: Base layer (Dvorak by default)
- Layer 1: Mac modifier swap layer
- Layer 2: Keypad
- Layer 3: Function keys
- Layer 4: Mod layer (Bluetooth, bootloader, versioning)

## Version Tracking

Firmware includes auto-versioning:
- Press `Mod+V` to type version string: `YYYYMMDD-XXXX-YYYYYY`
  - Date: Compilation date
  - XXXX: First 4 chars of Git branch
  - YYYYYY: 7-char Git commit hash
- Versioning macros defined in `config/version.dtsi` (auto-generated)
- Scripts: `bin/get_version.sh` (CI), `bin/get_version_local.sh` (local builds)

## Working with the Kinesis ZMK Fork

The repository points to a custom ZMK fork via `config/west.yml`:
- Remote: https://github.com/ReFil/zmk
- Branch: `adv360-z3.5-2`

This fork includes:
- RGB indicator LED support for CAPS/NUM/SCROLL lock
- Layer status LEDs with 32-layer color system
- Pointing device support
- ZMK Studio/Clique integration

Base ZMK compatibility: The keyboard is compatible with upstream ZMK, but custom features (indicator LEDs, layer colors) won't work.

## Key ZMK Behaviors Used

- `&tog` - Toggle layer
- `&mo` - Momentary layer
- `&sl` - Sticky layer (one-shot)
- `&sk` - Sticky key (one-shot modifier)
- `&mac_tog` - Custom hold-tap behavior for Mac modifier toggle
- `&smrtsft` / `&smrtsftr` - Smart shift (single tap = sticky shift, double tap = caps word)
- `&bt BT_SEL` - Bluetooth profile selection
- `&bt BT_CLR` - Clear Bluetooth bonds
- `&bootloader` - Enter bootloader mode
- `&studio_unlock` - Unlock ZMK Studio
- `&stp STP_BAT` - Show battery level via Smart Tap Protocol
- `&macro_ver` - Version macro (Mod+V)

## Documentation References

- ZMK Documentation: https://zmk.dev/docs (note: RGB Underglow, Backlight, Power Management sections don't apply to this custom fork)
- Kinesis Support: https://kinesis-ergo.com/support/kb360pro/
- Key Positions Reference: assets/key-positions.md
- Firmware Updates: https://kinesis-ergo.com/support/kb360pro/#firmware-updates
- Quick Start Guide: https://kinesis-ergo.com/wp-content/uploads/Advantage360-Professional-QSG-v8-25-22.pdf
- User Manual: https://kinesis-ergo.com/wp-content/uploads/Advantage360-ZMK-KB360-PRO-Users-Manual-v3-10-23.pdf
