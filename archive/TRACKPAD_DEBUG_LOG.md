# Trackpad Integration Debug Log

## Hardware Setup
- **Keyboard**: Sofle split keyboard
- **MCU**: nice!nano v2 (nRF52840)
- **Trackpad**: Azoteq TPS43 (IQS5xx family)
- **I2C Address**: 0x74
- **Breakout Board**: Handles power regulation and pull-up resistors internally
- **Split Config**: Right side is central (CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y)

## Wiring
- VCC → nice!nano 3.3V
- GND → nice!nano GND
- SDA → nice!nano D2 (P0.17)
- SCL → nice!nano D3 (P0.20)

## Driver
- **Module**: AYM1607/zmk-driver-azoteq-iqs5xx
- **Added to**: config/west.yml
- Successfully fetches during build

## Issues Encountered & Solutions

### Issue 1: Driver Never Initialized
**Symptom**: USB logs showed only BLE scanning, no I2C messages
**Root Cause**: Device tree overlay not being loaded by build system
**Build Error**: `DT_HAS_AZOTEQ_IQS5XX_ENABLED (=n)`

**Attempted Fixes**:
1. ❌ Changed GPIO references from `&gpio0` to `&pro_micro` - didn't help
2. ❌ Added `CONFIG_I2C=y` explicitly - didn't help
3. ❌ Placed overlay at `config/boards/shields/sofle_right.overlay` - **NOT LOADED**
4. ✅ **SOLUTION**: Moved to `config/sofle_right.overlay` (shield overlays go in config root)

### Issue 2: Invalid OLED Reference in Overlay
**Symptom**: Build failed with `parse error: undefined node label 'oled'`
**Root Cause**: Right side overlay tried to disable `&oled` which doesn't exist (OLED only on left)
**Build Error**: `devicetree error: /tmp/zmk-config/config/sofle_right.overlay:21 (column 1): parse error: undefined node label 'oled'`

**Attempted Fixes**:
1. ✅ **SOLUTION**: Removed `&oled { status = "disabled"; }` block (commit 76ab46d)
   - Setting `zephyr,display = &none;` in chosen node is sufficient
   - Can't reference device tree labels that don't exist

### Issue 3: Driver Requires RDY Pin But Hardware Doesn't Expose It
**Symptom**: No I2C or trackpad messages in USB logs after flashing (commit 76ab46d)
**Root Cause**: Driver requires `rdy-gpios` for initialization but TPS43 breakout only has 4 pins (VCC/GND/SDA/SCL)
**Hardware Limitation**: Breakout board doesn't expose RDY or NRST pins from TPS43's 6-pin FPC connector

**Driver Analysis** (from zmk-driver-azoteq-iqs5xx source):
- `rdy-gpios`: **REQUIRED** - Driver checks `gpio_is_ready_dt(&config->rdy_gpio)` and fails with `-ENODEV` if missing
- `reset-gpios`: **OPTIONAL** - Driver checks `if (config->reset_gpio.port)` and gracefully skips if not present
- RDY pin provides interrupt mechanism for trackpad events - driver cannot function without it

**Current Status**: ❌ **BLOCKED** - Cannot proceed without hardware modification or driver fork

**Potential Solutions**:
1. Check if TPS43 module has RDY/NRST pads that can be soldered to
2. Fork driver and modify to make RDY optional (polling mode instead of interrupts)
3. Replace hardware with Cirque Pinnacle trackpad (better ZMK support)

### Issue 4: Input Driver Pattern (NOT YET TESTED)
**Research Finding**: Documentation showed `zmk,input-listener` pattern
**Current Config**: Using `zmk,input-listener` (may need `zmk,input-split` for split keyboards)
**Status**: Cannot test until Issue #3 is resolved

## Current Configuration Files

### config/west.yml
```yaml
projects:
  - name: zmk-driver-azoteq-iqs5xx
    remote: aym1607
    revision: main
```

### config/sofle.conf
```conf
CONFIG_I2C=y
CONFIG_INPUT=y
CONFIG_INPUT_AZOTEQ_IQS5XX=y
CONFIG_ZMK_POINTING=y
CONFIG_ZMK_MOUSE=y
CONFIG_I2C_LOG_LEVEL_DBG=y
CONFIG_INPUT_LOG_LEVEL_DBG=y
CONFIG_LOG_PROCESS_THREAD_STARTUP_DELAY_MS=8000
```

### config/sofle_right.conf
```conf
CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y
```

### config/sofle_right.overlay
```dts
#include <dt-bindings/i2c/i2c.h>

/ {
    chosen {
        zephyr,display = &none;
    };

    trackpad_listener: trackpad_listener {
        compatible = "zmk,input-listener";
        device = <&trackpad>;
    };
};

&pro_micro_i2c {
    status = "okay";
    clock-frequency = <I2C_BITRATE_FAST>;

    trackpad: iqs5xx@74 {
        compatible = "azoteq,iqs5xx";
        reg = <0x74>;
        status = "okay";
        label = "TRACKPAD";

        reset-gpios = <&pro_micro 18 GPIO_ACTIVE_LOW>;
        rdy-gpios = <&pro_micro 19 GPIO_ACTIVE_HIGH>;
    };
};
```

## Next Steps (Latest Build - commit 76ab46d)

1. **Check build logs** for:
   - `-- Found devicetree overlay: /tmp/zmk-config/config/sofle_right.overlay`
   - `DT_HAS_AZOTEQ_IQS5XX_ENABLED (=y)` instead of `(=n)`

2. **If overlay loads successfully**:
   - Flash `sofle_right_debug.uf2`
   - Connect to USB serial
   - Look for I2C initialization messages

3. **Expected outcomes**:
   - ✅ Success: See I2C init + trackpad device ID
   - ⚠️  Hardware issue: See I2C init + NACK errors (wrong address/no power)
   - ❌ Still broken: No I2C messages (overlay still not loading)

## Debug Commands

### USB Serial Connection (macOS)
```bash
ls /dev/tty.usbmodem*
screen /dev/tty.usbmodem14201 115200
# Exit: Ctrl+A, K, Y
```

### Check Build Logs
Look in GitHub Actions → Build (sofle_right) step for:
- Overlay loading confirmation
- Kconfig warnings about DT_HAS_AZOTEQ_IQS5XX_ENABLED
- Device tree compilation errors

## Key Learnings

1. **Shield overlays go in `config/` root**, not `config/boards/shields/`
2. **Board overlays go in `config/boards/`**
3. **Build warnings are critical** - `DT_HAS_*_ENABLED` flags show if device tree nodes are recognized
4. **USB logging requires `zmk-usb-logging` snippet** in build.yaml
5. **GPIO pin references** use `&pro_micro NN` not `&gpio0 NN` in ZMK
6. **Can't reference non-existent device tree labels** - only reference labels that exist in the device tree hierarchy

## Alternative Approaches if Current Fails

1. Switch to `zmk,input-split` instead of `zmk,input-listener`
2. Try alternate I2C address (0x67 instead of 0x74)
3. Test different GPIO pins for reset/rdy
4. Consider Cirque Pinnacle trackpad (better ZMK support)
5. Test hardware with QMK to isolate firmware vs hardware issues
