# RESONANSIA MCP SERVER — GEN 2 SPEC

> Part of [Gen2 Spec](index.md)

## 4. AUTH MODEL

### 4.1 Token structure (OAuth 2.1)

```yaml
claims:
  sub: string          # user or agent identity
  iss: string          # token issuer
  aud: string          # "resonansia-mcp"
  tenant_ids: string[] # quick lookup: which tenants this token can access
  scopes: string[]     # fine-grained permissions (see below)
  exp: int             # expiration timestamp

scope_syntax: "tenant:{tenant_id}:{resource}:{action}"
scope_examples:
  - "tenant:T1:read"                  # read everything in T1
  - "tenant:T1:write"                 # write everything in T1
  - "tenant:T1:nodes:lead:read"       # read only leads in T1
  - "tenant:T1:nodes:lead:write"      # write only leads in T1
  - "tenant:*:read"                   # cross-tenant read (owner)
  - "admin"                           # full access
```

### 4.2 Auth spec version

**Gen1 implements against the MCP Authorization Specification (2025-11-25 revision)**, where:
- The MCP server acts ONLY as an OAuth 2.0 **Resource Server** (not Authorization Server).
- The MCP server MUST implement **RFC 9728** (Protected Resource Metadata).
- The MCP server MUST validate **Resource Indicators** (RFC 8707) — tokens are scoped to specific MCP servers.
- Token issuance is delegated to an external Authorization Server (Supabase Auth for users, custom JWT signing for agents).
- Dynamic Client Registration (RFC 7591) is OPTIONAL (relaxed in Nov 2025 revision).
- The MCP server MUST NOT pass through received tokens to upstream APIs (confused deputy prevention).
- PKCE (SHA-256) is REQUIRED for all public clients.

[RESOLVED:RESEARCH:gen1] MCP auth spec verified at 2025-11-25 revision. Key changes since June 2025:
- Protected Resource Metadata (RFC 9728) added for AuthZ server discovery
- Dynamic Client Registration made optional
- Enterprise-Managed Authorization extension added (not needed for gen1)
- Client ID Metadata Documents as alternative to DCR

References:
- https://modelcontextprotocol.io/specification/draft/basic/authorization
- https://auth0.com/blog/mcp-specs-update-all-about-auth/
- https://stackoverflow.blog/2026/01/21/is-that-allowed-authentication-and-authorization-in-model-context-protocol

### 4.3 Permission resolution (every tool call)

```
1. Extract tenant_id from params OR from token (if single-tenant scoped)
2. Verify token.scopes includes required scope for the operation
3. Cross-tenant operations: verify scopes for ALL involved tenants
4. For cross-tenant edge traversal: also check grants table
5. Database-level RLS enforces isolation even if server code has bugs
6. Every tool call → audit event: { caller, tool, params_hash, result_status }
```

### 4.4 Token types

```yaml
user_token:
  issued_by: platform auth provider
  scopes: based on user role within tenant(s)
  lifetime: short (1h), refreshable

agent_token:
  issued_by: platform admin or API
  scopes: specific to agent's purpose
  lifetime: medium (24h), non-refreshable

partner_token:
  issued_by: tenant admin
  scopes: read-only, specific entity types
  lifetime: long (30d), auto-expires, non-refreshable
```

### 4.5 Grants vs Scopes — two-layer access control

The system has two complementary layers of access control:

- **Scopes** (in JWT tokens): coarse-grained, tenant-level. Fast (in-memory check, no DB). Example: "This agent can read tenant T1." Checked on every tool call.
- **Grants** (in grants table): fine-grained, node-level. Requires DB lookup. Example: "Tenant T2 can TRAVERSE node N1 in tenant T1." Checked for cross-tenant operations.

Both must pass for an operation to succeed. Scopes are the fast outer gate; grants are the fine-grained inner gate.

**Trade-offs of this design:**
- Scopes in JWT mean permission changes require token re-issue (not instant). For gen1 with short-lived tokens (1h) this is acceptable.
- Grant changes take effect immediately (no token re-issue needed) because they are checked at query time.
- Token size grows with scope count. At gen1 scale (< 10 scopes per token) this is not a problem.
- [RESOLVED:gen1 D-014] Scope syntax is sufficient for gen1 (≤10 scopes/token, ~1KB JWT). [DECIDE:gen2] Move to opaque tokens + introspection if scope count exceeds 20 or JWT size exceeds 4KB.

### 4.6 Actor identity model

The `created_by` UUID on every row needs a defined source. This section specifies how actor identities are represented.

```yaml
# ── ACTOR IDENTITY ──
# [RESOLVED:gen1 D-015] Option B: Actors as graph nodes.
# Rationale: consistent with graph-native ontology, makes actors queryable,
# edges between actors and their creations are implicit via created_by.

actor_type_node:
  name: "actor"
  kind: "entity_type"
  label_schema:
    type: object
    required: [name, actor_type, external_id]
    properties:
      name: { type: string, description: "Display name of the actor" }
      actor_type: { type: string, enum: [user, agent, system], description: "Type of actor" }
      external_id: { type: string, description: "JWT sub claim value" }
      purpose: { type: string, description: "What this actor does (for agents)" }
      scopes: { type: array, items: { type: string }, description: "Scopes assigned to this actor" }

mapping_rules:
  jwt_sub_to_actor: |
    1. Extract sub claim from validated JWT
    2. For each tenant_id in token.tenant_ids:
       a. Query: SELECT node_id FROM nodes WHERE tenant_id = $tenant_id
          AND type_node_id = $actor_type_node_id
          AND data->>'external_id' = $sub
          AND is_deleted = false AND valid_to = 'infinity'
       b. If found: use node_id as created_by for this tenant
       c. If NOT found: auto-create actor node via store_entity:
          store_entity(entity_type="actor", data={
            name: sub,  // default name, can be updated later
            actor_type: token.actor_type || "agent",
            external_id: sub
          }, tenant_id=tenant_id)
          Use the returned entity_id as created_by
    3. Cache actor_node_id per (sub, tenant_id) for request lifetime

  created_by_semantics: |
    created_by on ALL rows = actor node's node_id (NOT the JWT sub claim directly).
    This means every row in nodes, edges, grants, events, blobs is traceable to a
    graph-queryable actor entity.

  actor_scope: |
    Actors are PER-TENANT. An agent with multi-tenant access has one actor node per tenant.
    Rationale: actor activity is tenant-scoped, and RLS naturally isolates per tenant.

  bootstrap_actors: |
    The metatype bootstrap (section 2.6.3) creates the system tenant but does NOT create
    actor nodes. Actor nodes for the Pettson scenario are created as part of seed data
    (section 7.8). The bootstrap event uses metatype_id as created_by (self-referential
    bootstrap exception).
```

### 4.7 Auth decisions (resolved in gen1)

```yaml
# [RESOLVED:gen1 D-011] Token format: Stateless JWT
decision: "Stateless JWT for gen1. No DB lookup for auth. Short-lived (1h) mitigates revocation lag."
question_this_if: "Token revocation latency (up to 1h) is unacceptable"
[DECIDE:gen2] "Consider opaque tokens + introspection endpoint"

# [RESOLVED:gen1 D-012] Auth provider: Supabase Auth + custom JWT
decision: |
  - User tokens: issued by Supabase Auth (OAuth 2.0 flows, social login, magic link)
  - Agent tokens: issued by a custom admin-only signing endpoint
    - Admin provides: sub, tenant_ids, scopes, expiry
    - Endpoint signs JWT with shared secret (Supabase JWT secret)
    - Returns signed JWT that the MCP server validates identically to user tokens
  - Partner tokens: same as agent tokens but with read-only scopes and 30-day expiry
question_this_if: "Supabase Auth cannot add custom claims (tenant_ids, scopes) to JWTs"

# [RESOLVED:gen1 D-013] RLS pattern: Application-level + RLS safety net (Option C)
decision: |
  Edge Function connects to PostgreSQL with service_role key (bypasses RLS).
  Before every query, sets: SET LOCAL 'app.tenant_ids' = '{uuid1,uuid2}';
  RLS policies use current_setting('app.tenant_ids') as the tenant filter.
  Cross-tenant edge traversal is verified in application code by querying grants table.
  RLS serves as defense-in-depth — even if application code has a bug, RLS prevents
  accessing data outside the token's tenant_ids.

# [gen1.1] ATTACK SCENARIOS — technical analysis of D-013 blast radius
#
# These scenarios define the concrete security boundary of Option C.
# They serve as technical triggers for D-013's question_this_if.
#
# SCENARIO A: Application bug bypasses grants check
#   Attack: A code path in explore_graph or connect_entities fails to call
#           the grants table check before traversing a cross-tenant edge.
#   What RLS catches: Nothing for same-token tenants. If the token has
#           tenant_ids=[T1,T2], SET LOCAL includes both, so RLS permits
#           reading data from T1 and T2. The grants check is the ONLY
#           barrier for node-level access within permitted tenants.
#   What RLS misses: An agent with T1+T2 access could read ALL T2 nodes
#           without any grant. Grants provide node-level filtering that
#           RLS does not enforce.
#   Blast radius: WITHIN-TOKEN ONLY. An agent cannot access T3 if T3
#           is not in its tenant_ids. The blast is limited to over-broad
#           access within the set of tenants the token already covers.
#   Mitigation: Audit log (§4.9) records every cross-tenant traversal.
#           Periodic audit: flag traversals without matching grants.
#
# SCENARIO B: Token with excessive tenant_ids
#   Attack: Admin issues a token with tenant_ids=[T1,T2,T3,T4,...] covering
#           tenants the agent should not access (misconfiguration, not breach).
#   What RLS catches: SET LOCAL sets all tenant_ids from the token. RLS
#           policies permit access to all of them. There is no "minimum
#           privilege" enforcement at the RLS layer.
#   What RLS misses: Scope-based filtering (§4.8) is the only limit.
#           If scopes include "tenant:*:read", the agent reads everything.
#   Blast radius: ALL DATA in all tenants listed in the token. This is
#           a configuration error, not a code bug.
#   Mitigation:
#     - Token issuance MUST be admin-only (D-012).
#     - Token lifetime: 1h max for agents, 30d for partners (§4.4).
#     - [REQUIRED:gen1-impl] Log and alert when a token has >3 tenant_ids.
#     - [DECIDE:gen2] Consider per-tenant token issuance (one token per tenant)
#       instead of multi-tenant tokens to eliminate this class of error.
#
# SCENARIO C: Edge Function code path skips SET LOCAL
#   Attack: A code path reaches the database query without first calling
#           SET LOCAL 'app.tenant_ids'. The GUC is unset or retains a
#           value from a previous request in the same connection.
#   What RLS catches: If app.tenant_ids is unset, current_setting()
#           throws an error (unless a default is set). If a default of
#           '{}' (empty array) is set, RLS policies deny all reads
#           (tenant_id = ANY('{}') is always false). This is SAFE.
#   What RLS misses: If the previous request's tenant_ids leak into the
#           current request (connection pooling without RESET), the current
#           request could access the previous request's tenants.
#   Blast radius: PREVIOUS REQUEST'S TENANTS. Not unbounded — limited to
#           whatever the previous request had access to.
#   Mitigation:
#     - CRITICAL: gen1-impl MUST call SET LOCAL (not SET) so the GUC
#       is scoped to the current transaction. After COMMIT/ROLLBACK,
#       the value reverts.
#     - If using connection pooling: DISCARD ALL or RESET at connection
#       checkout. Supabase Edge Functions use per-request connections
#       (no pooling concern for gen1).
#     - [REQUIRED:gen1-impl] Integration test: verify that a request
#       without SET LOCAL receives zero rows (not previous request's data).
#
# TECHNICAL TRIGGERS FOR question_this_if:
#   1. Security audit finding: "service_role key stored in Edge Function
#      environment is extractable" → migrate to per-user Supabase client
#      or pure RLS (Option A).
#   2. Any Scenario C occurrence in production (SET LOCAL missed) →
#      add middleware-level enforcement (abort request if GUC not set).
#   3. Tenant count per token exceeds 5 regularly → per-tenant tokens.

question_this_if: |
  (1) Security audit requires pure RLS without service_role bypass, OR
  (2) Scenario C (SET LOCAL skip) occurs in production, OR
  (3) Average tenant_ids per token exceeds 5.
[DECIDE:gen2] "Migrate to JOIN-based RLS policies (Option A) for cross-tenant edges"
```

### 4.8 Auth precision: scope requirements per tool

```yaml
# Every tool call checks scopes in this order:
# 1. Extract tenant_id from params or token (single-tenant token)
# 2. Check token.scopes includes required scope
# 3. For cross-tenant: check scopes for ALL involved tenants
# 4. For cross-tenant edge traversal: also check grants table

store_entity:
  required_scope: "tenant:{tenant_id}:write" OR "tenant:{tenant_id}:nodes:{entity_type}:write"
  cross_tenant: N/A (entities are single-tenant)

find_entities:
  required_scope: "tenant:{tenant_id}:read" OR "tenant:{tenant_id}:nodes:{entity_type}:read"
  cross_tenant: N/A (searches within one tenant)

connect_entities:
  required_scope: "tenant:{source_tenant_id}:write" AND (if cross-tenant) "tenant:{target_tenant_id}:read"
  cross_tenant: YES — also requires grants table entry for target node

explore_graph:
  required_scope: "tenant:{start_tenant_id}:read"
  cross_tenant: YES — each traversal step checks grants for cross-tenant nodes

remove_entity:
  required_scope: "tenant:{tenant_id}:write"
  cross_tenant: N/A

query_at_time:
  required_scope: "tenant:{tenant_id}:read"
  cross_tenant: N/A

get_timeline:
  required_scope: "tenant:{tenant_id}:read"
  cross_tenant: N/A

capture_thought:
  required_scope: "tenant:{tenant_id}:write"
  cross_tenant: N/A (all created entities belong to one tenant)

get_schema:
  required_scope: "tenant:{tenant_id}:read"
  cross_tenant: N/A

get_stats:
  required_scope: "tenant:{tenant_id}:read"
  cross_tenant: N/A

propose_event:
  required_scope: "tenant:{tenant_id}:write"
  cross_tenant: N/A

verify_lineage:
  required_scope: "tenant:{tenant_id}:read"
  cross_tenant: N/A

store_blob:
  required_scope: "tenant:{tenant_id}:write"
  cross_tenant: N/A

get_blob:
  required_scope: "tenant:{tenant_id}:read"
  cross_tenant: N/A

lookup_dict:
  required_scope: "tenant:{tenant_id}:read"
  cross_tenant: N/A

# Scope matching pseudocode:
# function has_scope(token, required):
#   return token.scopes.some(s =>
#     s === "admin" ||
#     s === required ||
#     required.startsWith(s.replace(':read',':').replace(':write',':'))  // broader scope covers narrower
#   )
# Example: "tenant:T1:write" covers "tenant:T1:nodes:lead:write"
# Example: "tenant:T1:nodes:campaign:read" does NOT cover "tenant:T1:nodes:lead:read"
#
# Type-scoped read behavior:
# When find_entities is called without entity_types filter under a type-scoped token
# (e.g., "tenant:T1:nodes:campaign:read"), results are SILENTLY FILTERED to only
# return entities of types the token has read access to. No error is returned.
# This allows agents with narrow scopes to use find_entities safely.
```

### 4.9 Audit event schema

```yaml
# Every tool call emits an audit trail entry (stored as part of the tool's event
# if it's a mutation, or as a lightweight log entry for read operations).
# Read audit entries are NOT stored in the events table (to avoid write amplification).
# They are logged to the Edge Function's stdout for external log aggregation.

audit_log_entry:
  timestamp: datetime           # ISO 8601
  actor_id: string              # actor node_id
  tool: string                  # tool name
  tenant_id: string             # primary tenant
  params_summary: object        # key params (entity_type, entity_id) — NOT full payload
  result_status: string         # "success" | "denied" | "error" | "not_found"
  duration_ms: int              # request processing time
  cross_tenant: boolean         # whether operation crossed tenant boundaries
  error_code: string?           # if result_status != "success"

# For mutations: the event itself serves as the audit record (event_id, created_by, timestamp).
# For reads: audit_log_entry is logged to stdout in JSON format.
```

---
