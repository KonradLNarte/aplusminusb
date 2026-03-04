# RESONANSIA MCP SERVER — GEN 2 SPEC

> Part of [Gen2 Spec](index.md)

## 9. DECISION LOG

```yaml
decisions:

# ═══ RESOLVED DECISIONS ═══

- id: D-001
  generation: gen1
  marker: "[DECIDE:gen1] UUIDv7 for event_id — retain recorded_at?"
  decision: "Use UUIDv7 for event_id. Retain recorded_at column."
  alternatives: ["Derive system time from UUIDv7 only (drop recorded_at)", "Use random UUID + recorded_at only"]
  rationale: "UUIDv7 provides natural ordering and implicit timestamp. recorded_at retained because: (1) extracting timestamp from UUID requires custom function, (2) recorded_at enables direct indexed range scans, (3) query readability. Storage cost: 8 bytes/row, negligible."
  confidence: high
  question_this_if: "uuid_extract_timestamp() becomes a standard PostgreSQL built-in"
  references: ["https://wiki.postgresql.org/wiki/SQL2011Temporal"]
  spec_section_updated: "2.1 — events table: event_id type changed to UUID DEFAULT uuid_generate_v7(), recorded_at marked NOT NULL"

- id: D-002
  generation: gen1
  marker: "[DECIDE:gen1] Edge versioning strategy"
  decision: "Option B — Simple PK (edge_id), immutable edges with soft-delete"
  alternatives: ["Option A: Composite PK (edge_id, valid_from) — full bitemporality like nodes"]
  rationale: "Edges represent relationships, not entities. Edge 'updates' are rare — soft-delete + recreate covers all gen1 use cases. Removes EXCLUDE constraint complexity from edges. No validation test (T01-T14) requires edge versioning. Simplifies cross-tenant RLS. Gen0 recommended Option A for consistency, but consistency doesn't justify the complexity when the use case doesn't demand it."
  confidence: high
  question_this_if: "Edge update frequency exceeds 5% of edge operations OR temporal edge state queries are needed"
  references: []
  spec_section_updated: "2.1 — edges table: changed to simple PK, removed EXCLUDE constraint"

- id: D-003
  generation: gen1
  marker: "[DECIDE:gen1] Grants soft-delete"
  decision: "Include is_deleted flag for pattern consistency"
  alternatives: ["Rely solely on valid_to for revocation"]
  rationale: "Pattern consistency with nodes and edges. Queries use same WHERE is_deleted = false AND valid_to > now() pattern across all fact tables."
  confidence: high
  question_this_if: "Grants are never revoked mid-validity, making is_deleted truly redundant"
  references: []
  spec_section_updated: "2.1 — grants table: is_deleted retained with resolved comment"

- id: D-004
  generation: gen1
  marker: "[DECIDE:gen1] Optimistic concurrency — expected_version"
  decision: "Optional for gen1 (last-write-wins if omitted)"
  alternatives: ["Required from gen1"]
  rationale: "Gen1 is primarily single-agent-per-entity. Concurrent conflicts unlikely. Optional reduces adoption friction while making the mechanism available for agents that want it."
  confidence: high
  question_this_if: "Multiple agents concurrently edit the same entity before gen2"
  references: []
  spec_section_updated: "2.2 — INV-CONCURRENCY updated. 3.2 — store_entity expected_version marked optional"

- id: D-005
  generation: gen1
  marker: "[DECIDE:gen1] Embedding model and dimensions"
  decision: "text-embedding-3-small, 1536 dimensions"
  alternatives: ["text-embedding-3-small at 768d (~4.7% accuracy loss)", "text-embedding-3-large at 3072d (higher cost)"]
  rationale: "1536d is the default for text-embedding-3-small. Best accuracy for this model. At gen1 scale (<100K nodes): 240 QPS, 126ms p95 on 4GB RAM (pgvector HNSW). Storage ~600KB per 1K nodes. Cost: $0.02/1M tokens. Graphiti uses 1024d, Open Brain uses 768d — we choose higher for richer entity data."
  confidence: high
  question_this_if: "Storage cost exceeds $10/month OR search latency exceeds 200ms p95"
  references: ["https://supabase.com/docs/guides/ai/choosing-compute-addon", "https://platform.openai.com/docs/models/text-embedding-3-small"]
  spec_section_updated: "2.4 — embedding strategy fully specified"

- id: D-006
  generation: gen1
  marker: "[DECIDE:gen1] Embed edge data?"
  decision: "No for gen1"
  alternatives: ["Embed edge data field"]
  rationale: "Edges represent relationships with typically sparse data. Discovery is via graph traversal from entity search. Embedding edges doubles volume/cost without clear search benefit."
  confidence: high
  question_this_if: "Users frequently need to search for relationships by descriptive content"
  references: []
  spec_section_updated: "2.4 — edges_and_events section"

- id: D-007
  generation: gen1
  marker: "[DECIDE:gen1] Embed events?"
  decision: "No for gen1"
  alternatives: ["Embed event payloads"]
  rationale: "Events are internal audit records. Searched via get_timeline with temporal/entity filters. No semantic search use case."
  confidence: high
  question_this_if: "A use case emerges for semantic search over event payloads"
  references: []
  spec_section_updated: "2.4 — edges_and_events section"

- id: D-008
  generation: gen1
  marker: "[DECIDE:gen1] Bulk embedding strategy"
  decision: "Sequential with batching: 100 texts/batch, 5 concurrent, 3000 RPM"
  alternatives: ["Queue-based async (Redis/BullMQ)", "Database trigger-based"]
  rationale: "At gen1 scale (seed: ~50 nodes, import: <10K), sequential batching completes in seconds/minutes. No need for queue infrastructure. OpenAI batch API supports 100 texts per call."
  confidence: mid
  question_this_if: "Import volume exceeds 10K nodes or embedding queue exceeds 1000 pending"
  references: ["https://platform.openai.com/docs/guides/embeddings"]
  spec_section_updated: "2.4 — bulk_strategy section"

- id: D-009
  generation: gen1
  marker: "[DECIDE:gen1] Entity deduplication approach"
  decision: "Hybrid: exact match → embedding similarity → LLM disambiguation"
  alternatives: ["Option A: Embedding only", "Option B: Fuzzy text only", "Option D: LLM-only (Graphiti v1 approach)"]
  rationale: "Inspired by Graphiti's three-tier approach but simplified for gen1 scale (<10K entities/tenant). Exact match handles identical strings. Embedding similarity catches semantic equivalence. LLM disambiguation resolves the uncertain zone (0.85-0.95 cosine similarity). Full MinHash+LSH (Graphiti v2) deferred — not needed at gen1 entity volumes."
  confidence: mid
  question_this_if: "Duplicate entities exceed 5% of total OR false positive matches exceed 2%"
  references: ["https://arxiv.org/abs/2501.13956", "https://github.com/getzep/graphiti"]
  spec_section_updated: "3.2 — capture_thought deduplication_strategy with full algorithm"

- id: D-010
  generation: gen1
  marker: "[DECIDE:gen1] Filter operators in find_entities"
  decision: "Equality + $in for gen1"
  alternatives: ["Full set ($gte, $lte, $contains, $exists)"]
  rationale: "Covers most common search patterns. Range and text operators require careful JSONB indexing strategy deferred to gen2."
  confidence: high
  question_this_if: "Agents frequently need range or text-contains queries before gen2"
  references: []
  spec_section_updated: "3.2 — find_entities filter_operators section"

- id: D-011
  generation: gen1
  marker: "[DECIDE:gen1] Token format"
  decision: "Stateless JWT"
  alternatives: ["Opaque token + introspection endpoint"]
  rationale: "JWT validation is in-memory (no DB roundtrip). Critical for Edge Functions where latency matters. Short-lived (1h) mitigates revocation lag. Scope count ≤10, JWT size ~1KB."
  confidence: high
  question_this_if: "Token revocation latency (up to 1h) is unacceptable"
  references: ["https://modelcontextprotocol.io/specification/draft/basic/authorization"]
  spec_section_updated: "4.7 — auth decisions resolved"

- id: D-012
  generation: gen1
  marker: "[DECIDE:gen1] Auth provider integration"
  decision: "Supabase Auth for users + custom JWT signing for agents"
  alternatives: ["Fully custom auth server", "Auth0 integration"]
  rationale: "Leverages deployment platform's built-in auth. Agent tokens signed with Supabase JWT secret via admin-only endpoint."
  confidence: high
  question_this_if: "Supabase Auth cannot add custom claims (tenant_ids, scopes)"
  references: []
  spec_section_updated: "4.7 — auth decisions resolved"

- id: D-013
  generation: gen1
  marker: "[DECIDE:gen1] RLS policy pattern for cross-tenant edges"
  decision: "Option C — Application-level enforcement with RLS safety net"
  alternatives: ["Option A: JOIN-based RLS", "Option B: Security-definer function"]
  rationale: "Edge Function connects with service_role, sets app.tenant_ids via SET LOCAL. Application code checks grants table for cross-tenant operations. RLS serves as defense-in-depth. Pure RLS for cross-tenant edges requires complex JOINs in policies."
  confidence: mid
  question_this_if: "Security audit requires pure RLS without service_role bypass"
  references: ["https://github.com/aws-samples/sample-multi-tenant-saas-mcp-server"]
  spec_section_updated: "2.6.2 — RLS policies, 4.7 — auth decisions"

- id: D-014
  generation: gen1
  marker: "[DECIDE:gen1] Scope granularity"
  decision: "Current tenant:{id}:{resource}:{action} syntax sufficient for gen1"
  alternatives: ["Node-level scopes", "Opaque tokens with DB-stored permissions"]
  rationale: "At ≤10 scopes per token, JWT size ~1KB. Scope syntax provides adequate granularity."
  confidence: high
  question_this_if: "Scope count exceeds 20 per token OR JWT size exceeds 4KB"
  references: []
  spec_section_updated: "4.5 — scope granularity note"

- id: D-015
  generation: gen1
  marker: "[DECIDE:gen1] Actor identity"
  decision: "Option B — Actors as graph nodes"
  alternatives: ["Option A: Separate actors table", "Option C: Raw JWT sub"]
  rationale: "Consistent with graph-native ontology. Actors queryable as entities. Actor-entity relationships implicit via created_by. Overhead: one node creation per agent lifecycle per tenant (not per request)."
  confidence: high
  question_this_if: "Actor node creation overhead measurable at >100 concurrent new agents"
  references: []
  spec_section_updated: "4.6 — actor identity fully specified with type node schema and mapping rules"

- id: D-016
  generation: gen1
  marker: "[DECIDE:gen1] capture_thought execution strategy"
  decision: "Option A — Synchronous"
  alternatives: ["Option B: Async (job_id)", "Option C: Split (sync LLM, async linking)"]
  rationale: "Analysis: LLM call ~1-3s wall (<50ms CPU) + dedup ~100ms wall (~20ms CPU) + DB ~100ms wall (~30ms CPU). Total: ~2-5s wall, <200ms CPU. Fits Supabase limits (2s CPU, 150s wall-clock) because LLM/DB are async I/O."
  confidence: mid
  question_this_if: "capture_thought p95 latency exceeds 10s or CPU exceeds 1.5s"
  references: ["https://supabase.com/docs/guides/functions/limits"]
  spec_section_updated: "6.1 — constraints section"

- id: D-017
  generation: gen1
  marker: "[DECIDE:gen1] License"
  decision: "Apache 2.0"
  alternatives: ["BSL 1.1 (restricts commercial use)"]
  rationale: "Maximizes adoption. No usage restrictions. Compatible with enterprise environments."
  confidence: high
  question_this_if: "Commercial exploitation by competitors becomes a concern"
  references: []
  spec_section_updated: "N/A — license is a project-level decision, not a spec section"

- id: D-018
  generation: gen1
  marker: "[DECIDE:gen1] Blob storage"
  decision: "External (Supabase Storage)"
  alternatives: ["Inline BYTEA"]
  rationale: "Already decided in gen0 as invariant INV-BLOB. Confirmed."
  confidence: high
  question_this_if: "N/A — invariant"
  references: []
  spec_section_updated: "2.1 — blobs table unchanged"

- id: D-019
  generation: gen1
  marker: "[DECIDE:gen1] Event emission"
  decision: "Synchronous (atomic with projection)"
  alternatives: ["Async event emission"]
  rationale: "INV-ATOMIC requires event + projection in same transaction. Async would violate this invariant."
  confidence: high
  question_this_if: "N/A — required by invariant"
  references: []
  spec_section_updated: "3.5 — transaction model unchanged"

# ═══ CHALLENGED ASSUMPTIONS ═══

- id: C-001
  generation: gen1
  type: challenged_assumption
  assumption: "Gen0 recommends composite PK (edge_id, valid_from) for edges for consistency with nodes (section 2.1, line 201-204)"
  challenge: "Edges represent relationships, not entities. Edge 'updates' are nearly always soft-delete + recreate. Full bitemporality adds EXCLUDE constraints and composite PK complexity with near-zero benefit. No validation test requires edge versioning."
  outcome: revised
  action: "Changed edges to simple PK (edge_id). Removed EXCLUDE constraint. Temporal fields (valid_from, valid_to) retained for relationship lifecycle but without versioning."
  spec_section_updated: "2.1 — edges table"

- id: C-002
  generation: gen1
  type: challenged_assumption
  assumption: "JSONB data field has optional schema enforcement — entities can be stored without matching type_node's label_schema (section 2.1 type_nodes note)"
  challenge: "Without mandatory validation, data quality depends entirely on agent discipline. This undermines the Epistemic Honesty principle — confirmed entities should have validated data."
  outcome: revised
  action: "When a type node has a label_schema, store_entity MUST validate data against it. Validation failure returns SCHEMA_VIOLATION error. Exception: capture_thought creates hypothesis entities with relaxed validation (LLM output may not match schema perfectly). Type nodes without label_schema remain schema-free."
  spec_section_updated: "3.2 — store_entity auto_side_effects, 3.5 — error model"

- id: C-003
  generation: gen1
  type: challenged_assumption
  assumption: "capture_thought is a single atomic tool — all-or-nothing for LLM extraction + entity creation + edge creation (section 3.2)"
  challenge: "Single tool means agent cannot review/correct LLM extraction before it's committed. A failure at any entity/edge creation step aborts everything, including valid extractions."
  outcome: deferred
  action: "Kept as single tool for gen1 (splitting would push tool count to 16, exceeding 5-15 range). But added extraction_result to the response so agents can inspect what was extracted. [DECIDE:gen2] Consider splitting into extract_thought (returns plan) + apply_extraction (commits)."
  spec_section_updated: "3.2 — capture_thought response includes full extraction details"

- id: C-004
  generation: gen1
  type: challenged_assumption
  assumption: "15 tools total (12 primary + 3 utility) at the upper limit of the 5-15 recommended range (section 3)"
  challenge: "query_at_time could be a parameter on find_entities. get_timeline overlaps with explore_graph over time. Merging could reduce tool count."
  outcome: upheld
  action: "Each tool has a distinct mental model: query_at_time answers 'what did THIS entity look like THEN?' (point-in-time state), find_entities answers 'which entities match THESE criteria?' (discovery). Merging would increase parameter complexity and reduce discoverability. 15 tools is at the limit but justified by distinct purposes."
  spec_section_updated: "N/A — no change, assumption upheld"

# ═══ GEN1.1 REMEDIATION ENTRIES ═══

- id: R-001
  generation: gen1.1
  type: remediation
  gap: "blob_stored missing from projection matrix (§3.4)"
  severity: low
  action: "Added blob_stored projection with full pseudocode: event INSERT + blob INSERT, with pre-tx object storage upload and orphan mitigation notes."
  spec_section_updated: "3.4 — projection matrix"

- id: R-002
  generation: gen1.1
  type: remediation
  gap: "Projection matrix pseudocode was skeleton-level prose, not implementation-ready"
  severity: high
  action: "Expanded all 10 intent_types to full SQL pseudocode with: exact column values, pre_validate steps, edge case handling, error conditions, and post_tx side effects. An implementer needs zero design decisions."
  spec_section_updated: "3.4 — projection matrix (complete rewrite)"

- id: R-003
  generation: gen1.1
  type: remediation
  gap: "capture_thought prompt_skeleton was too abstract for implementation"
  severity: high
  action: "Replaced skeleton with production-ready prompt template including: system/user message split, JSON output schema, 3 few-shot examples (Pettson domain), LLM call parameters (model, temperature, max_tokens, timeout), token budget analysis, and post-processing rules."
  spec_section_updated: "3.2 — capture_thought prompt_template"

- id: R-004
  generation: gen1.1
  type: remediation
  gap: "D-013 cross-tenant RLS had governance triggers but no technical attack scenarios"
  severity: medium
  action: "Added 3 concrete attack scenarios (A: grants bypass, B: excessive tenant_ids, C: SET LOCAL skip) with blast radius analysis, what RLS catches vs misses, and measurable trigger conditions. D-013 question_this_if updated from governance to technical triggers."
  spec_section_updated: "4.7 — D-013 attack scenarios"
  cross_reference: "D-013 in this file"

- id: R-005
  generation: gen1.1
  type: remediation
  gap: "Broken feedback loop — gen1-impl has not run, no runtime data available"
  severity: process
  action: "Added gen1.1 generation summary (§12.1) acknowledging the gap. Listed 4 decisions with [REQUIRED:gen1-impl] markers (D-008, D-009, D-016, D-013 Scenario C) that need runtime measurements. Stated gen2-spec SHOULD NOT change these without gen1-impl data."
  spec_section_updated: "12.1 — gen1.1 generation summary"

- id: R-006
  generation: gen1.1
  type: remediation
  gap: "A2A spec stability is an unresolved dependency for gen2"
  severity: medium
  action: "Added [RESEARCH:gen2] marker to federation.md §5.4 requiring A2A version stability check before any gen2 A2A work. Added depends_on link from DECIDE:gen2 AgentCard to the new RESEARCH marker."
  spec_section_updated: "5.4 — A2A integration roadmap gen2 block"
```

---
