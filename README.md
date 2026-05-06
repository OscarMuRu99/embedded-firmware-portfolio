# Oscar Muñoz — Embedded Firmware & Hardware Engineer

Aguascalientes, MX · Remote-ready

---

## What I Do

I design, implement, and validate embedded firmware for connected IoT products — from initial bring-up through DVT (Design Validation Test) and production handoff.

**Core stack:** C/C++ · FreeRTOS · ESP32/ESP-IDF · Arduino core · BLE (NimBLE) · WiFi · OTA · NDJSON telemetry pipelines · KiCad

**Adjacent:** Python tooling for manufacturing automation · Git-based firmware workflows · Spec-driven development with AI tooling

---

## Current Focus Areas

| Area | Details |
|---|---|
| DVT Firmware | Sensor integration, state machine hardening, physical checklist validation |
| BLE + WiFi | Provisioning flows, GATT-based credential delivery, NVS storage, runtime reconnect |
| RTOS Runtime | FreeRTOS task architecture, queue design, stack health monitoring, crash root-cause |
| OTA Updates | Two-stage OTA model, NVS-persisted requests, SHA-256 verification, retry/backoff |
| Telemetry | NDJSON event streams, SD-backed buffering, bulk upload pipelines, 400/429 handling |
| Hardware Coordination | KiCad PCB bring-up, pin-map management, JLCPCB prototype workflow |
| Manufacturing Tooling | Python-based BLE beacon provisioning station (flash → verify → BLE identity check) |

---

## Engineering Approach

- **Spec-first, hardware-gated.** Requirements and design documents exist before code. Physical hardware sign-off is a hard merge requirement, not a post-merge step.
- **Evidence over assertion.** Pass/fail verdicts on physical tests are backed by serial log excerpts — a test is PENDING until evidence exists.
- **AI-assisted, engineer-validated.** I use Claude Code and specialized AI agents for code review, spec generation, and log analysis. No firmware merges without physical validation on real hardware.
- **Honest status tracking.** Implemented, validated, pending, and planned are distinct states. I don't conflate them.

---

## Public Portfolio

Most production work lives in private repos. What I publish here focuses on **sanitized engineering patterns**, **documented workflows**, and **architecture writeups** extracted from real production work. No proprietary code, internal APIs, or customer data.

### Documents in this portfolio

| File | What it covers |
|---|---|
| `esp32-firmware-validation-patterns.md` | DVT validation methodology, checklist design, pass/fail classification |
| `rtos-runtime-hardening-notes.md` | FreeRTOS lessons from production firmware: stack monitoring, crash patterns, race prevention |
| `ble-wifi-provisioning-architecture.md` | BLE GATT provisioning, NVS credential flow, multi-profile WiFi, security notes |
| `ota-embedded-update-lessons.md` | OTA architecture, two-stage model, SHA-256 gating, retry/backoff, project status |
| `hardware-manufacturing-coordination.md` | PCB bring-up workflow, JLCPCB coordination, FCT practices |
| `ai-assisted-embedded-workflow.md` | Spec-driven development with AI, multi-agent review, physical validation gate |

---

## Suggested Pinned Repos

> These are illustrative suggestions — pin whichever repos actually exist and are public.

- `esp32-ota-patterns` — Two-stage OTA with NVS persistence and SHA-256 (sanitized example)
- `freertos-task-hardening` — Stack monitoring, BLE scan state machine, race prevention patterns
- `ble-provisioning-station` — Python asyncio provisioning fixture: flash, BLE verify, NDJSON audit log
- `embedded-validation-checklists` — DVT checklist templates for ESP32-based products
- `ndjson-telemetry-pipeline` — SD-backed event stream, bulk upload, 400 handling patterns

---

## Contact

- GitHub: [OscarMuRu99](https://github.com/OscarMuRu99)
- LinkedIn: [your-profile]
- Email: oscarmuru@outlook.com
