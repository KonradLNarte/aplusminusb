You are assembling a unified specification for Resonansia from four independently written parts. Your job is not just to concatenate — it is to UNIFY. Every inconsistency must be found and resolved. Every gap must be filled.

## Step 1: Read ALL Inputs

### Wave 2 outputs (the four spec parts you are assembling):
- `torvalds-output/wave2/2A-data-layer.md` — Complete data layer specification: schemas, indexes, RLS, temporal behavior, migration
- `torvalds-output/wave2/2B-operations.md` — Complete operation layer specification: all 15 tools, auth middleware, embedding pipeline
- `torvalds-output/wave2/2C-integration.md` — Integration specification: events, workflows, deployment, monitoring
- `torvalds-output/wave2/2D-testing.md` — Testing specification: invariant tests, contract tests, behavioral tests, chaos scenarios

### Wave 1 outputs (for cross-reference and conflict resolution):
- `torvalds-output/wave1/1A-reconciler.md` — The unified foundation (invariants, data model, operations)
- `torvalds-output/wave1/1B-interfaces.md` — Interface contracts
- `torvalds-output/wave1/1C-security.md` — Security analysis

### Original spec (for reference only — the Wave 2 outputs SUPERSEDE these):
- `index.md` — System definition, architectural principles
- `decisions.md` — Decision log

Read ALL of these files completely before starting assembly.

## Step 2: Consistency Audit

Read all four Wave 2 parts. Find EVERY inconsistency:

### Data ↔ Operations:
- Does the data spec (2A) define every table and column that the operations spec (2B) references?
- Does every operation in 2B produce events with intent_types that exist in 2A's CHECK constraint?
- Do the column types in 2A match the types used in 2B's pseudocode?
- Does every operation's INSERT/UPDATE in 2B match the schema defined in 2A?

### Operations ↔ Integration:
- Do the integration sequences (2C) reference operations that exist in 2B?
- Are the event schemas in 2C consistent with the events produced by operations in 2B?
- Do the projection logic descriptions in 2C match the step-by-step in 2B?
- Are the error codes consistent?

### Integration ↔ Testing:
- Do the tests in 2D cover every workflow in 2C?
- Are the test inputs consistent with the operation signatures in 2B?
- Are the expected test outputs consistent with the data schemas in 2A?

### Data ↔ Testing:
- Do the invariant tests in 2D actually verify the invariants defined in 2A?
- Do the seed data requirements in 2D match the schemas in 2A?

### Cross-cutting:
- Are type names consistent across all four parts? (entity_type vs type_node, edge vs relationship, etc.)
- Are error codes consistent? (same failure → same error code everywhere)
- Are scope names consistent? (exact scope strings referenced in operations match auth spec)
- Are UUIDv7 references consistent? (generation function, extraction of timestamp)

## Step 3: Gap Filling

For each inconsistency found:
- Determine the correct resolution (which part is right, or if both need adjustment)
- Write the corrected text
- Document why you chose this resolution

For each gap found (something not covered by any part):
- Write the missing specification
- Document what triggered the identification of the gap

## Step 4: Assembly

Produce a single, unified document with this structure:

```markdown
# RESONANSIA — Complete Specification
## Version: 2.0-evolved
## Produced by: Torvalds Evolution Process (5-wave, 14-agent)
## Status: ASSEMBLED — PENDING REVIEW

## 1. Overview
### 1.1 Purpose
One-sentence: A federated MCP server that exposes a bitemporal, event-sourced knowledge graph as AI-agent-accessible infrastructure with tenant isolation, semantic search, and temporal queries.

### 1.2 Core Invariants
Complete list from Wave 1A reconciliation, with enforcement mechanisms.

### 1.3 Architecture Diagram
Text diagram showing all components and their interfaces.

## 2. Data Layer
{from Wave 2A, corrected for consistency}
- Complete schemas for all tables
- All indexes and constraints
- All RLS policies
- Bootstrap sequence
- Scaling characteristics

## 3. Operations
{from Wave 2B, corrected for consistency}
- All 15 MCP tools fully specified
- Auth middleware
- Embedding pipeline
- capture_thought deep spec

## 4. Integration & Orchestration
{from Wave 2C, corrected for consistency}
- Event catalogue with projection logic
- Workflow sequence diagrams
- System lifecycle (startup, shutdown, health)
- Deployment topology

## 5. Security
{from Wave 1C, updated with any corrections from assembly}
- Threat model
- Authorization matrix
- Failure recovery

## 6. Testing & Verification
{from Wave 2D, corrected for consistency}
- Invariant tests
- Contract tests
- Behavioral tests
- Integration tests
- Property-based tests
- Chaos scenarios

## 7. Work Packages
### 7.1 Wave 0 — Foundation (no dependencies)
### 7.2 Wave 1 — Core (depends on Wave 0)
### 7.3 Wave 2 — Features (depends on Wave 1)
### 7.4 Wave 3 — Integration (depends on Waves 1-2)
### 7.5 Wave 4 — Hardening (depends on Wave 3)

For each work package:
- ID: WP-{number}
- Name
- Dependencies: [WP-{ids}]
- Scope: What to build
- Inputs: What you receive (which spec sections)
- Outputs: What you produce (which files/artifacts)
- Acceptance criteria: Testable conditions (reference specific tests from section 6)
- Estimated complexity: S / M / L / XL

## 8. Tech Profile
Supabase binding from tech-profile.md

## 9. Open Questions
{anything unresolved — be brutally honest}

## 10. Decision Log
{every decision made during the evolution process, with reasoning}
{include original D-001 through D-019 plus any new decisions from the evolution}

## 11. Consistency Matrix
A table showing: each invariant × which schema enforces it × which operations preserve it × which tests verify it
```

## Step 5: Self-Verification

After assembly, verify:
- [ ] Every invariant appears in the data layer, is enforced by operations, and is tested
- [ ] Every operation has a complete specification (signature, auth, validation, happy path, edge cases, failures, concurrency)
- [ ] Every interface has a contract
- [ ] Every workflow has compensation logic
- [ ] Every event intent_type has complete projection logic
- [ ] Every table has complete RLS policies
- [ ] Every test has exact expected input and output
- [ ] The work packages cover the entire specification with no gaps
- [ ] The consistency matrix has no empty cells
- [ ] Type names are consistent throughout
- [ ] Error codes are consistent throughout
- [ ] Scope strings are consistent throughout

## Step 6: Work Package Definition

Define work packages that map the spec to implementable units. Use the implementation brief from index.md (section 8.5) as a starting point, but refine based on the evolved spec.

Each work package must be:
- **Self-contained:** An implementer can build it from the spec alone
- **Testable:** Has specific acceptance criteria referencing tests from section 6
- **Dependency-aware:** Lists exactly which other WPs must be complete first
- **Sized:** S (1-2 hours), M (2-4 hours), L (4-8 hours), XL (8+ hours)

## Rules

- If you find a gap that cannot be resolved from the available material, add it to Open Questions with a clear description of what's missing.
- The final document must be SELF-CONTAINED. A reader should need nothing else to implement the system.
- Do not lose detail during assembly. If a Wave 2 part has a 50-line pseudocode block for an operation, keep it.
- The consistency matrix in section 11 is your proof of completeness. Every cell must be filled.
- Spend as many tokens as needed. This is the most important output of the entire process.

## Output

Write the complete unified specification to:
`torvalds-output/wave3/unified-spec.md`

Start with:
```
# RESONANSIA — Complete Specification
## Version: 2.0-evolved
## Produced by: Torvalds Evolution Process (5-wave, 14-agent)
## Status: ASSEMBLED — PENDING REVIEW
## Date: {today}
```

When the file is written, output: **AGENT 3 COMPLETE**
