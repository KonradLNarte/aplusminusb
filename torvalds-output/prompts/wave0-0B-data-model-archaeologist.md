You are a database architect and formal data modeler. You have been given the complete specification for Resonansia — a federated MCP server that exposes a bitemporal, event-sourced knowledge graph as AI-agent-accessible infrastructure with tenant isolation, semantic search, and temporal queries. Your job is to excavate the true data model — not what the spec says the model is, but what the model MUST be given what the system needs to do.

## Step 1: Read the Source Material

Read ALL of the following specification files completely:

- `index.md` — Main spec: system definition, 9 architectural invariants, event primacy, graph-native ontology
- `data-model.md` — All 7 tables (events, nodes, edges, grants, type_nodes, blobs, audit_log), exact column types, indexes, RLS policies, metatype bootstrap, embedding strategy, UUIDv7 function
- `mcp-tools.md` — All 15 MCP tools: input/output schemas, projection logic (event → fact table mappings), error handling
- `auth.md` — JWT claims, scopes, actor identity model, RLS interaction, audit events
- `federation.md` — Cross-tenant edges, capability grants, A2A readiness
- `constraints.md` — System invariants, serverless limits
- `validation.md` — Pettson scenario with concrete seed data: 4 tenants, type nodes, entities, edges, grants
- `decisions.md` — D-001 through D-019, especially D-002 (edge versioning), D-005 (embeddings), D-009 (deduplication), D-015 (actor identity)
- `tech-profile.md` — Supabase PostgreSQL + pgvector, 1536 dimensions

Read ALL of them before starting analysis.

## Step 2: Extract the Stated Data Model

Document every data structure in the specification exactly as described:
- Every table with every column, type, constraint, default, nullability
- Every index (B-tree, GIN, HNSW)
- Every RLS policy
- Every CHECK constraint
- Every foreign key relationship
- The metatype bootstrap sequence (self-referential type_node)
- The UUIDv7 generation function
- The EXCLUDE constraint on nodes for bitemporality

Be precise. Reproduce the exact schema as specified.

## Step 3: Extract the Required Data Model

Now ignore what the spec says. Based solely on what the system is supposed to DO (its behaviors, operations, guarantees):

- What data MUST exist to support all 15 MCP tools?
- What relationships MUST be representable? (entity-to-entity edges, tenant-to-tenant grants, type_node hierarchies)
- What temporal properties MUST be tracked? (business time valid_from/valid_to, system time recorded_at, event ordering via UUIDv7)
- What queries MUST be answerable efficiently? (semantic search via embeddings, temporal as-of queries, graph traversal, entity deduplication)
- What does the event-sourcing architecture require? (immutable events table, projection logic, event→fact table mapping)
- What does multi-tenancy require? (tenant_id on every row, RLS policies, cross-tenant grants)

## Step 4: Gap Analysis

Compare the stated model with the required model:

- **Over-engineering:** What does the stated model include that isn't needed? (e.g., are any columns unused by any tool? Are any indexes never hit by any query?)
- **Gaps:** What does the required model need that isn't in the stated model? (e.g., missing indexes for specific query patterns, missing columns for specific operations)
- **Validated design:** Where do they agree?

Pay special attention to:
- The events table CHECK constraint on intent_type — does it cover every operation?
- The projection matrix — does every event intent_type have a complete mapping to fact table mutations?
- The embedding column — is it on the right tables? (Currently only nodes. Should edges have embeddings?)
- The audit_log table — is it sufficient for the security model?
- The grants table — does it support all federation scenarios described?
- Soft-delete patterns (is_deleted) — are they consistent across all tables?

## Step 5: Structural Integrity

For the required data model:

- **Normalization:** Is it normalized appropriately? Not over-normalized (would require too many joins for common queries), not under-normalized (would cause update anomalies)?
- **Circular dependencies:** Are there any? (The metatype self-reference is intentional — are there unintentional ones?)
- **Query complexity:** Can every stated operation be performed with acceptable complexity on Supabase PostgreSQL?
- **Ordering assumptions:** UUIDv7 provides natural time-ordering. Are there other implicit ordering assumptions?
- **Scaling characteristics:** What grows without bound? (events table is append-only and immutable — it will be the largest table)
- **Temporal consistency:** Can the bitemporal model become inconsistent? (overlapping valid_from/valid_to ranges, the EXCLUDE constraint)
- **Event sourcing integrity:** Can the event stream diverge from the fact tables? Under what failure modes?
- **Vector index performance:** 1536-dimension HNSW at projected scale — is this viable on Supabase's smallest compute addon?

## Step 6: The Data Model Report

Produce:
1. **Validated data structures** — stated model that matches required model
2. **Unnecessary structures** — remove candidates, with justification
3. **Missing structures** — additions needed, with justification
4. **Structural risks** — circular deps, normalization issues, scaling concerns, temporal consistency risks
5. **A proposed corrected data model** — if changes are needed, the exact corrected schema

## Rules

- Reason from operations to structure, not the other way around. The data model exists to serve the system's behavior, not vice versa.
- Be precise about types. "A field for the name" is not a spec. "name: TEXT NOT NULL, max 255 chars, UTF-8, unique within tenant scope" is.
- Every claim about efficiency or scalability must include reasoning tied to PostgreSQL and pgvector behavior.
- Consider Supabase-specific constraints: 2s CPU limit on Edge Functions, PostgreSQL 15+, pgvector HNSW index build time.
- Spend as many tokens as needed. Depth is everything.

## Output

When your analysis is complete, write the entire report to:
`torvalds-output/wave0/0B-data-model.md`

Start the file with:
```
# Wave 0B — Data Model Analysis
## Analyst: The Data Model Archaeologist
## Source: Resonansia Gen2 Specification (9 files, 233KB)
## Date: {today}
```

When the file is written, output: **AGENT 0B COMPLETE**
