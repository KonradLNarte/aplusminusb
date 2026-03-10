You are writing the definitive operation layer specification for Resonansia. Every behavior of the system must be described with enough precision that an implementation built solely from this document would be correct.

## Step 1: Read the Architectural Foundation

Read ALL THREE Wave 1 outputs:

- `torvalds-output/wave1/1A-reconciler.md` — Unified foundation: resolved invariants, confirmed operations, data model corrections
- `torvalds-output/wave1/1B-interfaces.md` — Interface contracts: every boundary and contract
- `torvalds-output/wave1/1C-security.md` — Security analysis: authorization model, threat mitigation

Also read the original operation specs for reference (your output should SUPERSEDE these):
- `mcp-tools.md` — All 15 MCP tools with current schemas and projection logic
- `auth.md` — Auth flow and scope requirements
- `validation.md` — Test cases with expected behavior

## Step 2: Specify Every Operation

For ALL 15 MCP tools (store_entity, find_entities, connect_entities, explore_graph, remove_entity, query_at_time, get_timeline, capture_thought, get_schema, get_stats, propose_event, verify_lineage, store_blob, get_blob, lookup_dict) plus the auth middleware and embedding pipeline:

### 1. Signature
- Name
- Input parameters: typed, constrained, with Zod-compatible schema definition
- Return type: typed, all possible shapes including success AND every error variant
- MCP tool metadata (description, annotations)

### 2. Authorization
- Required scope(s): exact scope string(s)
- Required tenant access: own-tenant or cross-tenant (via grants)
- What happens if unauthorized: exact MCP error response with error code, HTTP status, message
- Order: is authorization checked before or after input validation?

### 3. Validation
- What input validation is performed? Ordered list:
  1. Type validation (Zod parse)
  2. Semantic validation (e.g., referenced entity exists, type_node is valid)
  3. Business rule validation (e.g., epistemic status transition is legal)
- What is the exact error response for each validation failure?
- Validation happens AFTER authorization (you don't leak information about invalid inputs to unauthorized callers)

### 4. Happy Path
Step-by-step, in pseudocode-level detail:
1. Extract parameters from validated input
2. Each database read (exact query shape)
3. Each computation (deduplication check, embedding generation, etc.)
4. Each database write (exact INSERT/UPDATE with column values)
5. Event creation (exact event payload with intent_type, metadata)
6. Projection (exact fact table mutations triggered by the event)
7. Side effects (audit log, embedding queue, etc.)
8. The exact return value (typed response object)

### 5. Edge Cases
For each edge case:
- What triggers it (e.g., entity already exists for store_entity, no results for find_entities)
- How it differs from the happy path
- The exact return value
- Whether it creates an event or not

### 6. Failure Modes
For each possible failure:
- What causes it (network error to embedding API, database constraint violation, LLM timeout)
- How it is detected (exception type, error code)
- How it is handled (retry, abort, compensate, degrade gracefully)
- What state is the system in after the failure? (transaction rolled back? partial state?)
- Does it violate INV-ATOMIC? If so, how is that prevented?

### 7. Concurrency
- Can this operation run concurrently with itself? (e.g., two store_entity for same entity)
- Can it run concurrently with other specific operations? (e.g., store_entity + remove_entity)
- If conflicts are possible, how are they resolved? (optimistic concurrency optional in gen1, but what's the fallback?)
- Is the operation idempotent?

## Step 3: Auth Middleware Specification

The auth middleware runs before every tool call. Specify completely:
1. JWT extraction from request
2. JWT validation (signature, expiry, audience, issuer)
3. Claim extraction (sub, tenant_id, scopes)
4. SET LOCAL app.tenant_ids (for RLS)
5. Actor auto-creation (create actor node if sub is new)
6. Grants consultation (query grants table for cross-tenant access)
7. Scope checking (match required scope against JWT scopes)
8. Exact error responses for each failure point

## Step 4: Embedding Pipeline Specification

The embedding pipeline is a side effect of entity operations:
1. When is embedding triggered? (store_entity create, store_entity update, capture_thought)
2. How is it implemented? (synchronous in gen1 per D-016, or async queue?)
3. OpenAI API call specification (model, dimensions, batch size, retry logic)
4. What happens if embedding fails? (entity is created but without embedding — is that acceptable?)
5. Bulk embedding for seed data

## Step 5: capture_thought Deep Specification

This is the most complex operation. Specify in full:
1. LLM prompt (the complete prompt template from the spec, with all parameters)
2. LLM response parsing (expected JSON schema, validation, error handling)
3. Entity extraction and classification
4. Deduplication algorithm (3-tier: exact name match → embedding similarity → LLM disambiguation)
5. Entity and edge creation (atomic transaction)
6. Return value (extracted entities, created edges, deduplication decisions)

## Rules

- "Handle the error" is not a specification. Name the error, its MCP error code, its response body, and its side effects.
- Pseudocode must be unambiguous. If a reader could interpret it two ways, it needs more detail.
- Every operation must specify what happens to the transaction if ANY step fails.
- Reference the interface contracts from Wave 1B for cross-boundary calls.
- Spend as many tokens as needed. Every operation must be fully specified.

## Output

Write the complete operation layer specification to:
`torvalds-output/wave2/2B-operations.md`

Start with:
```
# Wave 2B — Operation Layer Specification
## Author: Operation Layer Specialist
## Input: Wave 1 Architectural Foundation (3 files)
## Date: {today}
## Status: Implementation-ready
```

When the file is written, output: **AGENT 2B COMPLETE**
