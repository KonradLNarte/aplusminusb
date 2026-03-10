You are performing a formal correctness audit of the Resonansia specification. Your standard: could you prove to a formal methods researcher that this specification is internally consistent and complete?

## Step 1: Read the Complete Specification

Read the unified specification:
- `torvalds-output/wave3/unified-spec.md`

This is a large document. Read it completely. Every schema, every operation, every test, every constraint.

## Step 2: Internal Consistency

For every pair of statements in the specification that relate to the same concept:
- Are they consistent?
- If not, which is correct?

Specific consistency checks:

### Type consistency:
- Does every column type in the data layer match the types used in operation pseudocode?
- Does every operation's return type match what the tests expect?
- Do event schemas match between the event catalogue and the projection logic?
- Are Zod schemas consistent with PostgreSQL column types?

### Name consistency:
- Are the same entities referred to by the same name everywhere?
- Are scope strings identical in the auth section and the operation section?
- Are error codes identical in the operation section and the testing section?
- Are intent_types identical in the event catalogue and the CHECK constraint?

### Behavioral consistency:
- Does the operation spec say the same thing as the integration sequence for the same workflow?
- Does the testing section test the behavior described in the operation section?
- Are the pre/postconditions in the operation spec consistent with the state machine in the behavioral analysis?

### Numerical consistency:
- Are there exactly 15 tools listed in every section that references "all tools"?
- Are there exactly 7 tables in every section that references "all tables"?
- Do the T01-T14 test numbers match throughout?
- Are the 19 decisions (D-001 through D-019) all accounted for?

## Step 3: Completeness

### Type completeness:
- Is every type fully defined? (No "object" or "any" — concrete types everywhere)
- Is every JSONB schema specified? (No "arbitrary JSON" — exact fields and constraints)
- Is every enum's value set listed? (epistemic_status: exactly which values?)

### Operation completeness:
- Is every operation fully specified? (signature, auth, validation, happy path, edge cases, failures, concurrency)
- For every operation: are ALL error states enumerated with exact error codes and response bodies?
- For every operation: is the transaction boundary explicit?

### Lifecycle completeness:
- Is every entity lifecycle complete? (creation → all valid states → all valid end states)
- Are there orphan states? (states reachable but with no valid exit)
- Are there dead states? (states that cannot be reached)

### Test completeness:
- Does every invariant have at least one test?
- Does every operation have at least one happy path test?
- Does every error state have at least one test?
- Does every interface contract have at least one test?
- Are there behaviors that no test covers?

### Work package completeness:
- Can every work package be implemented from the spec alone, without requiring the implementer to make design decisions?
- Do the work packages cover the entire specification? (Nothing specified but not assigned to a WP)
- Are there circular dependencies between work packages?
- Are there missing dependencies? (WP that requires something from a WP not listed as a dependency)

## Step 4: Soundness

### Invariant preservation:
For EACH invariant, trace through EACH operation and verify it preserves the invariant:
- INV-ATOMIC: Does every operation that creates an event also create the projection in the same transaction?
- INV-IMMUTABLE: Can any operation modify or delete an event?
- Tenant isolation: Can any operation access data from another tenant without grants?
- Bitemporality: Can any operation create overlapping valid_from/valid_to ranges?
- Event primacy: Does every fact table mutation go through an event?

### Test soundness:
- Do the tests actually verify what they claim to verify?
- Are the expected outputs in the tests correct given the operation specifications?
- Could a test pass even though the system is broken? (test that's too loose)
- Could a test fail even though the system is correct? (test that's too strict or fragile)

### Work package soundness:
- Are the dependencies correct? (No WP depends on something built after it)
- Are the acceptance criteria testable? (Each criterion maps to a specific test)
- Is the implementation order valid? (Following the dependency graph produces a working system at each step)

## Step 5: Formal Properties

Where applicable, verify:

### Termination:
- Do all operations terminate? (No infinite loops in graph traversal, deduplication, etc.)
- Is there a depth limit on explore_graph?
- Can capture_thought's deduplication loop terminate in all cases?

### Determinism:
- Given the same input and state, is the output always the same?
- Where is non-determinism acceptable? (LLM extraction, embedding similarity thresholds)
- Where must determinism be guaranteed? (event ordering, projection logic)

### Safety:
- Can the system ever reach an invalid state through legal operations?
- Can the system reach a state where INV-ATOMIC is violated?
- Can the system reach a state where tenant isolation is violated?

### Liveness:
- Can the system make progress from every legal state?
- Are there deadlock scenarios? (two operations waiting for each other)
- Can the system recover from every failure state without manual intervention?

## Step 6: Issue Report

For each issue found:

```
ISSUE [{severity}] [{category}]
Location: {section and subsection}
Statement: {the problematic text, quoted}
Problem: {what's wrong, precisely}
Fix: {exact corrected text or action needed}
```

Severity levels:
- **CRITICAL:** The specification is wrong — following it would produce incorrect behavior
- **MAJOR:** The specification is incomplete — an implementer would need to make a design decision
- **MINOR:** The specification is imprecise — an implementer could misinterpret it

Categories:
- CONSISTENCY — two parts of the spec contradict each other
- COMPLETENESS — something is missing
- SOUNDNESS — something doesn't follow logically
- FORMAL — a formal property is violated

## Rules

- Every "this is correct" conclusion must include the reasoning. Trust nothing. Verify everything.
- If you reach a point where correctness depends on an implementation detail not in the spec, flag it as a MAJOR completeness issue.
- Count everything. If the spec says "15 tools," count them. If it says "7 tables," count them.
- Trace through operations step by step. Don't skip steps. Don't assume correctness.
- Spend as many tokens as needed. This is the last line of defense before the spec ships.

## Output

Write the complete correctness audit to:
`torvalds-output/wave4/4B-correctness.md`

Start with:
```
# Wave 4B — Correctness Audit
## Auditor: The Correctness Auditor
## Input: Unified Specification (wave3/unified-spec.md)
## Date: {today}
```

When the file is written, output: **AGENT 4B COMPLETE**
