You are writing the definitive data layer specification for Resonansia. This must be so precise that an implementation built solely from this document would be correct. You have the full architectural foundation from three independent analyses that have been reconciled.

## Step 1: Read the Architectural Foundation

Read ALL THREE Wave 1 outputs:

- `torvalds-output/wave1/1A-reconciler.md` — The unified foundation: resolved invariants, corrected data model assessment, confirmed operation set, open questions
- `torvalds-output/wave1/1B-interfaces.md` — Interface contracts: every boundary, every contract, coupling analysis
- `torvalds-output/wave1/1C-security.md` — Security analysis: threat model, authorization model, data integrity under failure

Also read the original data model spec for reference (your output should SUPERSEDE this):
- `data-model.md` — Current table schemas, indexes, RLS policies
- `constraints.md` — System invariants

## Step 2: Write the Complete Data Layer Specification

### For each entity/table (events, nodes, edges, grants, type_nodes, blobs, audit_log):

1. **Schema** — every field, its exact PostgreSQL type, constraints, default value, nullability, indexing requirements. Be EXACT:
   - Use concrete types: `UUID`, `TIMESTAMPTZ`, `JSONB`, `TEXT`, `BOOLEAN`, `VECTOR(1536)`, `SMALLINT`
   - Specify every constraint: `NOT NULL`, `CHECK(...)`, `UNIQUE(...)`, `REFERENCES ...`
   - Specify defaults: `DEFAULT gen_uuidv7()`, `DEFAULT now()`, `DEFAULT false`

2. **Invariants** — constraints that must hold across all instances at all times. Reference the proven invariants from Wave 0/1:
   - Which invariants does this table enforce at the schema level?
   - Which invariants require application-level enforcement?
   - Are there invariants that SHOULD be schema-level but currently aren't?

3. **Lifecycle** — creation, mutation, deletion:
   - Creation: what triggers it, what fields are set, what event intent_type produces it
   - Mutation: which fields can change, under what conditions, what triggers the change
   - Deletion/archival: soft vs hard delete, cascading effects, what happens to related entities
   - For each lifecycle stage: the exact SQL (INSERT/UPDATE) that implements it

4. **Temporal behavior** — bitemporality:
   - `valid_from` / `valid_to` semantics: what do they represent for this table?
   - `recorded_at` semantics: system time, from UUIDv7 extraction or explicit
   - How corrections work: new version with adjusted valid_from/valid_to
   - How as-of queries work: `WHERE valid_from <= $time AND (valid_to IS NULL OR valid_to > $time)`
   - EXCLUDE constraint: how it prevents overlapping temporal ranges

5. **Relationships** — every foreign key, every join path:
   - Cardinality (1:1, 1:N, N:M)
   - Required vs optional
   - Cascade rules (ON DELETE, ON UPDATE)
   - Cross-table consistency: what happens if a referenced entity is soft-deleted?

6. **RLS Policies** — exact SQL for every Row-Level Security policy:
   - Policy name, operation (SELECT/INSERT/UPDATE/DELETE), USING clause, WITH CHECK clause
   - How `app.tenant_ids` (SET LOCAL) interacts with the policy
   - Cross-tenant access: how grants are consulted

7. **Migration** — exact SQL to create this table, including:
   - CREATE TABLE statement
   - CREATE INDEX statements
   - CREATE POLICY statements
   - ALTER TABLE for constraints added after creation
   - Order of execution (dependency-aware)

### For the data layer as a whole:

1. **Consistency guarantees**:
   - Transaction isolation level (READ COMMITTED is Supabase default — is this sufficient?)
   - INV-ATOMIC enforcement: event + projection in one transaction
   - Cross-table consistency: how is it maintained?

2. **Indexing strategy**:
   - Which queries must be fast? (list the query patterns from each tool)
   - What indexes support them? (B-tree, GIN on JSONB, HNSW on vectors)
   - Index build time and maintenance cost

3. **Scaling characteristics**:
   - Events table: append-only, grows without bound. At what size does it need partitioning?
   - Nodes table: grows with entity count. Embedding column adds ~6KB per row (1536 × 4 bytes).
   - HNSW index: rebuild time at 10K, 100K, 1M vectors?
   - What's the realistic scaling ceiling on Supabase's smallest compute addon?

4. **Bootstrap sequence**:
   - Exact order of operations for metatype self-reference
   - Idempotency: can the bootstrap run multiple times safely?
   - UUIDv7 function creation

## Rules

- A field without a type, constraint, and nullability spec is not specified.
- A relationship without cardinality and cascade rules is not specified.
- An RLS policy without exact SQL is not specified.
- If the architecture left open questions, resolve them here and document your reasoning.
- Every claim about performance must reference PostgreSQL or pgvector behavior.
- The output must be sufficient for a developer to write a complete, correct migration script without making any design decisions.
- Spend as many tokens as needed.

## Output

Write the complete data layer specification to:
`torvalds-output/wave2/2A-data-layer.md`

Start with:
```
# Wave 2A — Data Layer Specification
## Author: Data Layer Specialist
## Input: Wave 1 Architectural Foundation (3 files)
## Date: {today}
## Status: Implementation-ready
```

When the file is written, output: **AGENT 2A COMPLETE**
