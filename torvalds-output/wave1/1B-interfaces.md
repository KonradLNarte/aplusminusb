# Wave 1B — Interface Architecture
## Analyst: The Interface Architect
## Input: Wave 0 Foundation Analyses (3 files) + Original Spec
## Date: 2026-03-11

## Executive Summary

I have identified **12 critical interfaces** in the Resonansia system spanning external boundaries, internal component boundaries, and trust/transaction boundaries. These interfaces form the complete communication fabric between every subsystem.

**Key Findings:**
- **4 external boundaries** where data enters/exits the system (MCP clients, LLM APIs, embedding services, object storage)
- **5 internal boundaries** between core subsystems (auth, event engine, database layers)
- **3 transaction/trust boundaries** where security context or atomicity scopes change
- **Consistent error model** using 9 standardized codes across all interfaces
- **Missing interfaces** identified for grant management and edge removal operations

## Part I: Boundary Identification

### 1.1 External System Boundaries

**B1: MCP Client ↔ MCP Server**
- **What crosses it:** HTTP requests with MCP tool calls and streaming responses
- **Trust level change:** Unauthenticated → Authenticated (JWT validation)
- **Transaction scope:** Each tool call = separate HTTP request/response cycle

**B2: MCP Server ↔ LLM Service (OpenAI)**
- **What crosses it:** Structured extraction prompts and JSON responses
- **Trust level change:** Internal → External API (authenticated via API key)
- **Transaction scope:** External call happens before DB transaction starts

**B3: MCP Server ↔ Embedding Service (OpenAI)**
- **What crosses it:** Text arrays for vectorization and embedding arrays back
- **Trust level change:** Internal → External API
- **Transaction scope:** Async, outside main transaction boundary

**B4: MCP Server ↔ Object Storage (Supabase)**
- **What crosses it:** Binary blob uploads/downloads
- **Trust level change:** Internal → External service
- **Transaction scope:** Upload happens before DB transaction (potential orphans)

### 1.2 Internal Component Boundaries

**B5: HTTP Handler ↔ Auth Middleware**
- **What crosses it:** Raw JWT tokens and authenticated request context
- **Trust level change:** Unauthenticated → Tenant-scoped authenticated
- **Transaction scope:** Per-request auth resolution

**B6: Auth Middleware ↔ Tool Layer**
- **What crosses it:** Validated actor identity, tenant context, and scoped permissions
- **Trust level change:** Authenticated → Authorized for specific operations
- **Transaction scope:** Request-scoped context passing

**B7: Tool Layer ↔ Event Engine**
- **What crosses it:** Event creation requests and projection commands
- **Trust level change:** Application logic → Event sourcing layer
- **Transaction scope:** **CRITICAL** - Single atomic transaction boundary (INV-ATOMIC)

**B8: Event Engine ↔ PostgreSQL Database**
- **What crosses it:** SQL commands for event storage and fact table projections
- **Trust level change:** Application → Database (service_role bypass)
- **Transaction scope:** Database transaction boundary (BEGIN/COMMIT/ROLLBACK)

**B9: Tool Layer ↔ Vector Index (pgvector)**
- **What crosses it:** Embedding similarity queries and result sets
- **Trust level change:** None (internal subsystem)
- **Transaction scope:** Read-only operations outside main transaction

### 1.3 Trust and Authorization Boundaries

**B10: Single-Tenant ↔ Cross-Tenant Operations**
- **What crosses it:** Entity IDs and edge traversals crossing tenant boundaries
- **Trust level change:** Single-tenant scope → Cross-tenant (grants table consulted)
- **Transaction scope:** Grant validation happens within main transaction

**B11: RLS Context ↔ Database Queries**
- **What crosses it:** SET LOCAL tenant context and query execution
- **Trust level change:** Unscoped DB access → Tenant-filtered queries
- **Transaction scope:** Per-transaction scope setting

**B12: Async Job Queue ↔ Embedding Pipeline**
- **What crosses it:** Node IDs and text for embedding generation
- **Trust level change:** Main request context → Background job context
- **Transaction scope:** Outside main transaction (eventual consistency)

## Part II: Interface Contract Specification

### INTERFACE [IF-001]
**Name:** MCP Tool Request/Response
**Boundary:** MCP Client → MCP Server
**Direction:** Bidirectional (HTTP request/response)
**Input:** 
```typescript
{
  tool: string,           // Tool name from 15-tool registry
  params: ToolParams,     // Tool-specific Zod schema (§3.3a)
  auth: {
    bearer: string        // JWT token
  }
}
```
**Output:**
```typescript
// Success case
{
  result: ToolResult      // Tool-specific Zod schema
}
// Error case
{
  error: {
    code: string,         // E001-E009 from error registry
    message: string,      // Human-readable error message
    details?: object      // Optional additional context
  }
}
```
**Error states:**
- HTTP 400 + E001 VALIDATION_ERROR: Invalid tool params
- HTTP 401 + Invalid JWT: Malformed/expired token
- HTTP 403 + E003 AUTH_DENIED: Insufficient scope
- HTTP 404 + E002 NOT_FOUND: Entity not found
- HTTP 409 + E004 CONFLICT: Optimistic concurrency violation
- HTTP 429 + E008 RATE_LIMITED: Too many requests
- HTTP 500 + E009 INTERNAL_ERROR: System failure
**Idempotency:** NO (mutations create new events with unique UUIDs)
**Ordering:** UNORDERED (each request independent)
**Consistency:** STRONG (each tool call atomic)

### INTERFACE [IF-002]
**Name:** JWT Token Validation
**Boundary:** HTTP Handler → Auth Middleware
**Direction:** HTTP Handler → Auth Middleware
**Input:**
```typescript
{
  authorization_header: string,    // "Bearer <jwt>"
  tool_name: string,              // Tool being invoked
  tool_params: object             // Tool parameters (for scope checking)
}
```
**Output:**
```typescript
{
  actor_id: UUID,                 // Resolved actor node ID
  tenant_ids: UUID[],             // Accessible tenant IDs
  scopes: string[],               // Permission scopes
  primary_tenant?: UUID           // Single tenant if token is tenant-scoped
}
```
**Error states:**
- Invalid JWT signature → 401 Unauthorized
- Expired token → 401 Unauthorized
- Wrong audience → 401 Unauthorized
- Missing required scope → 403 E003 AUTH_DENIED
**Idempotency:** YES (same token always produces same result)
**Ordering:** UNORDERED
**Consistency:** STRONG

### INTERFACE [IF-003]
**Name:** Actor Resolution and Auto-Creation
**Boundary:** Auth Middleware → Database
**Direction:** Auth Middleware → Database
**Input:**
```typescript
{
  jwt_sub: string,         // Subject claim from JWT
  tenant_id: UUID,         // Target tenant
  token_metadata: {
    actor_type?: string,   // "user" | "agent" | "system"
    display_name?: string
  }
}
```
**Output:**
```typescript
{
  actor_node_id: UUID,     // Existing or newly created actor
  was_created: boolean     // True if actor was auto-created
}
```
**Error states:**
- Database constraint violation → E009 INTERNAL_ERROR
- Tenant not found → E002 NOT_FOUND
**Idempotency:** YES (idempotent auto-creation)
**Ordering:** BEST_EFFORT (concurrent same-actor requests may create duplicates, handled by subsequent dedup)
**Consistency:** STRONG (each auto-creation atomic)

### INTERFACE [IF-004]
**Name:** Event Creation and Projection
**Boundary:** Tool Layer → Event Engine
**Direction:** Tool Layer → Event Engine
**Input:**
```typescript
{
  intent_type: string,         // One of 10 registered intent types
  payload: object,             // Intent-specific data structure
  stream_id: UUID,             // Entity the event belongs to
  occurred_at?: DateTime,      // Business time (default: now)
  node_ids?: UUID[],           // Entities affected by this event
  edge_ids?: UUID[],           // Edges affected by this event
  actor_id: UUID,              // Who caused this event
  tenant_id: UUID              // Owning tenant
}
```
**Output:**
```typescript
{
  event_id: UUID,              // Created event identifier
  projected_changes: {         // What changed in fact tables
    nodes_created?: number,
    nodes_updated?: number,
    edges_created?: number,
    grants_created?: number,
    blobs_created?: number
  }
}
```
**Error states:**
- Unknown intent_type → E001 VALIDATION_ERROR
- Payload schema violation → E006 SCHEMA_VIOLATION
- Projection constraint violation → E009 INTERNAL_ERROR (full rollback)
- UUIDv7 generation failure → E009 INTERNAL_ERROR
**Idempotency:** NO (each event gets unique event_id)
**Ordering:** REQUIRED (events within same stream_id must be ordered by event_id/occurred_at)
**Consistency:** STRONG (event + projection atomic via single transaction)

### INTERFACE [IF-005]
**Name:** Database Transaction Boundary
**Boundary:** Event Engine → PostgreSQL Database
**Direction:** Event Engine → PostgreSQL Database
**Input:**
```typescript
{
  event_insert: {
    event_id: UUID,
    tenant_id: UUID,
    intent_type: string,
    payload: object,
    // ... all event fields
  },
  projections: Array<{
    table: "nodes" | "edges" | "grants" | "blobs",
    operation: "INSERT" | "UPDATE",
    data: object
  }>
}
```
**Output:**
```typescript
{
  rows_affected: number,       // Total rows inserted/updated
  event_recorded_at: DateTime, // When transaction committed
  version_numbers: {           // For entities that were versioned
    [entity_id: string]: number
  }
}
```
**Error states:**
- EXCLUDE constraint violation (temporal overlap) → ROLLBACK + E004 CONFLICT
- Foreign key violation → ROLLBACK + E009 INTERNAL_ERROR
- RLS policy denial → ROLLBACK + E003 AUTH_DENIED
- Deadlock → ROLLBACK + retry logic (internal)
**Idempotency:** NO (each transaction has side effects)
**Ordering:** REQUIRED (within single stream_id)
**Consistency:** STRONG (ACID transaction boundary)

### INTERFACE [IF-006]
**Name:** RLS Context Setting
**Boundary:** Auth Middleware → Database Connection
**Direction:** Auth Middleware → Database Connection
**Input:**
```typescript
{
  tenant_ids: UUID[],          // Accessible tenants from JWT
  connection: DatabaseConnection
}
```
**Output:**
```typescript
{
  context_set: true,           // Confirmation GUC was set
  previous_value?: string      // Previous tenant context (for debugging)
}
```
**Error states:**
- Invalid UUID array → E009 INTERNAL_ERROR
- Database connection closed → E009 INTERNAL_ERROR
**Idempotency:** YES (SET LOCAL is idempotent)
**Ordering:** UNORDERED
**Consistency:** STRONG (per-transaction scope)

### INTERFACE [IF-007]
**Name:** LLM Extraction Service
**Boundary:** Tool Layer → OpenAI API
**Direction:** Tool Layer → OpenAI API
**Input:**
```typescript
{
  prompt: {
    system: string,            // Context + instructions + schema
    user: string               // Free-text content to extract from
  },
  model: "gpt-4o-mini",
  max_tokens: number,
  temperature: 0.1,
  response_format: { type: "json_object" },
  timeout_ms: 30000
}
```
**Output:**
```typescript
{
  extraction: {
    primary_entity: { type: string, data: object },
    mentioned_entities: Array<{
      type: string,
      data: object,
      existing_match_id?: UUID,
      match_confidence?: number
    }>,
    relationships: Array<{
      edge_type: string,
      source: string | number,
      target: string | number | string,  // "existing:<UUID>" format for refs
      data: object
    }>,
    action_items: string[]
  },
  usage: {
    input_tokens: number,
    output_tokens: number
  }
}
```
**Error states:**
- API timeout → E007 EXTRACTION_FAILED
- Invalid JSON response → E007 EXTRACTION_FAILED (retry once)
- Rate limit exceeded → E008 RATE_LIMITED
- API key invalid → E009 INTERNAL_ERROR
**Idempotency:** NO (LLM responses may vary)
**Ordering:** UNORDERED
**Consistency:** NONE (external service)

### INTERFACE [IF-008]
**Name:** Embedding Generation Service
**Boundary:** Tool Layer → OpenAI API (Async)
**Direction:** Tool Layer → OpenAI API
**Input:**
```typescript
{
  texts: string[],             // Batch of texts to embed (max 100)
  model: "text-embedding-3-small",
  dimensions: 1536,
  retry_config: {
    max_retries: 5,
    base_delay_ms: 1000,
    max_delay_ms: 60000
  }
}
```
**Output:**
```typescript
{
  embeddings: Array<{
    text: string,
    embedding: number[],       // 1536-dimensional vector
    token_count: number
  }>,
  usage: {
    total_tokens: number
  }
}
```
**Error states:**
- API timeout → Retry with exponential backoff
- Rate limit → Retry after delay
- All retries exhausted → Log error, mark embedding as failed
- Invalid text encoding → Skip text, log warning
**Idempotency:** YES (same text produces same/similar embeddings)
**Ordering:** BEST_EFFORT (batch processing with rate limits)
**Consistency:** EVENTUAL (embeddings updated async after entity creation)

### INTERFACE [IF-009]
**Name:** Cross-Tenant Grant Validation
**Boundary:** Tool Layer → Grants Table
**Direction:** Tool Layer → Grants Table
**Input:**
```typescript
{
  subject_tenant_id: UUID,     // Tenant requesting access
  object_node_id: UUID,        // Target node being accessed
  required_capability: "READ" | "WRITE" | "TRAVERSE",
  as_of_time?: DateTime        // Optional temporal query (default: now)
}
```
**Output:**
```typescript
{
  access_granted: boolean,
  grant_details?: {
    grant_id: UUID,
    capability: string,
    valid_from: DateTime,
    valid_to: DateTime,
    created_by: UUID
  }
}
```
**Error states:**
- Object node not found → E002 NOT_FOUND
- Subject tenant not found → E002 NOT_FOUND
- No valid grant found → E005 CROSS_TENANT_DENIED
**Idempotency:** YES (grant lookup is deterministic)
**Ordering:** UNORDERED
**Consistency:** STRONG (read-committed isolation)

### INTERFACE [IF-010]
**Name:** Vector Similarity Search
**Boundary:** Tool Layer → pgvector HNSW Index
**Direction:** Tool Layer → pgvector Index
**Input:**
```typescript
{
  query_embedding: number[],   // 1536-dimensional query vector
  tenant_id: UUID,             // RLS filter
  entity_types?: string[],     // Optional type filter
  limit: number,               // Max results (default 10, max 500)
  similarity_threshold?: number, // Cosine similarity cutoff (0.0-1.0)
  include_deleted?: boolean    // Include soft-deleted entities (default false)
}
```
**Output:**
```typescript
{
  results: Array<{
    node_id: UUID,
    entity_type: string,
    data: object,
    similarity: number,        // Cosine similarity score
    epistemic: string,
    valid_from: DateTime
  }>,
  total_candidates: number,    // How many vectors were considered
  search_time_ms: number       // Performance metric
}
```
**Error states:**
- Invalid vector dimensions → E001 VALIDATION_ERROR
- HNSW index corrupted → E009 INTERNAL_ERROR
- Similarity threshold out of range → E001 VALIDATION_ERROR
**Idempotency:** YES (same query vector produces same results at same point in time)
**Ordering:** UNORDERED (results sorted by similarity score)
**Consistency:** EVENTUAL (index reflects database state with possible lag)

### INTERFACE [IF-011]
**Name:** Object Storage Operations
**Boundary:** Tool Layer → Supabase Storage
**Direction:** Bidirectional (upload/download)
**Input (Upload):**
```typescript
{
  data: Buffer | Uint8Array,   // Binary content
  content_type: string,        // MIME type
  bucket: "resonansia-blobs",
  file_path: string,           // Generated path with blob_id
  metadata?: object            // Optional metadata
}
```
**Output (Upload):**
```typescript
{
  storage_ref: string,         // Reference path for retrieval
  size_bytes: number,          // Actual uploaded size
  upload_time_ms: number,      // Performance metric
  etag?: string                // Content hash if available
}
```
**Input (Download):**
```typescript
{
  storage_ref: string          // Reference from blob record
}
```
**Output (Download):**
```typescript
{
  data: Buffer,                // Binary content
  content_type: string,        // Original MIME type
  size_bytes: number,          // Content length
  last_modified?: DateTime     // Storage timestamp
}
```
**Error states:**
- Upload failure → E009 INTERNAL_ERROR (potential orphan)
- File not found → E002 NOT_FOUND
- Storage quota exceeded → E009 INTERNAL_ERROR
- Invalid storage_ref → E001 VALIDATION_ERROR
**Idempotency:** Upload NO (creates new storage objects), Download YES
**Ordering:** UNORDERED
**Consistency:** EVENTUAL (storage separate from DB transaction)

### INTERFACE [IF-012]
**Name:** Protected Resource Metadata (RFC 9728)
**Boundary:** External → MCP Server
**Direction:** External → MCP Server
**Input:**
```typescript
{
  method: "GET",
  path: "/.well-known/oauth-protected-resource",
  accept: "application/json"
}
```
**Output:**
```typescript
{
  resource: "resonansia-mcp",
  authorization_servers: [
    {
      issuer: "https://your-supabase-url.supabase.co/auth/v1",
      jwks_uri: "https://your-supabase-url.supabase.co/auth/v1/.well-known/jwks",
      scopes_supported: [
        "tenant:*:read",
        "tenant:*:write", 
        "tenant:{id}:read",
        "tenant:{id}:write",
        "tenant:{id}:nodes:{type}:read",
        "tenant:{id}:nodes:{type}:write"
      ],
      response_types_supported: ["token"],
      grant_types_supported: ["authorization_code", "refresh_token"]
    }
  ],
  resource_documentation: "https://docs.resonansia.com/api",
  revocation_endpoint?: "https://your-supabase-url.supabase.co/auth/v1/logout"
}
```
**Error states:**
- None (well-known endpoint always returns metadata)
**Idempotency:** YES (static metadata)
**Ordering:** UNORDERED
**Consistency:** STRONG (static configuration)

## Part III: Interface Consistency Analysis

### 3.1 Error Model Consistency

All interfaces use the **standardized 9-error-code registry** from mcp-tools.md §3.5:

| Code | HTTP Status | Usage Pattern | Cross-Interface |
|------|------------|---------------|-----------------|
| E001 VALIDATION_ERROR | 400 | Input validation failures | ✓ All interfaces |
| E002 NOT_FOUND | 404 | Resource lookup failures | ✓ All data interfaces |
| E003 AUTH_DENIED | 403 | Authorization failures | ✓ Auth boundaries |
| E004 CONFLICT | 409 | Optimistic concurrency | ✓ Transaction boundaries |
| E005 CROSS_TENANT_DENIED | 403 | Cross-tenant access denied | ✓ Federation boundaries |
| E006 SCHEMA_VIOLATION | 400 | Type schema validation | ✓ Entity operations |
| E007 EXTRACTION_FAILED | 422 | LLM processing failures | ✓ AI service boundaries |
| E008 RATE_LIMITED | 429 | Rate limiting | ✓ External APIs |
| E009 INTERNAL_ERROR | 500 | System failures | ✓ All interfaces |

**Consistency Achievement:** Every interface maps its failure modes to these 9 codes, ensuring agents can handle errors uniformly across all boundaries.

### 3.2 Authentication/Authorization Flow Consistency

The auth flow maintains consistent token handling across all interfaces:

1. **IF-001** receives JWT in Authorization header
2. **IF-002** validates JWT and extracts claims
3. **IF-003** resolves/creates actor identity per tenant
4. **IF-006** sets RLS context for database access
5. **IF-009** validates cross-tenant permissions when needed
6. All other interfaces inherit authenticated context

**No token passing:** The system correctly implements confused deputy prevention by not passing received tokens to upstream APIs (IF-007, IF-008, IF-011).

### 3.3 Transaction Boundary Consistency

**Strong Consistency Zone:** IF-004 → IF-005 (Event creation + projection)
- Single ACID transaction
- All-or-nothing semantics
- Consistent with INV-ATOMIC invariant

**Eventual Consistency Zone:** IF-008 (Async embedding generation)
- Operates outside main transaction
- Node creation succeeds even if embedding fails
- Retry logic handles temporary failures

**External Consistency:** IF-007, IF-011 (LLM, Storage)
- External calls happen before database transaction
- Potential for orphaned external resources
- Acceptable trade-off for performance and isolation

### 3.4 Data Type Consistency

All interfaces use **consistent data types:**
- Entity IDs: UUID v7 format
- Timestamps: ISO 8601 DateTime strings  
- Embeddings: Float32 arrays of dimension 1536
- JSONB payloads: Unconstrained objects (validated by application)
- Tenant IDs: UUID v4 format
- Scopes: String arrays with standardized syntax

## Part IV: Boundary Map

```
┌─── EXTERNAL WORLD ───┐
│                      │
│  MCP Clients  ┌─────┐│
│     ↕         │     ││  
│ [IF-001] ────▶│     ││
│ [IF-012] ◀────│     ││
│               │ MCP ││
│  LLM APIs     │     ││    ┌── INTERNAL SUBSYSTEMS ──┐
│     ↕         │ SVR ││    │                         │
│ [IF-007] ◀────┤     ││    │  ┌─────────────────────┐ │
│               │     ││    │  │   Auth Middleware   │ │
│ Embedding     │     ├┼────┼─▶│ [IF-002] [IF-003]   │ │
│ APIs ↕        └─────┘│    │  └─────────┬───────────┘ │
│ [IF-008] ◀────────────┤    │            │ [IF-006]    │
│                      │    │            ▼             │
│ Object Storage ↕     │    │  ┌─────────────────────┐ │
│ [IF-011] ◀───────────┤    │  │    Tool Layer       │ │
│                      │    │  │                     │ │
└──────────────────────┘    │  └─────────┬───────────┘ │
                            │            │ [IF-004]    │
                            │            ▼             │
                            │  ┌─────────────────────┐ │
                            │  │   Event Engine      │ │
                            │  │                     │ │
                            │  └─────────┬───────────┘ │
                            │            │ [IF-005]    │
                            │            ▼             │
                            │  ┌─────────────────────┐ │
                            │  │   PostgreSQL DB     │ │
                            │  │ ┌─────────────────┐ │ │
                            │  │ │ RLS Policies    │ │ │
                            │  │ │                 │ │ │
                            │  │ ├─────────────────┤ │ │
                            │  │ │ pgvector HNSW   │ │ │
                            │  │ │ [IF-010]        │ │ │
                            │  │ ├─────────────────┤ │ │
                            │  │ │ Grants Table    │ │ │
                            │  │ │ [IF-009]        │ │ │
                            │  │ └─────────────────┘ │ │
                            │  └─────────────────────┘ │
                            └─────────────────────────┘

Trust Level Transitions:
Unauthenticated ──[IF-001]──▶ JWT Validated ──[IF-002]──▶ 
Tenant Scoped ──[IF-006]──▶ RLS Enforced ──[IF-009]──▶ Cross-Tenant Authorized

Transaction Boundaries:
HTTP Request ──[IF-001]──▶ Tool Logic ──[IF-004]──▶ 
Database Transaction ──[IF-005]──▶ Async Jobs ──[IF-008]
```

## Part V: Coupling Analysis

### 5.1 Tight Coupling (Justified)

**IF-004 ↔ IF-005** (Event Engine ↔ Database)
- **Coupling type:** Transaction coupling  
- **Justification:** Required for INV-ATOMIC. Event and projections must be atomic.
- **Risk:** Database failure breaks event sourcing. Acceptable - database is foundational dependency.

**IF-002 ↔ IF-003** (JWT Validation ↔ Actor Resolution) 
- **Coupling type:** Identity coupling
- **Justification:** Actor nodes must exist for created_by references to be valid.
- **Risk:** Actor creation could fail. Mitigated by idempotent auto-creation.

### 5.2 Loose Coupling (Well-Designed)

**IF-001 ↔ IF-007/IF-008** (MCP Client ↔ External APIs)
- **Coupling type:** None (external calls happen within tool execution)
- **Design strength:** MCP client is isolated from external API failures/changes
- **Justification:** Client-server boundary should be stable regardless of internal dependencies

**IF-004 ↔ IF-008** (Event Creation ↔ Embedding Generation)
- **Coupling type:** Temporal decoupling (async)
- **Design strength:** Entity creation succeeds independently of embedding generation
- **Justification:** Embeddings are eventual consistency, not critical path

### 5.3 Problematic Coupling (Risks Identified)

**IF-004 ↔ IF-007** (Event Creation ↔ LLM Extraction)
- **Risk:** LLM failures break capture_thought operations entirely
- **Mitigation:** LLM calls happen pre-transaction, but failures cascade up
- **Recommendation:** Consider fallback to "note" entity type for LLM failures

**Missing Interface Coupling:**
- No interface for grant creation/revocation operations (identified gap)
- No interface for edge removal operations (identified gap)

## Part VI: Missing and Redundant Interfaces

### 6.1 Missing Interfaces

**MISSING-1: Grant Management Interface**
```typescript
interface GrantManagementInterface {
  name: "Grant Administration",
  boundary: "Admin Layer → Grants Table", 
  operations: ["create_grant", "revoke_grant", "list_grants"],
  input: { subject_tenant_id, object_node_id, capability, valid_until },
  output: { grant_id, effective_immediately: boolean }
}
```
**Impact:** Grant operations currently require propose_event or direct SQL
**Recommendation:** Add dedicated grant management tools

**MISSING-2: Edge Removal Interface** 
```typescript
interface EdgeRemovalInterface {
  name: "Edge Lifecycle Management",
  boundary: "Tool Layer → Event Engine",
  operations: ["remove_edge"],
  input: { edge_id, reason? },
  output: { removed: true, edge_type, event_id }
}
```
**Impact:** Edge removal currently requires propose_event with manual payload construction
**Recommendation:** Add remove_edge tool to complete edge lifecycle

### 6.2 Redundant Interfaces

**None identified.** All 12 interfaces serve distinct purposes and cannot be collapsed without losing functionality or violating architectural boundaries.

### 6.3 Interface Gaps

**GAP-1: Batch Operations**
- Current interfaces are single-entity focused
- No batch interface for bulk entity creation/updates
- **Impact:** High latency for large data imports
- **Recommendation:** Consider batch_store_entities interface for gen2

**GAP-2: Streaming Responses**
- Current interfaces use request-response pattern
- No streaming for long-running operations (large graph traversals)
- **Impact:** Timeout risk for complex explore_graph operations
- **Recommendation:** Consider Server-Sent Events for large result sets

## Part VII: Implementation Requirements

### 7.1 Interface Implementation Checklist

Each interface must implement:
- [ ] **Input validation** using exact Zod schemas
- [ ] **Output serialization** with type safety
- [ ] **Error mapping** to standardized error codes  
- [ ] **Timeout handling** with circuit breaker pattern
- [ ] **Audit logging** for security and debugging
- [ ] **Performance metrics** (latency, throughput)
- [ ] **Health checks** for external dependencies

### 7.2 Interface Testing Requirements

**Unit Test Coverage:**
- Happy path with valid inputs → expected outputs
- All error conditions → correct error codes and HTTP status
- Boundary value testing (max limits, empty inputs)
- Concurrent access patterns for stateful interfaces

**Integration Test Coverage:**
- End-to-end flows across multiple interfaces
- Error propagation from external APIs (IF-007, IF-008)
- Transaction rollback scenarios (IF-004, IF-005)
- Cross-tenant access patterns (IF-009)

**Performance Test Requirements:**
- Load testing for high-traffic interfaces (IF-001, IF-010)
- Latency testing for AI service interfaces (IF-007, IF-008) 
- Throughput testing for database interfaces (IF-005)
- Memory usage testing for large result sets

### 7.3 Monitoring and Observability

**Required Metrics per Interface:**
- Request rate and latency distribution
- Error rate by error code
- External API dependency health
- Database connection pool utilization
- Authentication success/failure rates
- Cross-tenant operation frequency

**Alert Conditions:**
- Error rate >5% for any interface
- External API latency >30s (IF-007, IF-008)
- Database transaction failures (IF-005)
- Authentication failures >10% (IF-002)

## Conclusion

The Resonansia system demonstrates well-architected interface boundaries with **strong consistency at the core** (event sourcing) and **eventual consistency at the edges** (embeddings, external APIs). The standardized error model and authentication flow provide consistency across all 12 interfaces.

**Key Strengths:**
- Clear separation of concerns between subsystems
- Consistent error handling and authentication patterns  
- Atomic transaction boundaries preserve data integrity
- Loose coupling to external services

**Critical Gaps:**
- Missing grant management interface
- Missing edge removal interface  
- No batch operation support

**Recommendations:**
1. **Immediate:** Add grant management and edge removal interfaces
2. **Gen2:** Consider batch operations and streaming responses for scalability
3. **Gen3:** Evaluate async messaging patterns for high-throughput scenarios

The interface architecture successfully supports the 15-tool MCP specification while maintaining clean boundaries and consistent contracts across all system interactions.

**AGENT 1B COMPLETE**