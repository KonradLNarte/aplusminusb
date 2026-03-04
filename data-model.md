# RESONANSIA MCP SERVER — GEN 2 SPEC

> Part of [Gen2 Spec](index.md)

## 2. DATA MODEL

### 2.0 Conceptual layers

The data model has three layers. Understanding the layers is essential before reading the tables.

```
┌─────────────────────────────────────────────┐
│  ONTOLOGY LAYER (graph describes itself)     │
│  type_nodes: what types of things exist      │
│  type_nodes are nodes — reflexive, temporal  │
├─────────────────────────────────────────────┤
│  EVENT LAYER (immutable truth)               │
│  events: append-only, never updated/deleted  │
│  every mutation originates here              │
├─────────────────────────────────────────────┤
│  FACT LAYER (materialized projections)       │
│  nodes, edges, grants: derived from events   │
│  every row references its creating event     │
│  blobs, dicts: utility storage               │
└─────────────────────────────────────────────┘
```

### 2.1 Schema

These table definitions are tech-agnostic SQL. Column types use standard SQL; the Tech Profile maps them to implementation-specific types (e.g., `VECTOR` → pgvector dimension).

```sql
-- Table 1: TENANTS — isolation boundary
tenants (
  tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id       UUID REFERENCES tenants NULLABLE,     -- hierarchy for holding companies
  name            TEXT NOT NULL,
  config          JSONB DEFAULT '{}',                    -- feature flags, limits, plan
  created_at      TIMESTAMPTZ DEFAULT now()
)

-- Table 2: EVENTS — append-only immutable truth
-- This is THE source of truth. Fact tables are projections.
-- NEVER updated. NEVER deleted. Corrections are new events referencing the original.
-- [RESOLVED:gen1 D-001] UUIDv7 for event_id: YES. recorded_at retained for query ergonomics.
events (
  event_id        UUID PRIMARY KEY DEFAULT uuid_generate_v7(),  -- UUIDv7: embeds millisecond timestamp for natural ordering
  tenant_id       UUID NOT NULL REFERENCES tenants,
  intent_type     TEXT NOT NULL CHECK (intent_type IN (
    'entity_created', 'entity_updated', 'entity_removed',
    'edge_created', 'edge_removed',
    'epistemic_change', 'thought_captured',
    'grant_created', 'grant_revoked',
    'blob_stored'
  )),
  payload         JSONB NOT NULL,                         -- event-specific data (schema per intent_type, see section 3.4)
  stream_id       UUID,                                   -- groups events about the same entity (= node_id)
  node_ids        UUID[] DEFAULT '{}',                    -- related nodes
  edge_ids        UUID[] DEFAULT '{}',                    -- related edges
  occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),     -- when it happened in reality (business time / valid time)
  recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),     -- when system learned about it (system time / transaction time)
                                                          -- Redundant with UUIDv7 timestamp but retained for:
                                                          -- 1. Direct WHERE recorded_at > X filtering without UUID parsing
                                                          -- 2. Index-friendly range scans on system time
                                                          -- 3. Query readability
  created_by      UUID NOT NULL                           -- actor node_id (see section 4.6 actor identity)
)
-- INDEXES on events:
-- CREATE INDEX idx_events_tenant ON events (tenant_id);
-- CREATE INDEX idx_events_stream ON events (stream_id) WHERE stream_id IS NOT NULL;
-- CREATE INDEX idx_events_intent ON events (tenant_id, intent_type);
-- CREATE INDEX idx_events_occurred ON events (tenant_id, occurred_at DESC);
-- CREATE INDEX idx_events_recorded ON events (recorded_at DESC);

-- Table 3: NODES — entities in the knowledge graph (FACT LAYER — projection of events)
-- Multiple rows per node_id: one per bitemporal version.
-- Current version: valid_to = 'infinity' AND is_deleted = false.
nodes (
  node_id         UUID NOT NULL,                           -- stable identity, never changes meaning or tenant
  tenant_id       UUID NOT NULL REFERENCES tenants,
  type_node_id    UUID NOT NULL REFERENCES nodes,          -- FK to type node (ontology layer)
  data            JSONB NOT NULL DEFAULT '{}',              -- entity fields
  embedding       VECTOR NULLABLE,                         -- dimension set by tech profile
  epistemic       TEXT NOT NULL DEFAULT 'hypothesis',      -- 'hypothesis' | 'asserted' | 'confirmed'
  valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),      -- bitemporal: business time start
  valid_to        TIMESTAMPTZ DEFAULT 'infinity',          -- bitemporal: business time end
  recorded_at     TIMESTAMPTZ DEFAULT now(),               -- bitemporal: system time
  created_by      UUID NOT NULL,
  created_by_event UUID NOT NULL REFERENCES events,        -- which event created this row
  is_deleted      BOOLEAN DEFAULT false,
  PRIMARY KEY (node_id, valid_from),                       -- composite PK: identity + time
  -- Prevent overlapping valid ranges for the same node:
  EXCLUDE USING gist (
    node_id WITH =,
    tstzrange(valid_from, valid_to) WITH &&
  )
)

-- Table 4: EDGES — relationships between nodes (FACT LAYER)
-- Cross-tenant edges are the federation mechanism.
-- source_id and target_id may belong to different tenants.
-- Auth must verify access to BOTH endpoints.
-- [RESOLVED:gen1 D-002] Edge versioning: Option B — immutable edges, simple PK.
-- Rationale: Edges represent relationships. "Updating" an edge = soft-delete + create new.
-- Temporal queries on edges use valid_from/valid_to but don't need versioning.
-- No validation test requires edge versioning. Simplifies schema and cross-tenant RLS.
edges (
  edge_id         UUID PRIMARY KEY DEFAULT uuid_generate_v7(),  -- stable identity, UUIDv7
  tenant_id       UUID NOT NULL REFERENCES tenants,       -- owner tenant of the edge
  type_node_id    UUID NOT NULL,                           -- FK to type node (predicate, e.g. "contacted_via")
  source_id       UUID NOT NULL,                           -- FK to nodes (by node_id only, not version)
  target_id       UUID NOT NULL,                           -- FK to nodes (by node_id only, not version)
  data            JSONB DEFAULT '{}',
  valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),      -- when relationship began (business time)
  valid_to        TIMESTAMPTZ DEFAULT 'infinity',          -- when relationship ended (business time)
  recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),      -- system time
  created_by      UUID NOT NULL,
  created_by_event UUID NOT NULL REFERENCES events,
  is_deleted      BOOLEAN DEFAULT false
)
-- INDEXES on edges:
-- CREATE INDEX idx_edges_tenant ON edges (tenant_id);
-- CREATE INDEX idx_edges_source ON edges (source_id) WHERE is_deleted = false;
-- CREATE INDEX idx_edges_target ON edges (target_id) WHERE is_deleted = false;
-- CREATE INDEX idx_edges_type ON edges (type_node_id);
-- CREATE INDEX idx_edges_active ON edges (tenant_id) WHERE is_deleted = false AND valid_to = 'infinity';
-- [DECIDE:gen2] If edge update frequency exceeds 5% of edge operations, reconsider composite PK.

-- Table 5: GRANTS — capability-based access control (FACT LAYER)
-- Grants are graph-native: they are nodes in the access control graph.
-- A grant says: "subject_tenant may perform capability on object_node during valid_range."
grants (
  grant_id        UUID PRIMARY KEY,
  tenant_id       UUID NOT NULL REFERENCES tenants,       -- tenant that owns (issued) this grant
  subject_tenant_id UUID NOT NULL REFERENCES tenants,     -- who receives the capability
  object_node_id  UUID NOT NULL REFERENCES nodes,         -- what the capability applies to
  capability      TEXT NOT NULL,                           -- 'READ' | 'WRITE' | 'TRAVERSE'
  valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_to        TIMESTAMPTZ DEFAULT 'infinity',
  is_deleted      BOOLEAN DEFAULT false,                   -- [RESOLVED:gen1 D-003] Include is_deleted for pattern consistency with nodes/edges.
  created_by      UUID NOT NULL,
  created_by_event UUID NOT NULL REFERENCES events
)

-- Table 6: BLOBS — binary storage (files, images, documents)
-- Blob content is NOT stored in the database. storage_ref points to object storage.
blobs (
  blob_id         UUID PRIMARY KEY,
  tenant_id       UUID NOT NULL REFERENCES tenants,
  content_type    TEXT NOT NULL,
  storage_ref     TEXT NOT NULL,                           -- path/key in object storage
  size_bytes      BIGINT NOT NULL,
  node_id         UUID REFERENCES nodes NULLABLE,
  created_at      TIMESTAMPTZ DEFAULT now(),
  created_by      UUID NOT NULL
)

-- Table 7: DICTS — reference data (currencies, countries, account codes)
dicts (
  dict_id         UUID PRIMARY KEY,
  tenant_id       UUID NOT NULL REFERENCES tenants,
  type            TEXT NOT NULL,
  key             TEXT NOT NULL,
  value           JSONB NOT NULL,
  valid_from      TIMESTAMPTZ DEFAULT now(),
  valid_to        TIMESTAMPTZ DEFAULT 'infinity',
  UNIQUE(tenant_id, type, key, valid_from)
)
```

**Note on type_nodes:** Type nodes are regular nodes whose `type_node_id` points to a bootstrap "metatype" node. Since `nodes` has a composite PK `(node_id, valid_from)`, SQL FK constraints referencing `node_id` alone cannot be used (PostgreSQL FKs must reference the PK or a unique constraint). Therefore:

- `REFERENCES nodes` in the DDL above is **LOGICAL, not enforced as SQL FK constraints**.
- These references are enforced at the **application level** in tool implementation code.
- `type_node_id` on nodes and edges is validated by querying `SELECT 1 FROM nodes WHERE node_id = $type_node_id AND is_deleted = false AND valid_to = 'infinity'` before insert.
- The metatype FK constraint uses `DEFERRABLE INITIALLY DEFERRED` for self-referential bootstrap.
- The EXCLUDE constraint guarantees at most one active version per `node_id` at any point in time.
- The metatype is self-referential (its `type_node_id` = its own `node_id`). This bootstrap node is created during schema migration.
- Type nodes carry a `label_schema` field in their `data` JSONB. [RESOLVED:gen1 C-002] When a type node has a `label_schema`, validation against it is MANDATORY for `store_entity` (returns SCHEMA_VIOLATION on failure). Exception: `capture_thought` creates hypothesis entities with relaxed validation.
- Type node resolution searches BOTH the caller's tenant AND the system tenant (`00000000-0000-7000-0000-000000000000`), since edge type nodes and the actor type node live in the system tenant.

### 2.6 Gen1 schema precision (Supabase PostgreSQL profile)

All tables use `uuid_generate_v7()` for UUIDv7 PKs where specified. Requires: `CREATE EXTENSION IF NOT EXISTS pgvector; CREATE EXTENSION IF NOT EXISTS btree_gist;`

#### 2.6.1 Exact index definitions

```sql
-- TENANTS
CREATE INDEX idx_tenants_parent ON tenants (parent_id) WHERE parent_id IS NOT NULL;

-- EVENTS (see table definition above for full list)

-- NODES
CREATE INDEX idx_nodes_tenant ON nodes (tenant_id);
CREATE INDEX idx_nodes_type ON nodes (type_node_id);
CREATE INDEX idx_nodes_active ON nodes (tenant_id, type_node_id) WHERE is_deleted = false AND valid_to = 'infinity';
CREATE INDEX idx_nodes_stream ON nodes (node_id, valid_from DESC);  -- version history lookup
CREATE INDEX idx_nodes_embedding ON nodes USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);

-- EDGES (see table definition above for full list)

-- GRANTS
CREATE INDEX idx_grants_tenant ON grants (tenant_id);
CREATE INDEX idx_grants_subject ON grants (subject_tenant_id);
CREATE INDEX idx_grants_object ON grants (object_node_id);
CREATE INDEX idx_grants_active ON grants (subject_tenant_id, object_node_id, capability) WHERE is_deleted = false AND valid_to = 'infinity';

-- BLOBS
CREATE INDEX idx_blobs_tenant ON blobs (tenant_id);
CREATE INDEX idx_blobs_node ON blobs (node_id) WHERE node_id IS NOT NULL;

-- DICTS
CREATE INDEX idx_dicts_lookup ON dicts (tenant_id, type, key);
```

#### 2.6.2 RLS policies

```sql
-- Enable RLS on all tables
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE events ENABLE ROW LEVEL SECURITY;
ALTER TABLE nodes ENABLE ROW LEVEL SECURITY;
ALTER TABLE edges ENABLE ROW LEVEL SECURITY;
ALTER TABLE grants ENABLE ROW LEVEL SECURITY;
ALTER TABLE blobs ENABLE ROW LEVEL SECURITY;
ALTER TABLE dicts ENABLE ROW LEVEL SECURITY;

-- TENANTS: read own tenant(s) only
CREATE POLICY tenants_select ON tenants FOR SELECT
  USING (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));

-- EVENTS: append-only, read own tenant(s)
CREATE POLICY events_select ON events FOR SELECT
  USING (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));
CREATE POLICY events_insert ON events FOR INSERT
  WITH CHECK (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));
-- NO UPDATE OR DELETE POLICIES ON EVENTS (INV-APPEND)

-- NODES: read own tenant(s), insert own tenant(s), no hard delete
CREATE POLICY nodes_select ON nodes FOR SELECT
  USING (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));
CREATE POLICY nodes_insert ON nodes FOR INSERT
  WITH CHECK (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));
CREATE POLICY nodes_update ON nodes FOR UPDATE
  USING (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));
-- NO DELETE POLICY ON NODES (INV-SOFT)

-- EDGES: read if source or target tenant is accessible
-- [RESOLVED:gen1 D-013] Cross-tenant edge reads use application-level enforcement
-- RLS permits reading edges owned by accessible tenants; cross-tenant traversal
-- verified in application code via grants table lookup.
CREATE POLICY edges_select ON edges FOR SELECT
  USING (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));
CREATE POLICY edges_insert ON edges FOR INSERT
  WITH CHECK (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));
CREATE POLICY edges_update ON edges FOR UPDATE
  USING (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));

-- GRANTS: read own tenant(s) or where subject_tenant is accessible
CREATE POLICY grants_select ON grants FOR SELECT
  USING (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[])
    OR subject_tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));
CREATE POLICY grants_insert ON grants FOR INSERT
  WITH CHECK (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));

-- BLOBS
CREATE POLICY blobs_select ON blobs FOR SELECT
  USING (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));
CREATE POLICY blobs_insert ON blobs FOR INSERT
  WITH CHECK (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));

-- DICTS
CREATE POLICY dicts_select ON dicts FOR SELECT
  USING (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));
CREATE POLICY dicts_insert ON dicts FOR INSERT
  WITH CHECK (tenant_id = ANY(current_setting('app.tenant_ids')::uuid[]));

-- Note: The Edge Function connects with service_role (bypasses RLS) and sets
-- app.tenant_ids via SET LOCAL before queries. This is the D-013 pattern:
-- application-level enforcement with RLS as safety net.
-- SET LOCAL 'app.tenant_ids' = '{uuid1,uuid2}';
```

#### 2.6.3 Metatype bootstrap sequence

The metatype is a self-referential node: its `type_node_id` points to itself. This requires a specific migration order:

```sql
-- Step 1: Create tables WITHOUT the nodes self-referencing FK
-- (type_node_id FK is deferred or not enforced at table creation)

-- Step 2: Insert metatype node (self-referential)
-- This is the ONLY node created outside the normal event→projection flow.
-- It bootstraps the ontology layer.
DO $$
DECLARE
  v_metatype_id UUID := '00000000-0000-7000-0000-000000000001'; -- well-known fixed UUID
  v_bootstrap_event_id UUID := uuid_generate_v7();
  v_bootstrap_tenant_id UUID; -- the platform/system tenant
BEGIN
  -- Create system tenant
  INSERT INTO tenants (tenant_id, name, config)
  VALUES ('00000000-0000-7000-0000-000000000000', '__system__', '{"is_system": true}')
  ON CONFLICT DO NOTHING
  RETURNING tenant_id INTO v_bootstrap_tenant_id;

  v_bootstrap_tenant_id := '00000000-0000-7000-0000-000000000000';

  -- Create bootstrap event
  INSERT INTO events (event_id, tenant_id, intent_type, payload, stream_id, occurred_at, created_by)
  VALUES (v_bootstrap_event_id, v_bootstrap_tenant_id, 'entity_created',
    '{"type": "metatype", "data": {"name": "metatype", "description": "The type of all type nodes. Self-referential bootstrap node."}}',
    v_metatype_id, now(), v_metatype_id);  -- created_by = metatype itself (bootstrap)

  -- Create metatype node (self-referential: type_node_id = node_id)
  INSERT INTO nodes (node_id, tenant_id, type_node_id, data, epistemic, valid_from, valid_to, created_by, created_by_event, is_deleted)
  VALUES (v_metatype_id, v_bootstrap_tenant_id, v_metatype_id,
    '{"name": "metatype", "description": "The type of all type nodes. Self-referential bootstrap node.", "kind": "entity_type"}',
    'confirmed', now(), 'infinity', v_metatype_id, v_bootstrap_event_id, false);

  -- Create actor type node (needed for actor identity, see D-015)
  -- (Created via normal event→projection in seed data, not here)
END $$;

-- Step 3: Add FK constraints now that metatype exists
-- ALTER TABLE nodes ADD CONSTRAINT fk_nodes_type FOREIGN KEY (type_node_id) REFERENCES nodes(node_id) DEFERRABLE INITIALLY DEFERRED;
-- Note: FK references node_id only (not composite PK), validated against any version.
-- The DEFERRABLE constraint allows self-referential inserts within a transaction.
```

#### 2.6.4 UUIDv7 generation function

```sql
-- If uuid_generate_v7() is not available natively, use this PL/pgSQL implementation:
CREATE OR REPLACE FUNCTION uuid_generate_v7() RETURNS UUID AS $$
DECLARE
  v_time BIGINT;
  v_bytes BYTEA;
BEGIN
  v_time := (extract(epoch from clock_timestamp()) * 1000)::BIGINT;
  v_bytes := decode(lpad(to_hex(v_time), 12, '0'), 'hex') || gen_random_bytes(10);
  -- Set version (7) and variant (10xx)
  v_bytes := set_byte(v_bytes, 6, (get_byte(v_bytes, 6) & x'0f'::int) | x'70'::int);
  v_bytes := set_byte(v_bytes, 8, (get_byte(v_bytes, 8) & x'3f'::int) | x'80'::int);
  RETURN encode(v_bytes, 'hex')::UUID;
END $$ LANGUAGE plpgsql VOLATILE;
```

### 2.2 Invariant rules

These rules are absolute. No generation may weaken them. Each invariant has an ID for referencing in code comments and tests.

1. **INV-TENANT: Tenant isolation** — `tenant_id` on every row. No exceptions. Enforcement: RLS policies on every table. Violation example: a query returning nodes from a tenant not in the token's `tenant_ids`.

2. **INV-APPEND: Events are append-only** — Never update, never delete. Corrections are new events referencing the original via payload. Enforcement: RLS policy denying UPDATE and DELETE on `events` table; no application code path for event mutation. Violation example: an UPDATE statement on the `events` table.

3. **INV-BITEMP: Bitemporality** — `valid_from`/`valid_to` = business time. `recorded_at` (or UUIDv7 timestamp of PK) = system time. Enforcement: EXCLUDE constraint on `(node_id, tstzrange(valid_from, valid_to))` prevents overlapping versions. Violation example: two active node versions with overlapping time ranges.

4. **INV-SOFT: Soft deletes only** — `is_deleted = true` + `valid_to = now()`. Hard deletes only for GDPR erasure (gen2+). Enforcement: application-layer check; no DELETE permission via RLS on fact tables. Violation example: a DELETE statement on the `nodes` table.

5. **INV-XTEN: Cross-tenant edges** — `source_id` and `target_id` may belong to different tenants. Auth must verify both. Enforcement: tool-level check before creating edge; grants table consulted for cross-tenant access. Violation example: creating a cross-tenant edge without grants for both endpoints.

6. **INV-TYPE: Type nodes are the schema** — System works with only the bootstrap metatype (fully dynamic). Type nodes add validation and discovery. Enforcement: metatype bootstrap in migration script; `type_node_id` FK on every node and edge. Violation example: a node without a `type_node_id`.

7. **INV-BLOB: Blobs use external storage** — Only metadata in the database. `storage_ref` points to object storage. Enforcement: no BYTEA columns for file content. Violation example: storing file bytes directly in a JSONB field.

8. **INV-IDENT: Identity persistence** — A `node_id` never changes meaning or tenant. Once assigned, it is permanent. Enforcement: `node_id` is set at creation (= `stream_id` of first event) and never reassigned. Violation example: reusing a `node_id` for a different entity type.

9. **INV-LINEAGE: Event lineage** — Every fact row (`nodes`, `edges`, `grants`) MUST have a non-null `created_by_event` FK. Enforcement: NOT NULL constraint on `created_by_event` column; verified by acceptance test T13. Violation example: a row in `nodes` with `created_by_event = NULL`.

10. **INV-NOJSON: No business logic in JSONB** — Relationships between entities MUST be modeled as edges, not as foreign keys buried in `data` payloads. The `data` field is for descriptive attributes only. Enforcement: code review + schema validation in type nodes. Violation example: storing `{"related_to": "node-xyz"}` in `node.data` instead of creating an edge.

11. **INV-ATOMIC: Atomic event+projection** — An event and its projection to fact tables (`nodes`, `edges`, `grants`) MUST be written within the same database transaction. If projection fails, the event MUST be rolled back. There are no "orphaned events" and no "eventual consistency" between the event layer and the fact layer. Enforcement: all tool mutations wrap event creation + projection in a single `BEGIN...COMMIT` block. Violation example: an event row exists in `events` but its corresponding node row is missing from `nodes`. **Exception**: external side effects (embedding generation, LLM calls) happen OUTSIDE the transaction boundary — they run before (LLM extraction in `capture_thought`) or after (async embedding queue) the atomic write. See section 3.5 for details.

12. **INV-CONCURRENCY: Optimistic concurrency on updates** — When updating an existing entity via `store_entity(entity_id=X)`, the caller SHOULD provide `expected_version` (the `valid_from` timestamp of the version they read). If the current version's `valid_from` does not match, the operation MUST fail with a `CONFLICT` error. This prevents silent overwrites when two agents update the same entity concurrently. Enforcement: application-level check in `store_entity` projection logic. [RESOLVED:gen1 D-004] `expected_version` is OPTIONAL for gen1 (last-write-wins if omitted). [DECIDE:gen2] Make `expected_version` REQUIRED on `store_entity` updates. Violation example: two concurrent `store_entity` calls both succeed, with the second silently overwriting the first without the caller's awareness.

### 2.3 Epistemic status model

Nodes carry an `epistemic` field that reflects the provenance and certainty of the information:

```yaml
hypothesis:
  meaning: "System inferred this, not yet verified by a human."
  set_by: capture_thought tool (LLM extraction), automated imports
  transitions_to: [asserted, confirmed]

asserted:
  meaning: "A human or trusted agent explicitly stated this."
  set_by: Direct store_entity calls, manual data entry
  transitions_to: [confirmed]

confirmed:
  meaning: "Verified against an authoritative source."
  set_by: Explicit confirmation action, integration with source systems
  transitions_to: [] # terminal state; corrections are new entities
```

Epistemic status changes emit events. Search results MAY be filtered or ranked by epistemic status.

### 2.4 Embedding strategy

```yaml
# [RESOLVED:gen1 D-005] Embedding model: text-embedding-3-small, 1536 dimensions
# [RESOLVED:gen1 D-006] Embed edges: NO for gen1
# [RESOLVED:gen1 D-007] Embed events: NO for gen1
# [RESOLVED:gen1 D-008] Bulk embedding: sequential with batching

model: "text-embedding-3-small"
dimensions: 1536
provider: "OpenAI (via OpenRouter or direct)"
cost: "$0.02 per 1M tokens (~$0.002 per 1K entity embeddings)"

trigger: on node create or update, queue async embedding job
input: |
  Concatenation formula:
    text = type_node_name + ": " + JSON.stringify(node.data, null, 0)
  Example:
    "lead: {\"name\":\"Johan Eriksson\",\"interest\":\"cabin+tickets\",\"source\":\"cabin fair\"}"
  Max input length: 8191 tokens (model limit). Truncate data fields if exceeded.

storage: nodes.embedding column (VECTOR(1536) via pgvector)
search: |
  SELECT node_id, data, 1 - (embedding <=> query_embedding) AS similarity
  FROM nodes
  WHERE tenant_id = $tenant_id
    AND is_deleted = false
    AND valid_to = 'infinity'
    AND type_node_id = ANY($type_node_ids)  -- optional type filter
  ORDER BY embedding <=> query_embedding
  LIMIT $limit;
  -- Uses pgvector HNSW index for approximate nearest neighbor
  -- Performance at 1536d: 240 QPS, 126ms p95 at 100K vectors (4GB RAM)

index_definition: |
  CREATE INDEX idx_nodes_embedding ON nodes
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);
  -- HNSW parameters: m=16 (connections per layer), ef_construction=64 (build quality)
  -- Suitable for <100K vectors. Increase ef_construction for larger datasets.

constraints:
  - Embedding generation MUST be async (not in the request path)
  - Embedding pipeline MUST implement exponential backoff with jitter
  - Respect OpenAI rate limits: 3500 RPM, 1M TPM for text-embedding-3-small
  - Search latency target: <200ms p95 including RLS filtering at 100K nodes
  - Nodes without embeddings ARE queryable (excluded from semantic search, included in structured search)

bulk_strategy:
  batch_size: 100          # texts per API call (OpenAI batch endpoint)
  concurrency: 5           # concurrent API calls
  rate_limit: 3000         # requests per minute (leave headroom below 3500 RPM)
  retry_policy: "exponential backoff, base=1s, max=60s, max_retries=5"
  seed_data_estimate: "~50 nodes = 1 batch = <2 seconds"
  import_10k_estimate: "100 batches × 5 concurrent = ~30 seconds"

edges_and_events: |
  Edges are NOT embedded in gen1. Edge discovery is via graph traversal from entity search.
  Events are NOT embedded in gen1. Event search is via get_timeline with temporal filters.
  [DECIDE:gen2] Re-evaluate edge embedding if users need semantic search over relationships.
```

### 2.5 Research findings (resolved in gen1)

```yaml
# [RESOLVED:RESEARCH:gen1] PostgreSQL SQL:2011 temporal table support
finding: |
  PostgreSQL 18 (current stable, Sep 2025) adds temporal PRIMARY KEY and UNIQUE
  constraints with WITHOUT OVERLAPS syntax for application-time periods, and temporal
  FOREIGN KEY constraints with PERIOD clause. However, PostgreSQL STILL LACKS:
  - System-versioned tables (automatic row versioning by the database)
  - Temporal DML (UPDATE FOR PORTION OF, DELETE FOR PORTION OF)
  - Temporal queries (AS OF, BETWEEN...AND, FROM...TO)
  The temporal_tables extension (arkhipov) provides trigger-based system-time versioning
  but is a C extension NOT available on Supabase managed PostgreSQL.
decision: |
  DO NOT use PostgreSQL native temporal features. Retain manual bitemporality
  (valid_from/valid_to + recorded_at/UUIDv7). The event-sourced append-only architecture
  with explicit version management is more powerful and portable than SQL:2011 system-time.
  PostgreSQL 18's WITHOUT OVERLAPS could theoretically replace the EXCLUDE USING gist
  constraint on nodes, but the syntax difference is cosmetic and the EXCLUDE constraint
  is well-tested. No schema change.
references:
  - https://wiki.postgresql.org/wiki/SQL2011Temporal (updated Dec 2025)
  - https://neon.com/postgresql/postgresql-18/temporal-constraints
  - https://hashrocket.com/blog/posts/postgresql-18-temporal-constraints
  [RESEARCH:gen2] Re-evaluate when PostgreSQL adds system-time versioning.

# [RESOLVED:RESEARCH:gen1] UUIDv7 implications
finding: |
  UUIDv7 embeds millisecond-precision timestamp. recorded_at is technically redundant
  but extracting timestamp from UUID requires uuid_extract_timestamp() function which
  is not yet a standard PostgreSQL built-in (requires extension or custom function).
decision: |
  Retain recorded_at column. Cost: 8 bytes per row. Benefit: direct indexing, readable
  queries, no function dependency. See D-001 in decision log.
```

---
