# RESONANSIA SPECIFICATION — IMPLEMENTER'S REVIEW
## Agent 4C: The Implementer's Advocate
**Date**: 2026-03-11  
**Stack**: Deno + Hono + Supabase Edge Functions + pgvector + @hono/mcp@^0.2.3 + @modelcontextprotocol/sdk@^1.27.1  
**Review Perspective**: Developer readiness for production implementation

---

## EXECUTIVE SUMMARY

**Assessment**: IMPLEMENTATION-READY with P1 clarifications needed

The unified specification is architecturally sound and comprehensive. The database schema is production-ready, the MCP tool interfaces are well-defined, and the work package structure provides clear implementation guidance. However, several critical areas need clarification before coding begins, particularly around Supabase Edge Functions limits, cross-tenant security enforcement, and LLM integration patterns.

**Confidence Level**: 85% - Ready to start with risk mitigation plan

---

## STEP 2: IMPLEMENTATION WALKTHROUGH

### WP-01: Database Schema & Bootstrap
**Files to Create**:
- `migrations/001_create_tables.sql` - Core schema with UUIDv7, RLS, indexes
- `migrations/002_bootstrap_metatype.sql` - Self-referential type system
- `migrations/003_sample_types.sql` - Core entity types (person, org, note)

**Judgment Calls Needed**:
- UUIDv7 function implementation - the spec provides PostgreSQL-specific bytea manipulation. Need to verify Supabase supports all used functions (`set_bit`, `int8send`).
- EXCLUDE constraint performance - `tstzrange(valid_from, valid_to) WITH &&` on nodes table could be expensive. Need monitoring strategy.
- RLS policy ordering - multiple policies per table, PostgreSQL evaluates them with OR logic. Need to verify performance.

**Questions for Architect**:
- Should we add indexes for audit queries? `audit_log` table has no indexes specified.
- Bootstrap sequence order - if metatype bootstrap fails halfway, how do we recover?
- Vector index parameters (`m=16, ef_construction=64`) - are these tuned for expected data size?

**Ambiguous Behavior**:
- What happens if UUIDv7 generation fails? Should we fall back to regular UUIDs?
- Cross-tenant edge trigger validation - spec shows trigger fires BEFORE INSERT, but what if grant lookup itself fails?

### WP-04: Authentication & Authorization  
**Files to Create**:
- `src/auth/jwt-validator.ts` - JWT signature/claims validation
- `src/auth/actor-manager.ts` - Auto-creation logic  
- `src/auth/scope-checker.ts` - Tool permission mapping
- `src/auth/tenant-context.ts` - SET LOCAL enforcement

**Judgment Calls Needed**:
- JWT algorithm allowlist - spec says "HS256 only" but real JWTs might use RS256. Need fallback strategy.
- Actor auto-creation race conditions - multiple requests with same `sub` could create duplicate actors. Need UPSERT pattern.
- Tenant context verification - spec requires verifying SET LOCAL succeeded, but what's the recovery path if it fails?

**Questions for Architect**:
- Should we cache validated JWTs in-memory? Edge Functions restart frequently.
- Actor node creation - should actors get embeddings like other entities?
- Scope inheritance - if user has `tenant:A:write`, do they automatically get `tenant:A:read`?

**Ambiguous Behavior**:
- JWT expiry handling - should we validate `exp` ourselves or rely on library?
- Tenant ID array format in JWT - spec shows string array, but SET LOCAL needs specific PostgreSQL array literal format.

### WP-05: Event Engine
**Files to Create**:
- `src/event-engine/event-creator.ts` - Event record generation
- `src/event-engine/projection-executor.ts` - Intent-type dispatch
- `src/event-engine/transaction-manager.ts` - Atomic event+projection wrapper

**Judgment Calls Needed**:
- Transaction timeout - Supabase has connection limits. How long should we hold transactions open?
- Event payload validation - should we validate against JSON schema before storing, or trust callers?
- Stream ID generation - spec mentions stream_id but doesn't define the grouping strategy clearly.

**Questions for Architect**:
- Projection failure recovery - if event succeeds but projection fails, do we retry the projection or rollback the event?
- Event ordering guarantees - are UUIDv7 timestamps sufficient for strict ordering?
- Concurrent event handling - can we process multiple events for different entities simultaneously?

### WP-10: LLM Integration & Deduplication
**Files to Create**:  
- `src/llm/openai-client.ts` - API wrapper with retry logic
- `src/llm/extraction-prompt.ts` - Production prompt template
- `src/dedup/three-tier-matcher.ts` - Exact → trigram → embedding → LLM chain
- `src/dedup/entity-merger.ts` - Handle deduplication results

**Judgment Calls Needed**:
- LLM timeout strategy - OpenAI can be slow. How long do we wait before failing capture_thought?
- Deduplication confidence thresholds - spec suggests 0.85/0.95 but provides no tuning guidance.
- Entity merge strategy - when deduplication finds a match, do we update the existing entity or create a relation?

**Questions for Architect**:
- OpenRouter vs direct OpenAI - any preference for cost/reliability?
- LLM response parsing - what if GPT returns invalid JSON?
- Embedding dimension mismatch - what if OpenAI changes their embedding model?

**Ambiguous Behavior**:
- Deduplication cross-tenant - should we dedupe across tenant boundaries using grants?
- Large text extraction - what if input exceeds token limits?

---

## STEP 3: TECHNOLOGY ASSESSMENT

### Supabase Edge Functions Analysis

**CPU Limit (2s)**: ⚠️ **CONCERNING**
- `capture_thought` with LLM calls will likely hit this limit
- OpenAI API calls typically 1-3s, plus deduplication processing
- Mitigation: Implement request queuing with background processing
- Alternative: Switch to async pattern with status polling

**Memory Limit (300MB)**: ✅ **FEASIBLE**  
- Vector embeddings: 1536 floats × 4 bytes × 1000 entities = 6MB
- JSON parsing should be fine for typical entity sizes
- LLM response caching fits comfortably

**Bundle Size (20MB)**: ✅ **FEASIBLE**
- @modelcontextprotocol/sdk@^1.27.1 is ~2MB
- @hono/mcp@^0.2.3 is ~500KB  
- OpenAI client libraries ~1MB
- Deno standard library is external
- Total estimate: ~8MB with dependencies

**Wall Clock (150s)**: ✅ **SUFFICIENT**
- Covers LLM processing time comfortably
- HTTP streaming for long-running operations possible

### Deno Compatibility Check

**@hono/mcp@^0.2.3**: ✅ **VERIFIED**
- Built for Deno/Edge runtime
- No Node.js-specific dependencies

**@modelcontextprotocol/sdk@^1.27.1**: ⚠️ **NEEDS VERIFICATION**
- Documentation unclear on Deno support
- May require import map configuration
- Fallback: Use protocol directly with Hono

### PostgreSQL + pgvector Performance

**HNSW Index Build Time**: ✅ **ACCEPTABLE**
- ~1000 vectors: <10 seconds
- Build happens async after initial data load
- Can defer index creation until after seed data

**EXCLUDE Constraint Efficiency**: ⚠️ **MONITOR NEEDED**
- Temporal exclusion on nodes table could become bottleneck
- Each insert requires range overlap check
- Consider partitioning for large datasets

**RLS with Grant Subqueries**: ❌ **PERFORMANCE RISK**
- Cross-tenant operations require grants table lookup in RLS policy
- Could result in N+1 query patterns
- Recommend application-level grant checking instead

### External API Concerns

**OpenAI Rate Limits**: ⚠️ **DESIGN AROUND**
- GPT-4o-mini: 10,000 RPM (should be sufficient)
- Embeddings: 5,000 RPM (could be limiting for bulk operations)
- Need exponential backoff with jitter

**LLM Response Robustness**: ❌ **BRITTLE**
- JSON parsing failures will break capture_thought
- Need robust error handling and response validation
- Consider fallback extraction methods

**Network Timeout Handling**: ⚠️ **EDGE FUNCTION CONTEXT**
- Edge Functions can't maintain persistent connections
- Each request creates fresh HTTP clients
- Connection pooling benefits are lost

---

## STEP 4: IMPLEMENTATION RISKS

### Top 5 Hardest to Implement

1. **Cross-Tenant Edge Validation Trigger (SEVERITY: HIGH)**
   - Database trigger + application logic coordination
   - Grant lookup within trigger context
   - Error handling when grant validation fails
   - Race conditions between grant creation/revocation and edge creation

2. **3-Tier Deduplication Algorithm (SEVERITY: HIGH)**
   - Multiple async operations (embedding, LLM calls)
   - Complex confidence scoring and threshold tuning
   - Error handling when any tier fails
   - Performance optimization for real-time operation

3. **Synchronous capture_thought within CPU limits (SEVERITY: CRITICAL)**
   - LLM + embedding generation + deduplication + database writes
   - All must complete within 2-second CPU window
   - No clear fallback if processing exceeds limits

4. **RLS Policy Performance with Cross-Tenant Grants (SEVERITY: MEDIUM)**
   - Grants table lookup in every RLS check
   - Potential N+1 query pattern
   - Complex policy logic with multiple tenant contexts

5. **Event-Sourcing Transaction Boundaries (SEVERITY: MEDIUM)**
   - Ensuring atomicity between events and projections
   - Error handling when projection logic fails
   - Preventing orphaned events or incomplete projections

### Bug Hiding Locations

**Authentication Context Pollution**: SET LOCAL tenant context not properly isolated between requests. Could leak tenant data across requests in same Edge Function instance.

**Embedding Vector Staleness**: Entities updated but embeddings not regenerated. Semantic search returns outdated results with no clear indication of staleness.

**Grant Revocation Race Conditions**: Edge created just before grant revocation. Database sees valid grant, but by the time edge is created, grant is invalid.

**UUID Timestamp Extraction**: UUIDv7 parsing for temporal ordering. Clock skew or incorrect implementation could break event ordering assumptions.

### Hardest to Test

**Cross-Tenant Security**: Difficult to test all permission combinations. Edge cases in grant inheritance and revocation cascades.

**LLM Extraction Quality**: Non-deterministic responses make consistent testing challenging. Need large test data sets for statistical validation.

**Concurrent Access Patterns**: Edge Functions auto-scale, making race condition testing difficult in development.

### Hardest to Debug in Production

**RLS Policy Violations**: Limited visibility into why queries return empty results. Could be permissions or data issues.

**Embedding Generation Failures**: Async process with limited error visibility. Entities might never get embeddings without clear indication.

**Memory Leaks in Edge Functions**: Function instances may persist longer than expected, accumulating state.

---

## STEP 5: DEVELOPER EXPERIENCE

### Organization Assessment

**✅ Strengths**:
- Clear work package dependency structure
- Comprehensive tool specifications with input/output schemas
- Complete database schema with exact DDL
- Detailed authentication and authorization model

**❌ Weaknesses**:
- Implementation details scattered across multiple sections
- No quick-start guide for setting up development environment
- Missing deployment troubleshooting guide
- No performance benchmarking guidelines

### Can You Find Things?

**Yes, but requires navigation skill**:
- Database schema: Section 2 (clear)
- Tool implementations: Section 3 (well-organized)  
- Auth model: Section 4 (complete)
- Work packages: Section 7 (good structure)

**Navigation challenges**:
- Cross-references between sections not hyperlinked
- Some implementation details in decision log rather than main spec
- Error handling patterns distributed throughout

### What Would Make It Better?

**P1 CRITICAL**:
1. **Code examples** for each work package - Show TypeScript patterns for common operations
2. **Development environment setup** - Complete Supabase + Deno setup instructions
3. **Integration test patterns** - How to test cross-tenant operations locally

**P2 SIGNIFICANT**:
4. **Dependency diagram** - Visual representation of work package dependencies  
5. **Performance testing guide** - How to validate against Supabase limits during development
6. **Configuration reference** - All environment variables and their purposes

**P3 NICE-TO-HAVE**:
7. **Debug logging strategy** - What to log for production troubleshooting
8. **Monitoring recommendations** - Key metrics to track

### Can You Implement in WP Order?

**Mostly yes, with caveats**:
- WP-01 through WP-03 are clearly independent
- WP-04 (Auth) requires some WP-05 (Events) understanding for actor creation
- WP-10 (LLM) depends on WP-07 (Embeddings) but this isn't explicit
- Cross-dependencies between tools not clearly documented

**Backtracking Required**:
- Tool implementations may need to return to auth module for scope additions
- Database indexes may need tuning based on actual query patterns
- Error handling patterns will evolve during implementation

---

## STEP 6: IMPROVEMENT REPORT

### IMPLEMENTER ISSUE [P0 BLOCKER] - Section 3.6, Embedding Pipeline

**What spec says**: "Async embedding generation (non-blocking)" but also "Synchronous embedding generation for gen1"

**What I'd need**: Clear decision on sync vs async. If sync, how do we handle OpenAI API timeouts within Supabase CPU limits?

**Proposed text**: 
```
DECISION: Synchronous embedding generation with timeout fallback

During store_entity and capture_thought operations, attempt embedding generation 
with 1.5-second timeout. If timeout exceeded:
1. Store entity without embedding
2. Queue for background regeneration (store in pending_embeddings table)
3. Continue with operation success

Background regeneration runs on separate Edge Function invocation via database trigger.
```

### IMPLEMENTER ISSUE [P0 BLOCKER] - Section 2.2, UUIDv7 Function

**What spec says**: PostgreSQL-specific implementation using `set_bit`, `int8send`, `gen_random_bytes`

**What I'd need**: Verification that all functions are available in Supabase PostgreSQL. Alternative implementation if functions missing.

**Proposed text**:
```sql
-- Supabase-compatible UUIDv7 implementation
-- Tests for required functions and provides fallback

DO $$ 
BEGIN
    -- Test required functions exist
    PERFORM set_bit('\x00'::bytea, 0, 1);
    PERFORM int8send(1);
    PERFORM gen_random_bytes(8);
EXCEPTION
    WHEN OTHERS THEN
        RAISE EXCEPTION 'Required functions not available. Use fallback implementation.';
END $$;
```

### IMPLEMENTER ISSUE [P1 SIGNIFICANT] - Section 3.3.4, capture_thought CPU Budget

**What spec says**: Full LLM extraction + 3-tier deduplication must complete within Supabase Edge Function limits

**What I'd need**: Breakdown of CPU budget allocation across steps. What to do if budget exceeded?

**Proposed text**:
```
CPU BUDGET ALLOCATION (2000ms total):
- JWT validation + auth: 100ms
- LLM extraction call: 1200ms  
- Deduplication (all tiers): 500ms
- Database transaction: 200ms

If any step exceeds budget:
1. Return partial results with status indicators
2. Queue remaining work for background completion
3. Provide handle for status checking
```

### IMPLEMENTER ISSUE [P1 SIGNIFICANT] - Section 4.7, RLS vs Application Auth

**What spec says**: "Application-level + RLS safety net (Option C)" but RLS policies show complex grant lookups

**What I'd need**: Clear guidance on when to use RLS vs application checks. Performance implications of grant subqueries in RLS.

**Proposed text**:
```
ENFORCEMENT STRATEGY:
- Application layer: All permission checks for performance
- RLS layer: Simple tenant_id isolation only
- Complex grant logic: Application-only, not in RLS policies

RLS policies simplified to:
tenant_id = ANY(current_setting('app.tenant_ids')::uuid[])

Cross-tenant grant validation happens in application code before database calls.
```

### IMPLEMENTER ISSUE [P1 SIGNIFICANT] - Section 7.6, Test Environment Setup

**What spec says**: "Test against seed data" but no instructions for local Supabase setup

**What I'd need**: Complete test environment bootstrap instructions

**Proposed text**:
```
TEST ENVIRONMENT SETUP:

Prerequisites:
- Supabase CLI installed
- Deno 2.0+ installed  
- OpenAI API key

Steps:
1. supabase init
2. supabase start
3. supabase db push
4. deno run scripts/seed-data.ts
5. deno test --allow-net --allow-env

Environment variables for testing:
SUPABASE_URL=http://localhost:54321
SUPABASE_ANON_KEY=(from supabase status)
OPENAI_API_KEY=(your key)
```

### IMPLEMENTER ISSUE [P2 IMPROVEMENT] - Section 3.5, Error Handling

**What spec says**: Standardized error codes E001-E009 but no error handling patterns

**What I'd need**: Consistent error handling patterns across all tools. How to map database errors to MCP errors.

**Proposed text**:
```typescript
// Standard error wrapper for all tools
async function withErrorHandling<T>(
  operation: () => Promise<T>,
  context: string
): Promise<T> {
  try {
    return await operation();
  } catch (error) {
    if (error.code === '23505') { // Unique violation
      throw new MCPError('E004', 'Conflict detected', 409);
    }
    if (error.code === '23503') { // Foreign key violation  
      throw new MCPError('E002', 'Referenced entity not found', 404);
    }
    // ... other mappings
    throw new MCPError('E009', `Internal error in ${context}`, 500);
  }
}
```

### IMPLEMENTER ISSUE [P2 IMPROVEMENT] - Section 2.6, Index Strategy

**What spec says**: Comprehensive index definitions but no guidance on creation order or performance monitoring

**What I'd need**: Index creation strategy for large datasets. How to monitor index performance.

**Proposed text**:
```sql
-- Index creation order for performance
-- 1. Create basic indexes first
CREATE INDEX CONCURRENTLY idx_nodes_tenant_basic ON nodes (tenant_id, is_deleted);

-- 2. Add complex indexes after data load  
CREATE INDEX CONCURRENTLY idx_nodes_embedding_hnsw ON nodes 
  USING hnsw (embedding vector_cosine_ops) 
  WHERE embedding IS NOT NULL AND is_deleted = false;

-- Monitor index usage
CREATE VIEW index_usage_stats AS
SELECT schemaname, tablename, indexname, 
       idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes;
```

### IMPLEMENTER ISSUE [P3 NICE-TO-HAVE] - Section 8.5, Development Workflow

**What spec says**: Work package implementation order but no development workflow guidance

**What I'd need**: Git branching strategy, testing checkpoints, deployment pipeline

**Proposed text**:
```
DEVELOPMENT WORKFLOW:

Branch Strategy:
- main: Production-ready code only
- dev: Integration branch  
- feature/WP-XX: Work package implementation branches

Testing Checkpoints:
- After WP-01: Database tests pass
- After WP-04: Auth tests pass  
- After WP-06: Basic tool tests pass
- After WP-10: Full integration tests pass

Deployment Pipeline:
1. Local testing with Supabase CLI
2. Staging deployment to test project
3. Production deployment with migration scripts
```

---

**AGENT 4C COMPLETE**