# AI-Assisted Embedded Firmware Workflow

How I integrate Claude Code and AI agents into embedded firmware development
without letting AI-generated code skip physical hardware validation.

---

## 1. Core Principle

> AI writes or edits. Engineer validates physically.

AI tools are genuinely useful for:
- Spec generation and review
- Code review and pattern auditing
- Debugging hypothesis generation
- Documentation drafting
- Test invariant design

AI tools cannot replace:
- Physical hardware bring-up
- Serial log validation on real sensors
- DVT pass/fail verdict
- Judgment calls about hardware-dependent behavior

The workflow below keeps these roles explicit.

---

## 2. Spec-Driven Development

Before any code is written, the specification exists in structured documents:

```
.kiro/specs/<feature>/
  requirements.md   — what the system must do (user-visible behavior)
  design.md         — architecture, state machines, data models, interfaces
  tasks.md          — implementation checklist (generated from design)
```

**Why this matters for AI-assisted work:**

When you ask an AI coding tool to implement a feature from a clear spec, the output is
auditable against the spec. When you ask it to "add OTA support" with no spec, the output
reflects the AI's training distribution, not your hardware constraints.

The spec is what makes AI-generated code reviewable by a domain expert who didn't write
it — including yourself six weeks later.

### Example: BLE Provisioning Station Spec

The provisioning station design spec included:
- A 9-state state machine with explicit transition table
- Dependency injection requirements (no global state, all hardware through ABCs)
- 20 named correctness properties that must be verified by property-based tests
- Module dependency rules (which modules may import which)

With this spec, Claude Code produced an implementation that was directly auditable
against the design document. Deviations from the spec were visible as diffs, not
hidden inside generated boilerplate.

---

## 3. Agent Roles

I use Claude Code with a set of specialized agents. Each agent has a defined role
in the development pipeline.

| Agent role | Function | When to invoke |
|---|---|---|
| **Code Reviewer** | Audits diffs for correctness, safety, edge cases | Before any PR merge |
| **Evidence Validator** | Validates serial log evidence against claimed test verdicts; defaults to skeptical verdict | After any DVT test run |
| **Firmware Specialist** | Architecture decisions, driver patterns, FreeRTOS design | Feature design, bring-up debugging |
| **Technical Writer** | Documentation drafts from code and context | After feature stabilization |

### Phase separation rule

Never combine audit, implementation, and verification in a single agent call.
Run them in sequence:

```
1. Audit / inspection   → Code Reviewer + Evidence Validator
2. Implementation       → Direct model or Firmware Specialist agent
3. Verification         → Evidence Validator (on actual hardware serial output)
```

Collapsing these phases produces outputs that are internally consistent but
not validated against physical reality.

---

## 4. Workflow: Feature Branch to Merge

```
┌──────────────────────────────────────────────────────────┐
│  1. SPEC                                                  │
│     requirements.md + design.md written or reviewed      │
│     before any code                                       │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│  2. IMPLEMENT                                             │
│     Feature branch: feature/<name>                        │
│     Claude Code generates or assists with implementation  │
│     Engineer reviews every diff before commit             │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│  3. AI CODE REVIEW                                        │
│     Code Reviewer agent audits the diff                   │
│     Scope: specific files or modules only — never whole   │
│     repo unscoped                                         │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│  4. PHYSICAL VALIDATION (gates merge)                     │
│     DVT checklist executed on real hardware               │
│     Serial log evidence captured                          │
│     Evidence Validator reviews log against verdicts       │
│     FAILs and BLOCKEDs documented before merge            │
└──────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│  5. MERGE                                                 │
│     PR with commit hash, DVT checklist link, verdict      │
│     Only after hardware validation gate passed            │
└──────────────────────────────────────────────────────────┘
```

The physical validation step is not optional. A PR that passes AI code review
but has not been run on hardware is not merge-ready.

---

## 5. How to Avoid Blindly Trusting AI-Generated Code

### Problem patterns

**AI hallucination in embedded code:**
- Correct-looking API calls that don't match the actual library version.
- Plausible-sounding timing constants that are wrong for the specific sensor.
- Generic patterns that are correct for most MCUs but broken for your specific chip variant.
- Confident explanations of behavior that contradict the datasheet.

**Mitigation practices:**

1. **Read every diff before committing.** "Claude wrote it" is not a substitute for understanding what was written.

2. **Verify against the datasheet for every driver change.** If AI code touches SPI configuration, IMU register writes, or GPIO modes — cross-check against the datasheet or SDK reference, not just the AI's explanation.

3. **Ask the AI to explain its own code.** If it can't explain *why* a register value is correct, the value is suspect.

4. **Use a dedicated review agent on serial log output.** Its role is to be skeptical — it defaults to "NEEDS WORK" rather than approving based on surface-level consistency.

5. **Never promote PENDING to PASS without physical evidence.** AI tools can confidently predict that a test will pass. That confidence is not evidence.

---

## 6. Prompt Discipline for Firmware Tasks

### Scope narrowly

```
// Too broad — produces generic patterns
"Review the BLE code"

// Correct — scoped and actionable
"Review BLEManager.cpp lines 190–215.
 Focus: does the START_PENDING → SCANNING transition include a settle delay?
 Does the scan use a single blocking getResults() call or a chunked loop?
 Return: verdict + specific line numbers for any issues."
```

### State the hardware constraint

AI tools don't know your board. Tell them:

```
"This runs on ESP32 Arduino core with NimBLE-Arduino.
 BLE host task runs on Core 0. Application task on Core 1.
 The NimBLE host stack corrupts on back-to-back gap_disc start/stop
 without a settle period. Identify any scan start calls that lack this guard."
```

### Separate implementation from verification

```
// Implement
"Add a START_PENDING state with 50ms settle before SCANNING.
 No other changes."

// Verify (separate call — evidence validator role)
"Here is the serial log from running this code.
 Confirm: no Guru Meditation on Core 0. BLE scan starts after pending state."
```

---

## 7. Git Discipline in AI-Assisted Workflows

AI tools can generate large diffs quickly. This creates pressure to commit large
batches without careful review. Resist this.

**Commit discipline:**
- One logical change per commit.
- Commit message explains *why*, not just *what*.
- Never commit a change you don't understand.
- Feature branch → PR → physical validation → merge. No shortcuts.

**Branch naming:**
```
feature/ble-scan-settle-delay
feature/ota-nvs-persistence
fix/nimble-core0-crash
dvt/product-sensor-validation
```

**Checkpoint commits during DVT:**
```
git commit -m "dvt checkpoint: BLE scan state machine — pre-settle-delay test"
```
This gives you a reproducible "before" state if a test uncovers a regression.

---

## 8. What AI Agents Are Actually Good For

Honest assessment from production use:

| Task | AI usefulness | Caveat |
|---|---|---|
| Spec document drafting | High | Requires engineer-provided hardware constraints |
| FreeRTOS task skeleton | High | Verify stack sizes and priorities for your product |
| NDJSON event schema design | High | Verify field names against backend contract |
| Driver code for common peripherals (MPU6050, DS3231, INA219) | Medium | Cross-check register values against datasheet |
| Debugging hypothesis generation | High | Hypotheses must be physically tested |
| Serial log analysis | High | A skepticism-first review agent is useful for structured verdict validation |
| Property-based test design | High | Verify invariants match actual firmware behavior |
| Predicting hardware behavior | Low | Hardware always wins; test physically |
| Deciding if a test passes | None | Only serial log evidence + physical observation counts |

---

## 9. The "AI Confidence ≠ Physical Truth" Rule

AI language models are trained to produce confident, coherent text. They apply this
tendency even when making claims about hardware behavior.

A model that says "this code correctly handles the NimBLE scan restart" is making a
claim based on training data patterns, not on running your firmware on your board.

A dedicated review agent with a skepticism-first prompt — defaulting to "NEEDS WORK"
and requiring serial log evidence before approving a verdict — acts as a deliberate
counterweight to the model's natural confidence bias.

In practice: **if you can't show serial log evidence, the test verdict is PENDING.**
