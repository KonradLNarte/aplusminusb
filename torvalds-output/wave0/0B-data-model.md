I've now read all 9 specification files completely. Let me produce the comprehensive data model analysis.

---

# Resonansia Data Model Archaeology Report

## Step 2: The Stated Data Model (Exact Reproduction)

### Tables (7 tables + 1 tenants table = 8 total)

**Note:** The spec says "7 tables" but actually defines 8: `tenants`, `events`, `nodes`, `edges`, `grants`, `blobs`, `dicts`. There is no separate `type_nodes` table ΓÇö type nodes are regular `nodes` rows with `type_node_id = METATYPE_ID`. There is no `audit_log` table ΓÇö read audits go to stdout; mutation audits are the events themselves.

#### Table 1: `tenants`
| Column | Type | Constraints |
|--------|------|-------------|
| `tenant_id` | UUID | PK, DEFAULT gen_random_uuid() |
| `parent_id` | UUID | FK tenants, NULLABLE |
| `name` | TEXT | NOT NULL |
| `config` | JSONB | DEFAULT '{}' |
| `created_at` | TIMESTAMPTZ | DEFAULT now() |

**Index:** `idx_tenants_parent ON tenants (parent_id) WHERE parent_id IS NOT NULL`

#### Table 2: `events`
| Column | Type | Constraints |
|--------|------|-------------|
| `event_id` | UUID | PK, DEFAULT uuid_generate_v7() |
| `tenant_id` | UUID | NOT NULL, FK tenants |
| `intent_type` | TEXT | NOT NULL, CHECK (10 values) |
| `payload` | JSONB | NOT NULL |
| `stream_id` | UUID | NULLABLE |
| `node_ids` | UUID[] | DEFAULT '{}' |
| `edge_ids` | UUID[] | DEFAULT '{}' |
| `occurred_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() |
| `recorded_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() |
| `created_by` | UUID | NOT NULL |

CHECK: `intent_type IN ('entity_created', 'entity_updated', 'entity_removed', 'edge_created', 'edge_removed', 'epistemic_change', 'thought_captured', 'grant_created', 'grant_revoked', 'blob_stored')`

**Indexes:** `idx_events_tenant (tenant_id)`, `idx_events_stream (stream_id) WHERE stream_id IS NOT NULL`, `idx_events_intent (tenant_id, intent_type)`, `idx_events_occurred (tenant_id, occurred_at DESC)`, `idx_events_recorded (recorded_at DESC)`

#### Table 3: `nodes`
| Column | Type | Constraints |
|--------|------|-------------|
| `node_id` | UUID | NOT NULL |
| `tenant_id` | UUID | NOT NULL, FK tenants |
| `type_node_id` | UUID | NOT NULL (logical FK to nodes) |
| `data` | JSONB | NOT NULL, DEFAULT '{}' |
| `embedding` | VECTOR(1536) | NULLABLE |
| `epistemic` | TEXT | NOT NULL, DEFAULT 'hypothesis' |
| `valid_from` | TIMESTAMPTZ | NOT NULL, DEFAULT now() |
| `valid_to` | TIMESTAMPTZ | DEFAULT 'infinity' |
| `recorded_at` | TIMESTAMPTZ | DEFAULT now() |
| `created_by` | UUID | NOT NULL |
| `created_by_event` | UUID | NOT NULL, FK events |
| `is_deleted` | BOOLEAN | DEFAULT false |

**PK:** `(node_id, valid_from)` ΓÇö composite for bitemporality
**EXCLUDE:** `USING gist (node_id WITH =, tstzrange(valid_from, valid_to) WITH &&)`

**Indexes:** `idx_nodes_tenant (tenant_id)`, `idx_nodes_type (type_node_id)`, `idx_nodes_active (tenant_id, type_node_id) WHERE is_deleted = false AND valid_to = 'infinity'`, `idx_nodes_stream (node_id, valid_from DESC)`, `idx_nodes_embedding USING hnsw (embedding vector_cosine_ops) WITH (m=16, ef_construction=64)`

#### Table 4: `edges`
| Column | Type | Constraints |
|--------|------|-------------|
| `edge_id` | UUID | PK, DEFAULT uuid_generate_v7() |
| `tenant_id` | UUID | NOT NULL, FK tenants |
| `type_node_id` | UUID | NOT NULL |
| `source_id` | UUID | NOT NULL |
| `target_id` | UUID | NOT NULL |
| `data` | JSONB | DEFAULT '{}' |
| `valid_from` | TIMESTAMPTZ | NOT NULL, DEFAULT now() |
| `valid_to` | TIMESTAMPTZ | DEFAULT 'infinity' |
| `recorded_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() |
| `created_by` | UUID | NOT NULL |
| `created_by_event` | UUID | NOT NULL, FK events |
| `is_deleted` | BOOLEAN | DEFAULT false |

**Indexes:** `idx_edges_tenant (tenant_id)`, `idx_edges_source (source_id) WHERE is_deleted = false`, `idx_edges_target (target_id) WHERE is_deleted = false`, `idx_edges_type (type_node_id)`, `idx_edges_active (tenant_id) WHERE is_deleted = false AND valid_to = 'infinity'`

#### Table 5: `grants`
| Column | Type | Constraints |
|--------|------|-------------|
| `grant_id` | UUID | PK (no DEFAULT) |
| `tenant_id` | UUID | NOT NULL, FK tenants |
| `subject_tenant_id` | UUID | NOT NULL, FK tenants |
| `object_node_id` | UUID | NOT NULL (logical FK to nodes) |
| `capability` | TEXT | NOT NULL |
| `valid_from` | TIMESTAMPTZ | NOT NULL, DEFAULT now() |
| `valid_to` | TIMESTAMPTZ | DEFAULT 'infinity' |
| `is_deleted` | BOOLEAN | DEFAULT false |
| `created_by` | UUID | NOT NULL |
| `created_by_event` | UUID | NOT NULL, FK events |

**Indexes:** `idx_grants_tenant (tenant_id)`, `idx_grants_subject (subject_tenant_id)`, `idx_grants_object (object_node_id)`, `idx_grants_active (subject_tenant_id, object_node_id, capability) WHERE is_deleted = false AND valid_to = 'infinity'`

#### Table 6: `blobs`
| Column | Type | Constraints |
|--------|------|-------------|
| `blob_id` | UUID | PK (no DEFAULT) |
| `tenant_id` | UUID | NOT NULL, FK tenants |
| `content_type` | TEXT | NOT NULL |
| `storage_ref` | TEXT | NOT NULL |
| `size_bytes` | BIGINT | NOT NULL |
| `node_id` | UUID | FK nodes, NULLABLE |
| `created_at` | TIMESTAMPTZ | DEFAULT now() |
| `created_by` | UUID | NOT NULL |

**Indexes:** `idx_blobs_tenant (tenant_id)`, `idx_blobs_node (node_id) WHERE node_id IS NOT NULL`

#### Table 7: `dicts`
| Column | Type | Constraints |
|--------|------|-------------|
| `dict_id` | UUID | PK (no DEFAULT) |
| `tenant_id` | UUID | NOT NULL, FK tenants |
| `type` | TEXT | NOT NULL |
| `key` | TEXT | NOT NULL |
| `value` | JSONB | NOT NULL |
| `valid_from` | TIMESTAMPTZ | DEFAULT now() |
| `valid_to` | TIMESTAMPTZ | DEFAULT 'infinity' |

**UNIQUE:** `(tenant_id, type, key, valid_from)`

**Index:** `idx_dicts_lookup (tenant_id, type, key)`

### RLS Policies (stated)

All 7 non-tenants tables + tenants have RLS enabled. Policies use `current_setting('app.tenant_ids')::uuid[]`:

- **tenants:** SELECT only
- **events:** SELECT + INSERT only (NO UPDATE/DELETE ΓÇö INV-APPEND)
- **nodes:** SELECT + INSERT + UPDATE (NO DELETE ΓÇö INV-SOFT)
- **edges:** SELECT + INSERT + UPDATE
- **grants:** SELECT (includes OR on subject_tenant_id) + INSERT
- **blobs:** SELECT + INSERT
- **dicts:** SELECT + INSERT

### Metatype Bootstrap

Self-referential node: `node_id = type_node_id = '00000000-0000-7000-0000-000000000001'`, in system tenant `'00000000-0000-7000-0000-000000000000'`. Created via a special migration block that also creates the bootstrap event.

### UUIDv7 Function

PL/pgSQL function extracting millisecond epoch into 6 bytes + 10 random bytes, setting version=7 and variant=10xx.

---

## Step 3: The Required Data Model (Derived from Operations)

### What the 15 tools require

| Tool | Tables Read | Tables Written | Special Requirements |
|------|------------|----------------|---------------------|
| `store_entity` | nodes (type resolution, current version) | events, nodes | Type node lookup in own + system tenant; schema validation; optimistic concurrency check |
| `find_entities` | nodes | ΓÇö | Embedding vector search (HNSW); JSONB filter queries; pagination |
| `connect_entities` | nodes (both endpoints), grants (cross-tenant check) | events, edges | Cross-tenant validation; edge type resolution |
| `explore_graph` | nodes, edges, grants | ΓÇö | Multi-hop traversal; cross-tenant grant checking per hop |
| `remove_entity` | nodes | events, nodes (UPDATE) | Soft-delete pattern |
| `query_at_time` | nodes | ΓÇö | Bitemporal point-in-time: `valid_from <= X < valid_to AND recorded_at <= Y` |
| `get_timeline` | events (by stream_id), nodes (versions) | ΓÇö | Merge event stream with node version history |
| `capture_thought` | nodes (type nodes, dedup candidates) | events, nodes, edges | LLM extraction; 3-tier dedup; multi-entity transaction |
| `get_schema` | nodes (WHERE type_node_id = METATYPE) | ΓÇö | Count queries per type |
| `get_stats` | nodes, edges, events, grants, blobs | ΓÇö | Aggregation queries |
| `propose_event` | varies by intent_type | events + fact tables | Generic event projection |
| `verify_lineage` | events, nodes, edges, grants | ΓÇö | FK integrity check |
| `store_blob` | nodes (if related_entity_id) | events, blobs | External storage upload |
| `get_blob` | blobs | ΓÇö | External storage download |
| `lookup_dict` | dicts | ΓÇö | Temporal filter on valid_from/valid_to |

### Required temporal properties

1. **Business time (valid_from/valid_to):** Required on nodes (versioning), edges (relationship lifecycle), grants (capability lifetime), dicts (reference data validity). NOT required on events (they use occurred_at/recorded_at instead) or blobs (no temporal lifecycle).
2. **System time (recorded_at):** Required on nodes and edges for `query_at_time` with `recorded_at` parameter. The events table has both occurred_at (business) and recorded_at (system). Grants have no recorded_at ΓÇö gap analysis below.
3. **Event ordering:** UUIDv7 provides natural ordering. `recorded_at` provides queryable timestamp.

### Required indexes (derived from query patterns)

| Query Pattern | Tool(s) | Index Needed |
|--------------|---------|-------------|
| Nodes by tenant + active | find_entities, get_schema | `idx_nodes_active` Γ£ô |
| Nodes by embedding similarity | find_entities | `idx_nodes_embedding` (HNSW) Γ£ô |
| Nodes by node_id + version | query_at_time, store_entity | `idx_nodes_stream` Γ£ô |
| Edges by source_id | explore_graph | `idx_edges_source` Γ£ô |
| Edges by target_id | explore_graph (incoming) | `idx_edges_target` Γ£ô |
| Events by stream_id | get_timeline, verify_lineage | `idx_events_stream` Γ£ô |
| Grants by subject + object + capability | connect_entities, explore_graph | `idx_grants_active` Γ£ô |
| Nodes by data->>'name' ILIKE | capture_thought dedup (tier 1) | **MISSING** ΓÇö no GIN/functional index on data |
| Nodes by type_node_id + tenant + active | get_schema counts | `idx_nodes_active` Γ£ô |
| Events by tenant + occurred_at | get_timeline with time_range | `idx_events_occurred` Γ£ô |
| Dicts by tenant + type + key | lookup_dict | `idx_dicts_lookup` Γ£ô |

---

## Step 4: Gap Analysis

### 4.1 Gaps (required but missing)

**GAP-1: Missing CHECK constraint on `nodes.epistemic`**
The column is `TEXT NOT NULL DEFAULT 'hypothesis'` with no constraint. The system enforces three values: `hypothesis`, `asserted`, `confirmed`. Without a CHECK, any string can be inserted.
- **Fix:** `CHECK (epistemic IN ('hypothesis', 'asserted', 'confirmed'))`

**GAP-2: Missing CHECK constraint on `grants.capability`**
The column is `TEXT NOT NULL` with no constraint. The system uses three values: `READ`, `WRITE`, `TRAVERSE`.
- **Fix:** `CHECK (capability IN ('READ', 'WRITE', 'TRAVERSE'))`

**GAP-3: Missing `created_by_event` on `blobs` table**
The spec's INV-LINEAGE says "Every fact row (nodes, edges, grants) MUST have a non-null created_by_event FK." Blobs are explicitly excluded from this list. However, the `blob_stored` projection creates both an event and a blob row within the same transaction. The event exists, but the blob has no FK back to it. This breaks traceability for `verify_lineage`-like checks on blobs and is inconsistent with the other fact tables.
- **Fix:** Add `created_by_event UUID NOT NULL REFERENCES events` to blobs. Or accept the inconsistency as intentional (blobs are "utility storage" per ┬º2.0).
- **Severity:** Low ΓÇö blobs are not in the formal lineage invariant.

**GAP-4: Missing `note` type node in seed data**
The capture_thought edge case says: "LLM returns entity type not in tenant's type_nodes: create as generic 'note' type (must exist as a fallback type_node; seed data includes it)." But ┬º7.8 seed data does NOT include a "note" type node for any tenant. Every tenant needs one, or it should be in the system tenant.
- **Fix:** Add a "note" type node to system tenant seed data.

**GAP-5: Missing index for deduplication exact match**
Tier 1 of the deduplication algorithm performs: `data->>'name' ILIKE $name`. No index supports this. At gen1 scale (<10K nodes/tenant), this is a sequential scan that's acceptable but will not scale.
- **Fix:** Add a GIN index: `CREATE INDEX idx_nodes_data_name ON nodes ((data->>'name')) WHERE is_deleted = false AND valid_to = 'infinity';` Or a trigram index for ILIKE.

**GAP-6: Missing `tenant_id` parameter on `store_blob` and `lookup_dict` Zod schemas**
Both tools need a tenant context for RLS filtering and data association. The Zod schemas (`StoreBlobParams`, `LookupDictParams`) lack `tenant_id`. For `store_blob`, tenant can be inferred from `related_entity_id` when present, but standalone blobs (no `related_entity_id`) with multi-tenant tokens have no way to specify the target tenant. Same for `lookup_dict`.
- **Fix:** Add `tenant_id: z.string().uuid().optional()` to both schemas.

**GAP-7: No mutation path for `dicts`**
There is no tool to create or update dict entries. `lookup_dict` is read-only. The events CHECK constraint has no `dict_created` intent_type. Dicts exist entirely outside the event-sourcing model.
- **Impact:** Dicts must be populated via direct SQL or a future admin tool. This is acceptable if dicts are seed data, but if agents need to create reference data at runtime, a tool is needed.
- **Severity:** Low ΓÇö dicts are reference data, likely admin-managed.

**GAP-8: Missing `recorded_at` on `grants`**
Grants have `valid_from`/`valid_to` (business time) but no `recorded_at` (system time). For full bitemporality, you'd want to know when the system learned about the grant. However, the `created_by_event` FK's event has `recorded_at`, so this is recoverable via a join.
- **Severity:** Very low ΓÇö grants are not queried bitemporally.

**GAP-9: `remove_entity` Zod schema lacks `tenant_id`**
`RemoveEntityParams` is just `{ entity_id: z.string().uuid() }`. The tenant is inferred by looking up the entity. This works but requires an extra query. More critically, scope checking needs the tenant_id before the operation. The implementation must do a lookup first, which is implied by the projection pseudocode but is a round-trip cost.
- **Severity:** Very low ΓÇö this is standard for entity-centric operations.

### 4.2 Over-engineering (stated but unnecessary or redundant)

**OVER-1: `edge_ids UUID[]` on events table**
Only `thought_captured` and `edge_created`/`edge_removed` populate this column. All other intent_types leave it as `'{}'`. It's 24 bytes per empty array per event row. At scale, this adds up on the append-only events table.
- **Verdict:** Keep ΓÇö enables "find all events affecting edge X" without JSONB parsing. The storage cost is marginal vs. the query benefit.

**OVER-2: `node_ids UUID[]` on events table**
Similar to edge_ids. Most events have a single node (already in `stream_id`). The array is primarily useful for `thought_captured` which creates multiple nodes.
- **Verdict:** Keep ΓÇö same reasoning. `stream_id` identifies the primary entity; `node_ids` identifies all affected entities.

**OVER-3: `recorded_at` on events (redundant with UUIDv7)**
D-001 explicitly analyzed this and decided to keep it. 8 bytes per row. Enables direct indexed scans without UUID parsing.
- **Verdict:** Keep ΓÇö justified by the decision log.

**OVER-4: `idx_events_intent (tenant_id, intent_type)`**
This index supports queries like "find all entity_created events for a tenant." No tool directly performs this query. `get_stats` counts by intent_type, but it could use `idx_events_tenant` with a filter.
- **Verdict:** Keep ΓÇö useful for `get_stats.recent_activity` and admin diagnostics.

**OVER-5: `idx_edges_active (tenant_id) WHERE is_deleted = false AND valid_to = 'infinity'`**
This partial index only includes tenant_id. It's useful for counting active edges per tenant (`get_stats`) but doesn't accelerate edge lookups by source/target (which have their own indexes).
- **Verdict:** Marginal ΓÇö could be removed if `get_stats` is acceptable without it. Keep for now.

### 4.3 Validated Design (stated matches required)

The following structures are correctly designed and well-justified:

1. **Events table as immutable append-only log** ΓÇö core of event-sourcing, no UPDATE/DELETE RLS policies
2. **Nodes composite PK `(node_id, valid_from)`** ΓÇö enables bitemporal versioning with EXCLUDE constraint preventing overlaps
3. **Edges simple PK** ΓÇö D-002's decision is correct; edge versioning adds complexity without gen1 use cases
4. **HNSW index on embeddings** ΓÇö appropriate for <100K vectors, correct operator class
5. **RLS policy pattern** ΓÇö application-level + RLS safety net (D-013) is pragmatic for cross-tenant edges
6. **Type nodes as regular nodes** ΓÇö reflexive ontology, no separate registry table
7. **Metatype self-reference** ΓÇö well-known UUID, DEFERRABLE constraint for bootstrap
8. **Actor nodes** ΓÇö D-015's decision for graph-native actor identity
9. **Grants table structure** ΓÇö sufficient for gen1 federation scenarios
10. **EventΓåÆfact table atomic transaction** ΓÇö INV-ATOMIC enforced by projection pseudocode
11. **Soft-delete pattern** ΓÇö consistent across nodes, edges, grants
12. **The 10 intent_types in the CHECK constraint** ΓÇö complete; every one has a projection entry
13. **UUIDv7 for natural time ordering** ΓÇö correct for event streams and the events PK

---

## Step 5: Structural Integrity Analysis

### 5.1 Normalization

The schema is appropriately normalized for its purpose:

- **Not over-normalized:** Common queries (find active nodes by tenant and type) can be answered from the `nodes` table alone without joins. The `data` JSONB column avoids the EAV anti-pattern while keeping entity-specific attributes flexible.
- **Not under-normalized:** Entity relationships are edges, not nested JSONB (INV-NOJSON). Tenant information is a separate table with FK references.
- **One concern:** The `data` column stores full snapshots on every version. For entities with large `data` that change one field, this duplicates unchanged data. Acceptable at gen1 scale; a diff-based approach would be a gen3 optimization.

### 5.2 Circular Dependencies

- **Intentional:** `nodes.type_node_id ΓåÆ nodes.node_id` (metatype self-reference). Handled via DEFERRABLE constraint and bootstrap migration.
- **Intentional:** `events.created_by ΓåÆ nodes.node_id` (actor nodes). Bootstrap uses metatype_id as created_by (self-referential exception).
- **None unintentional.** The dependency graph is: `tenants ΓåÉ events ΓåÉ nodes ΓåÉ edges/grants/blobs`. The `nodes ΓåÆ nodes` (via type_node_id) and `events ΓåÆ nodes` (via created_by) are the only cycles, both intentional.

### 5.3 Query Complexity

All 15 tools can be implemented with O(1) or O(n) PostgreSQL queries:

| Tool | Complexity | Notes |
|------|-----------|-------|
| `store_entity` (create) | O(1) ΓÇö 1 INSERT events + 1 INSERT nodes | Plus type resolution query |
| `store_entity` (update) | O(1) ΓÇö 1 INSERT events + 1 UPDATE nodes + 1 INSERT nodes | Plus version check |
| `find_entities` | O(log n) ΓÇö HNSW ANN search or B-tree scan | Plus JSONB filter |
| `connect_entities` | O(1) ΓÇö 2 lookups + 1 grant check + 2 INSERTs | |
| `explore_graph` | O(d * k) ΓÇö d=depth, k=branching factor | Bounded by max_results=500 |
| `query_at_time` | O(log n) ΓÇö B-tree range scan on (node_id, valid_from) | |
| `get_timeline` | O(k) ΓÇö k = events in stream | Uses idx_events_stream |
| `capture_thought` | O(m + dedup) ΓÇö m=mentioned entities | Dedup involves embedding search |
| `get_schema` | O(t) ΓÇö t=type nodes in tenant | |
| `get_stats` | O(count queries) ΓÇö 5-7 COUNT(*) queries | Could benefit from materialized views at scale |
| `verify_lineage` | O(f) ΓÇö f=fact rows for entity | |

**Concern:** `get_stats` performs multiple COUNT(*) queries. At >100K nodes, these become expensive without approximate counting or materialized views. Acceptable at gen1 scale.

### 5.4 Ordering Assumptions

- **UUIDv7 ordering = insertion order:** Correct within a single process. In a distributed system, clock skew could cause out-of-order UUIDs across nodes. Gen1 is single-instance (Supabase Edge Function), so this is safe.
- **Event stream ordering:** Events within a stream (stream_id) are ordered by event_id (UUIDv7). No explicit sequence number. This works if events are created sequentially per entity. Concurrent events on the same entity could have millisecond-close UUIDs, but the EXCLUDE constraint on nodes prevents overlapping versions, effectively serializing writes.
- **No explicit `version` column on nodes:** Version is derived as `COUNT(*) WHERE node_id = X`. This works but requires an aggregate query on every `store_entity` response. A denormalized `version INT` column on the latest node row would be more efficient.

### 5.5 Scaling Characteristics

| Table | Growth Pattern | Concern |
|-------|---------------|---------|
| `events` | Append-only, never deleted | **Largest table.** Every mutation adds a row. At 1000 mutations/day = 365K rows/year. Contains full JSONB payloads (old_data, new_data snapshots). This is the primary scaling concern. |
| `nodes` | Grows with entities ├ù versions | Second largest. Each update creates a new row (old row's valid_to is closed). Soft-deleted rows remain forever. |
| `edges` | Moderate | Grows with relationships. Soft-deleted rows remain. |
| `grants` | Small | Bounded by tenant count ├ù node count, practically very small. |
| `blobs` | Metadata only | Tiny ΓÇö actual data is in object storage. |
| `dicts` | Small | Reference data, bounded. |

**Unbounded growth mitigation options (gen2+):** Event archival, node version compaction, GDPR crypto-shredding.

### 5.6 Temporal Consistency

The EXCLUDE constraint `USING gist (node_id WITH =, tstzrange(valid_from, valid_to) WITH &&)` prevents overlapping versions. This is the primary defense against temporal inconsistency.

**Potential inconsistency scenarios:**

1. **Race condition on entity update:** Two concurrent `store_entity` calls for the same `node_id`. Both read `valid_to = 'infinity'`, both try to close the current version. The first UPDATE succeeds; the second finds 0 rows affected (WHERE valid_to = 'infinity' no longer matches). The projection pseudocode handles this: "If UPDATE affected 0 rows ΓåÆ ROLLBACK. Return E004 CONFLICT."
2. **Backdated valid_from:** If an entity is created with `valid_from = '2025-01-01'` and another version already exists at that time, the EXCLUDE constraint rejects it. Correct behavior.
3. **valid_to set to a time before valid_from:** No CHECK constraint prevents `valid_to < valid_from`. The EXCLUDE constraint would allow a zero-width or negative range. **Fix needed:** Add `CHECK (valid_to > valid_from OR valid_to = 'infinity')` on nodes.

### 5.7 Event Sourcing Integrity

The event stream can diverge from fact tables under these failure modes:

1. **Transaction failure between event INSERT and fact INSERT:** Handled by INV-ATOMIC ΓÇö both are in the same transaction, so they either both commit or both roll back.
2. **Application crash after COMMIT but before async embedding:** Acceptable ΓÇö entity exists without embedding. Embedding is retried via queue.
3. **Manual database modification:** Direct SQL bypassing the event layer. The `verify_lineage` tool detects this (orphaned facts without events).
4. **Bug in projection logic:** An event is stored but the fact table mutation is incorrect (wrong column values). `verify_lineage` checks FK integrity but NOT data correctness. Event replay (gen3) would be needed to reconstruct correct state.

**Missing capability:** There is no event replay mechanism in gen1. If projection logic has a bug and produces incorrect fact table state, the only recovery is manual SQL correction. Gen3 roadmap includes "Event replay: rebuild fact tables from event stream."

### 5.8 Vector Index Performance

At 1536 dimensions with HNSW (m=16, ef_construction=64):
- 15K vectors: 480 QPS, 16ms p95, 1GB RAM
- 100K vectors: 240 QPS, 126ms p95, 4GB RAM

Supabase's smallest compute addon (Micro, ~1GB RAM) can handle ~15K vectors. The Small addon (~2GB RAM) handles ~50K. For the Pettson scenario (~50 seed nodes + growth to ~1000), the Micro/Small addon is sufficient. The HNSW index with these parameters is viable.

**Concern:** The HNSW index covers ALL node rows, including historical versions (closed valid_to) and soft-deleted nodes. The index should ideally only cover active nodes. The `find_entities` query already filters `WHERE is_deleted = false AND valid_to = 'infinity'`, but the HNSW index indexes all rows, wasting space on inactive versions.

**Fix:** Add a partial HNSW index:
```sql
CREATE INDEX idx_nodes_embedding ON nodes
USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64)
WHERE is_deleted = false AND valid_to = 'infinity' AND embedding IS NOT NULL;
```
**Caveat:** pgvector HNSW may not support partial indexes well (implementation-dependent). Needs verification against the specific pgvector version on Supabase.

---

## Step 6: The Data Model Report

### 6.1 Validated Data Structures

The following are correctly designed and should remain as-is:

- `tenants` table structure and hierarchy
- `events` table: immutable append-only with CHECK on intent_type, UUIDv7 PK, recorded_at retention
- `nodes` table: composite PK, EXCLUDE constraint, embedding column, epistemic field
- `edges` table: simple PK (D-002), soft-delete, cross-tenant source/target
- `grants` table: subject_tenantΓåÆobject_node capability model with temporal bounds
- `blobs` table: metadata-only pattern with storage_ref
- `dicts` table: temporal reference data with UNIQUE constraint
- All stated RLS policies
- All stated indexes (with modifications noted below)
- Metatype bootstrap sequence
- UUIDv7 generation function
- The 10-value CHECK constraint on events.intent_type
- The projection matrix (all 10 intent_types covered)

### 6.2 Unnecessary Structures (Remove Candidates)

None identified for removal. Every table, column, and index serves at least one tool or invariant. The `recorded_at` on events is technically redundant with UUIDv7 but is justified by D-001.

### 6.3 Missing Structures (Additions Needed)

| ID | What | Where | Justification | Priority |
|----|------|-------|---------------|----------|
| M-1 | `CHECK (epistemic IN ('hypothesis','asserted','confirmed'))` | nodes table | Domain constraint enforcement. Without it, arbitrary strings can be inserted, violating the epistemic model. | High |
| M-2 | `CHECK (capability IN ('READ','WRITE','TRAVERSE'))` | grants table | Domain constraint enforcement. | High |
| M-3 | `CHECK (valid_to > valid_from)` | nodes table | Prevents invalid temporal ranges. The EXCLUDE constraint handles overlaps but not negative ranges. | Medium |
| M-4 | `tenant_id: z.string().uuid().optional()` | StoreBlobParams, LookupDictParams Zod schemas | Multi-tenant tokens cannot disambiguate target tenant for standalone blobs or dict lookups. | Medium |
| M-5 | "note" type node | System tenant seed data | Required by capture_thought fallback logic but absent from ┬º7.8 seed data. | High |
| M-6 | GIN or functional index on `data->>'name'` | nodes table | Deduplication tier 1 (exact name match) does ILIKE on JSONB. Sequential scan at scale. | Low (gen1) |
| M-7 | `created_by_event UUID REFERENCES events` | blobs table (optional) | Consistency with other fact tables. Currently the only fact table without event lineage traceability via FK. | Low |
| M-8 | Partial HNSW index filter | nodes embedding index | Current index includes deleted/historical rows, wasting space and potentially degrading search quality. | Medium |

### 6.4 Structural Risks

**RISK-1: Events table unbounded growth (Medium)**
The events table is append-only and immutable. Every mutation creates a row with full JSONB payloads (including `old_data` and `new_data` snapshots). At scale, this becomes the dominant storage cost. No archival or compaction strategy exists in gen1.
- *Mitigation:* Acceptable at gen1 scale (<100K events). Gen2+ should plan event archival or cold storage.

**RISK-2: `get_stats` COUNT(*) performance (Low-Medium)**
Multiple COUNT(*) queries on large tables with RLS filtering. PostgreSQL COUNT(*) is inherently expensive (no row count cache).
- *Mitigation:* At gen1 scale, acceptable. Consider materialized views or approximate counts (pg_class.reltuples) if stats queries exceed 500ms.

**RISK-3: Derived `version` field requires aggregate query (Low)**
`StoreEntityResult.version` is defined as "COUNT(*) of node rows with this node_id." Every store_entity response requires this count. A denormalized `version INT` column would avoid this.
- *Mitigation:* At gen1 entity volumes, the count is trivially fast (a few rows per node_id). Not worth schema change.

**RISK-4: Grants are node-level, not type-level (Design constraint)**
The Pettson validation grants specific nodes (e.g., "the package node"), not entire types. To grant access to "all packages," you need N grants for N package nodes. This doesn't scale for type-level access patterns.
- *Mitigation:* This is a known design decision, not a bug. Gen2 could add wildcard grants (object_node_id = NULL means "all nodes of type X in tenant").

**RISK-5: HNSW index includes inactive node versions (Medium)**
The embedding index covers all rows, not just active ones. When nodes are updated, the old version's embedding remains indexed, consuming space and potentially appearing in ANN results (though the WHERE clause in the query filters them out, the ANN search may waste work on inactive rows).
- *Mitigation:* Add the partial index filter from M-8 above, or accept the overhead at gen1 scale.

**RISK-6: No dict mutation path (Low)**
Dicts exist outside the event-sourcing model with no creation tool and no event type. This is inconsistent but intentional ΓÇö dicts are reference data.
- *Mitigation:* Document that dicts are admin-managed via direct SQL. Add a dict mutation tool in gen2 if agents need runtime reference data management.

**RISK-7: Edge type nodes in system tenant require special RLS handling (Low)**
Edge type nodes live in the system tenant (`00000000-0000-7000-0000-000000000000`). RLS policies on nodes require `tenant_id = ANY(current_setting('app.tenant_ids'))`. Type resolution must include the system tenant in `app.tenant_ids`, or bypass RLS. Since D-013 uses service_role (bypassing RLS), this works. But if RLS is ever tightened (D-013 gen2 revision), system tenant access needs explicit handling.
- *Mitigation:* Always include system tenant in `app.tenant_ids` when setting SET LOCAL, OR perform type resolution with service_role bypassing RLS.

### 6.5 Proposed Corrected Schema

Changes from the stated schema are marked with `-- [ADDED]` or `-- [MODIFIED]`:

```sql
-- ΓòÉΓòÉΓòÉ EXTENSIONS ΓòÉΓòÉΓòÉ
CREATE EXTENSION IF NOT EXISTS pgvector;
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- ΓòÉΓòÉΓòÉ UUIDv7 FUNCTION ΓòÉΓòÉΓòÉ
CREATE OR REPLACE FUNCTION uuid_generate_v7() RETURNS UUID AS $$
DECLARE
  v_time BIGINT;
  v_bytes BYTEA;
BEGIN
  v_time := (extract(epoch from clock_timestamp()) * 1000)::BIGINT;
  v_bytes := decode(lpad(to_hex(v_time), 12, '0'), 'hex') || gen_random_bytes(10);
  v_bytes := set_byte(v_bytes, 6, (get_byte(v_bytes, 6) & x'0f'::int) | x'70'::int);
  v_bytes := set_byte(v_bytes, 8, (get_byte(v_bytes, 8) & x'3f'::int) | x'80'::int);
  RETURN encode(v_bytes, 'hex')::UUID;
END $$ LANGUAGE plpgsql VOLATILE;

-- ΓòÉΓòÉΓòÉ TABLE 1: TENANTS ΓòÉΓòÉΓòÉ
CREATE TABLE tenants (
  tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id       UUID REFERENCES tenants,
  name            TEXT NOT NULL,
  config          JSONB DEFAULT '{}',
  created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_tenants_parent ON tenants (parent_id) WHERE parent_id IS NOT NULL;

-- ΓòÉΓòÉΓòÉ TABLE 2: EVENTS ΓòÉΓòÉΓòÉ
CREATE TABLE events (
  event_id        UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
  tenant_id       UUID NOT NULL REFERENCES tenants,
  intent_type     TEXT NOT NULL CHECK (intent_type IN (
    'entity_created', 'entity_updated', 'entity_removed',
    'edge_created', 'edge_removed',
    'epistemic_change', 'thought_captured',
    'grant_created', 'grant_revoked',
    'blob_stored'
  )),
  payload         JSONB NOT NULL,
  stream_id       UUID,
  node_ids        UUID[] DEFAULT '{}',
  edge_ids        UUID[] DEFAULT '{}',
  occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_by      UUID NOT NULL
);

CREATE INDEX idx_events_tenant ON events (tenant_id);
CREATE INDEX idx_events_stream ON events (stream_id) WHERE stream_id IS NOT NULL;
CREATE INDEX idx_events_intent ON events (tenant_id, intent_type);
CREATE INDEX idx_events_occurred ON events (tenant_id, occurred_at DESC);
CREATE INDEX idx_events_recorded ON events (recorded_at DESC);

-- ΓòÉΓòÉΓòÉ TABLE 3: NODES ΓòÉΓòÉΓòÉ
CREATE TABLE nodes (
  node_id         UUID NOT NULL,
  tenant_id       UUID NOT NULL REFERENCES tenants,
  type_node_id    UUID NOT NULL,                          -- logical FK to nodes (application-enforced)
  data            JSONB NOT NULL DEFAULT '{}',
  embedding       VECTOR(1536),                           -- nullable, async-populated
  epistemic       TEXT NOT NULL DEFAULT 'hypothesis'
                  CHECK (epistemic IN ('hypothesis', 'asserted', 'confirmed')),  -- [ADDED]
  valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_to        TIMESTAMPTZ DEFAULT 'infinity'
                  CHECK (valid_to >= valid_from),                                 -- [ADDED]
  recorded_at     TIMESTAMPTZ DEFAULT now(),
  created_by      UUID NOT NULL,
  created_by_event UUID NOT NULL REFERENCES events,
  is_deleted      BOOLEAN DEFAULT false,
  PRIMARY KEY (node_id, valid_from),
  EXCLUDE USING gist (
    node_id WITH =,
    tstzrange(valid_from, valid_to) WITH &&
  )
);

CREATE INDEX idx_nodes_tenant ON nodes (tenant_id);
CREATE INDEX idx_nodes_type ON nodes (type_node_id);
CREATE INDEX idx_nodes_active ON nodes (tenant_id, type_node_id)
  WHERE is_deleted = false AND valid_to = 'infinity';
CREATE INDEX idx_nodes_stream ON nodes (node_id, valid_from DESC);
-- [MODIFIED] Added partial index filter for active rows only
CREATE INDEX idx_nodes_embedding ON nodes
  USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64)
  WHERE is_deleted = false AND valid_to = 'infinity' AND embedding IS NOT NULL;

-- ΓòÉΓòÉΓòÉ TABLE 4: EDGES ΓòÉΓòÉΓòÉ
CREATE TABLE edges (
  edge_id         UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
  tenant_id       UUID NOT NULL REFERENCES tenants,
  type_node_id    UUID NOT NULL,
  source_id       UUID NOT NULL,
  target_id       UUID NOT NULL,
  data            JSONB DEFAULT '{}',
  valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_to        TIMESTAMPTZ DEFAULT 'infinity',
  recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_by      UUID NOT NULL,
  created_by_event UUID NOT NULL REFERENCES events,
  is_deleted      BOOLEAN DEFAULT false
);

CREATE INDEX idx_edges_tenant ON edges (tenant_id);
CREATE INDEX idx_edges_source ON edges (source_id) WHERE is_deleted = false;
CREATE INDEX idx_edges_target ON edges (target_id) WHERE is_deleted = false;
CREATE INDEX idx_edges_type ON edges (type_node_id);
CREATE INDEX idx_edges_active ON edges (tenant_id)
  WHERE is_deleted = false AND valid_to = 'infinity';

-- ΓòÉΓòÉΓòÉ TABLE 5: GRANTS ΓòÉΓòÉΓòÉ
CREATE TABLE grants (
  grant_id        UUID PRIMARY KEY DEFAULT uuid_generate_v7(),  -- [MODIFIED] added DEFAULT
  tenant_id       UUID NOT NULL REFERENCES tenants,
  subject_tenant_id UUID NOT NULL REFERENCES tenants,
  object_node_id  UUID NOT NULL,                               -- logical FK to nodes
  capability      TEXT NOT NULL
                  CHECK (capability IN ('READ', 'WRITE', 'TRAVERSE')),  -- [ADDED]
  valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_to        TIMESTAMPTZ DEFAULT 'infinity',
  is_deleted      BOOLEAN DEFAULT false,
  created_by      UUID NOT NULL,
  created_by_event UUID NOT NULL REFERENCES events
);

CREATE INDEX idx_grants_tenant ON grants (tenant_id);
CREATE INDEX idx_grants_subject ON grants (subject_tenant_id);
CREATE INDEX idx_grants_object ON grants (object_node_id);
CREATE INDEX idx_grants_active ON grants (subject_tenant_id, object_node_id, capability)
  WHERE is_deleted = false AND valid_to = 'infinity';

-- ΓòÉΓòÉΓòÉ TABLE 6: BLOBS ΓòÉΓòÉΓòÉ
CREATE TABLE blobs (
  blob_id         UUID PRIMARY KEY DEFAULT uuid_generate_v7(),  -- [MODIFIED] added DEFAULT
  tenant_id       UUID NOT NULL REFERENCES tenants,
  content_type    TEXT NOT NULL,
  storage_ref     TEXT NOT NULL,
  size_bytes      BIGINT NOT NULL,
  node_id         UUID,                                         -- logical FK to nodes, nullable
  created_at      TIMESTAMPTZ DEFAULT now(),
  created_by      UUID NOT NULL,
  created_by_event UUID REFERENCES events                       -- [ADDED] optional lineage FK
);

CREATE INDEX idx_blobs_tenant ON blobs (tenant_id);
CREATE INDEX idx_blobs_node ON blobs (node_id) WHERE node_id IS NOT NULL;

-- ΓòÉΓòÉΓòÉ TABLE 7: DICTS ΓòÉΓòÉΓòÉ
CREATE TABLE dicts (
  dict_id         UUID PRIMARY KEY DEFAULT uuid_generate_v7(),  -- [MODIFIED] added DEFAULT
  tenant_id       UUID NOT NULL REFERENCES tenants,
  type            TEXT NOT NULL,
  key             TEXT NOT NULL,
  value           JSONB NOT NULL,
  valid_from      TIMESTAMPTZ DEFAULT now(),
  valid_to        TIMESTAMPTZ DEFAULT 'infinity',
  UNIQUE(tenant_id, type, key, valid_from)
);

CREATE INDEX idx_dicts_lookup ON dicts (tenant_id, type, key);
```

### Summary of Changes from Stated Schema

| Change | Type | Justification |
|--------|------|---------------|
| Added `CHECK (epistemic IN (...))` on nodes | Constraint | Enforces domain invariant at DB level |
| Added `CHECK (capability IN (...))` on grants | Constraint | Enforces domain invariant at DB level |
| Added `CHECK (valid_to >= valid_from)` on nodes | Constraint | Prevents invalid temporal ranges |
| Added partial filter to HNSW index | Index optimization | Excludes inactive/deleted rows from vector search |
| Added `DEFAULT uuid_generate_v7()` on grants, blobs, dicts PKs | Consistency | Matches events/edges/nodes pattern; application can still override |
| Added optional `created_by_event` on blobs | Column | Lineage traceability consistency |
| "note" type node needed in seed data | Seed data | Required by capture_thought fallback but missing |
| `tenant_id` needed on StoreBlobParams, LookupDictParams | Zod schema | Multi-tenant token disambiguation |

All other structures are validated and correct as specified.
