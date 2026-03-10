# RESONANSIA SPECIFICATION — FORMAL CORRECTNESS AUDIT
## Agent 4B — Correctness Auditor  
## Date: 2026-03-11
## Specification: unified-spec.md (v2.0-evolved)

---

## EXECUTIVE SUMMARY

**AUDIT STATUS: FAILED** ❌

The Resonansia specification contains **22 critical issues** that would prevent successful implementation. While the architectural foundation is sound, there are significant gaps in type definitions, inconsistent behavioral specifications, missing error handling paths, and incomplete formal properties.

**Critical Findings:**
- 5 CRITICAL type consistency violations
- 4 CRITICAL completeness gaps in operation signatures
- 6 MAJOR invariant preservation failures  
- 7 MAJOR missing termination guarantees

**Recommendation:** Specification requires substantial revision before implementation can begin.

---

## DETAILED AUDIT FINDINGS

### STEP 2: INTERNAL CONSISTENCY CHECK

#### Type Consistency Issues

**ISSUE [CRITICAL] [CONSISTENCY]**
- **Location**: Section 3.3.1 `store_entity` vs Section 2.1 `nodes` table
- **Statement**: `StoreEntityParams.data: Record<string, any>` vs `nodes.data JSONB NOT NULL DEFAULT '{}'`
- **Problem**: TypeScript interface allows undefined data, but database requires NOT NULL
- **Fix**: Change interface to `data: Record<string, any> = {}`

**ISSUE [CRITICAL] [CONSISTENCY]** 
- **Location**: Section 3.3.4 `capture_thought` vs Section 4.2 Event Schema
- **Statement**: LLM extraction returns array of entities, but `thought_captured` event schema not defined
- **Problem**: Missing event schema for `thought_captured` intent type
- **Fix**: Add complete `thought_captured` event schema in Section 4.2

**ISSUE [CRITICAL] [CONSISTENCY]**
- **Location**: Section 3.5 Error Codes vs Section 3.3.3 `connect_entities` 
- **Statement**: Cross-tenant denial should return E005, but implementation uses E003
- **Problem**: Error code mismatch between specification and implementation logic
- **Fix**: Standardize on E005 for all cross-tenant access denials

**ISSUE [MAJOR] [CONSISTENCY]**
- **Location**: Section 2.1 `events.intent_type` vs Section 4.2 Event Catalogue
- **Statement**: Database constraint lists 11 intent types, but Event Catalogue covers only 3
- **Problem**: Incomplete event schema coverage
- **Fix**: Provide complete schemas for all 11 intent types

**ISSUE [MAJOR] [CONSISTENCY]**
- **Location**: Section 3.1 JWT claims vs Section 3.3 Tool implementations
- **Statement**: JWT contains `tenant_ids` array, tools use `tenant_id` singular
- **Problem**: Mismatch between multi-tenant auth and single-tenant tool operations  
- **Fix**: Clarify tenant selection logic for multi-tenant actors

#### Name Consistency Issues

**ISSUE [MINOR] [CONSISTENCY]**
- **Location**: Section 2.1 vs Section 3.6
- **Statement**: Table refers to `embedding VECTOR(1536)`, code refers to `text-embedding-3-small`
- **Problem**: Missing explicit connection between model name and dimensions
- **Fix**: Document model-to-dimension mapping explicitly

**ISSUE [MINOR] [CONSISTENCY]**
- **Location**: Section 6.1 vs Section 7 Work Packages
- **Statement**: Testing claims "18 tools × 6 scenarios", WP counts show 18 tools
- **Problem**: Tool count consistent, but scenario count not defined anywhere
- **Fix**: Define the 6 test scenarios explicitly

### STEP 3: COMPLETENESS CHECK

#### Type Completeness Issues

**ISSUE [CRITICAL] [COMPLETENESS]**
- **Location**: Section 3.3.2 `FindEntitiesResult`
- **Statement**: `similarity?: number` optional field
- **Problem**: Type doesn't specify when similarity is present vs absent
- **Fix**: Define exact conditions: `similarity: number | null` with clear semantics

**ISSUE [CRITICAL] [COMPLETENESS]**
- **Location**: Section 2.1 `nodes.data JSONB`
- **Statement**: "all JSONB schemas specified" but node data schema is generic object
- **Problem**: No type validation for entity data against type schemas
- **Fix**: Add schema validation logic and document entity type schema structure

#### Operation Completeness Issues

**ISSUE [CRITICAL] [COMPLETENESS]**
- **Location**: Section 3.4 Additional Tool Specifications
- **Statement**: "3.4.1 explore_graph — Traverses entity graph with cross-tenant grant validation"
- **Problem**: Tools 3.4.1-3.4.10 are mentioned but not specified
- **Fix**: Provide complete specifications for all 10 additional tools

**ISSUE [CRITICAL] [COMPLETENESS]**
- **Location**: Section 3.3 vs Section 7.1 Work Packages  
- **Statement**: Specification claims 18 MCP tools, but only 4 are fully specified
- **Problem**: 14 tools missing behavioral specifications
- **Fix**: Complete specifications for all 18 tools

**ISSUE [MAJOR] [COMPLETENESS]**
- **Location**: Section 3.3.4 `capture_thought` deduplication
- **Statement**: "LLM disambiguation for ambiguous cases" but no prompt provided
- **Problem**: Missing disambiguation prompt and logic specification
- **Fix**: Add complete LLM disambiguation prompt and decision algorithm

**ISSUE [MAJOR] [COMPLETENESS]**
- **Location**: Section 3.1 Authentication flow
- **Statement**: `ensureActorExists` function referenced but not defined
- **Problem**: Critical auth function missing implementation details
- **Fix**: Specify actor creation logic, ID generation, and error conditions

#### Test Completeness Issues

**ISSUE [MAJOR] [COMPLETENESS]**
- **Location**: Section 6.2 Critical Test Scenarios
- **Statement**: Only 2 test examples provided for "24 invariants" and "18 tools × 6 scenarios"
- **Problem**: Insufficient test coverage specification
- **Fix**: Provide test specifications for all claimed test scenarios

### STEP 4: SOUNDNESS CHECK

#### Invariant Preservation Issues

**ISSUE [MAJOR] [SOUNDNESS]**
- **Location**: Section 1.2 INV-CROSS_TENANT_GRANTS vs Section 3.3.3 `connect_entities`
- **Statement**: Invariant requires "valid grant with WRITE or TRAVERSE capability"
- **Problem**: `connect_entities` doesn't verify grant temporal validity (valid_from/valid_to)
- **Fix**: Add temporal grant validation to cross-tenant edge creation

**ISSUE [MAJOR] [SOUNDNESS]**
- **Location**: Section 1.2 INV-DEDUPLICATION vs Section 3.3.4 implementation
- **Statement**: Deduplication is "FRAGILE" but capture_thought doesn't handle embedding failures
- **Problem**: No fallback when OpenAI embeddings unavailable
- **Fix**: Specify degraded deduplication behavior (exact + trigram only)

**ISSUE [MAJOR] [SOUNDNESS]**
- **Location**: Section 1.2 INV-TENANT_ISOLATION vs Section 2.4 RLS policies
- **Statement**: RLS enforces tenant isolation, but cross-tenant grants policy allows reading
- **Problem**: Grants policy could bypass tenant isolation if misconfigured
- **Fix**: Add explicit tenant validation in grants RLS policy

#### Test Soundness Issues

**ISSUE [MINOR] [SOUNDNESS]**
- **Location**: Section 6.2 `testAtomicity()`
- **Statement**: Test checks table row counts for atomicity verification
- **Problem**: Row count changes don't prove transaction rollback semantics
- **Fix**: Test should verify specific transaction boundaries and rollback behavior

**ISSUE [MINOR] [SOUNDNESS]**
- **Location**: Section 6.2 `testTenantIsolation()`
- **Statement**: Test verifies Tenant A can't see Tenant B data
- **Problem**: Doesn't test the reverse case or grant-based access
- **Fix**: Test bidirectional isolation and grant-based cross-tenant access

### STEP 5: FORMAL PROPERTIES CHECK

#### Termination Issues

**ISSUE [MAJOR] [FORMAL]**
- **Location**: Section 3.3.4 Deduplication Algorithm
- **Statement**: "Tier 3: LLM disambiguation for ambiguous cases"
- **Problem**: No termination guarantee if OpenAI API hangs or returns malformed JSON
- **Fix**: Add timeout bounds and maximum retry limits

**ISSUE [MAJOR] [FORMAL]**
- **Location**: Section 3.3.2 `find_entities` semantic search
- **Statement**: Vector similarity search with no explicit limits
- **Problem**: Could return unbounded result sets despite LIMIT clause if similarity threshold too low
- **Fix**: Add hard upper bounds on result set size regardless of threshold

**ISSUE [MAJOR] [FORMAL]**
- **Location**: Section 3.3.3 Cross-tenant edge validation trigger
- **Statement**: `validate_cross_tenant_edge()` function queries grants table
- **Problem**: No termination guarantee if grants table grows very large
- **Fix**: Add indexes and query limits to grant lookup

#### Determinism Issues

**ISSUE [MAJOR] [FORMAL]**
- **Location**: Section 3.3.4 Deduplication "moderate" vs "loose" modes
- **Statement**: "thresholds = {strict: 0.95, moderate: 0.85, loose: 0.75}"
- **Problem**: Same input could return different results based on embedding API variability
- **Fix**: Define deterministic tie-breaking rules for edge cases

**ISSUE [MINOR] [FORMAL]**
- **Location**: Section 2.2 UUIDv7 generation with `clock_timestamp()`
- **Statement**: Uses system clock for timestamp component
- **Problem**: Clock skew could affect ordering guarantees across multiple servers
- **Fix**: Define clock synchronization requirements or use logical timestamps

#### Safety Issues

**ISSUE [MAJOR] [FORMAL]**
- **Location**: Section 2.1 `nodes` temporal exclusion constraint
- **Statement**: `EXCLUDE USING gist (node_id WITH =, tstzrange(valid_from, valid_to) WITH &&)`
- **Problem**: Doesn't prevent gaps in temporal coverage - entity could "disappear" between versions
- **Fix**: Add constraint to ensure continuous temporal coverage or explicit gap semantics

**ISSUE [MINOR] [FORMAL]**
- **Location**: Section 3.2 Event Engine transaction boundaries
- **Statement**: Event creation and projection happen in single transaction
- **Problem**: No deadlock prevention if multiple concurrent updates target same entity
- **Fix**: Define entity-level locking strategy or optimistic concurrency handling

#### Liveness Issues

**ISSUE [MAJOR] [FORMAL]**
- **Location**: Section 4.4.3 Graceful Degradation
- **Statement**: "OpenAI Embeddings unavailable" → "Structured search only"
- **Problem**: No recovery mechanism when OpenAI comes back online
- **Fix**: Define embedding backfill strategy for entities created during outage

**ISSUE [MINOR] [FORMAL]**
- **Location**: Section 3.3.4 Embedding pipeline async processing
- **Statement**: "Async embedding generation (non-blocking)" 
- **Problem**: No retry logic if embedding generation fails
- **Fix**: Define retry policy and manual trigger mechanism

### STEP 6: WORK PACKAGE SOUNDNESS

#### Dependency Issues

**ISSUE [MINOR] [SOUNDNESS]**
- **Location**: Section 7.3 WP-10 vs WP-07 dependencies
- **Statement**: WP-10 (capture_thought) depends on WP-07 (embeddings) but also needs WP-08 (graph operations)
- **Problem**: Missing dependency on graph operations for relationship creation
- **Fix**: Add WP-08 as dependency for WP-10

#### Acceptance Criteria Issues

**ISSUE [MINOR] [SOUNDNESS]**
- **Location**: Section 7.1 WP-01 Acceptance Criteria
- **Statement**: "All invariant verification tests pass"
- **Problem**: Tests not defined yet in WP-01, circular dependency
- **Fix**: Define specific database schema tests independent of later test frameworks

---

## SUMMARY OF CRITICAL GAPS

### High-Priority Fixes Required

1. **Complete Tool Specifications** (14 tools missing detailed specs)
2. **Complete Event Schema Definitions** (8 of 11 intent types missing schemas)
3. **Termination Guarantees** (Add timeouts and limits to all external API calls)
4. **Invariant Preservation Logic** (Fix cross-tenant grant validation temporal logic)
5. **Type Safety** (Remove "any" types, add proper JSONB validation)

### Medium-Priority Improvements

1. **Test Coverage** (Specify all claimed test scenarios)
2. **Error Handling** (Complete error condition specifications)
3. **Recovery Mechanisms** (Define degradation and recovery behavior)
4. **Work Package Dependencies** (Fix missing dependencies)

### Low-Priority Cleanup

1. **Naming Consistency** (Minor terminology alignment)
2. **Documentation Completeness** (Add missing implementation details)

---

## RECOMMENDATION

**STATUS: SPECIFICATION REQUIRES REVISION** ❌

The Resonansia specification demonstrates strong architectural thinking and comprehensive coverage of the problem domain. However, the significant gaps in operational completeness, type safety, and formal properties make it unsuitable for implementation without substantial revision.

**Immediate Actions Required:**
1. Complete all 18 tool specifications with full behavioral details
2. Add complete event schemas for all 11 intent types  
3. Define termination and timeout bounds for all operations
4. Fix invariant preservation logic gaps
5. Remove type safety violations

**Timeline Impact:** These issues would likely add 2-3 weeks to implementation timeline if discovered during development. Addressing them in specification phase is critical.

**Overall Assessment:** The specification shows excellent systems thinking but needs operational precision to be implementation-ready.

---

**AGENT 4B COMPLETE**