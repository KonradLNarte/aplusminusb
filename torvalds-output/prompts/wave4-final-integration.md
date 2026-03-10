You are performing the final integration of the Resonansia specification. It has been through a rigorous 5-wave, 14-agent evolution process. Three independent reviewers have now attacked it from different angles. Your job: apply every fix, every improvement, every clarification, and produce the FINAL specification.

## Step 1: Read Everything

### The specification to correct:
- `torvalds-output/wave3/unified-spec.md` — The assembled specification

### The three reviews:
- `torvalds-output/wave4/4A-elegance.md` — Elegance review: abstraction quality, naming, conceptual integrity, longevity
- `torvalds-output/wave4/4B-correctness.md` — Correctness audit: consistency, completeness, soundness, formal properties, issue report
- `torvalds-output/wave4/4C-implementer.md` — Implementer's review: technology assessment, implementation risks, developer experience, improvement report

### Original context (for reference):
- `index.md` — Original system definition and architectural principles
- `decisions.md` — Original decision log

Read ALL of these completely before starting.

## Step 2: Triage All Review Findings

Create a master list of ALL findings from all three reviewers:

### From 4A (Elegance):
- Every dimension graded below EXEMPLARY
- Every specific improvement recommended

### From 4B (Correctness):
- Every CRITICAL issue (spec is wrong)
- Every MAJOR issue (spec is incomplete)
- Every MINOR issue (spec is imprecise)

### From 4C (Implementer):
- Every P0 blocker
- Every P1 significant issue
- Every P2 improvement

Sort all findings by priority:
1. CRITICAL correctness issues and P0 blockers — MUST fix
2. MAJOR correctness issues and P1 significant — SHOULD fix
3. Elegance issues graded DEFICIENT — SHOULD fix
4. MINOR correctness issues and P2 improvements — FIX if feasible
5. Elegance issues graded ADEQUATE — CONSIDER
6. P3 nice-to-haves — INCLUDE if they don't add noise

## Step 3: Conflict Resolution

If two or more reviewers make conflicting recommendations:
1. State the conflict explicitly
2. Analyze both perspectives
3. Determine the correct resolution (or the best compromise)
4. Document the reasoning in the Decision Log

Common conflict patterns to watch for:
- Elegance says "simplify X" but Correctness says "X is needed for soundness"
- Implementer says "change this to work with Supabase" but Elegance says "keep it tech-agnostic"
- Correctness says "add more tests" but Implementer says "spec is already overwhelming"

## Step 4: Apply All Fixes

Go through the unified specification section by section. Apply:

1. **Every CRITICAL and MAJOR fix** from the Correctness Auditor — these are non-negotiable
2. **Every P0 blocker** from the Implementer — an unimplementable spec is useless
3. **Every improvement from the Elegance Critic** where the grade was DEFICIENT — quality matters
4. **Every P1 issue** from the Implementer — prevent likely implementation errors
5. **Naming improvements** from the Elegance Critic — clarity is free
6. **Every MINOR fix** from the Correctness Auditor — precision matters
7. **P2 improvements** from the Implementer — smoother development experience
8. **Elegance improvements** graded ADEQUATE — raise to at least SOLID

## Step 5: Produce the FINAL Specification

The output document must have this structure:

```markdown
# RESONANSIA — Final Specification
## Produced by: Torvalds Evolution Process (5-wave, 14-agent)
## Status: REVIEWED AND CORRECTED
## Version: 2.0-final
## Date: {today}

## Evolution Provenance
| Wave | Agents | Purpose | Key Findings |
|------|--------|---------|--------------|
| 0 | 3 (Invariant Hunter, Data Model Archaeologist, Behavioral Semanticist) | Foundation interrogation | {summary} |
| 1 | 3 (Reconciler, Interface Architect, Security Analyst) | Architectural synthesis | {summary} |
| 2 | 4 (Data Layer, Operations, Integration, Testing) | Deep specification | {summary} |
| 3 | 1 (Assembler) | Unification | {summary} |
| 4 | 3+1 (Elegance, Correctness, Implementer + Final Integration) | Review & correction | {summary} |

## 1. Overview
### 1.1 Purpose
### 1.2 Core Invariants (proven and classified)
### 1.3 Architecture Diagram

## 2. Data Layer (corrected)

## 3. Operations (corrected)

## 4. Integration & Orchestration (corrected)

## 5. Security (corrected)

## 6. Testing & Verification (corrected)

## 7. Work Packages (implementation plan)

## 8. Tech Profile

## 9. Open Questions (be honest — what couldn't be resolved?)

## 10. Decision Log (complete, including evolution decisions)

## 11. Consistency Matrix (invariants × schemas × operations × tests)

## 12. Review Response Log
For every review finding: finding ID, reviewer, severity, resolution, section updated
```

## Step 6: Final Self-Verification

After producing the final spec, verify:

### Completeness:
- [ ] Every CRITICAL issue from 4B has been addressed
- [ ] Every MAJOR issue from 4B has been addressed
- [ ] Every P0 blocker from 4C has been addressed
- [ ] Every DEFICIENT grade from 4A has been improved
- [ ] No review finding was silently ignored

### Consistency:
- [ ] All type names are consistent throughout
- [ ] All error codes are consistent throughout
- [ ] All scope strings are consistent throughout
- [ ] All intent_types are consistent throughout
- [ ] The consistency matrix has no empty cells

### Implementability:
- [ ] A developer with the tech profile (Deno, Hono, Supabase) could implement each work package from this document alone
- [ ] No work package requires the implementer to make design decisions
- [ ] The implementation order is valid (following dependency graph)
- [ ] All environment variables and configuration are listed

### Quality:
- [ ] The document reads as if designed by one mind
- [ ] Naming is clear and consistent
- [ ] The document is self-contained

## Rules

- Do not skip any review finding. Address every single one — even if the resolution is "no change needed" (document why).
- If two review findings conflict, resolve the conflict explicitly and document the reasoning.
- The final document must be complete enough that any competent developer could implement the system from it alone.
- The Review Response Log (section 12) is your proof that nothing was missed.
- This is the DELIVERABLE. It must be as close to perfect as possible.
- Spend as many tokens as needed.

## Output

Write the complete final specification to:
`torvalds-output/final/FINAL-SPEC.md`

Start with:
```
# RESONANSIA — Final Specification
## Produced by: Torvalds Evolution Process (5-wave, 14-agent)
## Status: REVIEWED AND CORRECTED
## Version: 2.0-final
## Date: {today}
```

When the file is written, output: **FINAL INTEGRATION COMPLETE — SPECIFICATION DELIVERED**
