# Current Session Debug Log - 2025-12-31

## Session Goal
Fix trackpad that works once then stops responding / causes system freeze.

## Current State (End of Session)
- **Symptom**: Trackpad generates ONE mouse movement, then entire system freezes
- **Evidence**: Logs completely stop after mouse movement (no BLE, no I2C, nothing)
- **Firmware**: Using debug build with USB logging enabled
- **Latest commit**: 932a682 (main), 7bdff74 (driver fork)

---

## Timeline of Attempts and Learnings

### 1. Initial Analysis - Device Tree Instantiation Check
**Tried**: Added compile-time diagnostics with `BUILD_ASSERT` and `#warning` to check if device tree was instantiating the IQS5XX device.

**Result**: Device tree IS working correctly
- Build logs showed `CONFIG_DT_HAS_AZOTEQ_IQS5XX_ENABLED=y` on right side
- Build logs showed dependency unsatisfied on left side (correct)
- No "NO IQS5XX DEVICES FOUND" warning appeared

**Learning**: Device tree instantiation is NOT the problem. The driver is being compiled and the device node is being recognized.

**Commit**: f7bfaf2

---

### 2. printk() Debugging to Bypass Logging Subsystem
**Tried**: Added `printk()` calls throughout init sequence to see if init was running.

**Rationale**: `LOG_ERR()` messages weren't appearing. Suspected logging subsystem might not be ready during `POST_KERNEL` init.

**Result**: Init messages still not visible (but explained later - see #9)

**Learning**: Init DOES run, but happens before USB console connects (~3 seconds before user connects tio).

**Commits**: 200e8b8, 4a98a67

---

### 3. QMK-Style Power Management Init
**Tried**: Implemented QMK's proven initialization sequence:
- Set `IDLE_MODE_TIMEOUT=255` to disable LP1/LP2 sleep modes
- Enable `MANUAL_CONTROL` to disable automatic power management
- Enable `REATI` for continuous touch re-calibration
- Configure for streaming mode (not event mode)

**Rationale**: QMK's azoteq_iqs5xx driver successfully works with TPS43. Their key setting is disabling idle timeout to prevent trackpad from sleeping.

**Result**: Configuration likely not being applied during init (device not responsive yet)

**Learning**: The approach is correct (proven by QMK), but timing is wrong. Device needs ~2 seconds to wake up.

**Commit**: da22c89

**QMK Reference**: `/Users/lrs/qmk_firmware/drivers/sensors/azoteq_iqs5xx.c` line 337

---

### 4. Retry Logic for Device Wake-Up
**Tried**: Added retry loop (up to 3 attempts with 100ms delays) for configuration writes. Made init not fail if setup_device() fails.

**Rationale**: Device may not be responsive immediately after power-on.

**Result**: Still seeing NACK errors during init, but trackpad does work briefly

**Learning**: Retries help but aren't enough. Device needs longer to wake up.

**Commit**: 38d468f

---

### 5. Increased Power-On Delay
**Tried**: Increased initialization delay from 500ms to 2500ms before attempting configuration.

**Rationale**: Logs consistently showed device becoming responsive at ~2-3 second mark after boot.

**Result**: Unknown - masked by later issues

**Learning**: Device has a long power-on/calibration time (~2 seconds minimum).

**Commit**: 8853107

---

### 6. CRITICAL BUG: printk() Console Flooding
**Tried**: Reduced printk frequency in work handler from every poll (20Hz) to only first success + every 100th.

**Problem Discovered**: Work handler was calling `printk()` on EVERY successful read (20 times per second), causing:
- Console buffer overflow
- System blocking when buffer full
- Watchdog timeout
- Complete system freeze

**Result**: System no longer freezes from console flooding

**Learning**: **NEVER use printk() in hot paths (20Hz polling)**. Only use for rare events or throttled output.

**Commit**: d503718

---

### 7. CRITICAL BUG: K_FOREVER Deadlock in Input Reporting
**Tried**: Replaced all `K_FOREVER` with `K_NO_WAIT` in `input_report_*()` calls.

**Problem Discovered**: Work handler was calling `input_report_rel()` with `K_FOREVER`. If HID queue was full/blocked, this caused:
- Work handler to block forever
- Entire work queue system to freeze
- No other work items could run (logging, BLE, etc.)
- Complete system deadlock

**Changed**:
- `input_report_rel()` for mouse movement
- `input_report_key()` for button presses
- Scroll wheel reporting

**Result**: System still freezes after one movement

**Learning**: **NEVER use K_FOREVER in work queue handlers**. Always use K_NO_WAIT or a reasonable timeout.

**Commit**: d67990a

---

### 8. Debug Counters for Timer/Work Handler Tracking
**Tried**: Added printk counters to track:
- Timer firing (`iqs5xx_poll_timer_handler`)
- Work handler completion

**Rationale**: Determine if timer stops firing or work handler stops completing.

**Result**: Messages never appeared in logs (see #9)

**Learning**: Debug messages from early boot (before USB connect) are lost.

**Commit**: 7bdff74

---

### 9. USB Connection Timing Issue (Not a bug, but important context)
**Observation**: User connects USB serial ~3 seconds after device boots.

**Impact**:
- All init messages are lost (happen at boot time, ~0-3 seconds)
- First ~60 poll attempts are lost
- We only see logs starting mid-session

**Learning**: To see init messages, need to:
- Connect USB BEFORE powering device, OR
- Add delayed printk that fires after USB enumeration, OR
- Use persistent logging to flash/RAM

**Evidence**: Logs show first message around timestamp 00:00:03.x, device was already polling and failing

---

## Current Problem: System Freeze After One Mouse Movement

### Symptoms
1. **Pattern**:
   - Device boots
   - 40-90 failed I2C reads (NACK errors)
   - 10-20 successful reads
   - ONE mouse movement reported (e.g., `225/-23`)
   - **Complete system freeze** - no more logs of any kind (not just trackpad)

2. **Evidence from logs**:
   ```
   [00:00:04.385] I2C error (last error before movement)
   [00:00:04.538] Mouse movement set to -194/81
   [00:00:04.540] Mouse movement set to 0/0
   [NOTHING - complete silence]
   ```

3. **Not just logging stopped**: BLE scanning also stops (no more split_central logs)

### What We Know
- ✅ Device tree works
- ✅ Driver compiles and loads
- ✅ Polling timer starts
- ✅ I2C communication works (91 successes in one test)
- ✅ Mouse movement data is read correctly
- ✅ `input_report_rel()` is called successfully at least once
- ❌ System completely freezes after first movement
- ❓ Don't know if it's the input subsystem, HID stack, USB stack, or something else

### What We Fixed (But Didn't Solve The Freeze)
- ✅ printk flooding
- ✅ K_FOREVER deadlock in input reporting
- ✅ Retry logic for device wake-up
- ✅ Extended power-on delay

### Hypotheses to Test

#### Hypothesis 1: Input Subsystem Callback Deadlock
**Theory**: ZMK's input subsystem has a callback that's deadlocking on USB HID send.

**Evidence**:
- System freezes immediately after `input_report_rel()` completes
- Even with K_NO_WAIT, downstream callbacks might still block

**Test**: Temporarily disable actual input reporting, just log movement

#### Hypothesis 2: USB HID + USB CDC Conflict
**Theory**: USB HID (mouse) and USB CDC (logging) are conflicting when both active.

**Evidence**:
- Only happens in debug build with USB logging
- Freeze happens right after HID report

**Test**: Try non-debug build to see if issue persists without USB logging

#### Hypothesis 3: Work Handler Crash
**Theory**: Work handler is crashing/asserting after successful read, not deadlocking.

**Evidence**:
- Complete system freeze (not just hang)
- No logs at all after freeze

**Test**: Check for asserts, add more granular printk to pinpoint exact line

#### Hypothesis 4: I2C Bus Left in Bad State
**Theory**: After successful read, `end_comm_window()` or subsequent I2C operation puts bus in bad state, causing I2C driver to crash.

**Evidence**:
- Works fine with NACK errors
- Freezes after successful transaction

**Test**: Add printk immediately after `iqs5xx_end_comm_window()`

---

## Key Configuration Files (Current State)

### Driver: `/Users/lrs/zmk-driver-workspace/zmk-driver-azoteq-iqs5xx/drivers/input/iqs5xx.c`
- Polling mode: 50ms interval (20Hz)
- All input reports use K_NO_WAIT
- QMK-style configuration in setup_device()
- 2.5s power-on delay before configuration

### Config: `/Users/lrs/zmk-config.git/config/sofle_right.overlay`
```dts
&pro_micro_i2c {
    status = "okay";
    clock-frequency = <I2C_BITRATE_STANDARD>;  // 100kHz

    trackpad: iqs5xx@74 {
        compatible = "azoteq,iqs5xx";
        reg = <0x74>;
        status = "okay";
        // Polling mode - no GPIO pins
    };
};
```

### Build: `build.yaml`
```yaml
- board: nice_nano_v2
  shield: sofle_right
  snippet: zmk-usb-logging
  artifact-name: sofle_right_debug
```

---

## Next Steps (Prioritized)

1. **Add granular printk to pinpoint freeze location**
   - After each line in movement reporting code
   - Before/after `iqs5xx_end_comm_window()`
   - At work handler exit

2. **Test with input reporting disabled**
   - Comment out `input_report_rel()` calls
   - Just printk the movement values
   - See if freeze still happens

3. **Check for assertions/panics**
   - Look for Zephyr assert configuration
   - Add panic handler hook if available

4. **Try non-debug build**
   - Flash regular `sofle_right.uf2` (no USB logging)
   - See if issue is USB HID + USB CDC conflict
   - Will lose logging visibility though

---

## Important Notes

- **Don't chase old theories**: Device tree instantiation is confirmed working
- **Timing matters**: Device needs 2+ seconds to wake up after power-on
- **USB connection timing**: Init messages happen before user connects (~3s boot time)
- **printk limitations**: Can cause system issues if used in hot paths
- **K_FOREVER is dangerous**: Never use in work handlers - always use K_NO_WAIT

---

## Build Artifacts
- Latest successful build: Run #20627132207
- Main branch: 932a682
- Driver branch: 7bdff74 (polling-mode-support)
- Artifacts: `sofle_left.uf2`, `sofle_right.uf2`, `sofle_right_debug.uf2`
