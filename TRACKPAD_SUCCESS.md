# Trackpad Implementation - SUCCESS ✅

## Summary

Successfully implemented polling mode support for Azoteq TPS43 (IQS5XX) trackpad on Sofle keyboard with nice!nano v2.

**Date**: 2025-12-31 to 2026-01-01
**Status**: ✅ WORKING
**Driver**: https://github.com/suchobits/zmk-driver-azoteq-iqs5xx (branch: polling-mode-support)

---

## Final Configuration

### Hardware
- **Board**: nice!nano v2 (nRF52840)
- **Keyboard**: Sofle split keyboard
- **Trackpad**: Azoteq TPS43 (IQS5XX chipset)
- **Connection**: I2C (SDA: P0.17, SCL: P0.20)
- **Address**: 0x74
- **Pins Available**: Only 4 (VCC, GND, SDA, SCL) - no RDY or RESET pins

### Software
- **Mode**: Polling (k_timer based, 50ms interval / 20Hz)
- **Driver Commit**: de8987d
- **Config Commit**: d374ca8

---

## Key Findings

### Root Cause of System Freeze
**Problem**: System would freeze after ONE mouse movement when using debug build with USB logging.

**Cause**: USB HID + USB CDC conflict in Zephyr USB stack. When both USB HID (mouse) and USB CDC (logging) were active simultaneously, the kernel would crash after processing the first HID report.

**Solution**: Use regular build without `zmk-usb-logging` snippet for production.

**Evidence**:
- Driver was working perfectly (confirmed via Phase 1/1B debugging)
- Non-debug build works continuously without any freezing
- Debug build (with USB CDC) crashes after first `zmk_hid_mouse_movement_set`

### Implementation Decisions

1. **Polling Mode**: Implemented k_timer-based polling because TPS43 module only exposes 4 wires (no RDY pin)
2. **Power Management**: Set `IDLE_MODE_TIMEOUT=255` to disable LP1/LP2 sleep modes (based on QMK implementation)
3. **Device Configuration**: MANUAL_CONTROL + REATI + streaming mode (not event mode)
4. **Timeout Handling**: All `input_report_*()` calls use `K_NO_WAIT` to prevent work queue deadlock
5. **Axis Orientation**: Implemented `flip-x` and `flip-y` device tree properties for natural cursor movement

---

## Current Features

### Enabled
- ✅ Cursor movement (polling mode at 20Hz)
- ✅ Axis flipping for natural movement direction
- ✅ Continuous operation without freezing

### Available But Disabled
The driver supports these gestures (can be enabled in device tree overlay):
- Single finger tap → Left click
- Two finger tap → Right click
- Press and hold → Left click (configurable hold time)
- Two finger scroll → Vertical/horizontal scroll
- Natural scroll options for X and Y axes

---

## Configuration Files

### Device Tree Overlay
`config/sofle_right.overlay`:
```dts
&pro_micro_i2c {
    status = "okay";
    clock-frequency = <I2C_BITRATE_STANDARD>;  // 100kHz

    trackpad: iqs5xx@74 {
        compatible = "azoteq,iqs5xx";
        reg = <0x74>;
        status = "okay";

        // Polling mode - no GPIO pins needed
        // Flip axes for natural cursor movement
        flip-x;
        flip-y;

        // Optional gesture features (currently disabled):
        // one-finger-tap;      // Tap to left-click
        // two-finger-tap;      // Two-finger tap for right-click
        // scroll;              // Two-finger scroll
        // natural-scroll-x;    // Natural scroll direction X
        // natural-scroll-y;    // Natural scroll direction Y
    };
};
```

### Kconfig
`config/sofle.conf`:
```conf
# Trackpad support
CONFIG_ZMK_POINTING=y
CONFIG_INPUT_AZOTEQ_IQS5XX=y

# I2C debugging (optional)
CONFIG_I2C_LOG_LEVEL_DBG=y
```

### Build Configuration
`build.yaml`:
```yaml
include:
  - board: nice_nano_v2
    shield: sofle_left
  - board: nice_nano_v2
    shield: sofle_right
  - board: nice_nano_v2
    shield: sofle_right
    snippet: zmk-usb-logging
    artifact-name: sofle_right_debug
```

**Production**: Flash `sofle_right-nice_nano_v2-zmk.uf2`
**Debugging**: Flash `sofle_right_debug.uf2` (will freeze after first movement due to USB conflict)

---

## Driver Implementation Details

### Polling Mode
Instead of interrupt-driven (RDY pin), uses k_timer:
```c
static void iqs5xx_poll_timer_handler(struct k_timer *timer) {
    k_work_submit(&data->work);
}

// In init function:
k_timer_init(&data->poll_timer, iqs5xx_poll_timer_handler, NULL);
k_timer_start(&data->poll_timer, K_MSEC(50), K_MSEC(50));  // 20Hz
```

### QMK-Style Power Management
```c
// Disable idle timeout to prevent LP1/LP2 low power modes
iqs5xx_write_reg8(dev, IQS5XX_IDLE_MODE_TIMEOUT, 255);

// Enable MANUAL_CONTROL + REATI for continuous touch detection
uint8_t config0 = IQS5XX_SETUP_COMPLETE | IQS5XX_REATI | IQS5XX_MANUAL_CONTROL;
iqs5xx_write_reg8(dev, IQS5XX_SYSTEM_CONFIG_0, config0);

// Configure for streaming mode (not event mode)
uint8_t config1 = IQS5XX_TP_EVENT | IQS5XX_GESTURE_EVENT | IQS5XX_REATI_EVENT;
iqs5xx_write_reg8(dev, IQS5XX_SYSTEM_CONFIG_1, config1);
```

### Axis Flipping
```c
if (config->flip_x) {
    rel_x *= -1;
}
if (config->flip_y) {
    rel_y *= -1;
}
```

---

## Build Process

Builds are automated via GitHub Actions:
```bash
git add .
git commit -m "Description"
git push
```

Firmware artifacts available at: https://github.com/suchobits/zmk-config/actions

**Flash**: Download `.uf2`, double-tap reset, drag to `NICENANO` drive.

---

## Debugging Reference

For future debugging needs, archived files in `archive/`:
- `CURRENT_SESSION_DEBUG.md` - Complete debugging session log with all attempts
- `NEXT_DEBUG_STEPS.md` - Phase-based debugging plan
- `TRACKPAD_DEBUG_LOG.md` - Original debug log

### Key Debugging Tools Used
- USB CDC serial logging (`zmk-usb-logging` snippet)
- printk() for driver diagnostics
- Phase-based systematic debugging (Phase 1, 1B)
- GitHub Actions for reproducible builds

---

## Next Steps (Optional Enhancements)

1. **Enable Gestures**: Uncomment gesture features in overlay and test
2. **Tune Polling Rate**: Adjust from 50ms (20Hz) if needed for responsiveness vs power
3. **Clean Up Debug Code**: Remove printk statements from driver for production
4. **Add Scroll Support**: Enable two-finger scroll with natural scroll options
5. **Configure Click Gestures**: Single tap for left-click, two-finger for right-click

---

## References

- **ZMK Firmware**: https://zmk.dev/
- **Driver Fork**: https://github.com/suchobits/zmk-driver-azoteq-iqs5xx
- **Original Driver**: https://github.com/aym1607/zmk-driver-azoteq-iqs5xx
- **QMK Reference**: https://github.com/qmk/qmk_firmware/blob/master/drivers/sensors/azoteq_iqs5xx.c
- **TPS43 Product**: Azoteq TPS43 trackpad module (EOL, uses IQS5XX chipset)
