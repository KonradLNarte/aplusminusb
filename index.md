# RESONANSIA MCP SERVER — GEN 2 SPEC (v0)

> **Lineage:** gen0-v5 → gen1-v1 → gen1.1 → gen2-v0
> **Base:** gen1-v1 (2026-03-03, Claude Opus 4.6)
> **Protocol version target:** MCP 2025-11-25, A2A v0.3
> **Tech profile:** Supabase (see section 10.2)


## Table of Contents

This spec is split into domain-aligned files. Section numbers are preserved from gen1.

| File | Section | Content |
|------|---------|---------|
| [data-model.md](data-model.md) | §2 | Tables, columns, indexes, RLS, metatype bootstrap, embeddings |
| [mcp-tools.md](mcp-tools.md) | §3 | All 15 MCP tools, projections, error handling, resources, prompts |
| [auth.md](auth.md) | §4 | OAuth 2.1, JWT, scopes, grants, RLS interaction, audit |
| [federation.md](federation.md) | §5 | Cross-tenant edges, grants, projections, A2A readiness |
| [constraints.md](constraints.md) | §6 | Invariants, serverless limits, cost model |
| [validation.md](validation.md) | §7 | Pettson scenario, seed data, T01-T14 test cases |
| [decisions.md](decisions.md) | §9 | Decision log (D-001 through D-019) |
| [tech-profile.md](tech-profile.md) | §10 | Supabase tech profile, runtime, dependencies |

---

## 0. GENERATIONAL PROTOCOL

This spec evolves through AI generations. Each generation MUST follow this protocol.

### 0.1 Generation contract

```yaml
gen0: Seed spec. Defines invariants, tool interface, validation scenario, open questions.
gen1-spec: Resolve all [DECIDE:gen1] and [RESEARCH:gen1] markers. Question ≥3 gen0 assumptions. Produce gen1 spec: a fully resolved, implementation-ready specification with no remaining ambiguity. No code.
gen1-impl: A SEPARATE coding agent receives the gen1 spec + tech profile. Produces deployable code + tests. Reports pass/fail against validation scenario. Feeds implementation learnings back as [FEEDBACK:gen1-impl] markers for gen2-spec.
gen2-spec: Inherit gen1 spec + gen1-impl feedback. Resolve [DECIDE:gen2] markers. Question ≥3 gen1-spec decisions. Produce gen2 spec.
gen2-impl: Separate coding agent implements gen2 spec.
genN: Pattern continues. Spec and implementation are always separate sessions with separate agents.
```

### 0.2 What every generation MUST do

1. **INHERIT** — Read the full spec. Read the DECISION LOG. Understand prior rationale.
2. **QUESTION** — Challenge ≥3 assumptions or decisions from prior generations. Document challenges in DECISION LOG with outcome: `upheld`, `revised`, or `deferred`.
3. **RESOLVE** — Every `[DECIDE]` assigned to your generation must be resolved with: decision, alternatives considered, rationale, confidence (high/mid/low), and a `question_this_if` trigger condition for future generations.
4. **RESEARCH** — Every `[RESEARCH]` assigned to your generation requires web research before deciding. Document findings.
5. **BUILD** — Produce the artifacts listed in your generation's DELIVERABLES section.
6. **VALIDATE** — Run the validation scenario. Report pass/fail per test case.
7. **SEPARATE** — Spec generations produce ONLY an updated spec file. Implementation cycles produce ONLY code artifacts. A spec generation must NEVER produce implementation code. An implementation cycle must NEVER modify the spec (it may propose changes as [FEEDBACK] markers for the next spec generation).
8. **FEEDBACK LOOP** — Implementation cycles MUST report back to the next spec generation:
   - Which spec sections were ambiguous (required interpretation during implementation)
   - Which spec sections were wrong (could not be implemented as specified)
   - Which spec sections were missing (implementation required decisions not covered by spec)
   - Performance measurements that inform future DECIDE markers
   - Format: [FEEDBACK:genN-impl] markers appended to the spec by the implementation agent.
9. **HAND OFF** — Update the DECISION LOG. Write a GENERATION SUMMARY (≤500 words): what you built, what you learned, what the next generation should watch for.

### 0.3 Decision log entry format

```yaml
- id: D-001
  generation: gen1
  marker: "[DECIDE:gen1] embedding dimensions"
  decision: "1536 dimensions with text-embedding-3-small"
  alternatives: ["768 with text-embedding-3-small", "1024 with custom model"]
  rationale: "Best recall for knowledge graph entities. Storage cost acceptable at <100K nodes."
  confidence: mid
  question_this_if: "Node count exceeds 500K or embedding latency exceeds 100ms p95"
  references: ["https://supabase.com/docs/guides/ai/choosing-compute-addon"]
  spec_section_updated: "2.1 — column type for event_id changed from UUID to UUIDv7"
```

### 0.4 Anti-drift rules

- **No scope creep**: If a feature is not in the SYSTEM DEFINITION or your GENERATION DELIVERABLES, do not build it. Propose it for a future generation instead.
- **No phantom requirements**: Do not invent requirements. If the spec does not mention it, ask whether it is needed before building it.
- **Spec stays in sync**: After resolving a DECIDE marker, update the spec text. Never leave resolved decisions only in the log.
- **Minimal viable first**: When a DECIDE has a simple option and a complex option, prefer the simple option unless the complex option is required by a validation test.
- **Spec and code never share a session**: A spec-refining agent must not have access to a code execution environment. A code-producing agent must not modify the spec. This separation prevents the spec from becoming a post-hoc description of what was built rather than a prescriptive definition of what should be built.

---

## 1. SYSTEM DEFINITION

### 1.0 One-sentence summary

A federated MCP server that exposes a bitemporal, event-sourced knowledge graph as AI-agent-accessible infrastructure with tenant isolation, semantic search, and temporal queries.

### 1.1 Architectural principles

These principles are **invariants** that no generation may violate:

1. **Event Primacy** — The `events` table is the single immutable source of truth. Fact tables (`nodes`, `edges`, `grants`) are projections of the event stream. Every row in a fact table MUST reference the `event_id` that created it. The same event stream + same projection logic = identical state.
2. **Graph-native Ontology** — The schema is part of the graph. Entity types, relationship types, and event types are first-class nodes (`type_nodes`), not a separate registry table. To understand what a "booking" is, you traverse its type node. Critical consequence: an agent can use the same tools to learn the schema as it uses to learn the data. `explore_graph(start_id=METATYPE_NODE)` returns all type nodes. `find_entities(entity_types=["type_node"])` searches type definitions by name or content. `get_schema` is a convenience wrapper — it MUST query the graph internally, not a separate registry. No generation may break this reflexive property.
3. **Epistemic Honesty** — Not all facts are equally certain. Every node carries an epistemic status (`hypothesis`, `asserted`, `confirmed`) reflecting the provenance of the information.
4. **Tenant Isolation** — `tenant_id` on every row. Row-Level Security enforces isolation. No data leaks between tenants, ever.
5. **Federation via Edges** — Cross-tenant data sharing happens through edges and capability grants, never through data copying.
6. **Bitemporality** — Every mutable entity tracks business time (`valid_from`/`valid_to`) and system time (`recorded_at`). Queries can ask "what was true then?" and "what did we know then?"
7. **Tech Agnosticism** — This spec defines WHAT, not HOW. Technology choices are bound via a separate Tech Profile (section 10) injected at generation start.
8. **Spec Primacy** — The specification is the source of truth for system design. Code is a derived artifact. If the spec and the code disagree, the spec is authoritative and the code must be corrected. No implementation cycle may introduce design decisions not present in the spec; if implementation reveals a gap, it must be reported as a [FEEDBACK] marker for the next spec generation, not silently resolved in code.
9. **A2A Readiness** — MCP defines how agents access tools and data. A2A (Agent2Agent Protocol) defines how agents discover and collaborate with each other. Resonansia sits at this intersection: it is an MCP server (agents use its tools) AND its tenants should be discoverable as A2A-compatible agents (agents find each other through Resonansia). No generation may design features that preclude future A2A compliance. See section 5.4 for details.

### 1.2 Unique value proposition

No existing system combines all of these:

| Property | Graphiti/Zep | Neo4j MCP | Open Brain | Mem0 | **This system** |
|---|---|---|---|---|---|
| Generalized knowledge graph | Partial (agent memory focus) | Yes | No (flat table) | Partial | **Yes** |
| Bitemporal data model | Yes (event_time + ingestion_time) | No | No | No | **Yes** |
| Event sourcing / immutable audit | No | No | No | No | **Yes** |
| Multi-tenancy (RLS) | group_id only | No | No | No | **Yes** |
| Cross-tenant federation | No | No | No | No | **Yes** |
| Schema-flexible + typed | Fixed schema | Cypher schema | Fixed | Fixed | **Yes (JSONB + type nodes)** |
| Graph-native ontology | No | Partial | No | No | **Yes** |
| Epistemic status tracking | No | No | No | No | **Yes** |
| Capability-based access (grants) | No | No | No | No | **Yes** |
| MCP server interface | Yes | Yes | Yes | No | **Yes** |
| A2A agent discoverability | No | No | No | No | **Yes (gen2+)** |

### 1.3 Competitive landscape (for research context)

| System | Repository / URL | Relevance |
|---|---|---|
| Graphiti (Zep) | https://github.com/getzep/graphiti | Temporal knowledge graph, MCP 1.0, multiple backends. Closest competitor. Study: entity deduplication, embedding strategy, tool design. |
| Graphiti MCP Docs | https://help.getzep.com/graphiti/getting-started/mcp-server | MCP tool design patterns, episode management. |
| Graphiti Paper | https://arxiv.org/abs/2501.13956 | Bitemporal knowledge graph architecture decisions. |
| Open Brain | https://github.com/benclawbot/open-brain | Proves Supabase Edge + pgvector + MCP pattern. Single-user, flat table. |
| Open Brain Guide | https://promptkit.natebjones.com/20260224_uq1_guide_main | Step-by-step deployment guide. Model for zero-coding-experience deployment docs. |
| AWS Multi-tenant MCP | https://github.com/aws-samples/sample-multi-tenant-saas-mcp-server | Reference architecture for OAuth 2.1 + multi-tenancy in MCP. |
| MCP Plexus | https://github.com/super-i-tech/mcp_plexus | Multi-tenant MCP framework (FastMCP). Auth + tenant routing, but no data model. |
| StrongDM Attractor | https://github.com/strongdm/attractor | Spec-driven coding agent methodology. NLSpec format, pipeline-as-graph, generational handoff. |
| @hono/mcp | https://github.com/honojs/middleware/tree/main/packages/mcp | Hono MCP middleware with auth router support. |
| Mem0 | https://github.com/mem0ai/mem0 | Graph-enhanced memory, managed service. No self-hosting, no federation. |
| MindBase | https://github.com/agiletec-inc/mindbase | PostgreSQL + pgvector + Ollama, local-first conversation knowledge management. |
| Anthropic KG Memory | https://github.com/modelcontextprotocol/servers | Reference MCP implementation. JSON file, no production use. |
| A2A Protocol | https://a2a-protocol.org/latest/specification/ | Agent-to-agent interoperability standard (v0.3). AgentCard discovery, task delegation, SSE streaming. Complements MCP — MCP for tools, A2A for agent collaboration. |
| A2A Security Guide | https://semgrep.dev/blog/2025/a-security-engineers-guide-to-the-a2a-protocol/ | Security analysis of A2A: AgentCard spoofing risks, token lifetime issues, OAuth considerations. |

---

## 8. GENERATION DELIVERABLES

### 8.1 Gen1-spec deliverables

Gen1-spec receives this spec + a TECH PROFILE (see section 10). Gen1-spec produces exclusively spec refinements — no code.

```yaml
deliverable_1_resolved_decisions:
  what: Every [DECIDE:gen1] marker resolved with full decision log entry
  includes:
    - Decision, alternatives considered, rationale, confidence, question_this_if
    - Web research findings for each [RESEARCH:gen1] marker
    - Updated spec text reflecting each resolved decision (not just log entries)
  done_when: Zero [DECIDE:gen1] or [RESEARCH:gen1] markers remain in spec.

deliverable_2_schema_precision:
  what: Section 2 (DATA MODEL) refined to implementation-ready precision
  includes:
    - Exact column types, index definitions, and constraint expressions for the chosen tech profile
    - Exact RLS policy SQL for every table (written in spec, not in a migration file)
    - Exact bootstrap migration sequence for metatype self-reference
    - Embedding dimension and model specified (from resolved DECIDE)
    - Any schema changes resulting from RESEARCH (e.g., SQL:2011 temporal support findings)
  done_when: A coding agent can produce a working migration by translating the spec SQL to the target platform without making any design decisions.

deliverable_3_tool_precision:
  what: Section 3 (MCP TOOL INTERFACE) refined to implementation-ready precision
  includes:
    - Exact input/output JSON schemas for every tool (using chosen schema_validation library syntax, e.g., Zod)
    - Exact error codes and error message templates for every failure mode per tool
    - Exact event payload schemas for every intent_type
    - Exact projection logic described in pseudocode: for each event intent_type, what rows are inserted/updated in which fact tables
    - capture_thought LLM prompt template (complete, not sketched)
    - Entity deduplication algorithm described step-by-step (from resolved DECIDE)
  done_when: A coding agent can implement each tool by translating pseudocode to the target language without making any design decisions.

deliverable_4_auth_precision:
  what: Section 4 (AUTH MODEL) refined to implementation-ready precision
  includes:
    - Exact JWT claim structure and validation rules
    - Exact scope-checking logic per tool (which scopes are required for each tool + operation)
    - Exact grants table consultation logic for cross-tenant operations (pseudocode)
    - Exact RLS policy interaction with application-level auth (from resolved DECIDE)
    - Exact audit event schema
  done_when: A coding agent can implement auth middleware by translating the spec to code without making any security decisions.

deliverable_5_validation_precision:
  what: Section 7 (VALIDATION SCENARIO) refined to executable precision
  includes:
    - Exact seed data: every tenant, type node, entity, edge, grant, and agent token as concrete JSON/SQL
    - Exact test assertions: for each T01-T14, the exact request payload and exact expected response shape (not prose descriptions)
    - Test execution order and dependencies made explicit
  done_when: A coding agent can produce executable test code by translating test definitions to the target test framework without inventing any test data.

deliverable_6_implementation_brief:
  what: A new section (section 8b) that specifies what gen1-impl must produce
  includes:
    - List of implementation artifacts (migration, server, tools, auth, embedding, capture, tests, deployment, cost estimate)
    - For each artifact: which spec sections it draws from, what "done" means, which validation tests verify it
    - Tech profile binding: which spec abstractions map to which concrete technology choices
  done_when: A coding agent receiving the gen1 spec + this brief can start work without asking clarifying questions.

deliverable_7_questioned_assumptions:
  what: ≥3 gen0 assumptions challenged, with outcomes documented in DECISION LOG
  includes:
    - Each challenge documented as: assumption, challenge rationale, outcome (upheld/revised/deferred)
  done_when: DECISION LOG contains ≥3 entries from gen1-spec.

deliverable_8_gen1_spec_file:
  what: The complete updated spec file (resonansia-mcp-server-gen1-spec.md)
  includes:
    - All resolved decisions integrated into spec text
    - All research findings integrated
    - New [DECIDE:gen2] and [RESEARCH:gen2] markers where appropriate
    - Updated GENERATION SUMMARY
  done_when: The file is self-contained, internally consistent, and has zero unresolved gen1 markers.
```

### 8.2 Gen1-spec success criteria

```yaml
minimum_viable:
  - All [DECIDE:gen1] markers resolved with decision log entries
  - All [RESEARCH:gen1] markers researched and findings documented
  - Schema section updated with resolved types and constraints
  - Tool section updated with exact input/output schemas
  - ≥3 assumptions questioned
  grade: "C — spec is more precise but still has ambiguity"

target:
  - All deliverables 1-8 complete
  - Zero ambiguity in schema, tool, and auth sections (a coding agent needs zero design decisions)
  - Validation scenario has exact seed data and exact assertions
  - Implementation brief is complete
  - DECISION LOG is populated
  grade: "A — spec is implementation-ready"

stretch:
  - Spec reviewed by a second AI agent (critique/revise loop) with findings documented
  - Alternative architectural options explored and documented for key decisions
  - Cost model researched and estimated (in spec, not as a separate document)
  grade: "A+ — spec has been adversarially tested"
```

### 8.3 Gen1 DECIDE markers — ALL RESOLVED

```yaml
# All markers resolved. See DECISION LOG (section 9) for full entries.
# Summary:
- "D-001: UUIDv7 for event_id" → YES, retain recorded_at
- "D-002: Edge versioning" → Option B (immutable, simple PK) — REVISED from gen0 recommendation
- "D-003: Grants soft-delete" → Include is_deleted
- "D-004: Optimistic concurrency" → Optional for gen1
- "D-005: Embedding model" → text-embedding-3-small, 1536 dimensions
- "D-006: Embed edges" → No for gen1
- "D-007: Embed events" → No for gen1
- "D-008: Bulk embedding" → Sequential with batching
- "D-009: Entity deduplication" → Hybrid (exact + embedding + LLM disambiguation)
- "D-010: Filter operators" → Equality + $in only
- "D-011: Token format" → Stateless JWT
- "D-012: Auth provider" → Supabase Auth + custom JWT signing
- "D-013: RLS pattern" → Application-level + RLS safety net (Option C)
- "D-014: Scope granularity" → Current syntax sufficient
- "D-015: Actor identity" → Actors as graph nodes (Option B)
- "D-016: capture_thought strategy" → Synchronous (Option A)
- "D-017: License" → Apache 2.0
- "D-018: Blob storage" → External (was gen0 invariant, confirmed)
- "D-019: Event emission" → Synchronous (atomic with projection per INV-ATOMIC)
```

### 8.4 Gen1 RESEARCH markers — ALL RESOLVED

```yaml
# All markers researched and findings integrated into spec. Summary:
- "PostgreSQL SQL:2011" → PG 18 lacks system-time versioning. Retain manual bitemporality. (section 2.5)
- "MCP SDK versions" → @modelcontextprotocol/sdk@1.27.1, @hono/mcp@0.2.3. Target spec 2025-11-25. (section 6.3)
- "Serverless limits" → Supabase: 2s CPU, 150s wall-clock, 300MB RAM, 20MB bundle. (section 6.1)
- "Graphiti dedup" → Three-tier: exact → entropy-gated MinHash/LSH → LLM fallback. Adapted for gen1. (section 3.2 capture_thought)
- "OAuth 2.1 in MCP" → 2025-11-25 spec: Resource Server only, RFC 9728, RFC 8707. DCR optional. (section 4.2)
- "A2A Protocol v0.3" → Complementary to MCP. AgentCard at .well-known/agent-card.json. (section 5.4)
- "A2A AgentCard auto-gen" → Feasible, multiple bridges exist. Auto-gen + manual augmentation. (section 5.4)
```

### 8.5 Implementation brief (section 8b — for gen1-impl)

This section specifies what a coding agent (gen1-impl) must produce. Gen1-impl receives this spec + the Supabase tech profile (section 10.2).

```yaml
# ═══ IMPLEMENTATION ARTIFACTS ═══

artifact_1_database_migration:
  what: "PostgreSQL migration script for Supabase"
  draws_from: [section 2.1, section 2.6]
  includes:
    - CREATE EXTENSION pgvector, btree_gist
    - UUIDv7 generation function (section 2.6.4)
    - All 7 tables with exact column types from section 2.1
    - All indexes from section 2.6.1
    - All RLS policies from section 2.6.2
    - Metatype bootstrap (section 2.6.3)
    - EXCLUDE constraint on nodes table
  done_when: "Migration runs idempotently on fresh Supabase project"
  verified_by: [T13]

artifact_2_seed_data:
  what: "Pettson scenario seed data"
  draws_from: [section 7.1-7.5, section 7.8]
  includes:
    - 4 tenants (holding + 3 subsidiaries)
    - Type nodes per tenant (section 7.2)
    - Actor nodes for 4 agents (section 7.5)
    - Sample entities (≥3 per tenant)
    - Cross-tenant edges (section 7.3)
    - Grants (section 7.4)
    - Agent tokens (JWT) for each role
  done_when: "All T01-T14 tests can run against seed data"
  verified_by: [T01-T14]

artifact_3_mcp_server:
  what: "Hono + @hono/mcp server on Supabase Edge Functions"
  draws_from: [section 3, section 6.3]
  includes:
    - HTTP Streaming transport endpoint at /mcp
    - All 15 tools registered with MCP SDK
    - MCP resources (section 3.6)
    - MCP prompts (section 3.7)
    - Protected Resource Metadata endpoint (RFC 9728)
  done_when: "MCP Inspector can connect and list all tools"
  verified_by: [T01-T14]

artifact_4_tool_implementations:
  what: "Implementation of all 15 tools"
  draws_from: [section 3.2, section 3.3, section 3.4, section 3.5]
  includes:
    - store_entity, find_entities, connect_entities, explore_graph, remove_entity
    - query_at_time, get_timeline
    - capture_thought (with LLM extraction and deduplication)
    - get_schema, get_stats
    - propose_event, verify_lineage
    - store_blob, get_blob, lookup_dict
    - All projection logic per section 3.4
    - All error handling per section 3.5
  done_when: "Each tool returns correct results for its validation test"
  verified_by: [T01-T14]

artifact_5_auth_middleware:
  what: "JWT validation + scope checking + grants lookup"
  draws_from: [section 4]
  includes:
    - JWT validation (signature, expiry, audience)
    - Scope extraction and matching per section 4.8
    - SET LOCAL app.tenant_ids for RLS (section 4.7 D-013)
    - Actor auto-creation (section 4.6)
    - Grants table consultation for cross-tenant operations
    - Audit logging (section 4.9)
  done_when: "T03 (permission denial) and T05 (partner read-only) pass"
  verified_by: [T03, T05, T04]

artifact_6_embedding_pipeline:
  what: "Async embedding generation via OpenAI API"
  draws_from: [section 2.4]
  includes:
    - text-embedding-3-small, 1536 dimensions
    - Async queue (in-memory for gen1, database-backed if needed)
    - Batch API calls (100 texts per call)
    - Exponential backoff with jitter
    - Bulk embedding for seed data
  done_when: "Seed data nodes have embeddings, T01 semantic search returns results"
  verified_by: [T01, T09, T11]

artifact_7_capture_thought:
  what: "LLM extraction pipeline for capture_thought"
  draws_from: [section 3.2 capture_thought, section 3.2 deduplication_strategy]
  includes:
    - Complete LLM prompt (section 3.2 prompt_skeleton)
    - Entity extraction and classification
    - Deduplication algorithm (3-tier: exact → embedding → LLM)
    - Entity and edge creation within atomic transaction
  done_when: "T09 and T11 pass (extraction + deduplication)"
  verified_by: [T09, T11]

artifact_8_test_suite:
  what: "Acceptance tests T01-T14"
  draws_from: [section 7.6, section 7.8]
  includes:
    - Test runner: Deno test (or Bun test per tech profile)
    - Each test as an independent function
    - Seed data setup/teardown
    - MCP client calls via SDK
    - Exact assertions from section 7.8
  done_when: "All 14 tests pass on a fresh Supabase project with seed data"
  verified_by: [self-verifying]

artifact_9_deployment:
  what: "Supabase Edge Function deployment config"
  draws_from: [section 10.2]
  includes:
    - supabase/functions/ directory structure
    - Environment variables for API keys
    - CORS configuration
    - Deployment script (supabase functions deploy)
  done_when: "Server accessible via deployed URL"
  verified_by: [manual smoke test]

artifact_10_cost_estimate:
  what: "Monthly cost estimate for Pettson-scale usage"
  draws_from: [section 6.4]
  includes:
    - Database hosting (Supabase Pro plan)
    - Embedding API cost per 1K/10K/100K nodes
    - LLM extraction cost per 1K capture_thought calls
    - Expected monthly cost for Pettson (4 tenants, ~1000 nodes, ~100 capture_thought/month)
  done_when: "Cost table with monthly estimates produced"
  verified_by: [N/A — informational]

# ═══ TECH PROFILE BINDING ═══

tech_profile: "Supabase (section 10.2)"
runtime: "Deno (Supabase Edge Functions native)"
framework: "Hono"
mcp_adapter: "@hono/mcp@^0.2.3"
mcp_sdk: "@modelcontextprotocol/sdk@^1.27.1"
database: "Supabase PostgreSQL + pgvector"
auth_provider: "Supabase Auth + custom JWT"
embedding_api: "text-embedding-3-small via OpenAI (or OpenRouter)"
extraction_llm: "gpt-4o-mini via OpenAI (or OpenRouter)"
object_storage: "Supabase Storage"
test_runner: "Deno test"
schema_validation: "Zod"

# ═══ IMPLEMENTATION ORDER ═══

recommended_order:
  1: "artifact_1 (migration) — tables + indexes + RLS"
  2: "artifact_2 (seed data) — Pettson scenario"
  3: "artifact_5 (auth middleware) — JWT + scopes + actor auto-creation"
  4: "artifact_3 (MCP server skeleton) — Hono + @hono/mcp + transport"
  5: "artifact_4 (tools) — implement in order: get_schema, store_entity, find_entities, connect_entities, explore_graph, remove_entity, query_at_time, get_timeline, get_stats, verify_lineage, propose_event, store_blob, get_blob, lookup_dict, capture_thought (last — most complex)"
  6: "artifact_6 (embedding pipeline) — async queue + bulk seeding"
  7: "artifact_7 (capture_thought LLM) — prompt + dedup"
  8: "artifact_8 (tests) — T01-T14 in numeric order"
  9: "artifact_9 (deployment) + artifact_10 (cost)"

# ═══ START CHECKLIST ═══

before_starting:
  - "Read this spec cover to cover (all 12+ sections)"
  - "Create Supabase project"
  - "Enable pgvector extension"
  - "Set up OpenAI API key (or OpenRouter)"
  - "Run migration (artifact_1)"
  - "Run seed data (artifact_2)"
  - "Verify metatype node exists: SELECT * FROM nodes WHERE node_id = '00000000-0000-7000-0000-000000000001'"
```

### 8.6 Gen2+ roadmap (for context only — gen1 does not build these)

Note: Each future generation follows the same spec/impl separation. gen2-spec resolves gen2 markers and produces gen2 spec. gen2-impl implements gen2 spec. And so on.

```yaml
gen2:
  - Consent protocol for cross-tenant edges (automated grant negotiation)
  - Token introspection endpoint
  - Rate limiting per agent
  - GDPR erasure (crypto-shred pattern)
  - Grant delegation (tenant A can allow tenant B to grant access to tenant C)
  - Performance optimization based on gen1 benchmarks
  - Migrate cross-tenant RLS to pure database-level enforcement
  - A2A AgentCard generation per tenant (/.well-known/agent-card.json)
  - A2A server endpoint — translate A2A tasks to MCP tool calls
  - Optimistic concurrency: make expected_version required on store_entity updates
  - Extended filter operators ($gte, $lte, $contains, $exists) on find_entities

gen3:
  - Federated query forwarding to remote Resonansia instances
  - Cryptographic event hardening (hash chains, signatures) for trustless federation
  - Horizontal scaling (read replicas)
  - Multi-region deployment
  - Real-time subscriptions (MCP sampling)
  - Epistemic branching (representing conflicting assertions from different sources)
    question_this_if: "Multiple federated instances need to reconcile conflicting facts"
  - A2A client capability: Resonansia agents discover and delegate tasks to external A2A agents
  - A2A federated mesh: Resonansia tenants across instances discover each other
  - AgentCard signing for trust between federated instances
  - Event replay: rebuild fact tables from event stream (disaster recovery, schema migration)
```

---


## 11. REFERENCES

Verified March 2026. **Research for latest versions before implementing.**

### 11.1 Core dependencies

| Dependency | URL | Why |
|---|---|---|
| MCP TypeScript SDK | https://github.com/modelcontextprotocol/typescript-sdk | MCP server implementation |
| MCP Specification | https://modelcontextprotocol.io/specification/2025-11-25 | Protocol reference (gen1 target: 2025-11-25) |
| MCP Auth (June 2025+) | https://modelcontextprotocol.io/specification/draft/basic/authorization | OAuth 2.1 for MCP — Resource Server model |
| Hono MCP Middleware | https://github.com/honojs/middleware/tree/main/packages/mcp | Hono + MCP integration (if using Hono) |
| pgvector | https://github.com/pgvector/pgvector | Vector similarity search in Postgres |
| A2A Protocol Spec | https://a2a-protocol.org/latest/specification/ | Agent-to-agent interoperability (v0.3). Required reading for gen1 RESEARCH. |
| A2A GitHub | https://github.com/a2aproject/A2A | Reference implementations, SDKs, AgentCard examples |

### 11.2 Reference implementations

| Project | URL | Learn from |
|---|---|---|
| Open Brain | https://github.com/benclawbot/open-brain | Supabase Edge + MCP + pgvector pattern |
| Open Brain Guide | https://promptkit.natebjones.com/20260224_uq1_guide_main | Step-by-step deployment guide |
| Graphiti (Zep) | https://github.com/getzep/graphiti | Temporal knowledge graph, MCP tools, entity deduplication |
| Graphiti MCP Docs | https://help.getzep.com/graphiti/getting-started/mcp-server | MCP tool design patterns |
| AWS Multi-tenant MCP | https://github.com/aws-samples/sample-multi-tenant-saas-mcp-server | OAuth 2.1 + multi-tenancy reference |
| MCP Plexus | https://github.com/super-i-tech/mcp_plexus | Multi-tenant MCP framework |
| StrongDM Attractor | https://github.com/strongdm/attractor | Spec-driven coding agent methodology |
| MindBase | https://github.com/agiletec-inc/mindbase | PostgreSQL + pgvector + Ollama, local-first |
| Mem0 | https://github.com/mem0ai/mem0 | Graph-enhanced memory (study API design) |

### 11.3 Best practices

| Topic | URL |
|---|---|
| MCP Tool Design | https://www.philschmid.de/mcp-best-practices |
| MCP Server Production | https://thenewstack.io/15-best-practices-for-building-mcp-servers-in-production/ |
| MCP Security | https://genai.owasp.org/resource/a-practical-guide-for-secure-mcp-server-development/ |
| Multi-tenant Auth in MCP | https://stackoverflow.blog/2026/01/21/is-that-allowed-authentication-and-authorization-in-model-context-protocol |
| MCP Auth Deep Dive | https://www.descope.com/blog/post/mcp-auth-spec |
| pgvector Benchmarks | https://supabase.com/docs/guides/ai/choosing-compute-addon |
| Bitemporal Postgres | https://wiki.postgresql.org/wiki/SQL2011Temporal |
| pg_bitemporal | https://github.com/scalegenius/pg_bitemporal |
| A2A Security Guide | https://semgrep.dev/blog/2025/a-security-engineers-guide-to-the-a2a-protocol/ |
| A2A + MCP Relationship | https://codelabs.developers.google.com/intro-a2a-purchasing-concierge |
| A2A Spring AI Integration | https://spring.io/blog/2026/01/29/spring-ai-agentic-patterns-a2a-integration/ |

### 11.4 Research papers

| Paper | URL | Relevance |
|---|---|---|
| Graphiti Architecture | https://arxiv.org/abs/2501.13956 | Bitemporal knowledge graph design, entity deduplication |

---


## 12. GENERATION SUMMARY (gen1)

```yaml
generation: gen1-v1
author: gen1-spec agent (Claude Opus 4.6)
date: 2026-03-03
lineage: gen0-v5 → gen1-v1

what_was_built: >
  Fully resolved, implementation-ready specification. All 19 DECIDE:gen1 markers
  resolved with full decision log entries. All 7 RESEARCH:gen1 markers researched
  with findings integrated. 4 gen0 assumptions challenged (C-001 through C-004):
  edge versioning revised (composite→simple PK), schema validation made mandatory
  when label_schema exists, capture_thought kept atomic but with extraction transparency,
  15-tool count upheld. Added: exact schema precision (indexes, RLS policies, metatype
  bootstrap, UUIDv7 function), exact seed data for Pettson scenario (4 tenants,
  16 type nodes, 9+ entities, 6 edges, 4 grants, 4 actors, 4 JWT token definitions),
  exact test assertions for T01-T14, auth precision (scope requirements per tool,
  audit event schema), deduplication algorithm (3-tier inspired by Graphiti),
  implementation brief with 10 artifacts and recommended build order.

what_was_learned:
  - PostgreSQL 18 adds WITHOUT OVERLAPS but still lacks system-time versioning — manual bitemporality remains correct
  - MCP SDK is at v1.27.1 (stable), v2 anticipated Q1 2026 but not yet released
  - @hono/mcp at v0.2.3 is viable for Supabase Edge Functions
  - Supabase Edge Functions: 2s CPU (hard limit all plans), 150s wall-clock, 300MB RAM
  - capture_thought fits within Supabase limits when implemented synchronously (LLM/DB are async I/O)
  - Graphiti's three-tier deduplication (exact → MinHash/LSH → LLM) is production-proven; simplified for gen1 scale
  - MCP auth spec (2025-11-25) now requires RFC 9728 (Protected Resource Metadata) and RFC 8707 (Resource Indicators)
  - A2A Protocol v0.3 is complementary to MCP; AgentCard auto-generation from MCP tools is feasible
  - Edge bitemporality (composite PK) is unnecessary at gen1 scale — immutable edges with soft-delete suffice
  - Schema enforcement on store_entity prevents garbage data without sacrificing flexibility for capture_thought hypotheses

what_next_gen_should_watch_for:
  - MCP SDK v2 may have breaking changes — check migration path before gen2
  - A2A Protocol may reach v1.0 before gen2 — verify AgentCard format stability
  - Edge immutability (D-002) may need revision if edge update patterns emerge
  - capture_thought synchronous execution (D-016) should be profiled — if CPU exceeds 1.5s, switch to async
  - Deduplication thresholds (0.85/0.95) are theoretical — tune based on gen1-impl experience
  - Supabase CPU limits may increase — monitor community discussions
  - gen1-impl should report [FEEDBACK:gen1-impl] on any spec ambiguity encountered
```

### 12.1 Generation summary (gen1.1 — remediation pass)

```yaml
generation: gen1.1
author: human review + Claude Opus 4.6
date: 2026-03-04
lineage: gen0-v5 → gen1-v1 → gen1.1
type: patch  # not a full generation — remediation pass before gen2 evolution

what_was_built: >
  Remediation pass closing 6 gaps identified by external review of the gen1-v1 spec.
  No new features or decisions — only precision improvements to make the spec
  truly implementation-ready. Changes: (1) blob_stored added to projection matrix
  (was missing despite being in events CHECK constraint), (2) projection matrix
  expanded from prose summaries to full SQL pseudocode with edge cases and error
  conditions for all 10 intent_types, (3) capture_thought prompt template expanded
  from skeleton to production-ready prompt with JSON schema, few-shot examples,
  LLM parameters, and post-processing rules, (4) cross-tenant RLS decision D-013
  augmented with 3 concrete attack scenarios and technical trigger conditions,
  (5) this gen1.1 summary added acknowledging the broken feedback loop,
  (6) A2A spec stability dependency given explicit [RESEARCH:gen2] marker.

feedback_loop_status: |
  BROKEN — gen1-impl has not run yet. The generational protocol (§0.2 step 8)
  requires implementation feedback before the next spec generation. Gen1.1 is a
  SPEC-ONLY remediation pass, not a substitute for gen1-impl feedback.

  The following decisions have question_this_if triggers that REQUIRE runtime data
  from gen1-impl. Gen2-spec SHOULD NOT change these decisions without gen1-impl
  measurement data:

  [REQUIRED:gen1-impl] D-008 (bulk embedding strategy):
    trigger: "Import volume exceeds 10K nodes or embedding queue exceeds 1000 pending"
    data_needed: actual embedding throughput, queue depth under load

  [REQUIRED:gen1-impl] D-009 (entity deduplication thresholds):
    trigger: "Duplicate entities exceed 5% of total OR false positives exceed 2%"
    data_needed: actual dedup precision/recall on Pettson scenario + real data

  [REQUIRED:gen1-impl] D-016 (capture_thought synchronous execution):
    trigger: "capture_thought p95 latency exceeds 10s or CPU exceeds 1.5s"
    data_needed: actual p50/p95 latency and CPU measurements on Supabase Edge

  [REQUIRED:gen1-impl] D-013 Scenario C (SET LOCAL skip):
    trigger: "SET LOCAL missed in any code path"
    data_needed: integration test results confirming SET LOCAL coverage

  Until gen1-impl provides these measurements, the decisions above are THEORETICAL.
  They are the best design choices given available information, but may need revision
  based on runtime reality.

what_was_learned:
  - The gen1-v1 spec was internally consistent (all DECIDE/RESEARCH markers resolved)
    but had precision gaps that would force an implementer to make design decisions
  - Projection matrix prose descriptions were the largest gap — an implementer would
    need to decide column values, edge case handling, and transaction boundaries
  - The capture_thought prompt skeleton was too abstract — critical details like output
    schema, few-shot examples, and LLM parameters were missing
  - RLS attack scenarios revealed that D-013's question_this_if was governance-oriented
    ("security audit requires it") rather than technically triggered — now has concrete
    scenarios with measurable trigger conditions
  - blob_stored was a simple omission but broke the completeness invariant (every
    intent_type in the CHECK constraint must have a projection entry)

what_next_gen_should_watch_for:
  - gen1-impl MUST report [FEEDBACK:gen1-impl] markers before gen2-spec begins
  - The 4 [REQUIRED:gen1-impl] decisions above are the highest-priority feedback items
  - A2A Protocol stability must be verified before any gen2 A2A work (see federation.md)
```

### 12.2 Previous generation summary (gen0)

```yaml
generation: gen0-v5
author: human + AI
date: 2026-03-03
what_was_built: >
  Seed spec defining invariant data model, tool interface, auth model,
  federation architecture, and validation scenario. v3 introduced four
  architectural advances over v2: event primacy (events are truth, facts
  are projections), graph-native ontology (type nodes replace labels table),
  epistemic status model (hypothesis/asserted/confirmed), and capability-based
  grants as a first-class table. v4 introduces spec/implementation separation:
  spec generations refine the spec, implementation cycles produce code from the
  refined spec. v5 addresses structural gaps identified in spec review:
  projection matrix (event→fact table mapping), transaction boundaries and
  error model, edge bitemporality fix (composite PK), optimistic concurrency,
  actor identity model, capture_thought prompt skeleton, filter operators,
  cursor pagination, grants soft-delete consistency, and A2A protocol awareness
  as a new architectural principle with gen2+ roadmap.

what_was_learned:
  - MCP best practices recommend 5-15 outcome-oriented tools, not CRUD wrappers
  - Multi-tenant MCP is not standardized; must be solved at application level
  - MCP auth spec has evolved significantly — June 2025 revision separates resource server from auth server
  - Serverless CPU limits (~2s) may constrain capture_thought; must measure before deciding sync vs async
  - pgvector HNSW performs well at <100K nodes on minimal infra
  - Entity deduplication is critical for capture_thought quality — Graphiti's approach is worth studying
  - Cryptographic event chains (hash linking, signatures) provide trustless audit but add significant
    complexity; deferred to gen3 when federation requires it
  - PostgreSQL has evolving SQL:2011 temporal support that may simplify bitemporal implementation
  - Spec-driven development requires strict separation between spec-refining and code-producing sessions. Mixing them causes the spec to become a post-hoc description rather than a prescriptive definition.
  - A2A Protocol (v0.3, Linux Foundation) has emerged as the dominant standard for agent-to-agent
    communication. It complements MCP (agent-to-tool) with agent-to-agent discovery and collaboration.
    Resonansia sits at the MCP/A2A intersection and must be A2A-aware from gen1 onward.
  - Edges table had broken bitemporality (simple PK prevented versioning) — fixed with composite PK
  - Event-sourcing requires explicit projection logic and transaction boundaries to prevent
    orphaned events and inconsistent state between event and fact layers
  - Optimistic concurrency is needed when multiple agents may update the same entity
  - Actor identity (created_by) needs a defined source — graph-native actors fit the architecture best

what_next_gen_should_watch_for:
  - Gen1-spec must resist the temptation to produce code. Its output is exclusively an updated spec file.
  - MCP auth spec may change — check latest before implementing
  - MCP adapter library API may have breaking changes — verify current version
  - Platform limits may have changed — verify before designing
  - capture_thought LLM extraction quality depends heavily on prompt engineering — invest time here
  - Entity deduplication accuracy will determine whether capture_thought is useful or frustrating
  - Cross-tenant RLS is the hardest auth problem — test thoroughly before building other features
  - Type node bootstrap (self-referential metatype) needs careful migration script — test idempotency
  - The grants table adds a DB lookup to every cross-tenant operation — measure performance impact early
  - A2A Protocol is evolving rapidly — verify v0.3 is still current before gen2 implementation
  - A2A AgentCard signing is supported but not enforced (v0.3) — security implications for federation
  - The projection matrix (section 3.6) is a skeleton — gen1-spec must expand every row to full pseudocode
  - Transaction boundary design (section 3.7) is critical — test rollback scenarios thoroughly
  - Actor-as-node pattern means every new JWT sub creates a node — measure overhead at scale
