# Hardware & Manufacturing Coordination

Lessons from coordinating embedded firmware development with PCB design,
prototype fabrication, and bring-up validation on ESP32-based products.

---

## 1. Firmware ↔ PCB Design Coordination

### AppConfig.h as the single source of truth for pin maps

Every product firmware has one canonical pin map file (e.g., `AppConfig.h` or `defines.h`).
All drivers, task code, and peripheral configuration reference this file — not literals scattered
across the codebase.

```c
// AppConfig.h — single source of truth for pin assignments
// Update this file when PCB changes; the rest of the firmware follows.
// (GPIO values below are illustrative — update per your schematic)

#define APP_PIN_SD_CS        5
#define APP_PIN_IMU_CS       14
#define APP_PIN_IMU_INT1     26
#define APP_PIN_RTC_SDA      21
#define APP_PIN_RTC_SCL      22
#define APP_PIN_ACTUATOR_IN  27   // e.g., optocoupler digital input
#define APP_PIN_PROV_BUTTON  33
```

When communicating a PCB spin change to the firmware engineer:

> "GPIO14 → GPIO15 for IMU CS (PCB rev 1.1 — routing conflict on rev 1.0)"

The firmware engineer updates one file and rebuilds. No grep hunt through driver code.

### Customization checklist for a new PCB spin

Before bringing up a new board revision, review:

- [ ] All GPIO assignments in `AppConfig.h` match the schematic
- [ ] SPI clock speeds are conservative for first bring-up (e.g., 4 MHz before raising to rated speed)
- [ ] I2C pullup resistors present; I2C speed matches pullup strength
- [ ] Active-high vs. active-low configuration matches hardware for each digital input (optocouplers, switches)
- [ ] Interrupt pin direction and pull configuration (e.g., IMU INT1 active-high requires pull-down on MCU side)
- [ ] Provisioning button polarity (active-low with internal pull-up is standard)

---

## 2. JLCPCB Prototype Workflow

For ESP32-based IoT products, the typical fabrication flow:

```
Schematic → PCB Layout (KiCad) → Gerber export → JLCPCB upload → DRC + quote
  → Prototype order (5 pcs, ~10 business days)
  → Board receive → visual inspection → bring-up
```

### Schematic → PCB handoff checklist

Before sending to fab:

- [ ] All net names match between schematic and PCB
- [ ] ESP32 antenna clearance zone respected (no copper under module antenna)
- [ ] Decoupling caps placed close to power pins
- [ ] SPI and I2C trace lengths reasonable (< 10 cm for 4 MHz SPI)
- [ ] SD card SPI lines include series resistors if traces are long
- [ ] Crystal load capacitors calculated for ESP32 module (if external crystal used)
- [ ] Test points on critical nets (UART, SPI CS lines, power rails)
- [ ] Silkscreen labels for connectors and test points

### PCB feedback loop to designer

When a bring-up reveals a hardware issue, document it precisely:

```
Issue: IMU INT1 line floating at boot — no pull resistor
Root cause: Pull-down omitted from schematic; INT1 active-high signal
            floats high through PCB noise, causing spurious WoM interrupts before IMU init
Fix for rev 1.1: Add 10kΩ pull-down on INT1 line (IMU side of series resistor)
Evidence: Serial log shows WoM interrupt fires within 100ms of power-on before IMU init
```

This format (issue, root cause, fix, evidence) lets the PCB designer make the correct
change without needing to understand the firmware in depth.

---

## 3. Bring-Up Procedure

### First power-on

1. Visual inspection before power: correct orientation of polarized caps, no solder bridges,
   module seated correctly.
2. Apply power with current limiting (~200 mA limit for an ESP32 board at 3.3V).
3. Check idle current before flashing. If > 300 mA at idle, there is likely a short.
4. Connect serial at 115200 baud.
5. Flash factory firmware (or a bare bring-up sketch).
6. Read boot log — look for `rst:0x1` (power-on reset, normal) not `rst:0x8` (WDT reset).

### Peripheral bring-up sequence

Bring up peripherals in order of decreasing criticality:

```
1. MCU + serial output (confirms flash and boot)
2. Power rails (measure 3.3V, any other regulated rail)
3. I2C scan (confirms RTC + any I2C sensors on bus)
4. SPI peripherals (IMU, SD) — confirm CS lines isolated
5. GPIO inputs (buttons, optocouplers) — measure voltage levels
6. BLE advertisement visible on host scanner
7. WiFi connect to known AP
8. Full firmware stack
```

Do not flash the full production firmware on a board that hasn't passed step 4.
You will not be able to distinguish firmware bugs from hardware defects.

---

## 4. Remote / Zoom-Guided Testing

When the PCB designer or manufacturing contact is not co-located, structured remote
testing sessions are necessary:

### What to prepare before the call

- Serial monitor capture template (which lines to watch for, expected values)
- Numbered test steps with explicit pass/fail criteria
- Known-good reference serial log for comparison
- List of manual actions required (press button, apply load, short a pin, etc.)

### Session structure

```
1. (5 min) Confirm firmware version flashed and board revision in hand
2. (10 min) Power-on sequence — share serial log in real time (screen share or paste to chat)
3. (20 min) Peripheral checks — walk through bring-up sequence above
4. (20 min) Failure mode injection — wrong credentials, removed SD, power cycle
5. (5 min) Capture and archive: serial log, photos of board, verdict
```

### Async testing protocol

When synchronous calls are not possible, the remote contact follows a written script:

```
Step 3.1: Power off. Remove SD card. Power on. Wait 30 seconds.
          Paste entire serial output here: [paste box]
Expected: No crash, no WDT reset. Line containing "SD mount failed" should appear.
          IMU init line should appear. BLE should start.
```

The script must be explicit enough that a technician with no firmware knowledge can
execute it and provide useful output.

---

## 5. FCT / Manufacturing Documentation

For a product moving from prototype to production, functional circuit test (FCT)
documentation must be in place before the first production build.

### FCT fixture requirements for an ESP32 product

- UART programming interface (pogo pins to test points)
- Power supply measurement points (3.3V rail, battery input)
- GPIO stimulus inputs (simulate button press, optocoupler trigger)
- BLE scanner on test station host (confirms BLE advertisement after flash)
- WiFi access point on test bench (confirms WiFi connect)
- SD card populated in fixture (confirms SD mount)
- Pass/fail LED or display on fixture

### Factory firmware vs. DVT firmware

Factory firmware is not the same as DVT firmware:

| DVT firmware | Factory firmware |
|---|---|
| Verbose serial logging | Silent or minimal output |
| Developer-friendly debug commands | Production-neutral |
| Test AP credentials hardcoded for bench | No credentials hardcoded |
| Extended WDT timeout for debugging | Production WDT timeout |
| All debug subsystems enabled | Only required subsystems |

Factory firmware should run a self-test routine on boot, write PASS/FAIL to serial,
and enter a low-power idle. DVT firmware should never be used as factory firmware.

---

## 6. Communicating with the PCB Designer

Engineers who write firmware think in signals, timing, and protocols.
PCB designers think in nets, layers, and footprints. Translation requires precision.

**Good feedback format:**

```
Net: SPI_SD_CS (GPIO5 on ESP32 → SD card pin 1)
Problem: CS line shows ~1.2V at idle (should be 3.3V HIGH = deselected)
         SD responds to SPI traffic intended for IMU
Suspected cause: CS line open-drain path, or trace shorted to adjacent net
Requested check: Measure continuity between SD_CS and IMU_CS in schematic
                 Verify pull-up resistor value and placement for SD_CS
```

**Information that helps the PCB designer:**
- The pin name on both the MCU and the peripheral
- The expected voltage level at rest
- The measured voltage level on the physical board
- Whether the behavior is "always wrong" or "intermittent"
- The schematic net name (not just the GPIO number)

**Information that does not help:**
- "The SD card doesn't work"
- "Something is wrong with SPI"
- "Try changing the firmware"

---

## 7. Version Traceability

Every firmware binary that leaves the bench must be traceable to:

1. A Git commit hash
2. A PCB hardware revision
3. A device serial number or MAC address

```c
// Emitted at boot in every firmware
DEBUG_LOG("[BOOT] FW: %s | HW: %s | MAC: %s",
          FIRMWARE_VERSION,   // from SystemConstants.h, e.g. "dvt-1.4.2"
          HARDWARE_REVISION,  // e.g. "rev1.1"
          deviceMac);
```

Without this, debugging a field failure is guesswork. With it, you can determine
whether the issue exists in the current firmware, was already fixed, or is hardware-specific.
