You are writing the specification for how all Resonansia components integrate. This covers event flows, orchestration sequences, and the system's behavior as a whole — not just individual operations, but how they compose.

## Step 1: Read the Architectural Foundation

Read ALL THREE Wave 1 outputs:

- `torvalds-output/wave1/1A-reconciler.md` — Unified foundation: invariants, data model, operations
- `torvalds-output/wave1/1B-interfaces.md` — Interface contracts: all boundaries and contracts
- `torvalds-output/wave1/1C-security.md` — Security analysis: threat model, failure modes

Also read for reference:
- `mcp-tools.md` — Tool specifications, projection logic
- `validation.md` — Pettson scenario and test sequence
- `tech-profile.md` — Deployment environment (Supabase Edge Functions)
- `federation.md` — Cross-tenant integration

## Step 2: Event Catalogue

Document every event the system produces. Events are the core of the event-sourcing architecture (INV-IMMUTABLE). For each event intent_type:

```
EVENT: {intent_type}
Schema: {exact JSONB structure of the event metadata}
Producer: {which operation/tool creates this event}
Consumers: {which projection logic processes this event}
Fact table mutations: {exact INSERT/UPDATE/DELETE on fact tables}
Delivery guarantee: ATOMIC (within same transaction as projection)
Ordering: UUIDv7 natural ordering
```

Event intent_types from the spec (verify completeness):
- `entity_stored` — store_entity (create or update)
- `entity_removed` — remove_entity
- `edge_created` — connect_entities
- `edge_removed` — disconnect/remove operations
- `thought_captured` — capture_thought
- `schema_defined` — type node creation/modification
- `grant_issued` — grant creation
- `grant_revoked` — grant revocation
- `blob_stored` — store_blob
- `event_proposed` — propose_event (custom)

For each, document the exact projection logic: what rows are INSERT/UPDATE/DELETE'd in which fact tables, with what column values.

## Step 3: Orchestration Sequences

For every multi-step workflow, provide a complete sequence diagram. Key workflows:

### 1. MCP Tool Call Flow (every request)
```
Client → HTTP → Hono → Auth Middleware → Tool Handler → Event Engine → Database → Response
```
Detail every step, including error handling at each point.

### 2. capture_thought Flow (most complex)
```
Input text → Auth → LLM extraction → Entity deduplication → Entity creation → Edge creation → Embedding → Response
```
This involves: MCP server, LLM API, embedding API, database (multiple tables), all within one request.

### 3. Cross-Tenant Operation Flow
```
Tool call → Auth (check JWT) → Grants lookup → Scope check → Operation with multi-tenant context → Response
```

### 4. Entity Store with Deduplication
```
store_entity → Check existing → Dedup check → Create/Update → Event → Projection → Embed → Response
```

### 5. Temporal Query Flow
```
query_at_time → Parse temporal params → Build temporal WHERE clause → Execute → Response
```

For each workflow:
- Sequence diagram (text/mermaid notation)
- Compensation logic: what happens if step N fails after steps 1..N-1 succeeded?
- Timeout behavior: what happens if the Supabase 2s CPU limit or 150s wall-clock limit is hit?
- Idempotency: can the workflow be safely retried?

## Step 4: System-Level Behaviors

### Startup sequence:
1. Supabase Edge Function cold start
2. Hono router initialization
3. MCP SDK registration of 15 tools
4. Database connection pool (Supabase client initialization)
5. Is there any warmup needed? (first request latency)

### Health checking:
- What constitutes "healthy"? (database reachable, embeddings API reachable)
- Is there a health endpoint? (Not in the MCP spec, but useful for monitoring)

### Error recovery:
- What happens after a failed request? (next request starts clean — Edge Functions are stateless)
- What about in-flight embedding requests? (lost on crash — embeddings can be regenerated)

### Monitoring:
- What metrics matter? (request latency, error rate, embedding queue depth, database connection pool usage)
- What thresholds trigger alerts?
- What does the audit_log provide for observability?

## Step 5: Deployment Topology

- **Edge Function:** Single Supabase Edge Function running Hono + MCP SDK
- **Database:** Supabase PostgreSQL with pgvector extension
- **External APIs:** OpenAI (embeddings + LLM extraction)
- **Object Storage:** Supabase Storage (for blobs)
- **Auth:** Supabase Auth + custom JWT signing

For each component:
- What configuration is required? (environment variables, API keys)
- What secrets are needed? (OPENAI_API_KEY, SUPABASE_SERVICE_ROLE_KEY, JWT_SECRET)
- What network connections exist?
- What happens if a dependency is unavailable?

## Step 6: The Pettson Scenario as Integration Test

Map the T01-T14 test sequence from validation.md as an integration flow:
- Which tests exercise which workflows?
- What's the data dependency between tests? (T01 must run before T02, etc.)
- Are there tests that exercise cross-cutting concerns? (auth + federation + temporal)

## Rules

- An event without a complete schema and projection logic is not specified.
- A workflow without compensation logic is not production-ready.
- Sequence diagrams must show error paths, not just happy paths.
- Consider the Supabase Edge Functions execution model: stateless, cold-startable, shared-nothing.
- Spend as many tokens as needed.

## Output

Write the complete integration specification to:
`torvalds-output/wave2/2C-integration.md`

Start with:
```
# Wave 2C — Integration & Orchestration Specification
## Author: Integration Specialist
## Input: Wave 1 Architectural Foundation (3 files)
## Date: {today}
## Status: Implementation-ready
```

When the file is written, output: **AGENT 2C COMPLETE**
