I've now read the complete specification. Let me produce the behavioral report.

---

# RESONANSIA ΓÇö Behavioral Specification Report

## 1. COMPLETE OPERATION CATALOGUE

### 1.1 MCP Tool Operations

---

#### **T1: store_entity**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with `entity_type` + `data`. Branches: `entity_id` absent ΓåÆ CREATE; `entity_id` present ΓåÆ UPDATE |
| **Preconditions** | (1) Token has `tenant:{tid}:write` or `tenant:{tid}:nodes:{type}:write`. (2) `entity_type` resolves to a type_node in tenant or system tenant. (3) If `label_schema` exists on type_node, `data` validates. (4) If UPDATE: entity exists, is not deleted, `valid_to = 'infinity'`. (5) If `expected_version` provided: must match `current.valid_from`. |
| **Postconditions ΓÇö CREATE** | New event (`entity_created`) + new node row with `valid_to = 'infinity'`, `is_deleted = false`, `created_by_event` = event_id. `node_id = stream_id` of event. |
| **Postconditions ΓÇö UPDATE** | New event (`entity_updated` or `epistemic_change`). Old node row closed (`valid_to = now()`). New node row opened. Version count incremented. |
| **Side effects** | Async embedding queued. Audit via event itself. |
| **Atomicity** | Event + projection in single BEGIN/COMMIT (INV-ATOMIC). |

---

#### **T2: find_entities**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with optional `query`, `entity_types`, `filters`, `epistemic`, `limit`, `cursor`, `sort_by` |
| **Preconditions** | Token has `tenant:{tid}:read` (or type-scoped read). tenant_id resolved. |
| **Postconditions** | No state change. Returns matching nodes where `is_deleted = false AND valid_to = 'infinity'`. |
| **Side effects** | Audit log entry to stdout (read operation). |
| **Atomicity** | Read-only, no transaction required. |

**Behavioral notes**: With type-scoped tokens, results are silently filtered to accessible types (no error). Semantic search uses `embedding <=> query_embedding` with pgvector HNSW. Nodes without embeddings are excluded from semantic search but included in structured search.

---

#### **T3: connect_entities**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with `edge_type`, `source_id`, `target_id`, optional `data` |
| **Preconditions** | (1) Source node exists, not deleted. (2) Target node exists, not deleted. (3) Edge type resolves to type_node with `kind = 'edge_type'` in system or caller tenant. (4) Write scope on source's tenant. (5) If cross-tenant: read scope on target's tenant AND grants table entry with TRAVERSE or WRITE capability for target node. |
| **Postconditions** | New event (`edge_created`). New edge row. Edge owned by source's tenant. |
| **Side effects** | Audit via event. |
| **Atomicity** | Event + edge insert in single transaction. |

**Behavioral notes**: Duplicate edges (same source/target/type) are allowed. Self-edges are allowed.

---

#### **T4: explore_graph**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with `start_id`, optional `edge_types`, `direction`, `depth`, `max_results`, `include_data`, `filters` |
| **Preconditions** | (1) Start node exists, not deleted. (2) Read scope on start node's tenant. |
| **Postconditions** | No state change. Returns subgraph centered on start node. |
| **Side effects** | Audit log to stdout. Cross-tenant nodes checked against grants ΓÇö inaccessible nodes silently omitted. |
| **Atomicity** | Read-only. |

---

#### **T5: remove_entity**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with `entity_id` |
| **Preconditions** | (1) Entity exists, not already deleted. (2) Write scope on entity's tenant. |
| **Postconditions** | Event (`entity_removed`). Node row updated: `is_deleted = true`, `valid_to = now()`. Connected edges become dangling (not auto-removed). |
| **Side effects** | Audit via event. |
| **Atomicity** | Event + node update in single transaction. |

---

#### **T6: query_at_time**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with `entity_id`, `valid_at`, optional `recorded_at` |
| **Preconditions** | Read scope on entity's tenant. |
| **Postconditions** | No state change. Returns the node version where `valid_from <= valid_at < valid_to` and `recorded_at <= recorded_at_param`. Returns null if no match. |
| **Side effects** | Audit log. |
| **Atomicity** | Read-only. |

---

#### **T7: get_timeline**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with `entity_id`, optional `include_related_events`, `time_from`, `time_to`, `limit`, `cursor` |
| **Preconditions** | Read scope on entity's tenant. Entity exists (any version). |
| **Postconditions** | No state change. Returns chronological list of version changes, events, edge creation/removal, epistemic changes. |
| **Side effects** | Audit log. |
| **Atomicity** | Read-only. |

---

#### **T8: capture_thought**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with `content` (free text), optional `source`, `tenant_id` |
| **Preconditions** | (1) Write scope on tenant. (2) LLM API available. (3) Embedding API available (for dedup search). |
| **Postconditions** | Event (`thought_captured`). Primary entity node created with `epistemic = 'hypothesis'`. Zero or more mentioned entity nodes created (also hypothesis) or linked to existing. Zero or more edges created. |
| **Side effects** | LLM call (pre-transaction). Embedding generation for dedup (pre-transaction). Async embedding for new nodes (post-transaction). |
| **Atomicity** | All DB writes (event + all nodes + all edges) in single transaction. LLM/embedding calls are external and happen before the transaction. If any DB insert fails, entire transaction rolls back. |

---

#### **T9: get_schema**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with optional `tenant_id` |
| **Preconditions** | Read scope on tenant. |
| **Postconditions** | No state change. Returns type nodes (entity_types, edge_types, event_types) with counts. |
| **Side effects** | Audit log. |
| **Implementation constraint** | MUST query nodes table (type_node_id = METATYPE), not a separate registry. Reflexive property. |

---

#### **T10: get_stats**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with optional `tenant_id` |
| **Preconditions** | Read scope. |
| **Postconditions** | No state change. Returns aggregate statistics. |

---

#### **T11: propose_event**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with `stream_id`, `intent_type`, `payload`, optional `occurred_at`, `tenant_id` |
| **Preconditions** | (1) Write scope. (2) `intent_type` matches a type_node of `kind = 'event_type'` (or is in the CHECK constraint list). (3) Payload validates against schema if defined. |
| **Postconditions** | Event created. Projection logic runs for the given intent_type (same as if triggered by a higher-level tool). |
| **Side effects** | Whatever the projection for that intent_type produces. |
| **Atomicity** | Event + projection in single transaction. |

**Behavioral note**: This is the escape hatch for operations without dedicated tools (e.g., `edge_removed`, `grant_created`, `grant_revoked`).

---

#### **T12: verify_lineage**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with `entity_id` |
| **Preconditions** | Read scope. |
| **Postconditions** | No state change. Returns integrity report: checks all node/edge/grant rows for this entity reference existing events. |
| **Side effects** | Audit log. |

---

#### **T13: store_blob**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with `data_base64`, `content_type`, optional `related_entity_id` |
| **Preconditions** | Write scope. If `related_entity_id`: entity exists. |
| **Postconditions** | Event (`blob_stored`). Blob row in blobs table. Binary uploaded to object storage. |
| **Side effects** | Object storage upload (pre-transaction ΓÇö potential orphan if DB fails). |
| **Atomicity** | Event + blob insert in single transaction. Object storage upload is external (pre-tx). |

---

#### **T14: get_blob**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with `blob_id` |
| **Preconditions** | Read scope on blob's tenant. |
| **Postconditions** | No state change. Returns blob metadata + data_base64 (fetched from object storage). |

---

#### **T15: lookup_dict**

| Aspect | Detail |
|--------|--------|
| **Trigger** | MCP tool call with `dict_type`, optional `key`, `valid_at` |
| **Preconditions** | Read scope. |
| **Postconditions** | No state change. Returns matching dict entries. |

---

### 1.2 Non-Tool Operations

#### **Auth Middleware**

| Aspect | Detail |
|--------|--------|
| **Trigger** | Every HTTP request to /mcp |
| **Sequence** | (1) Extract JWT from Authorization header. (2) Validate signature, expiry, audience=`resonansia-mcp`. (3) Extract `sub`, `tenant_ids`, `scopes`. (4) `SET LOCAL 'app.tenant_ids' = '{...}'`. (5) Resolve actor node_id per tenant via `jwt_sub_to_actor` mapping. (6) Auto-create actor node if not found. |
| **Side effects** | Potential actor node creation (store_entity internally). Audit log entry. |
| **Failure modes** | Invalid JWT ΓåÆ 401. Expired ΓåÆ 401. Wrong audience ΓåÆ 401. |

#### **Embedding Pipeline**

| Aspect | Detail |
|--------|--------|
| **Trigger** | Queued after any node create/update (post-transaction) |
| **Sequence** | (1) Generate text: `type_name + ": " + JSON.stringify(data)`. (2) Call OpenAI API (text-embedding-3-small, 1536d). (3) UPDATE nodes SET embedding = $vector WHERE node_id = $id AND valid_from = $vf. |
| **Failure modes** | API failure ΓåÆ retry with exponential backoff (base=1s, max=60s, max_retries=5). Failure does NOT affect entity availability. |
| **Batch mode** | 100 texts per API call, 5 concurrent, 3000 RPM limit. |

#### **Metatype Bootstrap**

| Aspect | Detail |
|--------|--------|
| **Trigger** | Database migration (one-time) |
| **Sequence** | (1) Create system tenant. (2) Insert bootstrap event. (3) Insert metatype node (self-referential: `type_node_id = node_id`). |
| **Special property** | Only entity created outside normal eventΓåÆprojection flow. `created_by = metatype_id` (bootstrap exception). |

---

## 2. STATE MACHINE DESCRIPTION

### 2.1 Entity (Node) Lifecycle

```
                          store_entity(new)
                                Γöé
                                Γû╝
ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé  ACTIVE                                      Γöé
Γöé  is_deleted=false, valid_to='infinity'       Γöé
Γöé  epistemic Γêê {hypothesis, asserted, confirmed}Γöé
Γöé                                              Γöé
Γöé  ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ store_entity  ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ    Γöé
Γöé  Γöé version N ΓöéΓöÇΓöÇ(update)ΓöÇΓöÇΓåÆΓöé version N+1Γöé    Γöé
Γöé  Γöé(closed)   Γöé              Γöé (current)  Γöé    Γöé
Γöé  ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ              ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ    Γöé
Γöé                                              Γöé
Γöé  Epistemic transitions:                      Γöé
Γöé  hypothesis ΓöÇΓöÇΓåÆ asserted ΓöÇΓöÇΓåÆ confirmed       Γöé
Γöé       ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓåÆ confirmed       Γöé
Γöé  (confirmed is terminal ΓÇö no backward moves) Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö¼ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
                       Γöé remove_entity
                       Γû╝
ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé  DELETED (absorbing state)                   Γöé
Γöé  is_deleted=true, valid_to=now()            Γöé
Γöé  No further operations (returns NOT_FOUND)   Γöé
Γöé  Remains in DB for audit/history             Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
```

**Key observations:**
- **DELETED is absorbing** ΓÇö there is no un-delete operation. The spec says: "To 'un-delete' ΓåÆ create a new entity with same type (gets a new node_id)." This means the original identity is permanently retired.
- `confirmed` epistemic status is absorbing within an entity's lifetime. Corrections to confirmed entities require new entities (spec ┬º2.3: "corrections are new entities").
- Entities without embeddings are a transient sub-state (queryable by structure, not by semantics).

### 2.2 Edge Lifecycle

```
      connect_entities
            Γöé
            Γû╝
ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé  ACTIVE               Γöé
Γöé  is_deleted=false      Γöé
Γöé  valid_to='infinity'  Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö¼ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
           Γöé propose_event(edge_removed) or
           Γöé implicit via future tool
           Γû╝
ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé  DELETED (absorbing)  Γöé
Γöé  is_deleted=true       Γöé
Γöé  valid_to=now()       Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
```

**Key observations:**
- Edges are immutable (D-002). No update ΓÇö only soft-delete + recreate.
- No dedicated `remove_edge` tool in gen1. Edge removal only via `propose_event` with `intent_type = 'edge_removed'`.
- When source or target entity is removed, edges become **dangling** ΓÇö not auto-deleted. `explore_graph` skips them; `get_timeline` still shows them.

### 2.3 Grant Lifecycle

```
      propose_event(grant_created) / admin action
            Γöé
            Γû╝
ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé  ACTIVE               Γöé
Γöé  is_deleted=false      Γöé
Γöé  valid_to >= now()    Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö¼ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
           Γöé propose_event(grant_revoked) or
           Γöé valid_to reached (temporal expiry)
           Γû╝
ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé  REVOKED/EXPIRED      Γöé
Γöé  is_deleted=true or    Γöé
Γöé  valid_to < now()     Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
```

**Key observations:**
- No dedicated MCP tool for grant creation/revocation in gen1. Must use `propose_event` or direct DB.
- Grant revocation does NOT cascade to existing cross-tenant edges. Those edges become inaccessible via `explore_graph` (grants check fails at traversal time) but remain in the database.

### 2.4 Event Lifecycle

```
      (any mutation)
            Γöé
            Γû╝
ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé  EXISTS (terminal)     Γöé
Γöé  Immutable forever     Γöé
Γöé  (INV-APPEND)          Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
```

Events are created once and never modified. Corrections are new events referencing originals in payload. This is a single-state lifecycle ΓÇö creation is the only transition.

### 2.5 System-Level States

```
ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ   migration    ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé  UNINITIALIZEDΓöéΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓåÆΓöé  BOOTSTRAPPED Γöé
Γöé  (no tables)  Γöé               Γöé  (metatype +   Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ               Γöé   system tenant)Γöé
                               ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö¼ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
                                      Γöé seed data
                                      Γû╝
                               ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
                               Γöé   OPERATIONAL Γöé
                               Γöé   (normal)    Γöé
                               ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓö¼ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
                                      Γöé LLM/embedding API down
                                      Γû╝
                               ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
                               Γöé  DEGRADED     Γöé
                               Γöé  (capture_    Γöé
                               Γöé   thought     Γöé
                               Γöé   fails, no   Γöé
                               Γöé   embeddings) Γöé
                               ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
                                      Γåæ
                               API restored Γöé
                                      ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
```

**Cold start**: Supabase Edge Functions have cold starts (first request after idle). This adds latency but doesn't affect correctness ΓÇö the database is persistent.

---

## 3. CONSISTENCY ISSUES FOUND

### CRITICAL

#### C1: Grant revocation does not invalidate existing cross-tenant edges

**Scenario**: Tenant A grants TRAVERSE on node N to Tenant B. Tenant B creates a cross-tenant edge to N. Later, the grant is revoked.

**Current behavior**: The edge remains. `explore_graph` will fail the grants check at traversal time (edge is effectively invisible), but `get_timeline` on the edge's source entity will still show the edge. The edge row references a now-revoked grant, but the edge itself is not modified.

**Issue**: The system has no mechanism to retroactively invalidate or flag edges created under revoked grants. The edge data (which may contain sensitive field references) persists. While `explore_graph` will skip it, direct database access (or `verify_lineage`) could expose it.

**Severity**: CRITICAL ΓÇö data sharing semantics are unclear after revocation.

**Recommendation**: Either (a) define that grant revocation soft-deletes all cross-tenant edges that relied on that grant, or (b) explicitly document that edges persist but are access-controlled at query time, and add a `grant_id` FK on edges so the dependency is traceable.

---

#### C2: `propose_event` bypasses tool-level validation for most intent_types

**Scenario**: An agent calls `propose_event(intent_type="entity_created", payload={...})` directly.

**Current behavior**: The spec says propose_event "validates intent_type against type nodes" and "runs projection logic." But the validation described is minimal ΓÇö it checks that `intent_type` matches a type_node of kind `event_type`. However, the standard intent_types (`entity_created`, `entity_updated`, etc.) are NOT type nodes ΓÇö they're CHECK constraint values. The propose_event schema says `intent_type: z.string().min(1)` ΓÇö any string that passes the CHECK constraint is valid.

**Issue**: If propose_event runs the same projection logic as the corresponding tool, it effectively duplicates every tool's behavior but potentially with weaker pre-validation (no schema checking, no scope granularity, no dedup). If it doesn't run projection logic, it creates events without projections (violating INV-ATOMIC unless it's a noop projection).

**Severity**: CRITICAL ΓÇö ambiguous whether propose_event is a full bypass or a constrained raw interface.

**Recommendation**: Clarify that `propose_event` for standard intent_types runs the full projection pipeline including all pre-validation. Or restrict it to custom intent_types only and require agents to use dedicated tools for standard operations.

---

### MAJOR

#### M1: No `remove_edge` tool ΓÇö only accessible via `propose_event`

**Impact**: An agent that creates an edge via `connect_entities` cannot remove it without knowing about `propose_event` and constructing a valid `edge_removed` payload manually. This breaks the "outcome over operation" design principle.

**Recommendation**: Add a `remove_edge` tool (bringing total to 16) or document this limitation prominently in tool descriptions.

---

#### M2: Actor auto-creation creates a node within auth middleware ΓÇö before the tool call's transaction

**Scenario**: New agent's first request. Auth middleware calls `store_entity` internally to create the actor node. This creates an event + node. Then the actual tool call runs in a separate transaction.

**Issue**: If the actual tool call fails, the actor node still exists (it was created in a prior, committed transaction). This is benign but means actor creation is NOT atomic with the first tool call. If the Edge Function crashes between actor creation and tool execution, the actor exists but performed no action ΓÇö an orphaned-but-valid actor.

**Severity**: MAJOR ΓÇö violates the spirit of INV-ATOMIC (though not the letter, since the actor creation IS atomic with its own event).

**Recommendation**: Accept this as a known non-issue (actor pre-creation is idempotent and benign) but document it explicitly.

---

#### M3: `store_entity` with epistemic change delegates to `epistemic_change` projection, but the branching logic is pre-transaction

**Issue**: The spec says (line 998): "IF $epistemic != $current.epistemic: ΓåÆ delegate to epistemic_change projection." This happens BEFORE the BEGIN. But what if between the pre-validation read and the BEGIN, another agent changes the entity's epistemic status? The pre-validation read is stale.

**Mitigation**: The UPDATE inside the transaction uses `WHERE valid_to = 'infinity' AND is_deleted = false` ΓÇö if the entity was modified concurrently, the UPDATE affects 0 rows, the transaction detects this, and returns CONFLICT. So correctness is maintained, but the wrong intent_type might be chosen (entity_updated vs epistemic_change).

**Severity**: MAJOR ΓÇö wrong event type recorded under race condition, but no data corruption.

**Recommendation**: Move the epistemic change detection inside the transaction, or document this as an acceptable audit imprecision.

---

#### M4: `capture_thought` LLM dedup uses embedding search, but embeddings are async

**Scenario**: Agent A calls `store_entity` to create "Johan Eriksson". Before the async embedding completes, Agent B calls `capture_thought` with text mentioning "Johan Eriksson."

**Issue**: The dedup algorithm's Tier 2 (embedding similarity search) will NOT find Johan because his embedding is NULL. Tier 1 (exact name match) might catch it if the name is identical, but Tier 1 uses ILIKE on `data->>'name'` which requires exact match after normalization.

**Severity**: MAJOR ΓÇö dedup fails for very recently created entities, leading to duplicates.

**Recommendation**: Either (a) make embedding synchronous for dedup-critical paths, (b) have Tier 1 use fuzzy matching (e.g., trigram similarity), or (c) document that dedup is eventual and accept temporary duplicates.

---

#### M5: `blobs` table has no `created_by_event` column

**Observation**: The blobs table schema (┬º2.1) has `created_by` and `created_at` but no `created_by_event` FK. The `blob_stored` projection creates both an event and a blob row, but the blob row has no FK back to the event.

**Issue**: Violates INV-LINEAGE ("Every fact row MUST have a non-null `created_by_event` FK"). The spec lists blobs as a "utility storage" layer alongside facts, but INV-LINEAGE says "nodes, edges, grants" ΓÇö not blobs. However, blobs DO have events (blob_stored), creating an inconsistency in the lineage model.

**Severity**: MAJOR ΓÇö blobs are partially event-sourced (event exists) but lack the FK to prove it.

**Recommendation**: Either add `created_by_event UUID NOT NULL REFERENCES events` to blobs table, or explicitly exclude blobs from INV-LINEAGE.

---

#### M6: `dicts` table is not event-sourced at all

**Observation**: The dicts table has no `created_by_event`, no `created_by`, no corresponding event intent_type. There is no `dict_created` or `dict_updated` in the events CHECK constraint.

**Issue**: Dict entries are created/updated without audit trail. There's no tool for creating dict entries ΓÇö only `lookup_dict` for reading. How do dicts get populated?

**Severity**: MAJOR ΓÇö dicts exist in the schema but have no creation mechanism and no event trail.

**Recommendation**: Either (a) add a `store_dict` tool with event sourcing, or (b) declare dicts as static reference data populated by migration/seed only, outside the event model.

---

### MINOR

#### m1: `find_entities` with type-scoped token and no `entity_types` filter ΓÇö silent filtering vs AUTH_DENIED inconsistency

**Observation**: ┬º4.8 says type-scoped reads are "SILENTLY FILTERED" ΓÇö no error. But T03 expects `AUTH_DENIED` when content_agent (type-scoped to campaign) calls `find_entities(entity_types=["lead"])`.

**Resolution**: These are consistent ΓÇö explicitly requesting an inaccessible type returns AUTH_DENIED; omitting the filter returns silently filtered results. But this is subtle and should be documented more clearly.

---

#### m2: `connect_entities` ΓÇö edge ownership is always source tenant's

**Observation**: Projection sets `$edge_tenant = $source.tenant_id`. For cross-tenant edges, the edge is always owned by the source's tenant.

**Issue**: If the source tenant is revoked/deleted, the edge is orphaned from an ownership perspective. The target tenant cannot manage it.

**Severity**: MINOR ΓÇö unlikely scenario at gen1 scale, but architecturally asymmetric.

---

#### m3: `explore_graph` with depth > 1 ΓÇö exponential expansion

**Observation**: Max depth is 5, max_results is 500. But a densely connected graph could require traversing thousands of edges to reach depth 5 before applying the limit.

**Severity**: MINOR ΓÇö performance issue, not correctness. The 2s CPU limit on Supabase Edge Functions is the natural bound.

---

#### m4: No `tenant_id` parameter on `remove_entity`, `query_at_time`, `get_timeline`, `verify_lineage`

**Observation**: These tools take `entity_id` but not `tenant_id`. The tenant must be inferred from the entity. This requires a pre-query to find the entity's tenant, which must succeed under RLS.

**Issue**: If the token covers multiple tenants, the system must query across all accessible tenants to find the entity. Not a correctness issue, but a performance concern.

---

#### m5: `store_entity` returns `version: int` defined as "COUNT(*) of node rows with this node_id"

**Issue**: This requires a COUNT query across all versions (including historical). For heavily-updated entities, this could be slow. More importantly, concurrent writes could cause version number conflicts (two writes both see COUNT = 5 and both claim version 6).

**Severity**: MINOR ΓÇö version is informational, not used for concurrency control (that's `expected_version` / `valid_from`).

---

#### m6: Scope matching pseudocode has a potential bug

**Observation** (┬º4.8):
```
required.startsWith(s.replace(':read',':').replace(':write',':'))
```
This means `"tenant:T1:read"` becomes `"tenant:T1:"` after replace. If required is `"tenant:T1:nodes:lead:read"`, then `required.startsWith("tenant:T1:")` is true. This works.

But `"tenant:T1:write"` becomes `"tenant:T1:"`. If required is `"tenant:T1:nodes:lead:read"`, then `required.startsWith("tenant:T1:")` is true ΓÇö meaning a WRITE scope grants READ access. This is likely intentional (write implies read) but not explicitly stated.

---

## 4. COMPLETENESS GAPS

### 4.1 Unhandled Inputs

| Input | Status |
|-------|--------|
| Malformed UUIDs | Handled: Zod `.uuid()` validation rejects at parse time ΓåÆ E001 |
| Empty strings | Handled: Zod `.min(1)` on required string fields |
| Null embeddings | Handled: Embedding column is NULLABLE. Entities work without embeddings. |
| Very large JSONB payloads | **NOT HANDLED**: No size limit on `data` field. A multi-MB JSONB payload would succeed. The 20MB function bundle limit and 300MB RAM limit are the only bounds. |
| Unicode edge cases | **NOT HANDLED**: No normalization spec for entity names. "├àngstr├╢m" vs "Angstr├╢m" would not match in dedup Tier 1. |
| SQL injection via JSONB filters | Should be safe via parameterized queries, but the filter-to-SQL translation for `$eq`/`$in` must use parameterized queries, not string interpolation. Not explicitly stated. |

### 4.2 Missing Operations

| Operation | Status |
|-----------|--------|
| Update a grant | **NOT SPECIFIED**. No way to modify a grant's capability or validity range. Must revoke and recreate. |
| Update a type_node | Implicitly supported via `store_entity(entity_id=TYPE_NODE_ID)`. But changing a type_node's `label_schema` could invalidate existing entities of that type. No cascade validation is specified. |
| Delete an event | **Prohibited** by INV-APPEND. Correct. |
| Remove an edge | Via `propose_event` only. No dedicated tool. |
| Create a grant | Via `propose_event` or direct DB. No dedicated tool. |
| Create a dict entry | **NOT SPECIFIED**. No tool, no event type. |
| Un-delete an entity | **NOT POSSIBLE**. Absorbing state. Must create new entity with new ID. |
| Update blob metadata | **NOT SPECIFIED**. Blobs are write-once. |
| Bulk import | **NOT SPECIFIED** as a tool. Must call store_entity repeatedly. |

### 4.3 Untested Behaviors

The Pettson T01-T14 test suite covers:

| Behavior | Covered by |
|----------|-----------|
| Semantic search | T01 |
| Entity creation | T02 |
| Permission denial (type-scoped) | T03 |
| Cross-tenant traversal | T04 |
| Write protection on read-only token | T05 |
| Audit trail | T06 |
| Bitemporal query | T07 |
| Timeline | T08 |
| Capture thought | T09 |
| Schema discovery | T10 |
| Deduplication | T11 |
| Epistemic transitions | T12 |
| Event primacy (DB assertion) | T13 |
| Verify lineage | T14 |

**NOT covered:**

| Behavior | Gap |
|----------|-----|
| Optimistic concurrency conflict (expected_version) | No test |
| Edge removal (via propose_event) | No test |
| Grant creation/revocation | No test (grants are seed data only) |
| store_blob / get_blob | No test |
| lookup_dict | No test |
| propose_event | No test |
| get_stats | No test |
| Actor auto-creation on first request | No test |
| Concurrent writes to same entity | No test |
| Entity with no embedding (structured-only search) | No test |
| Schema validation failure (SCHEMA_VIOLATION) | No test |
| Type node creation via store_entity | No test |
| Dangling edges after entity removal | No test |
| Cross-tenant edge creation (connect_entities with grant check) | No test (T04 tests traversal, not creation) |
| explore_graph at depth > 1 | No test |
| Pagination (cursor-based) | No test |
| capture_thought fallback (vague input ΓåÆ "note" type) | No test |

---

## 5. CONCURRENCY ANALYSIS

### 5.1 Two `store_entity` calls for the same entity simultaneously

**Without expected_version**: Last-write-wins. Both read current version. Both try to UPDATE `SET valid_to = $now WHERE valid_to = 'infinity'`. One succeeds, closing the original version. The other's UPDATE affects 0 rows (the version is already closed), triggering CONFLICT detection ΓåÆ E004 for the second caller. **Consistent.**

**With expected_version**: Both provide the same expected_version. Same race ΓÇö one succeeds, one gets E004. **Consistent.**

### 5.2 `capture_thought` + concurrent `store_entity` on same entity

The dedup search in capture_thought happens pre-transaction. If store_entity commits between the dedup search and capture_thought's BEGIN:
- If the entity was updated (new data), capture_thought may link to a stale version. But `node_id` is stable across versions, so the link is still correct. **Benign.**
- If the entity was deleted, capture_thought may link to a deleted entity. The edge would be dangling. **Minor inconsistency.**

### 5.3 Grant revoked during cross-tenant read

`explore_graph` checks grants at traversal time. If a grant is revoked mid-traversal (between depth 1 and depth 2 checks), depth 1 results include the cross-tenant node but depth 2 from that node fails. **Benign ΓÇö results are partial but not incorrect.**

### 5.4 Entity removed during `explore_graph` traversal

`explore_graph` skips edges where endpoints are `is_deleted = true`. If an entity is deleted between the start node read and the traversal, the connection is simply omitted. **Consistent.**

### 5.5 Two `capture_thought` calls extracting the same entity

Both run dedup pre-transaction. Both find no existing "Johan" entity. Both try to create "Johan" as a new node. Both succeed (different node_ids). Result: duplicate entity. The EXCLUDE constraint on nodes prevents temporal overlap for the SAME node_id, but these are different node_ids. **This is the expected failure mode per M4 above.**

---

## 6. RECOMMENDED ADDITIONS OR CORRECTIONS

### Priority 1 (CRITICAL fixes)

1. **Clarify `propose_event` scope**: Specify that propose_event for standard intent_types (entity_created etc.) runs the complete projection pipeline including all pre-validation from the corresponding tool. Or restrict propose_event to custom intent_types not covered by dedicated tools.

2. **Document grant revocation semantics for existing edges**: Specify that cross-tenant edges created under a revoked grant remain in the database but are inaccessible via query tools. Consider adding a `grant_id` column to edges for traceability.

### Priority 2 (MAJOR fixes)

3. **Add `created_by_event` to blobs table** or explicitly exclude blobs from INV-LINEAGE.

4. **Define dict creation mechanism**: Either add `store_dict` tool + `dict_created` event type, or declare dicts as migration-only data.

5. **Add `remove_edge` tool** (or prominently document that edge removal requires `propose_event`).

6. **Address dedup timing gap**: Document that dedup is best-effort for entities created within the embedding latency window. Consider adding fuzzy name matching (trigram) as Tier 1.5.

### Priority 3 (MINOR / gen2)

7. **Add JSONB data size limit** (e.g., 1MB) to prevent resource exhaustion.

8. **Add tests for**: propose_event, store_blob/get_blob, lookup_dict, get_stats, optimistic concurrency conflict, edge removal, cross-tenant edge creation, pagination, schema validation failure.

9. **Clarify write-implies-read** in scope matching pseudocode.

10. **Add `note` type_node to seed data** ΓÇö capture_thought fallback requires it but it's not in the seed data definition.

---

## Summary

The Resonansia specification is remarkably thorough ΓÇö all 15 tools have complete Zod schemas, projection pseudocode, and error codes. The event-sourced architecture is internally consistent: event primacy and INV-ATOMIC are well-defined and the projection matrix covers all 10 intent_types.

The most significant finding is that the system has **two critical ambiguities** (propose_event bypass potential, grant revocation semantics), **six major gaps** (blobs lacking lineage FK, dicts lacking any mutation mechanism, no remove_edge tool, dedup timing, actor creation atomicity, epistemic change race), and substantial **test coverage gaps** (only 14 of ~30+ behavioral paths are tested).

The state machine is well-defined with clear absorbing states (deleted entities, confirmed epistemic status). No unreachable states were found. The concurrency model relies on PostgreSQL's row-level WHERE clauses for conflict detection, which is sound but only effective within the transaction boundary ΓÇö pre-transaction reads (dedup, epistemic check) are inherently stale.
