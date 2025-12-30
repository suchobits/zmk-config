# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a ZMK firmware configuration repository for a Sofle split keyboard using nice!nano v2 microcontrollers. The repository contains user-specific configurations while ZMK firmware itself is managed as a Git submodule.

## Build System

### Building Firmware

Builds are automated via GitHub Actions. Push to trigger a build:

```bash
git add .
git commit -m "Description of changes"
git push
```

Firmware artifacts (`.uf2` files) are available in GitHub Actions after the build completes.

### Local Development (Advanced)

If you need to build locally, you'll need the Zephyr toolchain and West. See ZMK documentation at `.zmk/zmk/docs/docs/development/local-toolchain/setup/` for setup instructions.

```bash
# Initialize workspace (first time only)
west init -l config
west update

# Build for left side
west build -s zmk/app -b nice_nano_v2 -- -DSHIELD=sofle_left

# Build for right side
west build -s zmk/app -b nice_nano_v2 -- -DSHIELD=sofle_right
```

## Architecture

### Directory Structure

- **`/config`** - User configuration (this is what you modify)
  - `sofle.keymap` - Keymap shared by both halves
  - `sofle.conf` - Global feature configuration
  - `sofle_left.conf` / `sofle_right.conf` - Side-specific configs
  - `/boards/nice_nano_v2.overlay` - RGB underglow hardware config
  - `/boards/shields/sofle_right.overlay` - Trackpad configuration
  - `west.yml` - Dependency manifest (ZMK firmware + modules)

- **`/.zmk`** - Git submodules (ZMK firmware and dependencies)
  - `/zmk` - Main ZMK firmware repository
  - `/modules` - External modules (trackpad driver, nanopb)

- **`build.yaml`** - Build matrix for GitHub Actions

### Configuration System

ZMK uses two configuration systems that work together:

**Kconfig (`.conf` files)** - Compile-time feature flags:
- Controls what features are enabled (RGB, encoders, trackpad)
- Sets default values and behavior parameters
- Uses `CONFIG_*` variables

**Device Tree (`.overlay`, `.keymap` files)** - Hardware topology:
- Describes physical hardware (pins, I2C devices, matrix layout)
- Defines keymap bindings and layers
- Uses a hierarchical node system

Configuration priority (lowest to highest):
1. ZMK defaults
2. Shield defaults (in ZMK firmware)
3. `sofle.conf` (global)
4. `sofle_{side}.conf` (side-specific)
5. `.overlay` files (hardware overrides)

### Split Keyboard Architecture

- **Right side**: Central/primary (processes all input, sends HID to computer)
- **Left side**: Peripheral (scans matrix, forwards to right via BLE)
- Trackpad is on right side only to minimize latency (no BLE hop)
- OLED is disabled on right side (doesn't exist on hardware)

## Key Configuration Files

### sofle.keymap

Defines key bindings and layers using ZMK's keymap DSL:
- 4 layers: BASE (0), LOWER (1), RAISE (2), ADJUST (3)
- ADJUST layer activates automatically when LOWER+RAISE are both held
- Encoder bindings for volume/page control
- Shared by both keyboard halves

### sofle.conf

Global Kconfig settings for both sides:
- Encoder support: `CONFIG_EC11=y`
- RGB underglow with custom defaults (white, solid color)
- Trackpad support: `CONFIG_ZMK_POINTING=y`, `CONFIG_INPUT_AZOTEQ_IQS5XX=y`

### boards/nice_nano_v2.overlay

Configures RGB underglow hardware:
- SPI3 on pin P0.08 for WS2812/SK6812 LEDs
- 29 LEDs per side in chain
- Color mapping for SK6812 MINI E (RBG order)

### boards/shields/sofle_right.overlay

Configures Azoteq TPS43 trackpad on right side:
- I2C address 0x74 on `&pro_micro_i2c` bus
- Uses `zmk,input-split` driver for split keyboard integration
- Trackpad shares I2C bus with OLED (but OLED is disabled on right)
- Dummy GPIO pins for RDY/RESET (driver requires them but TPS43 module doesn't expose them)

### config/west.yml

Dependency manifest:
- Points to ZMK firmware v0.3 branch
- Includes `zmk-driver-azoteq-iqs5xx` module for trackpad support
- West uses this to fetch and manage submodules

## Common Modifications

### Changing Keymap

Edit [config/sofle.keymap](config/sofle.keymap). Key codes are defined in ZMK's `<dt-bindings/zmk/keys.h>`.

Example binding format:
```dts
&kp ESC      // Key press
&mo LOWER    // Momentary layer
&bt BT_CLR   // Bluetooth clear
```

### Enabling Trackpad Features

Edit [config/boards/shields/sofle_right.overlay](config/boards/shields/sofle_right.overlay) trackpad node:
```dts
trackpad: iqs5xx@74 {
    compatible = "azoteq,iqs5xx";
    reg = <0x74>;

    // Uncomment to enable:
    one-finger-tap;      // Tap to left-click
    two-finger-tap;      // Two-finger tap for right-click
    scroll;              // Two-finger scroll
};
```

### Adjusting RGB Settings

Edit [config/sofle.conf](config/sofle.conf):
```conf
CONFIG_ZMK_RGB_UNDERGLOW_HUE_START=180   # Hue (0-360)
CONFIG_ZMK_RGB_UNDERGLOW_SAT_START=100   # Saturation (0-100)
CONFIG_ZMK_RGB_UNDERGLOW_BRT_START=50    # Brightness (0-100)
CONFIG_ZMK_RGB_UNDERGLOW_EFF_START=2     # Effect (0=solid, 1=breathe, 2=spectrum, etc.)
```

### Adding New Layers

1. Add layer number define at top of [config/sofle.keymap](config/sofle.keymap):
   ```c
   #define NEWLAYER 4
   ```

2. Add layer definition in keymap block:
   ```dts
   newlayer_layer {
       display-name = "newlayer";
       bindings = <...>;
   };
   ```

3. Add layer activation in other layers (e.g., `&mo NEWLAYER`)

## Hardware-Specific Notes

### nice!nano v2
- nRF52840 microcontroller (ARM Cortex-M4)
- Built-in battery management for LiPo batteries
- USB-C connector for charging and flashing

### Split Communication
- Uses BLE for wireless split communication
- Central (right) side connects to computer via USB or BLE
- Pairing: Both halves must be powered on, they auto-pair on first boot

### Flashing Procedure
1. Download `.uf2` files from GitHub Actions artifacts
2. Double-tap reset button on nice!nano to enter bootloader mode
3. Drag and drop `.uf2` file to mounted drive (`NICENANO`)
4. Device automatically resets after flashing
5. Flash both left and right sides when updating firmware

### I2C Bus Architecture
- Right side I2C bus: Trackpad at 0x74 (OLED would be 0x3C but is disabled)
- Left side I2C bus: Could have OLED at 0x3C (not currently configured)
- I2C pins: SDA on P0.17 (D2), SCL on P0.20 (D3)

## Module System

External drivers/features are added via West modules in [config/west.yml](config/west.yml):

```yaml
projects:
  - name: zmk-driver-azoteq-iqs5xx
    remote: aym1607
    revision: main
```

Modules provide:
- Device drivers (`drivers/` directory)
- Device tree bindings (`dts/bindings/` directory)
- Kconfig options (`Kconfig` files)

## Debugging with USB Logging

### Enabling USB Logging

USB logging is enabled for the right side via the `zmk-usb-logging` snippet in [build.yaml](build.yaml). This allows you to view real-time logs including I2C errors, device initialization, and trackpad events.

**No JTAG required!** The nice!nano exposes a USB CDC ACM serial port when connected via USB-C.

### Viewing Logs

After flashing the debug firmware (`sofle_right_debug.uf2`), connect the right side to your computer via USB:

**macOS:**
```bash
# Find the device
ls /dev/tty.usbmodem*

# Connect with screen (built-in)
screen /dev/tty.usbmodem14201 115200

# Or use tio (install with: brew install tio)
tio /dev/tty.usbmodem14201
```

**Linux:**
```bash
# Find the device
ls /dev/ttyACM*

# Connect with tio
sudo tio /dev/ttyACM0

# Or use screen
sudo screen /dev/ttyACM0 115200
```

**Windows:**
```powershell
# Find COM port in Device Manager under "Ports (COM & LPT)"

# Use PuTTY or any serial terminal:
# - Connection type: Serial
# - Speed: 115200
# - COM port: (from Device Manager)
```

**Exit serial terminal:**
- `screen`: Press `Ctrl+A` then `K` then `Y`
- `tio`: Press `Ctrl+T` then `Q`

### What to Look For

After connecting, you should see:
```
*** Booting Zephyr OS build ... ***
[00:00:08.000,000] <inf> zmk: Welcome to ZMK!
[00:00:08.001,000] <dbg> i2c_nrfx_twim: i2c@40003000: Transfer 0x...
```

**Common I2C errors:**
- `I2C bus not ready` → I2C not enabled or wrong pins
- `scl timeout` → Missing pull-up resistors or wiring issue
- `NACK` → Device not responding (wrong address or not powered)
- `Data queue timed out` → Device not powered or reset stuck

### Debug Configuration

Debug logging is enabled in [config/sofle.conf](config/sofle.conf):
```conf
CONFIG_I2C_LOG_LEVEL_DBG=y
CONFIG_LOG_PROCESS_THREAD_STARTUP_DELAY_MS=8000
```

The 8-second delay ensures logs don't get lost during USB enumeration.

## Troubleshooting

### Build Failures
- Check GitHub Actions logs for specific errors
- Verify Kconfig syntax (no spaces around `=`, must be `y` or `n`)
- Verify Device Tree syntax (semicolons, proper node hierarchy)

### Trackpad Not Working
1. **Check USB logs first** (see Debugging section above)
2. Verify I2C address matches your hardware (default 0x74 for TPS43)
3. Check that right side is flashed with right firmware (not left)
4. **Flash settings_reset.uf2** to both halves after changing central role
5. Verify external pull-up resistors (2.2-4.7kΩ) on SDA/SCL lines
6. Try different GPIO pins for reset/rdy if current ones conflict

### RGB Not Working
- Verify LED chain length matches your hardware
- Check color-mapping matches LED type (SK6812 MINI E uses RBG)
- Ensure `CONFIG_ZMK_RGB_UNDERGLOW=y` in [config/sofle.conf](config/sofle.conf)

### Split Connection Issues
- Ensure right side has `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y`
- **Flash settings_reset.uf2** to both halves to clear pairing data
- Clear Bluetooth bonds: Hold ADJUST layer binding `&bt BT_CLR`
- Power cycle both halves

## Important Patterns

### Device Tree Node References
Nodes are referenced with `&node_name`:
```dts
&pro_micro_i2c {      // Reference existing node
    trackpad: ... {   // Add child node
    }
}

&oled {               // Reference to disable
    status = "disabled";
}
```

### Conditional Layers
Automatically activate a layer when multiple layers are held:
```dts
conditional_layers {
    compatible = "zmk,conditional-layers";
    adjust_layer {
        if-layers = <LOWER RAISE>;
        then-layer = <ADJUST>;
    };
};
```

### Split Input Devices
Pointing devices in split keyboards use `zmk,input-split`:
```dts
trackpad_split: trackpad_split {
    compatible = "zmk,input-split";
    device = <&trackpad>;
};
```

This routes input events through the split transport layer.
