# Next Debug Steps - Pinpointing System Freeze

## Current Status
**Problem**: System completely freezes after ONE mouse movement
- Last working commit: 7bdff74 (driver), 932a682 (config)
- Symptom: Logs stop completely after `zmk_hid_mouse_movement_set`
- Not just trackpad - entire system (BLE, logging, everything) freezes

## What We Know For Sure
1. ✅ Device tree works - driver instantiates correctly
2. ✅ Polling timer starts and runs
3. ✅ I2C communication works (proven by successful reads)
4. ✅ Mouse movement data is read correctly
5. ✅ Fixed: printk flooding (commit d503718)
6. ✅ Fixed: K_FOREVER deadlock (commit d67990a)
7. ❌ System freezes after first `input_report_rel()` call

## The Plan - Three-Phase Debug

### Phase 1: Pinpoint Exact Freeze Line (CURRENT)
**Goal**: Determine which exact line causes the freeze

**Method**: Add printk before and after every critical operation in movement reporting

**Implementation**:
```c
// In work handler around line 233-243:
if (rel_x != 0 || rel_y != 0) {
    printk("*** IQS5XX: About to report movement X=%d Y=%d\n", rel_x, rel_y);

    ret = input_report_rel(dev, INPUT_REL_X, rel_x, false, K_NO_WAIT);
    printk("*** IQS5XX: After report X, ret=%d\n", ret);

    if (ret < 0) {
        LOG_WRN("Failed to report X movement: %d (queue full?)", ret);
    }

    ret = input_report_rel(dev, INPUT_REL_Y, rel_y, true, K_NO_WAIT);
    printk("*** IQS5XX: After report Y, ret=%d\n", ret);

    if (ret < 0) {
        LOG_WRN("Failed to report Y movement: %d (queue full?)", ret);
    }

    printk("*** IQS5XX: Movement reporting complete\n");
}

// After movement reporting block:
printk("*** IQS5XX: About to end comm window\n");

// At end_comm label:
end_comm:
    printk("*** IQS5XX: At end_comm, about to call end_comm_window\n");
    iqs5xx_end_comm_window(dev);
    printk("*** IQS5XX: After end_comm_window\n");

// At work handler exit (already added):
// Work completion counter prints here
```

**Expected Results**:
- **If freeze before "About to report"**: Problem is earlier in work handler
- **If freeze after "After report X"**: Problem is in X reporting
- **If freeze after "After report Y"**: Problem is in Y reporting or downstream
- **If freeze after "Movement complete"**: Problem is in end_comm or after
- **If freeze after "After end_comm_window"**: Problem is at work handler exit

**Files to modify**:
- `/Users/lrs/zmk-driver-workspace/zmk-driver-azoteq-iqs5xx/drivers/input/iqs5xx.c`

**Commits**:
1. Add granular printk statements
2. Push to trigger build
3. Flash sofle_right_debug.uf2
4. Test and capture logs
5. **Update CURRENT_SESSION_DEBUG.md with results**

---

### Phase 2: Based on Results (FUTURE)

#### If Freeze is in input_report_rel():
**Theory**: Input subsystem callback is deadlocking

**Test**: Temporarily replace with mock:
```c
// Instead of input_report_rel(), just log:
printk("*** MOCK: Would report X=%d Y=%d\n", rel_x, rel_y);
// Skip actual reporting
```

**If mock works**: Problem is in ZMK input subsystem
**If mock still freezes**: Problem is elsewhere in work handler

#### If Freeze is in end_comm_window():
**Theory**: I2C bus corruption or driver issue

**Test**:
1. Check return value of `iqs5xx_end_comm_window()`
2. Try removing the end_comm_window call entirely
3. Add I2C bus reset before end_comm

#### If Freeze is After All Our Code:
**Theory**: Work queue system or Zephyr issue

**Test**: Add work handler exit barrier:
```c
// At very end of work handler:
k_sleep(K_MSEC(1)); // Force context switch
printk("*** After k_sleep\n");
```

---

### Phase 3: Implement Fix (FUTURE)

Based on Phase 1 and 2 results, implement targeted fix.

**Possible fixes**:
1. **If input system**: Defer reporting to different work queue or thread
2. **If I2C**: Add recovery logic or change transaction order
3. **If work queue**: Use dedicated thread instead of work queue
4. **If USB conflict**: Disable USB HID in debug build (use BLE only)

---

## Testing Procedure

### Before Each Test:
1. Verify latest build succeeded: `gh run list --limit 1`
2. Download artifacts from GitHub Actions
3. **Flash sofle_right_debug.uf2** (not regular build)
4. Double-tap reset on nice!nano
5. Wait for flash to complete

### During Test:
1. **Connect USB IMMEDIATELY after flash** (don't wait for boot)
2. Open serial: `tio /dev/tty.usbmodem2101`
3. If connected before boot, you'll see init messages
4. Watch for trackpad movement
5. Capture ALL logs until freeze

### After Test:
1. Save complete log to file
2. Identify last printk message
3. **Update CURRENT_SESSION_DEBUG.md** with:
   - Last message seen
   - Hypothesis about freeze location
   - Next test to run
4. Commit updates to debug docs

---

## Log Analysis Checklist

When analyzing logs, look for:
- [ ] Did we see `*** IQS5XX INIT STARTING ***`? (If yes, connected before boot)
- [ ] Did we see `Timer fired (count: 1-4)`? (Proves timer is running)
- [ ] Did we see `Work handler completed (count: 1-4)`? (Proves handler runs)
- [ ] Last printk message before freeze?
- [ ] Were there any error return codes (ret < 0)?
- [ ] Any assertions or panics in the logs?

---

## File Locations Reference

### Debug Logs
- Current session: `/Users/lrs/zmk-config.git/CURRENT_SESSION_DEBUG.md`
- This plan: `/Users/lrs/zmk-config.git/NEXT_DEBUG_STEPS.md`
- Old research: `/Users/lrs/zmk-config.git/Research.md`
- Old debug log: `/Users/lrs/zmk-config.git/TRACKPAD_DEBUG_LOG.md`

### Code
- Driver: `/Users/lrs/zmk-driver-workspace/zmk-driver-azoteq-iqs5xx/drivers/input/iqs5xx.c`
- Driver headers: `/Users/lrs/zmk-driver-workspace/zmk-driver-azoteq-iqs5xx/drivers/input/iqs5xx.h`
- Config overlay: `/Users/lrs/zmk-config.git/config/sofle_right.overlay`
- Kconfig: `/Users/lrs/zmk-config.git/config/sofle.conf`

### Repos
- Driver fork: https://github.com/suchobits/zmk-driver-azoteq-iqs5xx (branch: polling-mode-support)
- Config repo: https://github.com/suchobits/zmk-config (branch: main)

---

## If Session Gets Compacted

**Read these files first**:
1. This file (`NEXT_DEBUG_STEPS.md`) - for current plan
2. `CURRENT_SESSION_DEBUG.md` - for what we've tried and learned
3. Look for **Phase 1** section above to see current step

**Current task**: Adding granular printk to pinpoint freeze location in work handler

**Last known state**: System freezes after one mouse movement at `input_report_rel()` call

**Files to modify**: `iqs5xx.c` - add printk before/after input_report_rel() calls

**Don't repeat**: Device tree checks, logging subsystem fixes, K_FOREVER fixes - these are done
