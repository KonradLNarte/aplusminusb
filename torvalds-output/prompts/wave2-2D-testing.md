You are writing the complete testing and verification specification for Resonansia. Your goal: define tests that, if they all pass, PROVE the system is correct. Every invariant, every operation, every edge case, every failure mode.

## Step 1: Read the Architectural Foundation

Read ALL THREE Wave 1 outputs:

- `torvalds-output/wave1/1A-reconciler.md` — Unified foundation: complete invariant set, confirmed operations
- `torvalds-output/wave1/1B-interfaces.md` — Interface contracts: every boundary contract
- `torvalds-output/wave1/1C-security.md` — Security analysis: threats, auth model, failure modes

Also read the existing test specification:
- `validation.md` — Pettson scenario, T01-T14 test cases with seed data and expected outputs
- `mcp-tools.md` — Tool input/output schemas (needed for exact test inputs/outputs)
- `constraints.md` — System invariants to verify

## Step 2: Invariant Tests

For every proven invariant from the foundation analysis:

### INV-ATOMIC (Event + Projection atomicity):
- Test: After store_entity, verify both event row AND node row exist
- Test: Simulate projection failure (how?) — verify event is NOT committed either
- Test: Under concurrent operations, verify no orphaned events or orphaned projections

### INV-IMMUTABLE (Events are immutable):
- Test: After creating an event, attempt to UPDATE it — verify it fails
- Test: After creating an event, attempt to DELETE it — verify it fails
- Test: Verify events table has no UPDATE or DELETE RLS policies that allow mutation

### Tenant Isolation:
- Test: Agent A (tenant 1) attempts to read entity from tenant 2 — verify denial
- Test: Agent A attempts to store entity with tenant 2's tenant_id — verify denial
- Test: RLS policy enforcement: direct database query with wrong tenant_id returns empty
- Test: Cross-tenant read WITH valid grant — verify success
- Test: Cross-tenant read AFTER grant revocation — verify denial

### Event Primacy:
- Test: Create entity via store_entity, verify event exists with correct intent_type
- Test: Verify fact table row references the event_id
- Test: Verify the same event_id appears in both events table and nodes table

### Bitemporality:
- Test: Create entity, update it, query_at_time for original time — verify original version returned
- Test: Create entity, query_at_time for future time — verify current version returned
- Test: Create entity, remove it, query_at_time for active period — verify entity returned
- Test: Verify no overlapping valid_from/valid_to ranges for same entity (EXCLUDE constraint)

### Graph-Native Ontology:
- Test: get_schema returns type_nodes from the graph
- Test: Type nodes are queryable via find_entities
- Test: Metatype node exists and is self-referential
- Test: Creating an entity with a non-existent type_node_id fails (or succeeds with warning?)

### Epistemic Status:
- Test: Create entity with status "hypothesis"
- Test: Update to "asserted" — verify transition allowed
- Test: Update to "confirmed" — verify transition allowed
- Test: Attempt invalid transition (confirmed → hypothesis) — verify behavior

## Step 3: Contract Tests

For every interface contract identified in Wave 1B:

### MCP Client → Server:
- Valid tool call with correct params → success response
- Valid tool call with missing required param → validation error
- Valid tool call with wrong type for param → validation error
- Tool call with invalid tool name → MCP error

### Auth boundary:
- Request with valid JWT → proceeds
- Request with expired JWT → 401 with specific error
- Request with invalid signature → 401 with specific error
- Request with missing JWT → 401 with specific error
- Request with valid JWT but insufficient scope → 403 with specific error

### Database boundary:
- Connection established → operations succeed
- Connection pool exhausted → graceful error (not crash)

### Embedding API boundary:
- API available → embedding generated
- API unavailable → operation succeeds without embedding (degraded mode?)
- API returns error → specific error handling

### LLM API boundary (capture_thought):
- API available → extraction succeeds
- API unavailable → capture_thought fails with specific error
- API returns malformed response → handled gracefully

## Step 4: Behavioral Tests

For EVERY operation (all 15 MCP tools):

### store_entity:
- Create new entity: input, expected event, expected node row, expected response
- Update existing entity: input, expected new version, expected response
- Attempt to store with invalid type_node_id: expected error
- Attempt to store in foreign tenant: expected error
- Store with metadata: verify JSONB stored correctly
- Store with label_schema validation: verify schema enforcement

### find_entities:
- Search by entity_type: exact expected results
- Search by name substring: exact expected results
- Semantic search (embedding similarity): expected results (approximate)
- Combined filters: expected results
- Pagination: verify cursor behavior
- Empty results: expected response shape

### connect_entities:
- Create edge between two entities: expected event, expected edge row
- Create cross-tenant edge with grant: expected success
- Create cross-tenant edge without grant: expected error
- Create edge with non-existent endpoint: expected error
- Create duplicate edge: expected behavior

### explore_graph:
- Traverse from entity: expected neighbors
- Traverse with depth limit: verify depth respected
- Traverse across tenant boundary with grant: expected results
- Traverse across tenant boundary without grant: filtered results

### remove_entity:
- Soft-delete entity: verify valid_to set, event created
- Remove already-removed entity: expected behavior
- Remove entity with edges: what happens to edges?

### query_at_time:
- Query for current time: same as find_entities?
- Query for past time: returns historical version
- Query for time before entity existed: empty
- Query with both valid_time and system_time: bitemporal query

### get_timeline:
- Entity with one version: single entry
- Entity with multiple versions: chronological list
- Entity that was removed: includes removal event

### capture_thought:
- Simple text with one entity: entity extracted and stored
- Text with multiple entities and relationships: all extracted
- Text with entity that already exists: deduplication triggered
- Text with ambiguous entities: LLM disambiguation
- Empty text: expected error or empty result

### get_schema:
- Returns all type_nodes for current tenant
- Includes metatype node
- Response shape matches spec

### get_stats:
- Returns correct counts
- Respects tenant isolation

### propose_event:
- Create custom event: event stored, no automatic projection
- Verify custom event appears in verify_lineage

### verify_lineage:
- Entity with simple lineage: complete chain returned
- Entity created by capture_thought: includes thought event

### store_blob / get_blob:
- Store blob reference: expected storage
- Get existing blob: expected retrieval
- Get non-existent blob: expected error

### lookup_dict:
- Standard lookup: expected results

## Step 5: Integration Tests

For every multi-step workflow from the integration spec:

### Full Pettson Scenario (T01-T14):
Map each test case with:
- Exact input (MCP tool call JSON)
- Exact expected output (response JSON shape)
- Pre-conditions (what must exist in the database)
- Post-conditions (what must be true after the test)

### Cross-cutting scenarios:
- Full entity lifecycle: create → update → query historical → remove → query removed → timeline
- Full federation flow: create grant → cross-tenant edge → cross-tenant read → revoke grant → verify access denied
- Full capture_thought flow: input text → extraction → dedup → entities created → edges created → verify with explore_graph

## Step 6: Property-Based Test Specifications

Define properties that should hold regardless of input:

- **No operation can violate INV-ATOMIC**: For any valid tool call, either both the event and projection exist, or neither does
- **No operation can violate tenant isolation**: For any tool call from tenant A, no data from tenant B (without grants) is accessible
- **store_entity is idempotent** (if claimed): Calling store_entity twice with the same input produces the same result
- **Events are append-only**: The count of events never decreases
- **Bitemporality is consistent**: No entity has overlapping valid_from/valid_to ranges

## Step 7: Chaos Scenarios

- **Embedding API down**: All operations except semantic search should still work
- **LLM API down**: capture_thought should fail gracefully; all other operations unaffected
- **Database connection interrupted**: Operations should fail cleanly, no partial state
- **Supabase Edge Function cold start**: First request should succeed (possibly slower)
- **Concurrent store_entity for same entity**: Should not corrupt data (one wins, one gets conflict error or creates a new version)
- **Very large JSONB payload**: Should be rejected by validation, not crash the server

## Rules

- A test without exact expected output is not a test.
- "Verify the system works correctly" is not a test specification. Name the input, the operation, and the exact expected output.
- Every invariant must have at least one test. An untested invariant is an unproven invariant.
- Tests must be implementable with Deno test runner and MCP client SDK.
- Include the seed data requirements for each test group.
- Spend as many tokens as needed. Test completeness is non-negotiable.

## Output

Write the complete testing specification to:
`torvalds-output/wave2/2D-testing.md`

Start with:
```
# Wave 2D — Testing & Verification Specification
## Author: Testing Specialist
## Input: Wave 1 Architectural Foundation (3 files) + Validation Spec
## Date: {today}
## Status: Implementation-ready
```

When the file is written, output: **AGENT 2D COMPLETE**
