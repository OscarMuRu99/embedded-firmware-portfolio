# FreeRTOS Runtime Hardening Notes

Lessons from embedded firmware development on ESP32 + FreeRTOS across multiple products.
Patterns extracted from real DVT and production work; no proprietary source code included.

---

## 1. Task Architecture Decisions

### Task-based vs loop()-based

For complex products with multiple concurrent sensors and communication stacks, put all
real work in FreeRTOS tasks and leave `loop()` empty (or nearly so):

```
loop() {
    // intentionally empty
}
```

Reasons:
- The Arduino `loop()` runs as a low-priority FreeRTOS task on Core 1. Putting heavy work here means you're sharing that task's time budget with the FreeRTOS scheduler.
- Tasks with explicit stacks and priorities are easier to monitor and tune than one monolithic loop.
- It's easier to suspend, resume, or kill a specific task than to conditionally skip sections in a shared loop.

### Core pinning

On dual-core ESP32:
- BLE (NimBLE) host task runs on Core 0 by default. Keep your BLE application task there too to avoid cross-core scheduling conflicts.
- All other sensor, logger, and WiFi tasks can run on Core 1.

Mixing BLE application tasks between cores is a common source of intermittent crashes.

---

## 2. Shared State: Queues and Mutex Discipline

### Single-item latest-value queues

For sensor data that is sampled periodically and consumed once per logger cycle, a queue of length 1
with `overwrite` behavior is the right model:

- Producer overwrites stale data without blocking.
- Consumer always reads the latest snapshot.
- No memory accumulation.

For burst data (e.g., BLE advertisement events during a scan window), use a larger bounded queue
and monitor for overflow at DVT time.

### Mutex ownership rules

Establish explicit ownership:
1. Only one task "owns" a given resource (SD card, WiFi, RTC).
2. Other tasks access that resource only through the owning task's queue or accessor functions.
3. Never hold two mutexes simultaneously — always acquire in a fixed order to prevent deadlock.

A common trap: one task manages boot-time WiFi and another manages runtime upload WiFi.
If they both try to call `WiFi.begin()` and `WiFi.disconnect()` independently, you get a deadlock
or silent race. The fix is to make ownership explicit: the boot task stops WiFi before
the upload task starts, and they never overlap.

---

## 3. BLE Scan State Machine — Race Prevention

### The `START_PENDING` pattern

Never issue a BLE scan start immediately after a scan stop. The NimBLE host task (Core 0)
needs time to quiesce before it can safely start a new scan. Without a settle delay:

```
// UNSAFE — can corrupt NimBLE host stack on Core 0
ble_gap_disc_stop();
ble_gap_disc_start(params);  // immediate restart
```

The correct pattern uses an intermediate `START_PENDING` state with a minimum settle delay:

```
enum ScanState { IDLE, START_PENDING, SCANNING };

// On scan stop:
_state = START_PENDING;
_pendingDeadline = millis() + SCAN_SETTLE_MS;  // e.g., 50 ms

// In task loop:
if (_state == START_PENDING && millis() >= _pendingDeadline) {
    _state = SCANNING;
    startScan();
}
```

This single pattern eliminates an entire class of NimBLE host stack corruption crashes.

### Single blocking scan vs. chunked loop

Do not implement a scan window as a chunked loop:

```c
// PROBLEMATIC — repeated gap_disc start/stop corrupts NimBLE host on Core 0
for (int i = 0; i < 60; i++) {
    results = pScan->getResults(500);
    processResults(results);
    pScan->clearResults();
}
```

Use a single blocking call for the full window duration:

```c
// CORRECT — one gap_disc call per window
BLEScanResults results = pScan->getResults(SCAN_WINDOW_MS);
processResults(results);
```

The difference: the chunked approach issues `ble_gap_disc()` start/stop 60 times per window,
creating 60 opportunities for a host-stack race.

### Pending request handling during active scan

If `requestScanWindow()` is called while a scan is already active, do not start a second scan.
Set a flag and restart cleanly after the current window ends:

```c
void requestScanWindow() {
    if (_state == SCANNING) {
        _scanRequestPending = true;  // restart after current window
        return;
    }
    _state = START_PENDING;
    _pendingDeadline = millis() + SCAN_SETTLE_MS;
}
```

---

## 4. Stack Overflow Detection

### Enable stack overflow checking

In your FreeRTOS configuration, enable stack overflow detection:

```c
#define configCHECK_FOR_STACK_OVERFLOW 2
```

Then implement the hook:

```c
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
    Serial.printf("[FATAL] Stack overflow in task: %s\n", pcTaskName);
    esp_restart();  // or trap in debug builds
}
```

Mode 2 writes a pattern to the stack and checks it on each context switch — higher overhead but catches overflows faster.

### High-water mark logging

During DVT, log the stack high-water mark for every task periodically:

```c
UBaseType_t hwm = uxTaskGetStackHighWaterMark(NULL);
DEBUG_LOG("[TASK] stack HWM: %u words remaining", hwm);
```

Warning thresholds:
- **< 10% of allocated stack** — increase stack size or reduce locals/recursion.
- **Trending downward across runs** — unbounded growth; investigate dynamic allocation in the task.

Log HWM after any operation that is likely to consume peak stack (BLE scan completion,
HTTP upload, large JSON serialization).

---

## 5. Task Startup Race Prevention

### Problem: task B depends on task A's initialization

A common pattern:

```c
void setup() {
    wifiManager.begin();      // starts WiFiManagerTask
    createTask(loggerTask);   // loggerTask uses WiFi — but WiFiManagerTask may not have
                              // finished registration yet
}
```

If `loggerTask` tries to use WiFi before `WiFiManagerTask` completes registration and
stops the radio, you get unpredictable behavior.

**Fix:** Let the boot-time task own deferred task creation:

```c
void wifiManagerTask(void*) {
    connectWifi();
    registerDevice();
    storeCredentials();
    WiFi.disconnect();  // hand radio back before creating workers
    
    // Only now is it safe to create the upload-owning tasks
    xTaskCreate(accelerometerTask, ...);
    xTaskCreate(loggerTask, ...);
    xTaskCreate(bleTask, ...);
    
    while (true) { vTaskDelay(pdMS_TO_TICKS(IDLE_POLL_MS)); }  // idle
}
```

This eliminates an entire class of timing-sensitive startup races that are impossible to reproduce reliably under test conditions.

---

## 6. Watchdog Considerations

### Blocking calls and the WDT

The ESP32 task watchdog will reset the device if a task does not yield for its configured
timeout. Long-blocking operations to watch:

- `HTTPClient::POST()` with a high timeout (e.g., 12 s) — must yield during send/receive.
- `BLEScan::getResults(windowMs)` — single blocking call for the full window; ensure the scan window is shorter than the WDT timeout.
- SD `file.write()` on a cold mount — can be slow; wrap with yield points if writing large buffers.

Rules:
- Set the WDT timeout to be meaningfully larger than your longest expected blocking operation.
- In debug/DVT builds, use a longer WDT timeout than production to give headroom for serial logging overhead.
- Watch for `rst:0x8` in the boot log (WDT reset) during DVT — it indicates a blocking path.

---

## 7. BLE + WiFi Radio Contention

ESP32 shares the RF hardware between BLE and WiFi. Running both simultaneously is supported
but increases the probability of scan gaps and upload stalls.

Mitigation patterns applied in firmware development:
- Gate bulk uploads so they only start when BLE scanning is not in progress.
- Shut down BLE before long HTTP upload sessions; restart BLE after upload completes.
- Use the `senderBusy` flag as a gating input to the sleep/scan decision logic.

```c
bool canUpload() {
    return sdAvailable
        && !bleScanning
        && wifiConnected
        && !activeSession       // no in-progress device operation
        && motionWindowInactive;
}
```

This multi-condition gating eliminates upload failures caused by radio contention and
prevents uploads from interrupting in-flight BLE scan sessions.

---

## 8. Deep Sleep and Wake Reliability

### Prevent boot-sleep thrash

If the device enters deep sleep too quickly after boot, a unit placed on a surface immediately
after power-up will wake from motion → boot → detect no motion → sleep → wake → repeat.

Use a minimum-awake-after-boot timer:

```c
const uint32_t MIN_AWAKE_MS_AFTER_BOOT = 5000;
bool canSleep() {
    return (millis() - bootTimeMs) > MIN_AWAKE_MS_AFTER_BOOT
        && !actuatorsActive
        && !bleScanning
        && staticDurationMs > STATIC_SLEEP_THRESHOLD_MS;
}
```

### Clear interrupt latch before sleeping

If the wake source is a GPIO interrupt (e.g., IMU motion interrupt), the latch must be
cleared before entering sleep:

```c
void preSleep() {
    imu.clearInterruptLatch();  // read INT_STATUS to clear
    WiFi.disconnect(true);      // prevent WiFi-related wake
    esp_sleep_enable_ext0_wakeup(IMU_INT_PIN, HIGH);
    esp_deep_sleep_start();
}
```

Failing to clear the latch causes the device to wake immediately on the next boot cycle.
