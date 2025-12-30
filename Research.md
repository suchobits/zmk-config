# ZMK drivers for TPS43 Azoteq trackpads on split keyboards

**No official ZMK driver exists for IQS5xx trackpads**, but the community module **AYM1607/zmk-driver-azoteq-iqs5xx** provides full support for TPS43 including tap gestures, scrolling, and multi-touch. For your Sofle with nice!nano v2, the critical configuration step is **making the right side the central** via `ZMK_SPLIT_ROLE_CENTRAL`. The TPS43 specifications—**2048×1792 resolution** and **0x74 I2C address**—are confirmed correct, though note this trackpad has reached end-of-life status.

## The AYM1607 driver is your primary option

The main ZMK driver for IQS5xx trackpads lives at **github.com/AYM1607/zmk-driver-azoteq-iqs5xx** (not "rscires" as previously referenced—extensive searching found no such repository). This MIT-licensed module has **32 stars** and active community usage despite minimal maintenance (only 2 commits since creation from the ZMK module template).

The driver supports all features you'd expect from a trackpad:
- Single-finger tap → left click
- Two-finger tap → right click
- Press and hold → click-and-drag (configurable timeout)
- Vertical and horizontal scrolling with natural scroll options
- Axis orientation via `switch-xy` property

Add the module to your `west.yml` under `remotes` and `projects`, then define the trackpad node under an I2C bus with `compatible = "azoteq,iqs5xx"`. The **HumanComputingInc/zmk-keyboard-iqs5xx-dev** repository provides a useful development shield for testing with debug logging enabled by default.

## Device tree configuration requires precise syntax

Modern ZMK uses Zephyr's pinctrl subsystem rather than deprecated direct pin properties. For nice!nano v2, the default I2C0 pins are **P0.17 (SDA)** and **P0.20 (SCL)**, accessible via the `&pro_micro_i2c` alias.

Complete trackpad configuration:

```dts
#include <dt-bindings/zmk/input_transform.h>

/ {
    tps43_listener: tps43_listener {
        compatible = "zmk,input-listener";
        device = <&tps43>;
        /* Orientation transforms if needed */
        input-processors = <&zip_xy_transform (INPUT_TRANSFORM_XY_SWAP | INPUT_TRANSFORM_Y_INVERT)>;
    };
};

&pro_micro_i2c {
    status = "okay";
    
    tps43: iqs5xx@74 {
        status = "okay";
        compatible = "azoteq,iqs5xx";
        reg = <0x74>;
        
        reset-gpios = <&pro_micro 21 GPIO_ACTIVE_LOW>;
        rdy-gpios = <&pro_micro 20 GPIO_ACTIVE_HIGH>;
        
        one-finger-tap;
        two-finger-tap;
        press-and-hold;
        press-and-hold-time = <200>;
        
        scroll;
        natural-scroll-y;
    };
};
```

The TPS43 module uses a 6-pin FPC connector: RDY (interrupt), NRST (reset), GND, VDDHI (1.65-3.6V), SCL, SDA. Both **reset-gpios** and **rdy-gpios** properties are required for proper initialization sequencing.

## Split keyboard setup demands right-side-as-central configuration

ZMK defaults to left-side-as-central, so hosting the trackpad on the right requires explicit role reversal. In your right shield's `Kconfig.defconfig`:

```kconfig
if SHIELD_SOFLE_RIGHT

config ZMK_KEYBOARD_NAME
    default "Sofle"

config ZMK_SPLIT_ROLE_CENTRAL
    default y

endif
```

**After changing central roles, you must flash `settings_reset` to both halves**—the pairing information becomes invalid when roles swap.

Placing the trackpad on the central side is strongly recommended because mouse data routes directly to the host without an additional BLE hop. Central-to-host latency averages **3.75ms** versus **7.5ms** when data must traverse the split link first. For pointing devices, this difference is perceptible.

The nice!nano v2 provides these GPIO options for the trackpad's reset and ready pins: P1.04, P1.02, P1.00 on back pads, or P0.06/P0.08 if UART isn't needed. **Avoid P0.12** entirely—it has bootloader issues with I2C pull-down resistors.

## Known conflicts affect component placement decisions

Several resource conflicts matter for your build:

| Conflict | Impact |
|----------|--------|
| nice!view + RGB underglow | **Cannot coexist**—both use SPI3 pins (P1.13, P0.10, P1.11) |
| OLED + I2C trackpad | **Works fine**—use separate addresses on same bus (OLED: 0x3C, TPS43: 0x74) |
| Battery ADC | **P0.04 reserved**—cannot repurpose for trackpad GPIO |

I2C devices sharing `&pro_micro_i2c` work reliably as sibling nodes. If you're using an OLED display, simply add the trackpad node alongside it under the same I2C bus definition.

## Debugging requires USB logging and hardware verification

Enable USB logging with the modern snippet approach:

```yaml
# build.yaml
include:
  - board: nice_nano_v2
    shield: sofle_right
    snippet: zmk-usb-logging
```

Add I2C-specific debugging:
```
CONFIG_I2C_LOG_LEVEL_DBG=y
CONFIG_LOG_PROCESS_THREAD_STARTUP_DELAY_MS=8000
```

View output via serial console—`sudo tio /dev/ttyACM0` on Linux. Watch for these error patterns:

| Error | Likely cause |
|-------|--------------|
| `I2C bus not ready` | Wrong pins or I2C not enabled in Kconfig |
| `scl timeout` | Missing/weak pull-ups or wiring issue |
| `Data queue timed out` | Device not powered or reset stuck |

**External pull-up resistors (2.2-4.7kΩ) are strongly recommended** for 400kHz I2C operation. The nRF52840's internal pull-ups are too weak for reliable fast-mode communication. A $5 Saleae-compatible logic analyzer with PulseView can definitively diagnose I2C problems by showing actual bus traffic and ACK/NACK responses.

## Alternatives exist if the IQS5xx driver proves problematic

**QMK has mature, built-in IQS5xx support** with `POINTING_DEVICE_DRIVER = azoteq_iqs5xx`. If ZMK's module causes issues, testing the same hardware with QMK can isolate firmware versus hardware problems. QMK offers more configuration options and better documentation for this specific trackpad family.

The **Cirque Pinnacle** (petejohanson/cirque-input-module) is the most widely-used ZMK trackpad, available in 23/35/40mm sizes with SPI interface (I2C requires removing resistor R1). Community experience is deeper, making troubleshooting easier.

For trackball alternatives, **inorichi/zmk-pmw3610-driver** supports the ultra-low-power PMW3610 sensor—excellent for wireless builds concerned about battery life.

## Conclusion

The AYM1607 driver provides everything needed for TPS43 integration on your Sofle. Critical implementation details: configure right-side-as-central via Kconfig, use `&pro_micro_i2c` with proper pinctrl syntax, include both reset and ready GPIO assignments, and add external **2.2-4.7kΩ pull-up resistors** for reliable I2C. The TPS43's EOL status means planning for future hardware changes—Cirque trackpads offer a more actively-supported alternative if you encounter persistent issues.