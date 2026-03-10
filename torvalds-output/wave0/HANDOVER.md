# Wave 0 — HANDOVER

## Date: 2026-03-11
## Status: COMPLETE ✅

---

## Agents & Output

| Agent | Role | Output | Size |
|-------|------|--------|------|
| 0A | Invariant Hunter | `0A-invariants.md` | 29 KB |
| 0B | Data Model Archaeologist | `0B-data-model.md` | 38.8 KB |
| 0C | Behavioral Semanticist | `0C-behavioral.md` | 37.8 KB |

**Total analysis: 105.6 KB**

---

## Key Findings

### 0A — Invariant Hunter (47 invariants identified)
- **3 PROVEN** by logical necessity
- **31 AXIOMATIC** (sound design choices)
- **11 FRAGILE** with identified failure modes
- **2 FALSE** as currently stated
- **Most critical fragility:** Cross-tenant RLS pattern (I-37) creates security gap where application bugs can bypass grants checks within permitted tenants
- **Most dangerous assumption:** Metatype self-reference bootstrap (I-13) assumes PostgreSQL FK constraint ordering that may not hold across all implementations

### 0B — Data Model Archaeologist
- Spec says "7 tables" but actually defines 8 (tenants table undercounted)
- No separate `type_nodes` table — type nodes are regular `nodes` rows with `type_node_id = METATYPE_ID`
- No `audit_log` table — read audits go to stdout; mutation audits are events themselves
- FK constraints in bootstrap DDL are physically impossible (node_id not unique)
- HNSW index should be partial
- Missing CHECK constraints identified

### 0C — Behavioral Semanticist
- 15 MCP tools catalogued with full pre/post conditions and atomicity analysis
- `thought_captured` violates INV-APPEND (uses UPDATE on events table instead of append-only)
- `tenant_id` unchecked on updates — critical security gap
- Only 14 tests for 15 tools — coverage gap
- 2 critical, 6 major issues identified

---

## Cross-Cutting Themes

1. **Event primacy violation**: Multiple agents flagged `thought_captured` UPDATE as violating the append-only event store invariant (INV-APPEND). This is the single most important bug.

2. **RLS/tenant isolation gaps**: Both 0A and 0C identified that cross-tenant edge federation + the grants model creates potential security leaks if application code has bugs.

3. **Bootstrap fragility**: The metatype self-reference (`METATYPE_ID` type_node pointing to itself) creates a chicken-and-egg problem that depends on PostgreSQL-specific FK deferral behavior.

4. **Spec inconsistencies**: Table count discrepancy (7 vs 8), missing `note` type in seed data, test coverage gap (14/15 tools tested).

5. **Bitemporality complexity**: `valid_from`/`valid_to` overlap prevention depends entirely on application logic, not database constraints.

---

## Direction for Wave 1

Wave 1 agents (Reconciler, Interface Architect, Security Analyst) should:

1. **Reconciler**: Focus on resolving the contradictions found — especially the table count discrepancy, the `thought_captured` UPDATE vs append-only invariant, and the bootstrap FK impossibility.

2. **Interface Architect**: Design the MCP tool interfaces with the behavioral semantics from 0C in mind — especially atomicity boundaries and the missing `note` type.

3. **Security Analyst**: Deep-dive on the RLS gaps flagged by 0A and 0C — model the actual attack surface of cross-tenant edges + grants, and determine if the federation model is fixable or needs redesign.

---

## Process Notes

- 0A was the hardest to get running — failed 3 times via Claude Code CLI (hanging during generation). Successfully completed via OpenClaw sessions_spawn with Sonnet 4.
- 0B and 0C completed successfully via Claude Code CLI with `--dangerously-skip-permissions` pipe to file.
- All output files contain some ANSI artifacts from piped CLI output (0B, 0C). 0A is clean.
- Total wall time for Wave 0: ~3 hours (including retries)
