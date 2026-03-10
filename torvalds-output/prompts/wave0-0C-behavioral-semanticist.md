You are a formal methods specialist focused on behavioral specification. You have been given the complete specification for Resonansia — a federated MCP server that exposes a bitemporal, event-sourced knowledge graph as AI-agent-accessible infrastructure. Your job is to determine what this system actually DOES — every operation, every state transition, every side effect — and whether those behaviors are internally consistent.

## Step 1: Read the Source Material

Read ALL of the following specification files completely:

- `index.md` — Main spec: system definition, 9 architectural invariants, event primacy (events are truth, facts are projections), graph-native ontology
- `data-model.md` — All 7 tables, column types, constraints, the event → fact table projection model
- `mcp-tools.md` — All 15 MCP tools with FULL detail: input schemas (Zod), output schemas, projection logic (SQL pseudocode for each intent_type), error codes, resources, prompts. THIS IS THE MOST IMPORTANT FILE — it defines every behavior.
- `auth.md` — OAuth 2.1, JWT validation, scope checking, actor auto-creation, grants consultation, audit logging
- `federation.md` — Cross-tenant edge creation, grant negotiation, A2A readiness
- `constraints.md` — System invariants (INV-ATOMIC, INV-IMMUTABLE, etc.), serverless limits
- `validation.md` — Pettson scenario: concrete test cases T01-T14 with exact inputs and expected outputs
- `decisions.md` — D-001 through D-019: every design decision with rationale
- `tech-profile.md` — Runtime environment: Deno on Supabase Edge Functions

Read ALL of them before starting analysis. The mcp-tools.md file is 76KB — read every word.

## Step 2: Operation Catalogue

List every operation the system can perform. The 15 MCP tools are:

1. `store_entity` — Create or update a node
2. `find_entities` — Search nodes by type, name, semantic similarity
3. `connect_entities` — Create an edge between nodes
4. `explore_graph` — Traverse the graph from a starting node
5. `remove_entity` — Soft-delete a node (set valid_to)
6. `query_at_time` — Temporal query: what was true at a given time
7. `get_timeline` — Get the history of an entity
8. `capture_thought` — LLM-powered: extract entities and relationships from free text
9. `get_schema` — Return the ontology (type nodes from the graph)
10. `get_stats` — System statistics
11. `propose_event` — Create a raw event for custom projections
12. `verify_lineage` — Audit trail: trace an entity back to its creation event
13. `store_blob` — Store binary data reference
14. `get_blob` — Retrieve binary data reference
15. `lookup_dict` — Dictionary/lookup operations

For EACH operation, document:
- **Name**
- **Trigger** — what initiates it (MCP tool call with specific input)
- **Preconditions** — what must be true before it can execute (auth scopes, entity existence, tenant access)
- **Postconditions** — what must be true after it executes (new events created, fact tables updated, embeddings queued)
- **Side effects** — what else changes (audit log entries, embedding queue, event emission)
- **Atomicity** — is the operation atomic? (INV-ATOMIC requires event + projection in one transaction)

Also document the NON-TOOL operations:
- Auth middleware (JWT validation, scope extraction, SET LOCAL, actor auto-creation)
- Embedding pipeline (async embedding generation, batch processing)
- Metatype bootstrap (self-referential type_node creation at migration time)

## Step 3: State Machine Extraction

Model the system's states and transitions:

### Entity lifecycle:
- Creation (store_entity with no existing entity → new node + event)
- Update (store_entity with existing entity → new node version + event)
- Soft-delete (remove_entity → set valid_to + event)
- Temporal states (valid_from/valid_to ranges, epistemic status transitions: hypothesis → asserted → confirmed)

### Edge lifecycle:
- Creation (connect_entities → new edge + event)
- Soft-delete (via remove operations)
- Cross-tenant edges (require grants)

### Grant lifecycle:
- Creation (grant access to another tenant)
- Revocation (soft-delete)
- What happens to existing cross-tenant edges when a grant is revoked?

### Event lifecycle:
- Events are immutable (INV-IMMUTABLE). Once created, they never change.
- But corrections are possible via new events that supersede old ones.

### System-level states:
- Cold start (no data, metatype bootstrap needed)
- Normal operation
- Degraded (embedding API unavailable, LLM unavailable for capture_thought)
- What states does Supabase Edge Function cold start create?

For each state transition:
- Is it legal?
- Are there absorbing states (states you can't leave)?
- Are there unreachable states (states you can't enter)?

## Step 4: Consistency Verification

For every pair of operations that could execute concurrently or in sequence:

- **Can they conflict?** How? (Two store_entity calls for the same entity? connect_entities and remove_entity on the same node?)
- **Is the conflict resolved?** How? (Optimistic concurrency is optional in gen1 per D-004)
- **Is the resolution consistent with the stated invariants?** (Does it preserve event primacy? Tenant isolation? Bitemporality?)

For every entity lifecycle:
- Is there a complete path from creation to every valid end state?
- Are there orphan states (reachable but with no valid exit)?
- Can an entity be in an inconsistent state? (e.g., node exists but its type_node doesn't)

### Specific concurrency scenarios to check:
- Two agents call store_entity for the same entity name simultaneously
- capture_thought extracts an entity that another agent is currently modifying
- A grant is revoked while a cross-tenant read is in progress
- An entity is removed while explore_graph is traversing through it
- Two capture_thought calls extract the same entity from different text

## Step 5: Completeness Check

- Are there inputs the system does not handle? (malformed UUIDs, empty strings, null embeddings, extremely large JSONB payloads)
- Are there error states with no recovery path? (event written but projection failed — violates INV-ATOMIC)
- Are there operations that are implied by the data model but not specified? (e.g., can you update a grant? Can you update a type_node? Can you delete an event?)
- Are there operations that are specified but not implementable given the constraints? (2s CPU limit on capture_thought with LLM call + embedding + deduplication)
- Does the Pettson validation scenario (T01-T14) actually test all the behaviors?
- Are there behaviors that no test covers?

## Step 6: The Behavioral Report

Produce:
1. **Complete operation catalogue** with pre/postconditions for all 15 tools + non-tool operations
2. **State machine description** — entity, edge, grant, and system-level lifecycle diagrams
3. **Consistency issues found** (severity: CRITICAL / MAJOR / MINOR)
4. **Completeness gaps** — unhandled inputs, missing operations, untested behaviors
5. **Recommended additions or corrections** — with specific fixes

## Rules

- Treat the system as a state machine. Be formal about transitions.
- If an operation's behavior is ambiguous, enumerate all possible interpretations and assess each.
- "The system handles errors gracefully" is not a behavioral spec. Enumerate every error state and its specific handling.
- The mcp-tools.md file contains SQL pseudocode for every projection. Verify that the pseudocode is correct and complete.
- Spend as many tokens as needed. This behavioral analysis is the foundation for everything that follows.

## Output

When your analysis is complete, write the entire report to:
`torvalds-output/wave0/0C-behavioral.md`

Start the file with:
```
# Wave 0C — Behavioral Analysis
## Analyst: The Behavioral Semanticist
## Source: Resonansia Gen2 Specification (9 files, 233KB)
## Date: {today}
```

When the file is written, output: **AGENT 0C COMPLETE**
