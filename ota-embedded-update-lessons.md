# OTA Embedded Update Lessons

Architecture patterns and real-world lessons from implementing OTA (Over-the-Air)
firmware updates on ESP32-based IoT products.
Status of each project is stated honestly — implemented, validated, pending, or planned.

---

## Project Status Overview

| Product | OTA Status | Notes |
|---|---|---|
| Product A (mopping unit DVT) | Not implemented — not in scope | DVT focus was sensor validation and BLE; OTA planned post-DVT |
| Product B (IoT hub) | Assessed — conditional | Partition table and rollback flow are the blockers; not recommended before MVP |
| Product V (vacuum DVT) | Firmware-side implemented | Two-stage model + NVS persistence + SHA-256 + retry tracking; hardware/backend validation depends on board and server availability |

**Honest disclaimer:** OTA is the last mile of embedded software delivery. Firmware-side
code that compiles and passes offline review is not the same as validated end-to-end OTA.
For Product V, the firmware-side OTA layer is implemented; the full validation path
(backend-triggered, hardware-on-bench, rollback under power loss) requires dedicated
hardware availability and backend integration work not yet completed at time of writing.

---

## 1. Partition Table Requirements

OTA on ESP32 requires an OTA-capable partition table. The default Arduino "Single App"
table does not support OTA.

```
# Minimum partition table for OTA
# Name,   Type, SubType, Offset,  Size
nvs,      data, nvs,     0x9000,  0x5000
otadata,  data, ota,     0xe000,  0x2000
app0,     app,  ota_0,   0x10000, 0x1F0000
app1,     app,  ota_1,   0x200000,0x1F0000
spiffs,   data, spiffs,  0x3F0000,0x10000
```

Key constraints:
- `app0` and `app1` must be equal size and large enough for the firmware binary.
- `otadata` stores which partition to boot from — do not shrink it.
- OTA partition offsets must be 64KB-aligned.
- If a deployed device has the wrong partition table, you must physically reflash via UART.
  There is no OTA path to change the partition table.

---

## 2. Two-Stage OTA Model (Product V Implementation Pattern)

Rather than downloading and flashing immediately on receiving an OTA command, split
OTA into two stages separated by NVS persistence:

```
Stage 1 — Poll:
  Backend delivers OTA request (request_id, version, firmware_url, sha256, size)
  Firmware validates the request → saves to NVS → returns ACK to backend
  Firmware continues normal operation

Stage 2 — Execute:
  On next eligible loop tick (gating conditions met):
    Load OTA request from NVS
    Validate gating: not uploading, not scanning, SD available, heap sufficient
    Download + flash
    Verify SHA-256 of written partition
    Save boot confirmation to NVS
    esp_restart()
```

**Why two stages?**
- The download + flash operation takes 30–120 seconds and cannot be interrupted safely.
- By persisting the request in NVS, the device can choose the right moment to apply the update (e.g., device idle, not mid-session).
- The pending flag survives power loss. If the device loses power before flashing, it retries on next boot.
- `clearOtaRequest()` is called only on full verified success — not before.

### NVS OTA record structure (illustrative)

```c
struct OtaRequest {
    bool     pending;
    char     request_id[64];
    char     version[32];
    char     url[256];
    char     sha256[65];     // hex SHA-256 of expected firmware
    bool     force;
    uint32_t size_bytes;     // 0 if not provided by backend
    uint8_t  retry_count;
    char     last_error[64];
    uint32_t last_attempt_at; // unix seconds; 0 if clock not yet synced
};
```

---

## 3. Boot Confirmation Pattern

After a successful OTA flash and `esp_restart()`, the new firmware must confirm that it
booted correctly. Without this, a bad firmware update that boots but immediately crashes
cannot be detected by the bootloader.

### Pattern

Before restarting, save a boot confirmation token to NVS:

```c
saveOtaBootConfirm(request_id, new_version);
esp_restart();
```

On next boot (new firmware), load and clear the token:

```c
char confirmed_id[64], confirmed_version[32];
if (loadAndClearOtaBootConfirm(confirmed_id, sizeof(confirmed_id),
                               confirmed_version, sizeof(confirmed_version))) {
    logEvent("ota.boot_confirmed", confirmed_id, confirmed_version);
    // Notify backend that update succeeded
}
```

If the new firmware crashes before reading the token, the token stays in NVS.
On the next boot (rollback firmware), the token is stale and can be used to log the failure.

This is a lightweight alternative to the full ESP-IDF OTA rollback mechanism, suitable
for products that don't need automatic rollback partitions.

---

## 4. SHA-256 Verification

Always verify the downloaded firmware before writing to the partition:

```c
// Compute SHA-256 of downloaded bytes during stream
mbedtls_sha256_context sha_ctx;
mbedtls_sha256_init(&sha_ctx);
mbedtls_sha256_starts(&sha_ctx, 0);  // 0 = SHA-256 (not SHA-224)

while (bytesRemaining > 0) {
    int bytesRead = readChunk(buffer, CHUNK_SIZE);
    esp_ota_write(otaHandle, buffer, bytesRead);
    mbedtls_sha256_update(&sha_ctx, buffer, bytesRead);
    bytesRemaining -= bytesRead;
}

uint8_t digest[32];
mbedtls_sha256_finish(&sha_ctx, digest);

// Compare with expected SHA-256 from OTA request
if (memcmp(digest, expected_sha256, 32) != 0) {
    esp_ota_abort(otaHandle);
    saveLastError("SHA-256 mismatch");
    return OTA_ERR_HASH_MISMATCH;
}

esp_ota_end(otaHandle);
```

Never flash first and verify second — if the device loses power between flash and verify,
it will boot corrupted firmware.

---

## 5. Heap Gating

OTA download requires contiguous heap for the HTTP client, TLS buffers, and OTA write buffers.
On a busy ESP32 with active BLE, WiFi, and FreeRTOS tasks, heap fragmentation can prevent OTA
from starting even if `esp_get_free_heap_size()` looks adequate.

Minimum heap check before starting OTA:

```c
const size_t OTA_MIN_FREE_HEAP = 60000;  // calibrate per product

if (esp_get_free_heap_size() < OTA_MIN_FREE_HEAP) {
    logEvent("ota.deferred", "insufficient_heap");
    return;  // retry next eligible cycle
}
```

Additional heap mitigation:
- Suspend non-essential tasks before OTA (e.g., BLE scanner, sensor loggers).
- Disconnect WiFi STA, reconnect, then start OTA — this flushes stale sockets.

---

## 6. Retry and Cooldown

Transient failures (server busy, DNS failure, brief connectivity drop) should trigger retry
with exponential backoff. Permanent failures (SHA mismatch, partition write error) should
not retry indefinitely.

```c
const uint32_t OTA_RETRY_BACKOFF_BASE_MS = 60000;
const uint8_t  OTA_MAX_RETRIES          = 5;

if (otaRequest.retry_count >= OTA_MAX_RETRIES) {
    clearOtaRequest();   // give up; await new request from backend
    logEvent("ota.abandoned", "max_retries_exceeded");
    return;
}

uint32_t backoff = OTA_RETRY_BACKOFF_BASE_MS * (1 << otaRequest.retry_count);
if (now - otaRequest.last_attempt_at < backoff / 1000) {
    return;  // cooldown not elapsed
}

otaRequest.retry_count++;
otaRequest.last_attempt_at = now;
saveOtaRequest(otaRequest);
attemptOtaDownload();
```

---

## 7. Backend-Triggered OTA Validation Checklist

For a complete OTA validation (not just firmware-side):

**Server side:**
- [ ] Backend can deliver OTA request with `request_id`, `version`, `url`, `sha256`
- [ ] Backend can poll device for OTA status (pending / in_progress / success / failed)
- [ ] Backend receives `ota.boot_confirmed` event after successful update

**Device side:**
- [ ] OTA request saved to NVS correctly (survives power cycle before execution)
- [ ] Gating conditions prevent OTA during active session/upload
- [ ] SHA-256 mismatch → abort (no flash), increment retry count, save last_error
- [ ] Successful flash → boot confirmation saved, device restarts
- [ ] New firmware boots, reads and clears boot confirmation, logs event
- [ ] Backend notified of success

**Resilience:**
- [ ] Power loss after NVS save but before flash → retry on next boot
- [ ] Power loss during flash → ESP32 OTA handle cleanup; retry on next boot
- [ ] Power loss after flash but before restart → device stays on old firmware; request remains; retry applies new firmware again (idempotent if SHA-256 matches)
- [ ] Server returns 429 (throttle) → respect `Retry-After` header, backoff

---

## 8. Pre-MVP OTA Feasibility Assessment

One product in this portfolio was assessed for OTA feasibility before MVP:

**Verdict:** Conditional — possible but not recommended before MVP.

**Blockers identified:**
- Default partition table (4MB with SPIFFS) does not include OTA slots.
- Rollback-safe boot flow not implemented — a bad OTA that crashes on boot has no recovery path.
- WiFi usage must be arbitrated with the existing upload cycle; OTA download and normal upload cannot run simultaneously.
- Existing deployed devices would require physical reflash to migrate to an OTA-capable partition table.

**Recommendation:** Implement OTA after MVP, using the two-stage model described in section 2.
Do not attempt to add OTA to a device already deployed on a non-OTA partition table
without a physical reflash plan for existing units.

---

## 9. Security Notes

- Always use HTTPS for firmware downloads. HTTP firmware delivery is trivially interceptable.
- Validate the server's TLS certificate (do not use `setInsecure()` in production OTA paths).
- SHA-256 verification by the device provides integrity — the device can detect a tampered image even if the download channel is compromised.
- For higher security requirements, consider signing the firmware binary and verifying the signature before flashing (ESP32 Secure Boot V2).
- Store the OTA URL in the NVS record, not hardcoded. This allows the backend to rotate firmware delivery infrastructure without a firmware update.
