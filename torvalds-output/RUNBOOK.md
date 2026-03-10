# TORVALDS EVOLUTION — CLAUDE CODE RUNBOOK

## The 30-Second Version

You have 9 spec files describing Resonansia. You will run 5 waves of Claude Code agents (14 sessions total). Each wave produces markdown files that feed the next wave. Your job: open terminals, paste prompts, wait for "AGENT COMPLETE," move to the next wave. The agents do all the thinking.

---

## Prerequisites

- **4 terminal windows** (max parallel agents in any wave)
- **Claude Code** installed and working
- All terminals `cd` to: `C:\Development\let-there-be-light-20260309\aplusminusb`

## Directory Structure

```
torvalds-output/
├── RUNBOOK.md                              ← YOU ARE HERE
├── prompts/                                ← Copy-paste these into Claude Code
│   ├── wave0-0A-invariant-hunter.md
│   ├── wave0-0B-data-model-archaeologist.md
│   ├── wave0-0C-behavioral-semanticist.md
│   ├── wave1-1A-reconciler.md
│   ├── wave1-1B-interface-architect.md
│   ├── wave1-1C-security-analyst.md
│   ├── wave2-2A-data-layer.md
│   ├── wave2-2B-operation-layer.md
│   ├── wave2-2C-integration.md
│   ├── wave2-2D-testing.md
│   ├── wave3-assembler.md
│   ├── wave4-4A-elegance.md
│   ├── wave4-4B-correctness.md
│   ├── wave4-4C-implementer.md
│   └── wave4-final-integration.md
├── wave0/                                  ← Wave 0 agent outputs land here
│   ├── 0A-invariants.md
│   ├── 0B-data-model.md
│   └── 0C-behavioral.md
├── wave1/                                  ← Wave 1 agent outputs
│   ├── 1A-reconciler.md
│   ├── 1B-interfaces.md
│   └── 1C-security.md
├── wave2/                                  ← Wave 2 agent outputs
│   ├── 2A-data-layer.md
│   ├── 2B-operations.md
│   ├── 2C-integration.md
│   └── 2D-testing.md
├── wave3/                                  ← Wave 3 output
│   └── unified-spec.md
├── wave4/                                  ← Wave 4 review outputs
│   ├── 4A-elegance.md
│   ├── 4B-correctness.md
│   └── 4C-implementer.md
└── final/                                  ← Final deliverable
    └── FINAL-SPEC.md
```

## Source Material (the spec files agents will read)

| File | Size | Content |
|------|------|---------|
| `index.md` | 43KB | Main spec: system definition, invariants, gen protocol, deliverables |
| `data-model.md` | 29KB | Tables, columns, indexes, RLS, metatype bootstrap, embeddings |
| `mcp-tools.md` | 76KB | All 15 MCP tools, projections, error handling, resources, prompts |
| `auth.md` | 17KB | OAuth 2.1, JWT, scopes, grants, RLS interaction, audit |
| `federation.md` | 8KB | Cross-tenant edges, grants, projections, A2A readiness |
| `constraints.md` | 3KB | Invariants, serverless limits, cost model |
| `validation.md` | 36KB | Pettson scenario, seed data, T01-T14 test cases |
| `decisions.md` | 18KB | Decision log D-001 through D-019 |
| `tech-profile.md` | 3KB | Supabase tech profile, runtime, dependencies |

**Total: ~233KB of specification**

---

## HOW TO RUN EACH WAVE

### For each agent in a wave:

1. Open a terminal window
2. `cd C:\Development\let-there-be-light-20260309\aplusminusb`
3. Run `claude`
4. Open the prompt file listed below (e.g., `torvalds-output/prompts/wave0-0A-invariant-hunter.md`)
5. Copy the ENTIRE contents and paste into Claude Code
6. Wait until you see **"AGENT [ID] COMPLETE"** in the output
7. Verify the output file exists at the path listed below

### Parallel agents
When a wave has multiple agents, open separate terminal windows and run them simultaneously. They do not depend on each other within a wave.

### Wave gates
**DO NOT start a wave until ALL agents from the previous wave have completed and their output files exist.** Each wave reads the previous wave's outputs.

---

## WAVE 0 — FOUNDATION INTERROGATION

**Purpose:** Prove or disprove every foundational assumption in the Resonansia spec.
**Parallel agents:** 3
**Estimated time:** 15-30 min per agent (they run simultaneously)

### Step-by-step:

1. Open **3 terminal windows**
2. In each, `cd C:\Development\let-there-be-light-20260309\aplusminusb` then `claude`
3. Paste prompts:

| Terminal | Prompt file | Output file |
|----------|-------------|-------------|
| 1 | `torvalds-output/prompts/wave0-0A-invariant-hunter.md` | `torvalds-output/wave0/0A-invariants.md` |
| 2 | `torvalds-output/prompts/wave0-0B-data-model-archaeologist.md` | `torvalds-output/wave0/0B-data-model.md` |
| 3 | `torvalds-output/prompts/wave0-0C-behavioral-semanticist.md` | `torvalds-output/wave0/0C-behavioral.md` |

### Gate check before Wave 1:
```bash
ls -la torvalds-output/wave0/
# Must see: 0A-invariants.md, 0B-data-model.md, 0C-behavioral.md
# All three must be non-empty
wc -l torvalds-output/wave0/*.md
```

---

## WAVE 1 — ARCHITECTURAL SYNTHESIS

**Purpose:** Reconcile the three foundation analyses into a coherent architecture.
**Parallel agents:** 3
**Input:** All three Wave 0 output files
**Estimated time:** 15-25 min per agent

### Step-by-step:

1. Open **3 terminal windows**
2. In each, `cd C:\Development\let-there-be-light-20260309\aplusminusb` then `claude`
3. Paste prompts:

| Terminal | Prompt file | Output file |
|----------|-------------|-------------|
| 1 | `torvalds-output/prompts/wave1-1A-reconciler.md` | `torvalds-output/wave1/1A-reconciler.md` |
| 2 | `torvalds-output/prompts/wave1-1B-interface-architect.md` | `torvalds-output/wave1/1B-interfaces.md` |
| 3 | `torvalds-output/prompts/wave1-1C-security-analyst.md` | `torvalds-output/wave1/1C-security.md` |

### Gate check before Wave 2:
```bash
ls -la torvalds-output/wave1/
# Must see: 1A-reconciler.md, 1B-interfaces.md, 1C-security.md
wc -l torvalds-output/wave1/*.md
```

---

## WAVE 2 — DEEP SPECIFICATION DEVELOPMENT

**Purpose:** Full-depth specification of every layer.
**Parallel agents:** 4
**Input:** All three Wave 1 output files
**Estimated time:** 20-35 min per agent

### Step-by-step:

1. Open **4 terminal windows**
2. In each, `cd C:\Development\let-there-be-light-20260309\aplusminusb` then `claude`
3. Paste prompts:

| Terminal | Prompt file | Output file |
|----------|-------------|-------------|
| 1 | `torvalds-output/prompts/wave2-2A-data-layer.md` | `torvalds-output/wave2/2A-data-layer.md` |
| 2 | `torvalds-output/prompts/wave2-2B-operation-layer.md` | `torvalds-output/wave2/2B-operations.md` |
| 3 | `torvalds-output/prompts/wave2-2C-integration.md` | `torvalds-output/wave2/2C-integration.md` |
| 4 | `torvalds-output/prompts/wave2-2D-testing.md` | `torvalds-output/wave2/2D-testing.md` |

### Gate check before Wave 3:
```bash
ls -la torvalds-output/wave2/
# Must see: 2A-data-layer.md, 2B-operations.md, 2C-integration.md, 2D-testing.md
wc -l torvalds-output/wave2/*.md
```

---

## WAVE 3 — SPECIFICATION ASSEMBLY

**Purpose:** Merge the four spec parts into one unified document.
**Sequential:** 1 agent only (it needs all Wave 2 outputs)
**Input:** All four Wave 2 output files + Wave 1 outputs for cross-reference
**Estimated time:** 25-40 min

### Step-by-step:

1. Open **1 terminal window**
2. `cd C:\Development\let-there-be-light-20260309\aplusminusb` then `claude`
3. Paste prompt from: `torvalds-output/prompts/wave3-assembler.md`
4. Output: `torvalds-output/wave3/unified-spec.md`

### Gate check before Wave 4:
```bash
ls -la torvalds-output/wave3/
# Must see: unified-spec.md
wc -l torvalds-output/wave3/unified-spec.md
# Should be substantial (1000+ lines)
```

---

## WAVE 4 — THE TORVALDS REVIEW

**Purpose:** Three-angle adversarial review, then final integration.
**Phase 1:** 3 parallel reviewers
**Phase 2:** 1 final integrator (after all 3 reviewers finish)
**Input:** unified-spec.md from Wave 3
**Estimated time:** 15-25 min per reviewer, then 20-35 min for integrator

### Phase 1 — Reviews (parallel):

1. Open **3 terminal windows**
2. In each, `cd C:\Development\let-there-be-light-20260309\aplusminusb` then `claude`
3. Paste prompts:

| Terminal | Prompt file | Output file |
|----------|-------------|-------------|
| 1 | `torvalds-output/prompts/wave4-4A-elegance.md` | `torvalds-output/wave4/4A-elegance.md` |
| 2 | `torvalds-output/prompts/wave4-4B-correctness.md` | `torvalds-output/wave4/4B-correctness.md` |
| 3 | `torvalds-output/prompts/wave4-4C-implementer.md` | `torvalds-output/wave4/4C-implementer.md` |

### Gate check before Final Integration:
```bash
ls -la torvalds-output/wave4/
# Must see: 4A-elegance.md, 4B-correctness.md, 4C-implementer.md
wc -l torvalds-output/wave4/*.md
```

### Phase 2 — Final Integration (sequential):

1. Open **1 terminal window**
2. `cd C:\Development\let-there-be-light-20260309\aplusminusb` then `claude`
3. Paste prompt from: `torvalds-output/prompts/wave4-final-integration.md`
4. Output: `torvalds-output/final/FINAL-SPEC.md`

### Final verification:
```bash
ls -la torvalds-output/final/
# Must see: FINAL-SPEC.md
wc -l torvalds-output/final/FINAL-SPEC.md
# This is your deliverable
```

---

## PROGRESS TRACKER

Copy this checklist and check off as you go:

```
WAVE 0 — FOUNDATION INTERROGATION
  [ ] 0A — Invariant Hunter          → torvalds-output/wave0/0A-invariants.md
  [ ] 0B — Data Model Archaeologist  → torvalds-output/wave0/0B-data-model.md
  [ ] 0C — Behavioral Semanticist    → torvalds-output/wave0/0C-behavioral.md
  [ ] GATE CHECK: All 3 files exist and are non-empty

WAVE 1 — ARCHITECTURAL SYNTHESIS
  [ ] 1A — Reconciler                → torvalds-output/wave1/1A-reconciler.md
  [ ] 1B — Interface Architect       → torvalds-output/wave1/1B-interfaces.md
  [ ] 1C — Security Analyst          → torvalds-output/wave1/1C-security.md
  [ ] GATE CHECK: All 3 files exist and are non-empty

WAVE 2 — DEEP SPECIFICATION
  [ ] 2A — Data Layer Spec           → torvalds-output/wave2/2A-data-layer.md
  [ ] 2B — Operation Layer Spec      → torvalds-output/wave2/2B-operations.md
  [ ] 2C — Integration Spec          → torvalds-output/wave2/2C-integration.md
  [ ] 2D — Testing Spec              → torvalds-output/wave2/2D-testing.md
  [ ] GATE CHECK: All 4 files exist and are non-empty

WAVE 3 — ASSEMBLY
  [ ] 3  — Assembler                 → torvalds-output/wave3/unified-spec.md
  [ ] GATE CHECK: File exists and is 1000+ lines

WAVE 4 — REVIEW & FINAL
  [ ] 4A — Elegance Critic           → torvalds-output/wave4/4A-elegance.md
  [ ] 4B — Correctness Auditor       → torvalds-output/wave4/4B-correctness.md
  [ ] 4C — Implementer's Advocate    → torvalds-output/wave4/4C-implementer.md
  [ ] GATE CHECK: All 3 review files exist
  [ ] FINAL — Integration            → torvalds-output/final/FINAL-SPEC.md
  [ ] DONE: Final spec delivered
```

---

## SUMMARY

| Wave | Agents | Parallel? | Input | Output |
|------|--------|-----------|-------|--------|
| 0 | 3 | Yes | 9 spec files (233KB) | wave0/ (3 files) |
| 1 | 3 | Yes | wave0/ (3 files) | wave1/ (3 files) |
| 2 | 4 | Yes | wave1/ (3 files) | wave2/ (4 files) |
| 3 | 1 | No | wave2/ (4 files) | wave3/ (1 file) |
| 4 | 3+1 | Yes then No | wave3/ (1 file) | final/ (1 file) |

**Total sessions:** 14 Claude Code instances
**Max concurrent:** 4 (Wave 2)
**Your job:** Open terminals, paste prompts, check gate, repeat.
**The agents' job:** Everything else.

---

## TROUBLESHOOTING

**Agent seems stuck or running too long (>45 min):**
The prompt tells it to be exhaustive. This is normal for deep analysis. Only intervene if it's genuinely looping.

**Agent didn't write the output file:**
Scroll through the output. If it produced the analysis but forgot to write the file, copy its analysis text and manually save it to the expected path. Then proceed.

**Output file is tiny (<50 lines):**
The agent may have hit a context issue. Re-run that specific agent. Previous agents' outputs on disk are unaffected.

**Wave 1+ agent says it can't find Wave 0 files:**
Verify the files exist at the exact paths. Ensure the terminal is `cd`'d to the correct directory.

**Want to re-run just one agent:**
Each agent is independent within its wave. Re-run the specific agent's prompt in a fresh Claude Code session. It will overwrite its output file.
