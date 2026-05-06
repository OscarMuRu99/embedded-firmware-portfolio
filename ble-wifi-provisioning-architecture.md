# BLE + WiFi Provisioning Architecture

Patterns for provisioning WiFi credentials to embedded ESP32 devices via BLE,
storing them securely in NVS, and managing runtime WiFi connectivity.
All examples are generalized; no internal API endpoints, UUIDs, or credentials are included.

---

## 1. The Provisioning Problem

An ESP32 product ships without a preconfigured WiFi network. On first boot, it needs
a way to receive WiFi credentials from the user without a physical keyboard, touchscreen,
or hardcoded SSID.

Two approaches applied across multiple products in this portfolio:

| Approach | How it works | When to use |
|---|---|---|
| BLE GATT provisioning | Mobile app (or provisioning station) writes credentials to a BLE characteristic | Low-power devices; no browser available on host |
| WiFi captive portal | Device creates a temporary AP; user connects and submits via web form | Devices with sufficient flash and heap for a web server |

Both approaches store credentials in NVS and behave identically after the first successful provisioning.

---

## 2. BLE GATT Provisioning Flow

### High-level sequence

```
Device boots with no stored credentials
  → Start BLE advertisement (provisioning mode)
  → Mobile app scans → finds device → connects
  → App writes SSID:PASSWORD to provisioning characteristic
  → Device receives write → parses credentials
  → Device stops BLE advertisement
  → Device attempts WiFi connect
  → On success: save credentials to NVS, exit provisioning mode
  → On failure: restart BLE advertisement, report error
```

### Characteristic design

A single writable GATT characteristic is sufficient for basic credential delivery:

```
Service UUID:     [app-specific UUID]
Characteristic:   [app-specific UUID]
  Properties:     WRITE + WRITE_WITHOUT_RESPONSE
  Payload format: "SSID:PASSWORD\0"
```

Keeping it a single characteristic minimizes the BLE stack overhead and makes
the mobile app integration trivial.

### NVS namespace strategy

Organize NVS keys by functional domain:

```c
// WiFi credentials namespace
nvs_open("wifi_creds", NVS_READWRITE, &handle);
nvs_set_str(handle, "ssid",     ssid);
nvs_set_str(handle, "password", password);
nvs_commit(handle);
```

On boot, read credentials from NVS before starting any network operation.
If the keys are absent or the read fails, enter provisioning mode.

---

## 3. Multi-Profile WiFi Credentials

For devices that may roam between locations (office, factory floor, field test site),
support multiple saved credential profiles:

```c
// Save up to N profiles; identify by index
for (int i = 0; i < MAX_PROFILES; i++) {
    char ssidKey[16], passKey[16];
    snprintf(ssidKey, sizeof(ssidKey), "ssid_%d", i);
    snprintf(passKey, sizeof(passKey), "pass_%d", i);
    nvs_set_str(handle, ssidKey, profiles[i].ssid);
    nvs_set_str(handle, passKey, profiles[i].password);
}
```

Connection logic:
- Try profiles in order.
- On auth failure (reason 202) for all profiles, enter ERROR state (not endless retry).
- In ERROR state, accept new credentials via BLE — this allows field recovery without
  physical access to the device.

---

## 4. Captive Portal Provisioning (alternative approach)

For devices with a web server available:

```
Device boots with no credentials
  → Creates WiFi AP ("DeviceName-XXXX")
  → Serves a captive portal page from flash (PROGMEM HTML)
  → User connects to AP → portal redirects all HTTP traffic
  → User submits SSID + password via form POST
  → Device validates, stores in EEPROM/NVS, restarts in STA mode
```

Key implementation details:
- Store credentials encrypted in EEPROM — raw plaintext credential storage is not acceptable.
- Serve the portal HTML from flash (not SD or heap) to minimize memory usage during provisioning.
- Use `AsyncWebServer` (ESPAsyncWebServer) to handle captive portal redirect without blocking the main task.
- Set a captive portal timeout: if no credentials are submitted within N minutes, restart and try again.

---

## 5. WiFi Manager Architecture

### Boot-time vs. runtime ownership separation

A critical design decision: which task owns WiFi at boot vs. during runtime?

**Pattern applied in this portfolio's firmware:**

```
wifiManagerTask (boot):
  - Mounts storage
  - Reads/validates credentials from NVS
  - Starts captive portal OR connects to AP
  - Registers device with cloud
  - Stops WiFi
  - Creates worker tasks
  - Enters long idle loop (300s+ cycles)

loggerTask (runtime):
  - Owns runtime reconnect and upload cycle
  - Reconnects WiFi for upload, disconnects when done
  - Never calls WiFi APIs while wifiManagerTask is active
```

This separation prevents deadlock between boot-time provisioning and runtime upload.
The boot task explicitly releases the radio before the logger task starts.

### ERROR state management

Model WiFi as a state machine, not just "connected or not":

```
IDLE → CONNECTING → CONNECTED → DISCONNECTED → RECONNECTING → ERROR
                                                              ↓
                                                    (credentials failed)
                                                    BLE credential update
                                                              ↓
                                                           IDLE
```

The ERROR state with a cooldown timer is important:
- After N failed reconnect attempts across all profiles, enter ERROR.
- In ERROR: wait a configurable cooldown (e.g., 5 minutes) then retry.
- Accept new credentials via BLE at any time, even in ERROR state.
- On new credentials via BLE: immediately exit ERROR and retry without waiting for cooldown.

---

## 6. Runtime WiFi / BLE Gap Analysis

On ESP32, BLE and WiFi share the RF hardware. Known integration gaps:

| Scenario | Risk | Mitigation |
|---|---|---|
| BLE scan during HTTP upload | Scan gaps, upload stalls | Gate uploads: do not start while BLE is scanning |
| WiFi connect during BLE advertisement (provisioning) | Advertisement drops | Use NimBLE concurrent mode or sequence: stop BLE adv → connect WiFi → restart BLE |
| WiFi reconnect during BLE beacon scan | RSSI readings degraded | Defer WiFi reconnect until scan window ends |

A practical gating condition for uploads:

```c
bool canStartUpload() {
    return wifiConnected
        && !bleScanning
        && !inProvisioningMode
        && rtcValid
        && sdAvailable
        && authTokenAvailable;
}
```

---

## 7. Provisioning Button Pattern

For field recovery without a mobile app:

```c
void setup() {
    pinMode(PROV_BUTTON_PIN, INPUT_PULLUP);
    
    // Hold at boot for N seconds → clear all saved credentials
    uint32_t holdStart = millis();
    while (digitalRead(PROV_BUTTON_PIN) == LOW) {
        if (millis() - holdStart > PROV_CLEAR_HOLD_MS) {
            clearWifiCredentials();
            // Device will enter provisioning mode on next boot
            esp_restart();
        }
    }
}
```

This provides a hardware escape hatch when:
- Credentials are corrupt or for a network that no longer exists.
- The device is being reassigned to a different site.
- The mobile app is unavailable.

---

## 8. Security Considerations

### NVS credential storage
- NVS is stored in flash and is readable by anyone with physical UART access and `esptool`.
- For production: enable ESP32 NVS encryption (requires eFuse key provisioning at manufacturing time).
- At minimum, do not store credentials in plaintext EEPROM without encryption.

### TLS for cloud uploads
- Use TLS for all credential-bearing HTTP calls (registration, upload).
- `setInsecure()` (disabling certificate validation) is acceptable in development/DVT but **must not** ship to production. Mark it clearly:

```c
// TODO: replace setInsecure() with CA bundle before production
client.setInsecure();
```

### BLE provisioning window
- Limit the provisioning BLE advertisement to a time window. Do not advertise indefinitely.
- Consider requiring physical confirmation (button press) to enter provisioning mode on a device that already has credentials.

---

## 9. Validation Checklist

Before signing off provisioning on a new product:

- [ ] Device enters provisioning mode when no credentials stored
- [ ] BLE characteristic write delivers SSID + password correctly
- [ ] Credentials survive power cycle (NVS read on next boot succeeds)
- [ ] Wrong password → device reaches ERROR state (not infinite retry loop)
- [ ] BLE credential update in ERROR state → immediate reconnect (no cooldown wait)
- [ ] Multi-profile: tries all profiles before entering ERROR
- [ ] Provisioning button hold clears credentials and triggers re-provisioning
- [ ] WiFi and BLE radio do not contend during upload (gating confirmed in DVT)
- [ ] TLS cert validation planned/enabled for production build

---

## 10. Future Improvements

- **Encrypted NVS:** Provision a unique NVS encryption key per device at manufacturing time.
- **BLE bond/pairing:** Add BLE bonding to prevent unauthorized credential writes after initial setup.
- **Certificate pinning:** Replace `setInsecure()` with a bundled CA root or pinned leaf certificate.
- **Credential rotation:** Allow remote credential update via an authenticated OTA-style command (no physical access required).
- **Provisioning audit log:** Write a provisioning record to SD or NVS each time credentials are changed, including timestamp and MAC of the provisioning source device.
