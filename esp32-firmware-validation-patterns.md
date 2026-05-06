# ESP32 Firmware Validation Patterns — DVT Methodology

Lessons from physical DVT (Design Validation Test) validation of ESP32-based connected products.
All examples are sanitized; no proprietary code or internal APIs are included.

---

## 1. DVT Validation Philosophy

DVT firmware exists to answer one question: **does the hardware + firmware combination behave correctly under real operating conditions?**

This is distinct from unit testing (which validates logic in isolation) and from production FCT (which validates manufacturing). DVT sits between them: real hardware, real radio, real sensors, deliberate fault injection.

Key principles:
- Every pass verdict must be backed by observable evidence (serial log lines, LED behavior, measurements).
- Tests must be reproducible from a documented starting state.
- A partially-passing test is never silently promoted to PASS.
- Hardware-dependent tests that cannot run due to board limitations are blocked — not skipped and not guessed.

---

## 2. Test Status Classification

Use a consistent status vocabulary across all checklist entries:

| Status | Meaning |
|---|---|
| `PASS` | Behavior confirmed, evidence logged |
| `FAIL` | Expected behavior did not occur; root cause identified or under investigation |
| `PASS_WITH_NOTE` | Core behavior correct, but specific detail differs from spec wording (note explains delta) |
| `PASS_WITH_HW_LIMITATION` | Pass behavior observed organically, not under controlled conditions due to hardware constraint |
| `PARTIAL_PASS` | Subset of the test's pass conditions confirmed; remaining conditions untested |
| `PENDING` | Not yet executed |
| `BLOCKED_BY_HARDWARE` | Cannot execute until hardware dependency is resolved (e.g., peripheral not powered, board revision pending) |
| `INCONCLUSIVE` | Executed, result ambiguous; needs re-run under more controlled conditions |

`BLOCKED_BY_HARDWARE` is important: it explicitly documents that the test was not forgotten — it was blocked for a stated reason. This prevents confusion at handoff and during post-DVT reviews.

---

## 3. Checklist Structure

Organize by subsystem, not by feature. Each subsystem's tests should be independently runnable.

```
## 0. Subsystem Init (Pre-check)
  0.1  IMU init
  0.2  SD init
  0.3  RTC init

## 1. [First failure mode, e.g., RTC OSF / Invalid Time]
  1.1  Test case description | Status | Pass condition

## 2. [Next subsystem failure mode]
  ...
```

Pre-checks (section 0) serve as a sanity gate. If a peripheral fails to initialize and the test plan doesn't account for graceful degradation, block dependent tests before running them.

---

## 4. Evidence Practices

For each test result, attach a mini evidence block. Do not rely on memory.

```markdown
### 1.1 Evidence Log
Serial lines observed (in order):
```
[RTC][E] RTC readDateTime() failed on init
[E][MAIN] RTC.initialize() FAILED – check DS3231 wiring.
[W][RTC] RTC read failed -> setting RTC from NTP
```
→ Firmware detected RTC read failure before WiFi/NTP started.
→ After WiFi connected, NTP recovery path ran and set RTC from network time.
- No crash, no hang, no WDT reset
- IMU unaffected

**Verdict:** PASS_WITH_NOTE — invalid-RTC-before-NTP and NTP-recovery behavior both confirmed.
```

Evidence rules:
- Quote serial lines verbatim (copy-paste, don't paraphrase).
- Annotate with `→` to explain what the log line means.
- State what was NOT observed if that's part of the pass condition.
- Note the monitor baud rate and whether a WDT reset occurred.

---

## 5. Hardware-Conditional Test Handling

Some tests require hardware that may not be available on every board revision. Handle these explicitly:

```markdown
## 5. SD Full Behavior

| # | Test | Status | Pass condition |
|---|---|---|---|
| 5.1 | Fill SD to capacity, boot | BLOCKED_BY_HARDWARE | Serial: SD full warning; no crash |
| 5.2 | Trigger events while full | BLOCKED_BY_HARDWARE | No new writes; upload still reads existing data |

**Hardware notes:** SD card not powered on this board revision. SD-absent recovery
was observed organically (mount failures handled gracefully; no crash, no WDT).
SD-absent path classified PASS_WITH_HW_LIMITATION.
```

When a test is blocked:
- State the blocking condition explicitly.
- Note any organic (incidental) evidence that partially validates the path.
- Do not promote organic evidence to a controlled PASS.

---

## 6. BLE Scan Crash Pattern (and Diagnosis Workflow)

A recurring DVT failure class on ESP32: **BLE scan restart triggers a Core 0 Guru Meditation.**

Symptoms:
```
[I][BLE] Scan window complete, no known beacons found
[I][BLE] Pending scan request -> START_PENDING
[I][BLE] Scan window started
Guru Meditation Error: Core 0 panic'd (Unhandled debug exception)
Debug exception reason: BREAK instr
Backtrace corrupted
```

**Root cause pattern:** The NimBLE host task runs on Core 0. Issuing a scan start too quickly after a scan stop (e.g., via a chunked scan loop) can corrupt the NimBLE host stack through repeated `ble_gap_disc()` start/stop cycles before the host fully quiesces.

**Diagnosis checklist:**
1. Is the crash on Core 0 (NimBLE host) or Core 1 (application task)?
2. Is the BLE state machine using a pending-request path that allows immediate restart?
3. Is the scan implemented as a chunked loop (`getResults(500ms) × N`) or a single blocking call?
4. Is there a settle delay between scan stop and next scan start?

**Fix pattern:**
- Use a `START_PENDING` state with a fixed settle delay (e.g., 50 ms) before issuing the next scan start. This allows the NimBLE host task to fully quiesce.
- Replace chunked scan loops with a single blocking `getResults(windowMs)` call per scan window.
- Log the FreeRTOS stack high-water mark for the BLE task after each window. Values below ~1500 words on a 8192-word stack are an early warning.

---

## 7. Stack High-Water Mark Monitoring

Add stack HWM logging to every FreeRTOS task during DVT:

```c
// In the BLE task's main loop, after each scan window:
UBaseType_t hwm = uxTaskGetStackHighWaterMark(NULL);
DEBUG_LOG("[BLE] scan task HWM: %u words", hwm);
```

Thresholds (calibrate per task):
- **> 30% remaining**: healthy
- **10–30% remaining**: investigate — is stack growth bounded?
- **< 10% remaining** (e.g., < 800 words on an 8192-word stack): increase stack allocation before shipping

HWM logging costs nothing in production and eliminates one entire class of hard-to-reproduce stack overflow crashes.

---

## 8. Git Checkpoint / Branching Strategy

Before a DVT run, cut a named branch and commit:

```
feature/<subsystem-change>
  └── dvt checkpoint: <commit hash>
```

This gives you a reproducible "last known state" if the test uncovers a regression. If a fix is needed mid-DVT:

1. Fix on the feature branch.
2. Re-run only the tests affected by the fix.
3. Annotate the checklist with the commit hash for traceability: `Fixed in dvt a3b4c18`.

Never run DVT against an uncommitted working tree — "it was working before I changed X" is not evidence.

---

## 9. WiFi ERROR Recovery Validation

A class of tests that is easy to skip and expensive to miss at customer launch.

Validation procedure:
1. Configure wrong credentials for all WiFi profiles → boot → confirm device reaches ERROR state.
2. Wait through the full cooldown window → confirm reconnect attempt starts (no manual intervention needed).
3. In ERROR state, push correct credentials via BLE → confirm immediate exit from ERROR, no cooldown wait.
4. Flap AP power repeatedly → confirm no reconnect storm; backoff respected; device stays responsive.

Serial evidence to collect: WiFi state transitions, backoff timestamps, retry reason codes.

---

## 10. Pass/Fail Summary Template

At the end of a DVT run, summarize by section:

```markdown
## DVT Summary — [Product] [Branch] [Date]

| Section | Tests | PASS | FAIL | PENDING | BLOCKED |
|---|---|---|---|---|---|
| 0. Subsystem Init | 3 | 2 | 0 | 0 | 1 |
| 1. RTC / Timestamp | 4 | 1 | 1 | 2 | 0 |
| 2. WiFi Recovery | 4 | 0 | 0 | 4 | 0 |
| ... | | | | | |

### Open FAILs (must close before merge):
- 1.2 BLE scan crash under no-WiFi + pending restart [FAIL] — see issue #XX

### Open PENDINGs (must close before production):
- 2.x WiFi ERROR recovery — blocked by test AP availability

### BLOCKED items (hardware dependency):
- 5.x SD Full — SD not powered on this board revision; retest on DVT-2 board
```

This format survives handoffs and is a direct input to the PR checklist before merging DVT firmware to `main`.
