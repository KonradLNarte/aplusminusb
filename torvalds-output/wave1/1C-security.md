# Wave 1C — Security & Integrity Analysis
## Analyst: The Security & Integrity Analyst
## Input: Wave 0 Foundation Analyses (3 files) + Auth/Federation/Constraints Specs
## Date: 2026-03-11

## Executive Summary

Resonansia's security architecture demonstrates **sophisticated defense-in-depth** with strong foundational protections, but contains **7 critical vulnerabilities** that could enable data breaches, privilege escalation, or system compromise. The three-layer security model (JWT scopes + RLS policies + grants table) provides excellent baseline security, but **15% of the 47 identified system invariants are fragile** and depend on implementation discipline rather than enforced constraints.

**Critical Findings:**
- **3 CRITICAL vulnerabilities** requiring immediate remediation: grant revocation bypass, propose_event privilege escalation, SET LOCAL bypass potential
- **4 HIGH vulnerabilities** requiring gen1 remediation: dedup timing attack, actor creation atomicity violation, lineage bypass via blobs, cross-tenant edge orphaning  
- **8 MEDIUM vulnerabilities** that could become critical under load: embedding index exposure, concurrent epistemic race, JSON injection potential
- **Proven Security Strengths:** Event sourcing immutability, RLS defense-in-depth, bitemporal consistency, tenant isolation at database level

The **most severe risk** is the grants table bypass scenario where application bugs could expose cross-tenant data within a token's permitted tenant set. The **most likely attack vector** is through malformed JSONB payloads that could bypass validation or cause resource exhaustion.

## Step 1: Foundation Analyses Summary

Based on Wave 0 analysis, the security-critical invariants are:

**Proven Security Foundations (11 invariants):**
- **I-001 Tenant Isolation** — Database RLS + schema design prevents cross-tenant leakage
- **I-002 Append-Only Events** — Immutable audit trail cannot be tampered with
- **I-019 RLS Policy Completeness** — Every table with tenant_id has corresponding RLS policies
- **I-021 Scope Hierarchy** — Broader scopes correctly cover narrower scopes
- **I-009 Event Lineage** — Every fact row traces to its creating event
- **I-015 Epistemic Honesty** — Schema-enforced epistemic states prevent data quality degradation
- **I-016 Federation via Edges** — No cross-tenant FK relationships prevent unauthorized data access
- **I-017 UUIDv7 Ordering** — Cryptographically secure event ordering
- **I-032 JWT Token Lifetime Bounds** — Time-bounded access reduces exposure window
- **I-011 Atomic Event+Projection** — Prevents partial state corruption
- **I-006 Type Nodes Schema** — Self-describing ontology prevents schema corruption

**Fragile Security Points (7 invariants requiring disciplined implementation):**
- **I-005 Cross-Tenant Edges** — Grant validation depends on application code
- **I-022 Cross-Tenant Grant Validation** — No database-level enforcement
- **I-020 Actor Auto-Creation** — Could enable identity pollution attacks
- **I-023 Epistemic Status Transitions** — Validation exists only at application level
- **I-030 Schema Validation Conditionality** — Inconsistent enforcement could allow malformed data
- **I-004 Soft Deletes Only** — Hard deletes could bypass audit trail
- **I-012 Optimistic Concurrency** — Race conditions possible with optional versioning

## Step 2: Threat Model

### External Threats (Malicious AI Agent)

#### CRITICAL: Grant Table Bypass via Application Bug (Scenario A from D-013)
**Attack Vector:** Malicious agent exploits a code path in `explore_graph` or `connect_entities` that fails to call the grants table check before traversing cross-tenant edges.

**Mechanism:** 
1. Agent obtains legitimate token with tenant_ids=[T1,T2] 
2. Discovers application bug where certain edge traversal paths skip grants validation
3. Accesses all T2 nodes without any grant, limited only by token scope

**Impact:** **CRITICAL** — Complete tenant data exposure within token's permitted tenant set. RLS cannot catch this because both T1 and T2 are in the token's tenant_ids.

**Current Protections:** Audit logging records cross-tenant traversals for post-hoc detection.

**Required Mitigation:** 
- Integration tests covering all cross-tenant code paths
- Static analysis to verify grants table queries in all cross-tenant operations
- Runtime assertions that grants check occurred before cross-tenant access

#### HIGH: Malformed JSONB Injection
**Attack Vector:** Crafted input bypassing validation through JSONB field manipulation.

**Mechanisms:**
1. **Schema Bypass:** Entity with `type_node.label_schema` but malformed `data` field that passes Zod validation but fails semantic constraints
2. **Resource Exhaustion:** Multi-MB JSONB payload (no size limit identified) causing memory exhaustion in 150MB Edge Function
3. **Query Injection:** JSONB filter operators (`$eq`, `$in`) potentially allowing SQL injection if not properly parameterized

**Impact:** **HIGH** — Data corruption, resource exhaustion, potential SQL injection

**Current Protections:** Zod schema validation, PostgreSQL parameterized queries

**Required Mitigations:**
- Add JSONB data size limit (1MB recommended)  
- Verify all filter-to-SQL translation uses parameterized queries
- Add semantic validation beyond Zod for critical entity types

#### MEDIUM: Compromised JWT Privilege Escalation
**Attack Vector:** Compromised but valid JWT with excessive tenant_ids used for broader access than intended.

**Mechanism:**
1. Agent token issued with tenant_ids=[T1,T2,T3,T4] due to admin misconfiguration
2. Agent accesses data across all tenants within scope boundaries
3. Scope syntax allows broad access (e.g., "tenant:*:read")

**Impact:** **MEDIUM** — Broad data exposure, but limited to token's declared permissions

**Current Protections:** Token lifetime bounds (1h for users, 24h for agents), audit logging

**Required Mitigations:**
- Alert when token has >3 tenant_ids (per D-013)
- Consider per-tenant token issuance in gen2
- Implement principle of least privilege in scope assignment

#### MEDIUM: Cross-Tenant Edge Metadata Exposure  
**Attack Vector:** Malicious tenant creates cross-tenant edges with sensitive data in edge.data field, then grant is revoked but edge persists.

**Mechanism:**
1. Tenant A grants TRAVERSE on node N to Tenant B  
2. Tenant B creates edge with sensitive data: `{source: B_node, target: A_node, data: {secret: "sensitive_info"}}`
3. Tenant A revokes the grant
4. Edge remains in database with sensitive data accessible via direct queries

**Impact:** **MEDIUM** — Sensitive metadata persists after authorization revoked

**Current Protections:** Application-level access control prevents query-time access

**Required Mitigation:** 
- Add `grant_id` FK on edges for traceability
- Define grant revocation semantics (cascade deletion vs access-only restriction)

### Internal Threats (Implementation Bugs)

#### CRITICAL: SET LOCAL Bypass (Scenario C from D-013)
**Attack Vector:** Code path reaches database query without setting `app.tenant_ids`, allowing access to previous request's tenants.

**Mechanism:**
1. Request A sets `app.tenant_ids = '{T1,T2}'`
2. Request B on same connection skips SET LOCAL due to implementation bug
3. Request B inherits T1,T2 access without authorization

**Impact:** **CRITICAL** — Unauthorized cross-tenant access

**Current Protections:** SET LOCAL scopes to transaction (auto-resets after COMMIT)

**Required Mitigations:**
- Mandatory integration test: verify request without SET LOCAL receives zero rows
- Middleware-level enforcement: abort request if GUC not set
- Use SET LOCAL (transaction-scoped) not SET (session-scoped)

#### HIGH: Deduplication Timing Attack
**Attack Vector:** Race condition in `capture_thought` deduplication when entities lack embeddings.

**Mechanism:**
1. Agent A creates entity "Johan Eriksson" at time T
2. Agent B calls `capture_thought` mentioning "Johan Eriksson" at time T+100ms  
3. Embedding for original entity not yet generated (async process)
4. Dedup Tier 2 (embedding similarity) fails to find existing entity
5. Duplicate entity created

**Impact:** **HIGH** — Data quality degradation, potential information disclosure through duplicate entities

**Current Protections:** Tier 1 exact name matching (ILIKE on data->>'name')

**Required Mitigations:**
- Make embedding synchronous for dedup-critical paths
- Add Tier 1.5 fuzzy matching (trigram similarity)
- Document dedup limitations and eventual consistency model

#### HIGH: Actor Creation Atomicity Violation
**Attack Vector:** Actor node creation in auth middleware creates committed state before tool execution.

**Mechanism:**
1. New agent's first request triggers actor node creation in auth middleware
2. Auth middleware commits actor creation successfully  
3. Actual tool call fails due to validation error
4. Actor node exists without any associated action events

**Impact:** **HIGH** — Audit trail inconsistency, potential for orphaned actor nodes

**Current Protections:** Actor creation is idempotent

**Required Mitigation:** 
- Move actor creation into tool transaction
- Or accept and document this as known audit imprecision

#### MEDIUM: Projection Logic Bypass via propose_event  
**Attack Vector:** `propose_event` potentially bypasses tool-level validation for standard intent_types.

**Mechanism:**
1. Agent calls `propose_event(intent_type="entity_created", payload={...})` directly
2. Unclear whether this runs full `store_entity` validation pipeline
3. May create events without proper schema validation, scope checking, or deduplication

**Impact:** **MEDIUM** — Validation bypass, potential for malformed entities

**Current Protections:** Intent type CHECK constraint limits available values

**Required Mitigation:** 
- Clarify specification: does propose_event run full tool validation?
- If yes, document explicitly; if no, restrict to custom intent_types only

### Concurrency Threats

#### HIGH: Time-of-Check-Time-of-Use in store_entity
**Attack Vector:** Epistemic status check occurs before transaction, allowing race condition.

**Mechanism:**
1. Agent A reads entity with epistemic="hypothesis"
2. Agent B updates same entity to epistemic="confirmed" 
3. Agent A's transaction begins with stale epistemic decision
4. Wrong intent_type chosen (entity_updated vs epistemic_change)

**Impact:** **HIGH** — Incorrect audit trail, potential logical inconsistency

**Current Protections:** UPDATE with WHERE clause detects concurrent changes (returns CONFLICT)

**Required Mitigation:**
- Move epistemic change detection inside transaction
- Or document as acceptable audit imprecision

#### MEDIUM: Concurrent capture_thought Entity Duplication
**Attack Vector:** Two concurrent `capture_thought` calls extract the same entity.

**Mechanism:**
1. Both agents call `capture_thought` mentioning "Johan Eriksson"
2. Both run deduplication search pre-transaction
3. Both find no existing entity (neither has committed yet)
4. Both create separate entities with different node_ids

**Impact:** **MEDIUM** — Data quality degradation

**Current Protections:** EXCLUDE constraint prevents same node_id overlaps but not different node_ids

**Required Mitigation:**
- Advisory locks on entity names during deduplication
- Or accept as known limitation with eventual cleanup process

#### MEDIUM: Grant Revocation During Cross-Tenant Read
**Attack Vector:** Grant revoked mid-traversal in `explore_graph`.

**Mechanism:**
1. `explore_graph` starts traversal with valid grant to depth 1
2. Grant revoked between depth 1 and depth 2 queries
3. Partial results returned (depth 1 includes cross-tenant node, depth 2 fails)

**Impact:** **MEDIUM** — Information disclosure of first-level cross-tenant data

**Current Protections:** Each traversal step re-checks grants

**Acceptable Risk:** Partial results are not incorrect, just incomplete

### Failure Mode Threats

#### CRITICAL: Database Connection Pool Exhaustion
**Attack Vector:** Resource exhaustion leading to authentication bypass.

**Mechanism:**
1. High concurrent load exhausts PostgreSQL connection pool
2. New requests fail to establish database connections
3. Error handling may fail open rather than closed
4. Requests proceed without RLS enforcement

**Impact:** **CRITICAL** — Complete security bypass

**Current Protections:** Supabase handles connection pooling

**Required Mitigations:**
- Fail-closed error handling: no database connection = deny request
- Connection pool monitoring and alerting
- Rate limiting at Edge Function level

#### HIGH: Embedding API Timeout Exploitation
**Attack Vector:** Embedding API failure during `store_entity` leads to inconsistent state.

**Mechanism:**
1. Entity created successfully in transaction
2. Async embedding generation fails (API timeout, rate limit)
3. Entity exists without embedding, excluded from semantic search
4. Attacker creates entities that are "invisible" to semantic discovery

**Impact:** **HIGH** — Data integrity violation, potential information hiding

**Current Protections:** Embedding failure doesn't affect entity creation success

**Required Mitigation:**
- Retry mechanism with exponential backoff
- Monitor embedding generation success rate
- Periodic batch job to regenerate missing embeddings

#### MEDIUM: Edge Function Cold Start State Confusion
**Attack Vector:** Cold start leads to stale authentication state.

**Mechanism:**
1. Edge Function cold after idle period
2. Initialization code may have different auth context than runtime
3. First request after cold start may have inconsistent security context

**Impact:** **MEDIUM** — Authentication bypass on first request

**Current Protections:** Stateless JWT validation doesn't depend on persistent state

**Required Mitigation:**
- Integration tests for cold start scenarios
- Explicit auth state validation on each request

## Step 3: Protection Analysis

### Existing Strong Protections

#### Database-Level Security (PROVEN)
1. **RLS Policies** — Every table with tenant_id has RLS enabled using `current_setting('app.tenant_ids')::uuid[]`
   - **Protection:** Even with application bugs, RLS prevents access outside token's tenant_ids
   - **Limitation:** Cannot enforce node-level grants (grants table must be checked in application)

2. **Immutable Events Table** — RLS policies deny UPDATE/DELETE on events
   - **Protection:** Audit trail cannot be tampered with even by compromised application
   - **Coverage:** Complete mutation history preserved

3. **Bitemporal Consistency** — EXCLUDE constraint prevents overlapping time ranges  
   - **Protection:** Prevents temporal paradoxes and ensures consistent versioning
   - **Mechanism:** `EXCLUDE USING gist (node_id WITH =, tstzrange(valid_from, valid_to) WITH &&)`

4. **Foreign Key Integrity** — All fact tables reference creating event
   - **Protection:** Enables lineage verification and prevents orphaned data
   - **Exception:** Blobs table lacks `created_by_event` FK (identified gap)

#### Application-Level Security (STRONG)  
1. **JWT Validation** — OAuth 2.1 compliant token validation
   - **Protection:** Cryptographically verified identity and scopes
   - **Limitations:** Stateless tokens have revocation lag (1h for users, 24h for agents)

2. **Scope Hierarchy** — Broader scopes cover narrower scopes correctly  
   - **Protection:** Prevents privilege escalation through scope manipulation
   - **Example:** "tenant:T1:write" correctly covers "tenant:T1:nodes:lead:write"

3. **Three-Tier Deduplication** — Exact → Embedding → LLM disambiguation
   - **Protection:** Prevents most duplicate entity creation
   - **Limitation:** Timing window vulnerability for entities without embeddings

4. **Schema Validation** — Zod validation + optional type node label_schema
   - **Protection:** Prevents most malformed data injection
   - **Limitation:** No size limits on JSONB payloads

### Protection Gaps Requiring Mitigation

#### Authentication & Authorization
1. **Cross-Tenant Grant Validation** — Application-level only, no database enforcement
   - **Risk:** Application bugs can bypass grants checks entirely
   - **Mitigation Needed:** Integration tests + static analysis + runtime assertions

2. **Actor Identity Pollution** — Auto-creation without validation
   - **Risk:** Malicious agents could create many fake actor identities
   - **Mitigation Needed:** Rate limiting + actor validation + cleanup procedures

3. **Scope Validation Granularity** — Limited to tenant and type level
   - **Risk:** Cannot enforce field-level or record-level permissions
   - **Acceptable for gen1:** Node-level grants provide adequate granularity

#### Data Integrity  
1. **JSONB Size Limits** — No constraints on data field size
   - **Risk:** Resource exhaustion via large payloads
   - **Mitigation Needed:** Add CHECK constraint limiting JSONB size to 1MB

2. **Embedding Index Pollution** — HNSW includes deleted/historical rows
   - **Risk:** Embedding search quality degradation + resource waste
   - **Mitigation Needed:** Partial index with WHERE clause filtering active rows only

3. **Event Sourcing Consistency** — No event replay mechanism
   - **Risk:** If projection logic has bugs, fact tables cannot be reconstructed
   - **Acceptable for gen1:** Manual SQL correction is sufficient

#### Concurrency & Race Conditions
1. **Optimistic Concurrency** — Optional expected_version parameter
   - **Risk:** Last-write-wins conflicts without explicit version checking
   - **Acceptable for gen1:** Single-agent-per-entity expected usage pattern

2. **Grant State Consistency** — Grants checked at traversal time, not creation time
   - **Risk:** Time-of-check-time-of-use gaps in cross-tenant operations
   - **Mitigation Needed:** Document acceptable partial results behavior

### Attack Scenario Analysis (D-013)

#### Scenario A: Grants Bypass (CRITICAL)
- **Current Protection:** Audit logging records all cross-tenant traversals
- **Gap:** RLS cannot detect this - both tenants are in token's tenant_ids
- **Required:** Application-level prevention (tests, static analysis, runtime checks)

#### Scenario B: Excessive tenant_ids (MEDIUM)  
- **Current Protection:** Token lifetime bounds, audit logging
- **Gap:** No runtime detection of over-privileged tokens
- **Required:** Alert when tenant_ids >3, consider per-tenant tokens in gen2

#### Scenario C: SET LOCAL Skip (CRITICAL)
- **Current Protection:** SET LOCAL auto-resets after transaction
- **Gap:** Connection pooling could leak state between requests  
- **Required:** Integration test + middleware enforcement

## Step 4: Authorization Model

### Trust Levels

#### Level 1: Unauthenticated (No JWT)
- **Access:** None - all requests rejected with 401 Unauthorized
- **Use Case:** Invalid for any operations
- **Error Response:** `{"error": "missing_authorization_header", "code": 401}`

#### Level 2: Authenticated Single-Tenant (JWT with one tenant_id)
- **Access:** Read/write within single tenant, subject to scopes
- **Typical Scopes:** `["tenant:T1:read", "tenant:T1:write"]`
- **RLS Context:** `SET LOCAL 'app.tenant_ids' = '{T1}'`
- **Cross-Tenant:** Not possible - no other tenants in token

#### Level 3: Authenticated Multi-Tenant (JWT with multiple tenant_ids)  
- **Access:** Operations across multiple tenants, subject to scopes + grants
- **Typical Scopes:** `["tenant:T1:write", "tenant:T2:read", "tenant:T3:read"]`
- **RLS Context:** `SET LOCAL 'app.tenant_ids' = '{T1,T2,T3}'`
- **Cross-Tenant:** Subject to grants table validation

#### Level 4: System-Level (service_role or metatype bootstrap)
- **Access:** Bypass RLS, direct database operations  
- **Use Case:** Migration scripts, metatype bootstrap
- **Security:** Requires direct database access, not via Edge Functions

### Per-Operation Authorization Matrix

| Tool | Required Scope | Cross-Tenant Check | Validation Steps |
|------|---------------|-------------------|------------------|
| **store_entity** | `tenant:{tid}:write` OR `tenant:{tid}:nodes:{type}:write` | N/A | 1. Scope check<br>2. Type node resolution<br>3. Schema validation<br>4. Optimistic concurrency check |
| **find_entities** | `tenant:{tid}:read` OR `tenant:{tid}:nodes:{type}:read` | N/A | 1. Scope check<br>2. Silent type filtering for type-scoped tokens |
| **connect_entities** | `tenant:{source_tid}:write` AND (if cross-tenant) `tenant:{target_tid}:read` | YES | 1. Scope check on source<br>2. Scope check on target (if cross-tenant)<br>3. Grants table check for target node<br>4. Edge type resolution |
| **explore_graph** | `tenant:{start_tid}:read` | YES | 1. Scope check on start node<br>2. Per-hop grants validation<br>3. Silent filtering of inaccessible nodes |
| **remove_entity** | `tenant:{tid}:write` | N/A | 1. Scope check<br>2. Entity existence + ownership verification |
| **query_at_time** | `tenant:{tid}:read` | N/A | 1. Scope check<br>2. Temporal bounds validation |
| **get_timeline** | `tenant:{tid}:read` | N/A | 1. Scope check<br>2. Stream access validation |
| **capture_thought** | `tenant:{tid}:write` | N/A | 1. Scope check<br>2. LLM content validation<br>3. Deduplication process |
| **get_schema** | `tenant:{tid}:read` | N/A | 1. Scope check<br>2. Type node enumeration |
| **get_stats** | `tenant:{tid}:read` | N/A | 1. Scope check<br>2. Aggregation bounds |
| **propose_event** | `tenant:{tid}:write` | Depends on intent_type | 1. Scope check<br>2. Intent type validation<br>3. **UNCLEAR:** Full tool validation? |
| **verify_lineage** | `tenant:{tid}:read` | N/A | 1. Scope check<br>2. Entity access validation |
| **store_blob** | `tenant:{tid}:write` | N/A | 1. Scope check<br>2. Related entity validation (if provided) |
| **get_blob** | `tenant:{tid}:read` | N/A | 1. Scope check<br>2. Blob ownership validation |
| **lookup_dict** | `tenant:{tid}:read` | N/A | 1. Scope check<br>2. Dict type validation |

### Authorization Failure Response Matrix

| Failure Type | Error Code | HTTP Status | Response |
|-------------|------------|-------------|----------|
| **Missing JWT** | `E401` | 401 | `{"error": "missing_authorization_header", "message": "Authorization header required"}` |
| **Invalid JWT** | `E401` | 401 | `{"error": "invalid_token", "message": "Token signature invalid or expired"}` |
| **Insufficient Scope** | `E001` | 403 | `{"error": "AUTH_DENIED", "message": "Insufficient scope for operation", "required": "tenant:T1:write", "actual": ["tenant:T1:read"]}` |
| **Cross-Tenant No Grant** | `E001` | 403 | `{"error": "AUTH_DENIED", "message": "No grant for cross-tenant access", "target_node": "uuid", "capability": "TRAVERSE"}` |
| **Entity Not Found** | `E002` | 404 | `{"error": "NOT_FOUND", "message": "Entity not found or not accessible"}` |
| **Type-Scoped Filtering** | Success | 200 | Silent filtering - return accessible subset without error |

### Grant Model Deep Dive

#### Grant Creation and Management
- **Creation:** No dedicated MCP tool - must use `propose_event(intent_type="grant_created")` or direct DB
- **Validation:** Created grants must reference existing tenants and nodes
- **Temporal:** Grants have `valid_from`/`valid_to` for time-bounded access  
- **Immutable:** Grants cannot be updated - must revoke and recreate

#### Grant Capabilities
1. **READ** — View node data and metadata
2. **WRITE** — Modify node (requires READ on source tenant)  
3. **TRAVERSE** — Follow edges from/to this node in graph traversals

#### Grant Delegation
- **Gen1:** No delegation - grants cannot be re-granted
- **Question for Gen2:** Can tenant B grant access to nodes that tenant A granted to B?

#### Grant Revocation Effects
- **Immediate:** Cross-tenant queries fail grants check at traversal time
- **Existing Edges:** Remain in database but become inaccessible via query tools
- **Audit Trail:** Grant revocation creates `grant_revoked` event

#### Grant Resolution Algorithm
```
function check_cross_tenant_access(token, source_node, target_node, capability):
  if source_node.tenant_id == target_node.tenant_id:
    return true  # Same tenant - no grant required
  
  if target_node.tenant_id not in token.tenant_ids:
    return false  # Target tenant not in token scope
    
  grant = query(`
    SELECT 1 FROM grants 
    WHERE subject_tenant_id = $1 
      AND object_node_id = $2 
      AND capability IN ($3, 'WRITE')  -- WRITE implies READ/TRAVERSE
      AND is_deleted = false 
      AND valid_from <= now() 
      AND valid_to > now()
  `, [source_node.tenant_id, target_node.node_id, capability])
  
  return grant.exists()
```

## Step 5: Data Integrity Under Failure

### Write Interruption Scenarios

#### Event Written, Projection Fails
**Scenario:** Event INSERT succeeds but subsequent fact table INSERT fails within same transaction.

**Protection:** Transaction atomicity ensures both operations commit or both rollback together.

**Verification:** INV-ATOMIC is enforced by transaction boundaries in all projection logic.

**Recovery:** Not needed - atomicity guarantees consistency.

#### Partial Transaction Commit in Distributed System
**Scenario:** In a truly distributed setup, transaction coordinator might fail between event and fact commits.

**Current Risk:** **LOW** - Supabase PostgreSQL is single-node, transactions are local.

**Future Risk:** If migrating to distributed database, this becomes critical concern requiring 2PC or saga pattern.

#### Projection Logic Bug Creates Inconsistent State
**Scenario:** Event stored correctly but projection logic has bug that writes wrong values to fact table.

**Detection:** `verify_lineage` tool can detect missing events but not incorrect data values.

**Recovery:** No automatic recovery mechanism in gen1. Manual SQL correction required.

**Prevention:** Comprehensive integration tests for all projection logic.

### Partial State Read Scenarios

#### Node Exists but Type Node Missing  
**Scenario:** Node references `type_node_id` that was soft-deleted or not accessible due to RLS.

**Protection:** Type node resolution searches both tenant and system tenant. Bootstrap ensures metatype always exists.

**Failure Mode:** If type node is hard-deleted (violates INV-SOFT), node becomes uninterpretable.

**Mitigation:** Forbid hard deletion of type nodes, even under GDPR (replace data with tombstone).

#### Edge Exists but Endpoint Soft-Deleted
**Scenario:** Edge remains active while source or target node is soft-deleted.

**Current Behavior:** `explore_graph` silently skips edges to deleted nodes. `get_timeline` may include them.

**Consistency:** This is correct behavior - edges become "dangling" but remain for audit purposes.

#### Grant Exists but Target Tenant Removed
**Scenario:** Grant references tenant that was deleted.

**Current Protection:** FK constraint prevents tenant deletion while grants exist.

**Edge Case:** If grants are soft-deleted but tenant is hard-deleted, grants become meaningless.

**Mitigation:** Cascade grant deletion when tenant is deleted.

### System Crash Recovery

#### Supabase Edge Function Crash Mid-Operation
**Database State:** PostgreSQL transactions provide ACID guarantees. If Edge Function crashes before COMMIT, all changes rollback automatically.

**In-Memory Queues:** No persistent queues in gen1 - embedding generation failure doesn't affect entity creation.

**Recovery Process:** 
1. Next request creates new Edge Function instance (stateless)
2. Database is in consistent state due to transaction rollback
3. Failed embedding generation is retried via background process

#### Cold Start After Crash
**Authentication State:** Stateless JWT validation doesn't depend on persistent state.

**Database Connections:** New function instance establishes fresh connections.

**Risk:** First request after cold start may have slightly higher latency but no consistency issues.

#### Network Partition Between Edge Function and Database
**Scenario:** Edge Function loses database connectivity mid-transaction.

**Protection:** Database connection timeout causes transaction rollback.

**Client Experience:** Request returns 500 error, client should retry.

**Data Consistency:** Maintained - no partial commits possible.

### Data Loss Scenarios

#### Event Loss Analysis
**Immutability:** Events are append-only and immutable (INV-APPEND).

**Storage:** PostgreSQL WAL + streaming replication (Supabase handles this).

**Risk Assessment:** **VERY LOW** - would require PostgreSQL data loss, which is outside system boundary.

**Mitigation:** Trust Supabase backup and replication policies.

#### Embedding Loss Recovery  
**Scenario:** Embeddings table corruption or deletion.

**Recovery Capability:** **FULL RECOVERY POSSIBLE** - embeddings are derived data.

**Process:**
1. Query all nodes where embedding IS NULL
2. Regenerate embeddings via OpenAI API  
3. Batch update embedding vectors

**Cost:** Embedding regeneration cost scales with node count (~$0.02 per 1K nodes).

#### Audit Log Durability
**Mutation Audit:** Stored in immutable events table - same durability as events.

**Read Audit:** Logged to Edge Function stdout - durability depends on log aggregation system.

**Risk:** Read audit logs could be lost if not properly aggregated.

**Mitigation:** Ensure log aggregation system has appropriate retention and backup policies.

#### Blob Storage Data Loss
**Metadata:** Stored in database blobs table - same durability as other database data.

**Binary Data:** Stored in Supabase Storage (S3-compatible) - separate durability guarantees.

**Orphan Risk:** Upload-first pattern means blob binary could exist without metadata if database fails after upload.

**Recovery:** Orphaned binaries waste storage but don't affect system consistency. Periodic cleanup job can identify and remove orphans.

## Step 6: The Integrity Report

### 1. Threat Catalogue (Prioritized)

#### CRITICAL Threats (Immediate Remediation Required)

**C1: Grant Table Bypass via Application Bug**
- **Severity:** CRITICAL
- **Likelihood:** MEDIUM (implementation-dependent)
- **Impact:** Complete tenant data exposure within token scope
- **Remediation Timeline:** Before production deployment
- **Specific Mitigations:**
  - Integration test coverage for all cross-tenant code paths
  - Static analysis rules requiring grants table queries
  - Runtime assertions validating grants check occurred

**C2: SET LOCAL Bypass Vulnerability**  
- **Severity:** CRITICAL
- **Likelihood:** LOW (requires specific implementation bug)
- **Impact:** Unauthorized cross-tenant access via state leakage
- **Remediation Timeline:** Before production deployment
- **Specific Mitigations:**
  - Mandatory integration test: verify zero rows without SET LOCAL
  - Middleware enforcement: abort if GUC not set
  - Code review checklist item for SET LOCAL usage

**C3: Database Connection Pool Exhaustion**
- **Severity:** CRITICAL  
- **Likelihood:** LOW (depends on load patterns)
- **Impact:** Security bypass via fail-open error handling
- **Remediation Timeline:** Before high-load deployment
- **Specific Mitigations:**
  - Fail-closed error handling patterns
  - Connection pool monitoring and alerting
  - Rate limiting at Edge Function level

#### HIGH Threats (Gen1 Remediation Required)

**H1: Deduplication Timing Attack**
- **Severity:** HIGH
- **Likelihood:** MEDIUM (timing-dependent)
- **Impact:** Data quality degradation, duplicate entity creation
- **Remediation Timeline:** Gen1.1
- **Specific Mitigations:**
  - Implement Tier 1.5 fuzzy matching (trigram similarity)
  - Advisory locks during deduplication process
  - Document eventual consistency model

**H2: JSONB Resource Exhaustion**
- **Severity:** HIGH
- **Likelihood:** MEDIUM (easy to exploit)
- **Impact:** Function memory exhaustion, DoS
- **Remediation Timeline:** Gen1.1  
- **Specific Mitigations:**
  - Add CHECK constraint limiting JSONB to 1MB
  - Request size validation at Edge Function entry point

**H3: Actor Creation Atomicity Violation**
- **Severity:** HIGH
- **Likelihood:** LOW (requires specific failure timing)
- **Impact:** Audit trail inconsistency
- **Remediation Timeline:** Gen2
- **Specific Mitigations:**
  - Move actor creation into tool transaction
  - Document as acceptable audit imprecision if deferring fix

**H4: Cross-Tenant Edge Orphaning**
- **Severity:** HIGH
- **Likelihood:** MEDIUM (normal operation after grant revocation)
- **Impact:** Sensitive metadata persists after authorization revoked
- **Remediation Timeline:** Gen1.1
- **Specific Mitigations:**
  - Add grant_id FK to edges table for traceability
  - Define grant revocation cascade policy

#### MEDIUM Threats (Monitor and Plan)

**M1: Embedding Index Pollution**
- **Severity:** MEDIUM
- **Likelihood:** HIGH (guaranteed over time)  
- **Impact:** Search quality degradation, resource waste
- **Specific Mitigation:** Partial HNSW index excluding deleted/historical rows

**M2: Concurrent Epistemic Race**
- **Severity:** MEDIUM
- **Likelihood:** MEDIUM (concurrent operations)
- **Impact:** Incorrect audit event types
- **Specific Mitigation:** Move epistemic check inside transaction

**M3: Propose Event Validation Bypass**
- **Severity:** MEDIUM
- **Likelihood:** LOW (requires malicious agent)
- **Impact:** Validation circumvention
- **Specific Mitigation:** Clarify specification and restrict to custom intent_types

#### LOW Threats (Acceptable Risk)

**L1: Grant Revocation TOCTOU**
- **Severity:** LOW
- **Likelihood:** MEDIUM
- **Impact:** Partial information disclosure
- **Mitigation:** Document as acceptable behavior

**L2: Cold Start Authentication Confusion**  
- **Severity:** LOW
- **Likelihood:** LOW
- **Impact:** Single request authentication bypass
- **Mitigation:** Integration tests for cold start scenarios

### 2. Existing Protections Summary

#### Database-Level (Proven Strong)
- ✅ RLS policies on all tenant-scoped tables
- ✅ Immutable events table (UPDATE/DELETE denied)
- ✅ Bitemporal consistency via EXCLUDE constraints  
- ✅ Foreign key integrity for event lineage
- ✅ Transaction atomicity for event+projection writes

#### Application-Level (Strong with Gaps)
- ✅ JWT validation with cryptographic verification
- ✅ Scope hierarchy correctly implemented
- ✅ Three-tier entity deduplication
- ✅ Schema validation via Zod + type nodes
- ❌ Cross-tenant grant validation (application-only, no DB enforcement)
- ❌ Resource limits (no JSONB size constraints)

#### Operational (Good Foundation)
- ✅ Audit logging for all operations
- ✅ Time-bounded tokens (1h users, 24h agents)
- ✅ Fail-safe defaults (deny access on error)
- ❌ Monitoring and alerting for security events
- ❌ Automated threat detection

### 3. Required Additions (Implementation Priority)

#### Priority 1: Pre-Production (Security Critical)
1. **Integration Test Suite for Cross-Tenant Paths**
   - Test all code paths involving grants table validation
   - Verify SET LOCAL enforcement across request types
   - Test failure modes (missing grants, revoked grants)

2. **Resource Limit Enforcement**
   ```sql
   ALTER TABLE nodes ADD CONSTRAINT check_data_size 
   CHECK (pg_column_size(data) <= 1048576); -- 1MB limit
   ```

3. **Runtime Security Assertions**
   - Middleware to verify app.tenant_ids is set before queries
   - Assert grants check occurred for cross-tenant operations
   - Fail-closed error handling for database connection issues

#### Priority 2: Gen1.1 (Data Integrity)  
1. **Enhanced Deduplication**
   - Implement trigram similarity for fuzzy name matching
   - Add advisory locks during deduplication process
   - Background job to detect and merge duplicate entities

2. **Audit Trail Completion**
   - Add created_by_event to blobs table
   - Implement dict creation tool with event sourcing
   - Monitor embedding generation success rate

3. **Operational Security**
   - Alert when token contains >3 tenant_ids
   - Monitor cross-tenant operation frequency
   - Automate detection of potential privilege escalation

#### Priority 3: Gen2 (Architecture Evolution)
1. **Enhanced Authorization**
   - Per-tenant token issuance to reduce blast radius
   - Field-level access control for sensitive data
   - Delegatable grants for trusted partner scenarios

2. **Advanced Threat Detection**
   - Behavioral analysis of agent access patterns
   - Anomaly detection for unusual cross-tenant access
   - Automated response to potential security incidents

### 4. Authorization Model Specification

See Section 4 above for complete authorization model including:
- 4 trust levels with specific access patterns
- 15-tool authorization matrix with required scopes
- Grant resolution algorithm with capability checking
- Failure response specifications with error codes

### 5. Failure Recovery Procedures

#### Database Connection Failure
1. **Detection:** Database query timeout or connection error
2. **Response:** Immediate request termination with 503 Service Unavailable  
3. **Recovery:** Automatic retry with exponential backoff
4. **Escalation:** Alert after 3 consecutive failures

#### Authentication System Failure  
1. **Detection:** JWT validation failure due to external service unavailability
2. **Response:** Deny all requests (fail-closed)
3. **Recovery:** Retry JWT validation with cached issuer keys if available
4. **Escalation:** Alert immediately - authentication is critical path

#### Cross-Tenant Grant Verification Failure
1. **Detection:** Grants table query timeout or error
2. **Response:** Deny cross-tenant operation (fail-closed)
3. **Recovery:** Retry with reduced timeout
4. **Fallback:** Allow same-tenant operations only

#### Event Sourcing Consistency Violation
1. **Detection:** verify_lineage tool reports orphaned fact rows
2. **Response:** Investigation required - potential security breach
3. **Recovery:** Manual reconciliation of events to fact tables
4. **Prevention:** Enhanced integration testing of projection logic

#### Embedding System Degradation  
1. **Detection:** Embedding generation failure rate >10%
2. **Response:** Continue core operations (entities work without embeddings)
3. **Recovery:** Background retry with exponential backoff
4. **Escalation:** Alert if failure rate >50% for >1 hour

### 6. Security Recommendations (Priority Order)

#### Immediate (Pre-Production)
1. **Implement comprehensive integration tests** covering all cross-tenant security boundaries
2. **Add resource limits** to prevent DoS via large payloads
3. **Enhance error handling** to fail-closed in all security-critical paths
4. **Deploy runtime security assertions** to catch implementation bugs

#### Short-term (Gen1.1)
5. **Address deduplication timing vulnerabilities** with fuzzy matching and locking
6. **Complete audit trail** by adding lineage FKs to all fact tables
7. **Implement security monitoring** with automated alerting for anomalous access patterns
8. **Document and test failure recovery procedures** for all identified scenarios

#### Medium-term (Gen2)
9. **Evolve to per-tenant tokens** to reduce blast radius of token compromise
10. **Add behavioral analysis** to detect potential agent compromise or misuse
11. **Implement advanced grant delegation** for trusted partner scenarios
12. **Consider migration to pure RLS** if service_role bypass becomes unacceptable

#### Long-term (Gen3)
13. **Implement cryptographic event chains** for trustless federation verification
14. **Add zero-knowledge proofs** for cross-tenant data sharing without exposure
15. **Deploy automated threat response** with ability to revoke access and isolate threats
16. **Consider confidential computing** for processing sensitive data without exposure

## Conclusion

Resonansia's security architecture provides a **solid foundation** with sophisticated defense-in-depth, but contains **critical vulnerabilities** that must be addressed before production deployment. The three-layer security model (JWT + RLS + Grants) provides excellent baseline protection, but the **15% fragile invariant rate** indicates that implementation discipline is crucial for security.

**The single greatest security risk** is the grants table bypass scenario (C1) where application bugs could expose complete tenant data within a token's scope. This risk is inherent to the application-level enforcement model chosen in D-013 and requires vigilant implementation practices.

**The most practical attack vector** is JSONB resource exhaustion (H2) which is easily exploitable and requires immediate mitigation through resource limits.

The security analysis reveals that Resonansia is **ready for controlled deployment** after addressing the 3 critical vulnerabilities, but will require ongoing security investment to maintain trustworthiness as it scales to production volumes and gains wider adoption.

**AGENT 1C COMPLETE**