# RESONANSIA MCP SERVER — GEN 2 SPEC

> Part of [Gen2 Spec](index.md)

## 3. MCP TOOL INTERFACE

### 3.1 Design principles

These principles override implementation convenience:

1. **Outcome over operation** — Tools represent agent goals, not database operations. An agent should accomplish its intent in 1-2 tool calls, not 5.
2. **Flat arguments** — Top-level primitives and constrained types. No nested dicts as params.
3. **5-15 tools total** — Curate ruthlessly. Each tool has one clear purpose.
4. **Every mutation emits an event** — Automatic. The caller does not need to emit events separately. The event is created FIRST, then fact tables are projected.
5. **Every mutation returns the created/modified entity** — No need for a follow-up read.
6. **Errors enable self-correction** — Error messages tell the agent what went wrong and how to fix it.
7. **Epistemic defaults** — `capture_thought` creates `hypothesis` entities. `store_entity` creates `asserted` entities. Agents can promote status explicitly.
8. **Reflexive ontology** — Type nodes are queryable through the same tools as data nodes. An agent that knows how to `explore_graph` already knows how to discover the schema. `get_schema` is a convenience shortcut, not a separate system. Gen1 must not introduce a parallel type registry that diverges from the graph.

References:
- https://www.philschmid.de/mcp-best-practices
- https://thenewstack.io/15-best-practices-for-building-mcp-servers-in-production/

### 3.2 Tools (12 primary)

```yaml
# ─── ENTITY MANAGEMENT ───

store_entity:
  description: "Create or update an entity in the knowledge graph. Returns the full entity with ID. Creates an event FIRST, then projects to nodes table."
  params:
    entity_type: string            # type node name, e.g. "lead", "booking"
    data: object                   # entity fields as flat JSON
    entity_id: string?             # if provided, updates existing (creates new version)
    expected_version: datetime?    # optimistic concurrency: valid_from of the version being updated. If provided and does not match current version, returns CONFLICT error. See INV-CONCURRENCY.
    valid_from: datetime?          # backdate business time (default: now)
    epistemic: string?             # 'asserted' (default) or 'confirmed'. Never 'hypothesis' — use capture_thought for that.
    tenant_id: string?             # required only if token spans multiple tenants
  returns:
    entity_id: string
    entity_type: string
    data: object
    version: int
    epistemic: string
    valid_from: datetime
    event_id: string               # the event that created this projection
    previous_version_id: string?
  auto_side_effects:
    - creates event (intent_type: "entity_created" or "entity_updated")
    - projects to nodes table with created_by_event reference
    - queues async embedding generation
    - validates data against type node schema if schema defined
    - if expected_version provided and does not match current version: returns CONFLICT error (no event created)
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)

find_entities:
  description: "Search entities by semantic similarity, structured filters, or both. Use this to find entities matching a natural language query or specific field values."
  params:
    query: string?                 # semantic search (uses embeddings). omit for structured-only search.
    entity_types: string[]?        # filter by type node name(s)
    filters: object?               # field-level filters using operator syntax (see filter_operators below)
    epistemic: string[]?           # filter by epistemic status, e.g. ["asserted", "confirmed"]
    tenant_id: string?
    limit: int?                    # default 10, max 100
    cursor: string?                # opaque cursor from previous response for pagination
    sort_by: string?               # "relevance" (default for semantic) | "created_at"
  filter_operators: |
    Filters support the following operators applied to JSONB data fields:
      { "field": "value" }              — equality (shorthand, default)
      { "field": { "$eq": "value" } }   — explicit equality
      { "field": { "$contains": "sub" } } — text contains (case-insensitive)
      { "field": { "$gte": 100 } }      — greater than or equal
      { "field": { "$lte": 200 } }      — less than or equal
      { "field": { "$in": ["a","b"] } } — value in set
      { "field": { "$exists": true } }  — field exists in JSONB
    [RESOLVED:gen1 D-010] Gen1 supports equality + $in only.
    [DECIDE:gen2] Add $gte, $lte, $contains, $exists operators.
  returns:
    results: [{ entity_id, entity_type, data, similarity?, epistemic, valid_from }]
    total_count: int
    next_cursor: string?           # null if no more results
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)

connect_entities:
  description: "Create a typed relationship between two entities. Works across tenant boundaries if token has access to both."
  params:
    edge_type: string              # type node name, e.g. "contacted_via", "includes"
    source_id: string
    target_id: string
    data: object?
  returns:
    edge_id: string
    edge_type: string
    source: { entity_id, entity_type, tenant_id }
    target: { entity_id, entity_type, tenant_id }
    is_cross_tenant: boolean
    event_id: string
  auto_side_effects:
    - creates event (intent_type: "edge_created") FIRST
    - projects to edges table with created_by_event
    - validates both endpoints exist and are accessible
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)

explore_graph:
  description: "Starting from an entity, traverse relationships to discover connected entities. Returns a subgraph centered on the start node."
  params:
    start_id: string
    edge_types: string[]?          # filter by relationship types. omit for all.
    direction: "outgoing" | "incoming" | "both"?  # default "both"
    depth: int?                    # max traversal depth. default 1, max 5.
    max_results: int?              # max total connected entities returned. default 50, max 500.
    include_data: boolean?         # include full entity data. default true.
    filters: object?               # filter connected entities by field values
  returns:
    center: { entity_id, entity_type, data }
    connections: [{
      edge_id, edge_type, direction,
      entity: { entity_id, entity_type, data, tenant_id, epistemic },
      depth: int
    }]
    # Entities the caller cannot access are omitted silently (not errors).
    # Grants table is consulted for cross-tenant traversal.
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)

remove_entity:
  description: "Soft-delete an entity. It remains in history and audit trail but is excluded from search and list results."
  params:
    entity_id: string
  returns:
    removed: true
    entity_type: string
    event_id: string
  auto_side_effects:
    - creates event (intent_type: "entity_removed") FIRST
    - sets is_deleted=true, valid_to=now() on node
    - connected edges become dangling (queryable for audit, excluded from explore_graph)
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)

# ─── TEMPORAL ───

query_at_time:
  description: "Retrieve an entity's state at a specific point in time. Supports 'what was true then?' (business time) and 'what did we know then?' (system time)."
  params:
    entity_id: string
    valid_at: datetime             # business time
    recorded_at: datetime?         # system time (default: now)
  returns:
    entity_id: string
    entity_type: string
    data: object
    epistemic: string
    valid_from: datetime
    valid_to: datetime
    recorded_at: datetime
    # Returns null with explanation if no version matches.
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)

get_timeline:
  description: "Get chronological history of an entity: all versions, related events, and changes over time."
  params:
    entity_id: string
    include_related_events: boolean?  # include events from connected entities. default false.
    time_range: { from: datetime?, to: datetime? }?
    limit: int?                    # default 50
    cursor: string?                # opaque cursor from previous response for pagination
  returns:
    entity: { entity_id, entity_type, current_data }
    timeline: [{
      timestamp: datetime,
      type: "version_change" | "event" | "edge_created" | "edge_removed" | "epistemic_change",
      data: object,
      created_by: string,
      event_id: string
    }]
    next_cursor: string?           # null if no more results
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)

# ─── CAPTURE ───

capture_thought:
  description: "Submit free-text input. System extracts structured data, classifies entity type, identifies mentioned entities, and creates nodes + edges automatically. All created entities get epistemic status 'hypothesis'."
  params:
    content: string                # free-text: note, message, observation
    source: string?                # origin: "slack", "email", "manual", "agent"
    tenant_id: string?
  returns:
    created_node: { entity_id, entity_type, data, epistemic: "hypothesis" }
    extracted_entities: [{ entity_id, entity_type, relationship, is_new, match_confidence? }]
    action_items: string[]?
  implementation_notes: |
    1. Fetch tenant's type nodes (available entity types + schemas)
    2. Call LLM to extract: type classification, structured fields, mentioned entities, action items
    3. For each mentioned entity: DEDUPLICATE before creating
       - Search existing entities by name similarity (embedding + fuzzy text match)
       - If match confidence > threshold: link to existing entity
       - If match confidence < threshold: create new entity as 'hypothesis'
       - Return match_confidence in response for transparency
    4. Create primary node via store_entity (with epistemic override to 'hypothesis')
    5. Create edges between primary node and mentioned/found entities
    6. Emit event: "thought_captured"
    LLM must be type-node-aware: prompt includes tenant's available types and their schemas.
  prompt_template: |
    [gen1.1: expanded from skeleton to production-ready prompt]

    # ── SYSTEM MESSAGE ──────────────────────────────────────────
    # Sent as the system/developer message. Variables in {braces} are interpolated at runtime.

    You are a structured entity extraction agent for the "{tenant_name}" knowledge graph.
    Your task: given free-text input, extract structured entities, relationships, and action items.

    RULES:
    - Output ONLY valid JSON. No markdown, no explanation, no commentary.
    - Classify entities using ONLY the types listed below. If no type fits, use "note".
    - Map text attributes to the type's schema fields. Omit fields you cannot confidently extract.
    - For mentioned entities, check the EXISTING ENTITIES list for potential matches.
      Set existing_match_id to the ID if you are confident it's the same entity (>80% sure).
      Set it to null if unsure or no match. Never guess an ID — when in doubt, set null.
    - Relationships MUST use edge types from the AVAILABLE EDGE TYPES list.
    - Action items are optional. Only include clear, actionable next steps implied by the text.
    - All extracted data reflects what the text STATES. Do not infer facts not present in the input.
    - If the input is too vague to extract any entity, return the FALLBACK structure (see below).

    AVAILABLE ENTITY TYPES:
    {entity_types_block}
    # Runtime interpolation produces:
    #   - name: "lead"
    #     description: "A potential customer or contact"
    #     schema:
    #       required: [name]
    #       properties:
    #         name: { type: string }
    #         phone: { type: string }
    #         email: { type: string }
    #         interest: { type: string }
    #         source: { type: string }
    #   - name: "booking"
    #     ...
    # Include ALL entity type nodes for the tenant. Max ~20 types at gen1 scale.

    AVAILABLE EDGE TYPES:
    {edge_types_block}
    # Runtime interpolation produces:
    #   - name: "contacted_via"
    #     description: "Communication channel between entities"
    #   - name: "includes"
    #     ...

    EXISTING ENTITIES (potential deduplication targets):
    {existing_entities_block}
    # Runtime interpolation: fetch top 20 entities by embedding similarity to input text.
    # Format per entity:
    #   - id: "uuid-here"
    #     type: "lead"
    #     name: "Johan Eriksson"
    #     summary: "Interested in cabin renovation, source: cabin fair"
    # If no existing entities, this section reads: "(none)"

    # ── USER MESSAGE ────────────────────────────────────────────
    # Sent as the user message.

    Extract structured data from the following text:

    ---
    {content}
    ---

    Source: {source}

    # ── EXPECTED OUTPUT SCHEMA ──────────────────────────────────
    # This block is appended to the system message as a reference.

    OUTPUT JSON SCHEMA:
    ```json
    {
      "primary_entity": {
        "type": "string — entity type name from AVAILABLE ENTITY TYPES",
        "data": {
          "...schema fields mapped from text..."
        }
      },
      "mentioned_entities": [
        {
          "type": "string — entity type name",
          "data": { "...fields..." },
          "existing_match_id": "string (UUID) | null",
          "match_confidence": "number 0.0-1.0 (required if existing_match_id is set)"
        }
      ],
      "relationships": [
        {
          "edge_type": "string — edge type name from AVAILABLE EDGE TYPES",
          "source": "\"primary\" | integer (index into mentioned_entities, 0-based)",
          "target": "\"primary\" | integer | \"existing:<UUID>\"",
          "data": {}
        }
      ],
      "action_items": ["string — actionable next step implied by text"]
    }
    ```

    FALLBACK (when input is too vague to classify):
    ```json
    {
      "primary_entity": { "type": "note", "data": { "content": "<original text>" } },
      "mentioned_entities": [],
      "relationships": [],
      "action_items": []
    }
    ```

    # ── FEW-SHOT EXAMPLES (Pettson domain) ──────────────────────

    EXAMPLE 1 — Lead capture from a fair:
    Input: "Met Johan Eriksson at the cabin fair. He's interested in renovating his cabin near Åre. Wants a quote for new roof + insulation. Phone: 070-123-4567."
    Output:
    ```json
    {
      "primary_entity": {
        "type": "lead",
        "data": {
          "name": "Johan Eriksson",
          "phone": "070-123-4567",
          "interest": "cabin renovation (roof + insulation)",
          "source": "cabin fair"
        }
      },
      "mentioned_entities": [
        {
          "type": "property",
          "data": { "name": "Johan's cabin", "location": "near Åre", "property_type": "cabin" },
          "existing_match_id": null,
          "match_confidence": 0.0
        }
      ],
      "relationships": [
        { "edge_type": "owns", "source": "primary", "target": 0, "data": {} },
        { "edge_type": "project_for", "source": "primary", "target": 0, "data": { "scope": "roof + insulation" } }
      ],
      "action_items": ["Send quote for roof + insulation to Johan Eriksson"]
    }
    ```

    EXAMPLE 2 — Matching existing entity:
    Input: "Johan Eriksson called back. He accepted the quote for the Åre cabin. Start date: March 15. He'll pay 50% upfront."
    (Assuming existing entity: { id: "abc-123", type: "lead", name: "Johan Eriksson" })
    Output:
    ```json
    {
      "primary_entity": {
        "type": "booking",
        "data": {
          "description": "Åre cabin renovation",
          "start_date": "2026-03-15",
          "payment_terms": "50% upfront",
          "status": "confirmed"
        }
      },
      "mentioned_entities": [
        {
          "type": "lead",
          "data": { "name": "Johan Eriksson" },
          "existing_match_id": "abc-123",
          "match_confidence": 0.95
        }
      ],
      "relationships": [
        { "edge_type": "booked_by", "source": "primary", "target": "existing:abc-123", "data": {} }
      ],
      "action_items": ["Schedule start date March 15", "Send invoice for 50% upfront payment"]
    }
    ```

    EXAMPLE 3 — Vague input (fallback):
    Input: "Remember to check the thing about the permit."
    Output:
    ```json
    {
      "primary_entity": { "type": "note", "data": { "content": "Remember to check the thing about the permit." } },
      "mentioned_entities": [],
      "relationships": [],
      "action_items": ["Check permit status"]
    }
    ```

    # ── LLM CALL PARAMETERS ─────────────────────────────────────
    model: "gpt-4o-mini"           # via OpenAI or OpenRouter
    max_tokens: 2000               # output budget — sufficient for ~10 entities
    temperature: 0.1               # low for deterministic extraction
    response_format: { type: "json_object" }  # enforce JSON output (OpenAI feature)
    timeout: 30_000                # 30s — fail fast, fits within Supabase 150s wall-clock

    # Input token budget: system message + examples ≈ 1500 tokens.
    # User content: up to 10,000 chars ≈ 2500 tokens.
    # Existing entities context: up to 20 entities ≈ 500 tokens.
    # Total input: ≈ 4500 tokens. Well within gpt-4o-mini 128K context.

    # ── POST-PROCESSING ─────────────────────────────────────────
    # After LLM returns JSON:
    # 1. Parse JSON. If invalid → retry once with "Your output was not valid JSON. Try again."
    #    If still invalid → E007 EXTRACTION_FAILED("LLM returned invalid JSON")
    # 2. Validate primary_entity.type against tenant's type nodes. If unknown → fall back to "note".
    # 3. For each mentioned_entity:
    #    a. If existing_match_id is set → verify it exists in tenant. If not → treat as new entity.
    #    b. Run 3-tier deduplication (§3.2 deduplication_strategy) regardless of LLM's match.
    #       LLM's existing_match_id is a HINT, not authoritative. Dedup algorithm has final say.
    # 4. Validate all edge_types against tenant's edge type nodes. Drop edges with unknown types.
    # 5. Build execution plan → pass to thought_captured projection (§3.4).
  deduplication_strategy: |
    [RESOLVED:gen1 D-009] Entity deduplication: Hybrid embedding + LLM disambiguation.
    Inspired by Graphiti's three-tier approach but simplified for gen1 scale (<10K entities/tenant).

    DEDUPLICATION ALGORITHM (pseudocode):
    ```
    function deduplicate(extracted_entity, tenant_id):
      name = normalize(extracted_entity.name)  // lowercase, trim, collapse whitespace

      // TIER 1: Exact match (O(1))
      exact = query_nodes(tenant_id, type=extracted_entity.type, data->>'name' ILIKE name, active_only=true)
      if exact.length == 1:
        return { match: exact[0], confidence: 1.0, method: "exact" }

      // TIER 2: Embedding similarity
      query_embedding = generate_embedding(name + " " + JSON.stringify(extracted_entity.data))
      candidates = vector_search(tenant_id, query_embedding, types=[extracted_entity.type], limit=5)
      // Filter by similarity threshold
      strong_matches = candidates.filter(c => c.similarity > 0.95)
      if strong_matches.length == 1:
        return { match: strong_matches[0], confidence: strong_matches[0].similarity, method: "embedding" }

      // TIER 3: LLM disambiguation (only for ambiguous zone 0.85-0.95)
      uncertain_matches = candidates.filter(c => c.similarity > 0.85 && c.similarity <= 0.95)
      if uncertain_matches.length > 0:
        llm_result = ask_llm_to_disambiguate(extracted_entity, uncertain_matches)
        if llm_result.is_match:
          return { match: llm_result.matched_entity, confidence: llm_result.confidence, method: "llm" }

      // No match found — create new entity
      return { match: null, confidence: 0.0, method: "none" }
    ```

    THRESHOLDS:
      - > 0.95 cosine similarity: auto-link (high confidence)
      - 0.85 - 0.95: LLM disambiguation (uncertain zone)
      - < 0.85: create new entity
    question_this_if: "Duplicate entities exceed 5% of total OR false positives exceed 2%"
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)

# ─── DISCOVERY ───

get_schema:
  description: "Discover what entity types, relationship types, and event types exist in a tenant. Returns type nodes with their schemas and usage counts. Use this first to understand the data model before querying. Note: this is a convenience tool — the same information is available via explore_graph(start_id=METATYPE_NODE) or find_entities(entity_types=['type_node'])."
  params:
    tenant_id: string?
  returns:
    entity_types: [{ name, schema, node_count, example_fields, type_node_id }]
    edge_types: [{ name, schema, edge_count, type_node_id }]
    event_types: [{ name, event_count, last_occurred_at }]
  implementation_constraint: |
    get_schema MUST query the nodes table (where type_node_id = metatype), not a separate
    registry or hardcoded view. This preserves the reflexive property: type nodes are nodes,
    and any tool that works on nodes also works on type nodes. If a type node is created via
    store_entity, it must immediately appear in get_schema results without additional sync.
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)

get_stats:
  description: "Dashboard-level statistics for a tenant: totals, recent activity, data freshness."
  params:
    tenant_id: string?
  returns:
    totals: { nodes, edges, events, grants, blobs }
    by_type: { [entity_type]: count }
    by_epistemic: { hypothesis: count, asserted: count, confirmed: count }
    recent_activity: [{ event_type, count_last_7d, last_occurred_at }]
    data_freshness: { newest_node, newest_event, oldest_unembedded_node }
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)

# ─── ADVANCED / SYSTEM ───

propose_event:
  description: "Submit a raw event directly. For system-to-system integration and advanced use cases. Handles validation, event creation, and projection to fact tables. Most agents should use store_entity/connect_entities/remove_entity instead."
  params:
    stream_id: string              # the entity this event belongs to
    intent_type: string            # must match a type node of kind 'event_type'
    payload: object                # event-specific data
    occurred_at: datetime?         # business time (default: now)
    tenant_id: string?
  returns:
    event_id: string
    stream_id: string
    projected_changes: object      # summary of what changed in fact tables
  auto_side_effects:
    - validates intent_type against type nodes
    - validates payload against intent_type schema if defined
    - creates event row
    - runs projection logic to update fact tables
    - returns projected changes
  error_codes:
    INVALID_INTENT: "intent_type does not match any type node of kind 'event_type' in tenant."
    SCHEMA_VIOLATION: "payload does not match schema for intent_type."
    UNAUTHORIZED: "Actor lacks required scope for this operation."
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)

verify_lineage:
  description: "Verify event lineage integrity for an entity. Checks that all fact rows reference existing events and that the event stream is consistent. Returns integrity report."
  params:
    entity_id: string
  returns:
    entity_id: string
    event_count: int
    is_valid: boolean
    fact_rows_checked: int         # total nodes/edges/grants rows checked
    orphaned_facts: [{ table, row_id, missing_event_id }]?  # facts referencing non-existent events
    timeline_gaps: [{ expected_after_event_id, actual_next_event_id }]?
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)
```

### 3.3 Utility tools (3)

```yaml
store_blob:
  params: { data_base64: string, content_type: string, related_entity_id: string? }
  returns: { blob_id, content_type, size_bytes, storage_ref }
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)

get_blob:
  params: { blob_id: string }
  returns: { blob_id, content_type, data_base64 }
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)

lookup_dict:
  params: { dict_type: string, key: string?, valid_at: datetime? }
  returns: { entries: [{ key, value, valid_from, valid_to }] }
  gen1_spec_delivered:  # SATISFIED by sections 3.3a (Zod schemas), 3.4 (projection), 3.5 (errors)
    - exact JSON Schema for params (in Zod notation matching tech profile)
    - exact JSON Schema for return value
    - exact error codes with message templates
    - exact event payload schema for each emitted event
    - pseudocode for the projection logic (event → fact table mutations)
```

**Total: 15 tools** (12 primary + 3 utility). Within the recommended 5-15 range.

### 3.3a Tool schemas (gen1 — Zod notation)

Gen1-impl MUST implement these exact schemas. All types use Zod syntax.

```typescript
// ─── store_entity ───
const StoreEntityParams = z.object({
  entity_type: z.string().min(1).describe("Type node name, e.g. 'lead', 'booking'"),
  data: z.record(z.unknown()).describe("Entity fields as flat JSON"),
  entity_id: z.string().uuid().optional().describe("If provided, updates existing entity"),
  expected_version: z.string().datetime().optional().describe("Optimistic concurrency: valid_from of version being updated"),
  valid_from: z.string().datetime().optional().describe("Backdate business time (default: now)"),
  epistemic: z.enum(["asserted", "confirmed"]).optional().default("asserted"),
  tenant_id: z.string().uuid().optional().describe("Required if token spans multiple tenants"),
});

const StoreEntityResult = z.object({
  entity_id: z.string().uuid(),
  entity_type: z.string(),
  data: z.record(z.unknown()),
  version: z.number().int().min(1),  // monotonic version = COUNT(*) of node rows with this node_id
  epistemic: z.enum(["hypothesis", "asserted", "confirmed"]),
  valid_from: z.string().datetime(),
  event_id: z.string().uuid(),
  previous_version_id: z.string().uuid().nullable(),  // valid_from of previous version, null if version=1
});

// ─── find_entities ───
const FindEntitiesParams = z.object({
  query: z.string().optional().describe("Semantic search query (uses embeddings)"),
  entity_types: z.array(z.string()).optional().describe("Filter by type node name(s)"),
  filters: z.record(z.union([
    z.string(), z.number(), z.boolean(),                    // equality shorthand
    z.object({ "$eq": z.unknown() }),                        // explicit equality
    z.object({ "$in": z.array(z.unknown()) }),               // value in set
  ])).optional().describe("Structured filters on data fields"),
  epistemic: z.array(z.enum(["hypothesis", "asserted", "confirmed"])).optional(),
  tenant_id: z.string().uuid().optional(),
  limit: z.number().int().min(1).max(100).optional().default(10),
  cursor: z.string().optional().describe("Opaque cursor from previous response"),
  sort_by: z.enum(["relevance", "created_at"]).optional().default("relevance"),
});

const FindEntitiesResult = z.object({
  results: z.array(z.object({
    entity_id: z.string().uuid(),
    entity_type: z.string(),
    data: z.record(z.unknown()),
    similarity: z.number().optional(),
    epistemic: z.string(),
    valid_from: z.string().datetime(),
  })),
  total_count: z.number().int(),
  next_cursor: z.string().nullable(),
});

// ─── connect_entities ───
const ConnectEntitiesParams = z.object({
  edge_type: z.string().min(1),
  source_id: z.string().uuid(),
  target_id: z.string().uuid(),
  data: z.record(z.unknown()).optional().default({}),
});

const ConnectEntitiesResult = z.object({
  edge_id: z.string().uuid(),
  edge_type: z.string(),
  source: z.object({ entity_id: z.string(), entity_type: z.string(), tenant_id: z.string() }),
  target: z.object({ entity_id: z.string(), entity_type: z.string(), tenant_id: z.string() }),
  is_cross_tenant: z.boolean(),
  event_id: z.string().uuid(),
});

// ─── explore_graph ───
const ExploreGraphParams = z.object({
  start_id: z.string().uuid(),
  edge_types: z.array(z.string()).optional(),
  direction: z.enum(["outgoing", "incoming", "both"]).optional().default("both"),
  depth: z.number().int().min(1).max(5).optional().default(1),
  max_results: z.number().int().min(1).max(500).optional().default(50),
  include_data: z.boolean().optional().default(true),
  filters: z.record(z.unknown()).optional(),
});

const ExploreGraphResult = z.object({
  center: z.object({ entity_id: z.string(), entity_type: z.string(), data: z.record(z.unknown()) }),
  connections: z.array(z.object({
    edge_id: z.string().uuid(),
    edge_type: z.string(),
    direction: z.enum(["outgoing", "incoming"]),
    entity: z.object({
      entity_id: z.string(), entity_type: z.string(),
      data: z.record(z.unknown()), tenant_id: z.string(), epistemic: z.string(),
    }),
    depth: z.number().int(),
  })),
});

// ─── capture_thought ───
const CaptureThoughtParams = z.object({
  content: z.string().min(1).max(10000).describe("Free-text input"),
  source: z.enum(["slack", "email", "manual", "agent", "sms", "whatsapp"]).optional().default("manual"),
  tenant_id: z.string().uuid().optional(),
});

const CaptureThoughtResult = z.object({
  created_node: z.object({
    entity_id: z.string().uuid(),
    entity_type: z.string(),
    data: z.record(z.unknown()),
    epistemic: z.literal("hypothesis"),
  }),
  extracted_entities: z.array(z.object({
    entity_id: z.string().uuid(),
    entity_type: z.string(),
    relationship: z.string(),
    is_new: z.boolean(),
    match_confidence: z.number().optional(),
  })),
  action_items: z.array(z.string()).optional(),
});

// ─── query_at_time ───
const QueryAtTimeParams = z.object({
  entity_id: z.string().uuid(),
  valid_at: z.string().datetime(),
  recorded_at: z.string().datetime().optional(),
});

const QueryAtTimeResult = z.object({
  entity_id: z.string().uuid(),
  entity_type: z.string(),
  data: z.record(z.unknown()),
  epistemic: z.string(),
  valid_from: z.string().datetime(),
  valid_to: z.string().datetime(),
  recorded_at: z.string().datetime(),
}).nullable();

// ─── get_timeline ───
// Note: time_from/time_to are flat params (section 3.1 principle #2: flat arguments)
const GetTimelineParams = z.object({
  entity_id: z.string().uuid(),
  include_related_events: z.boolean().optional().default(false),
  time_from: z.string().datetime().optional().describe("Start of time range filter"),
  time_to: z.string().datetime().optional().describe("End of time range filter"),
  limit: z.number().int().min(1).max(500).optional().default(50),
  cursor: z.string().optional(),
});

// ─── remove_entity ───
const RemoveEntityParams = z.object({ entity_id: z.string().uuid() });
const RemoveEntityResult = z.object({ removed: z.literal(true), entity_type: z.string(), event_id: z.string().uuid() });

// ─── get_schema ───
const GetSchemaParams = z.object({ tenant_id: z.string().uuid().optional() });

// ─── get_stats ───
const GetStatsParams = z.object({ tenant_id: z.string().uuid().optional() });

// ─── propose_event ───
const ProposeEventParams = z.object({
  stream_id: z.string().uuid(),
  intent_type: z.string().min(1),
  payload: z.record(z.unknown()),
  occurred_at: z.string().datetime().optional(),
  tenant_id: z.string().uuid().optional(),
});

// ─── verify_lineage ───
const VerifyLineageParams = z.object({ entity_id: z.string().uuid() });

// ─── store_blob ───
const StoreBlobParams = z.object({
  data_base64: z.string().min(1),
  content_type: z.string().min(1),
  related_entity_id: z.string().uuid().optional(),
});

// ─── get_blob ───
const GetBlobParams = z.object({ blob_id: z.string().uuid() });

// ─── lookup_dict ───
const LookupDictParams = z.object({
  dict_type: z.string().min(1),
  key: z.string().optional(),
  valid_at: z.string().datetime().optional(),
});
const LookupDictResult = z.object({
  entries: z.array(z.object({
    key: z.string(), value: z.record(z.unknown()),
    valid_from: z.string().datetime(), valid_to: z.string().datetime(),
  })),
});

// ─── get_schema result ───
const GetSchemaResult = z.object({
  entity_types: z.array(z.object({
    name: z.string(), schema: z.record(z.unknown()).nullable(),
    node_count: z.number().int(), example_fields: z.array(z.string()),
    type_node_id: z.string().uuid(),
  })),
  edge_types: z.array(z.object({
    name: z.string(), schema: z.record(z.unknown()).nullable(),
    edge_count: z.number().int(), type_node_id: z.string().uuid(),
  })),
  event_types: z.array(z.object({
    name: z.string(), event_count: z.number().int(),
    last_occurred_at: z.string().datetime().nullable(),
  })),
});

// ─── get_stats result ───
const GetStatsResult = z.object({
  totals: z.object({
    nodes: z.number().int(), edges: z.number().int(), events: z.number().int(),
    grants: z.number().int(), blobs: z.number().int(),
  }),
  by_type: z.record(z.number().int()),
  by_epistemic: z.object({
    hypothesis: z.number().int(), asserted: z.number().int(), confirmed: z.number().int(),
  }),
  recent_activity: z.array(z.object({
    event_type: z.string(), count_last_7d: z.number().int(),
    last_occurred_at: z.string().datetime().nullable(),
  })),
  data_freshness: z.object({
    newest_node: z.string().datetime().nullable(),
    newest_event: z.string().datetime().nullable(),
    oldest_unembedded_node: z.string().datetime().nullable(),
  }),
});

// ─── get_timeline result ───
const GetTimelineResult = z.object({
  entity: z.object({
    entity_id: z.string().uuid(), entity_type: z.string(),
    current_data: z.record(z.unknown()),
  }),
  timeline: z.array(z.object({
    timestamp: z.string().datetime(),
    type: z.enum(["version_change", "event", "edge_created", "edge_removed", "epistemic_change"]),
    data: z.record(z.unknown()), created_by: z.string().uuid(),
    event_id: z.string().uuid(),
  })),
  next_cursor: z.string().nullable(),
});

// ─── propose_event result ───
const ProposeEventResult = z.object({
  event_id: z.string().uuid(), stream_id: z.string().uuid(),
  projected_changes: z.record(z.unknown()),
});

// ─── verify_lineage result ───
const VerifyLineageResult = z.object({
  entity_id: z.string().uuid(), event_count: z.number().int(),
  is_valid: z.boolean(), fact_rows_checked: z.number().int(),
  orphaned_facts: z.array(z.object({
    table: z.string(), row_id: z.string(), missing_event_id: z.string(),
  })),
  timeline_gaps: z.array(z.object({
    expected_after_event_id: z.string(), actual_next_event_id: z.string(),
  })),
});

// ─── store_blob result ───
const StoreBlobResult = z.object({
  blob_id: z.string().uuid(), content_type: z.string(),
  size_bytes: z.number().int(), storage_ref: z.string(),
});

// ─── get_blob result ───
const GetBlobResult = z.object({
  blob_id: z.string().uuid(), content_type: z.string(),
  data_base64: z.string(),
});
```

### 3.4 Projection matrix (event → fact table mutations)

This matrix defines how each event `intent_type` maps to mutations on fact tables. It is the skeleton of the event-sourcing engine. Gen1-spec MUST expand each row into full pseudocode; gen1-impl translates pseudocode to executable code.

```
# ══════════════════════════════════════════════════════════════
# PROJECTION MATRIX — gen1.1 expanded pseudocode
# ══════════════════════════════════════════════════════════════
#
# Format: intent_type → full pseudocode for event + fact table mutations.
# All mutations happen within a single database transaction (INV-ATOMIC, §3.5).
# Variables prefixed with $ are runtime values. $now = clock_timestamp().
# $actor = resolved actor node_id (§4.6). $tenant = resolved tenant_id.
#
# CONVENTION: every projection function returns the created/modified rows
# so the tool handler can build the response without a follow-up query.
# ──────────────────────────────────────────────────────────────

# ─── entity_created ───────────────────────────────────────────

entity_created:
  trigger: store_entity (new entity — entity_id NOT provided)
  pre_validate:
    - Resolve type_node_id:
        SELECT node_id, data->'label_schema' AS schema
        FROM nodes
        WHERE (tenant_id = $tenant OR tenant_id = '00000000-0000-7000-0000-000000000000')
          AND type_node_id = '00000000-0000-7000-0000-000000000001'  -- metatype
          AND data->>'name' = $entity_type
          AND is_deleted = false AND valid_to = 'infinity'
        LIMIT 1;
      If not found → E001 VALIDATION_ERROR("Invalid entity_type: '$entity_type'")
    - If type_node has label_schema → validate $data against it (JSON Schema).
      If invalid → E006 SCHEMA_VIOLATION
    - Check scope: has_scope(token, "tenant:$tenant:write") OR
      has_scope(token, "tenant:$tenant:nodes:$entity_type:write")
      If denied → E003 AUTH_DENIED
  pseudocode: |
    $node_id   = uuid_generate_v7()      -- new stable identity
    $event_id  = uuid_generate_v7()
    $valid_from = $params.valid_from ?? $now
    $epistemic  = $params.epistemic  ?? 'asserted'

    BEGIN;

    INSERT INTO events (event_id, tenant_id, intent_type, payload, stream_id,
                        node_ids, occurred_at, recorded_at, created_by)
    VALUES ($event_id, $tenant, 'entity_created',
            jsonb_build_object(
              'entity_type', $entity_type,
              'type_node_id', $type_node_id::text,
              'data', $data,
              'epistemic', $epistemic
            ),
            $node_id,          -- stream_id = node_id (first event in stream)
            ARRAY[$node_id],
            $valid_from, $now, $actor);

    INSERT INTO nodes (node_id, tenant_id, type_node_id, data, embedding,
                       epistemic, valid_from, valid_to,
                       recorded_at, created_by, created_by_event, is_deleted)
    VALUES ($node_id, $tenant, $type_node_id, $data, NULL,
            $epistemic, $valid_from, 'infinity',
            $now, $actor, $event_id, false);

    COMMIT;
  post_tx:
    - Queue async embedding: { node_id: $node_id, text: "$entity_type: " + JSON.stringify($data) }
  edge_cases:
    - valid_from in the past: allowed (backdating). EXCLUDE constraint prevents overlap
      with any existing version of the same node_id (impossible for new entity, but
      defensive against UUID collision — astronomically unlikely with UUIDv7).
  errors:
    - E001 if entity_type not found
    - E006 if data fails schema validation
    - E003 if scope check fails

# ─── entity_updated ───────────────────────────────────────────

entity_updated:
  trigger: store_entity (existing entity — entity_id IS provided, epistemic unchanged or not provided)
  pre_validate:
    - Fetch current version:
        SELECT node_id, tenant_id, type_node_id, data, epistemic, valid_from
        FROM nodes
        WHERE node_id = $entity_id AND is_deleted = false AND valid_to = 'infinity'
        LIMIT 1;
      If not found → E002 NOT_FOUND
    - If expected_version provided:
        If current.valid_from != $expected_version → E004 CONFLICT(expected=$expected_version, actual=current.valid_from)
    - If current.valid_to != 'infinity':
        The entity already has a closed version with no open successor.
        This means a concurrent update closed it. → E004 CONFLICT
    - Resolve type_node_id (may change if entity_type param differs from current type):
        If $entity_type provided and != current type name → re-resolve type_node_id.
        If not provided → retain current type_node_id.
    - If type_node has label_schema → validate $data against it → E006 on failure
    - Check scope: write on $tenant
  pseudocode: |
    $event_id   = uuid_generate_v7()
    $valid_from = $now                    -- new version starts now
    $epistemic  = $params.epistemic ?? $current.epistemic  -- preserve if not changed
    $type_node_id = $resolved_type_node_id ?? $current.type_node_id

    -- Detect epistemic change: if $epistemic != $current.epistemic, emit
    -- epistemic_change instead (see below). This check happens BEFORE BEGIN.
    IF $epistemic != $current.epistemic:
      → delegate to epistemic_change projection (below)

    BEGIN;

    INSERT INTO events (event_id, tenant_id, intent_type, payload, stream_id,
                        node_ids, occurred_at, recorded_at, created_by)
    VALUES ($event_id, $tenant, 'entity_updated',
            jsonb_build_object(
              'entity_id', $entity_id::text,
              'old_data', $current.data,       -- full snapshot for audit
              'new_data', $data,
              'old_epistemic', $current.epistemic,
              'new_epistemic', $epistemic,
              'expected_version', $params.expected_version
            ),
            $entity_id,        -- stream_id = node_id
            ARRAY[$entity_id],
            $valid_from, $now, $actor);

    -- Close current version
    UPDATE nodes
    SET    valid_to = $valid_from          -- close at exact moment new version opens
    WHERE  node_id = $entity_id
      AND  valid_to = 'infinity'
      AND  is_deleted = false;
    -- If UPDATE affected 0 rows → concurrent modification. ROLLBACK. Return E004.

    -- Open new version
    INSERT INTO nodes (node_id, tenant_id, type_node_id, data, embedding,
                       epistemic, valid_from, valid_to,
                       recorded_at, created_by, created_by_event, is_deleted)
    VALUES ($entity_id, $tenant, $type_node_id, $data, NULL,
            $epistemic, $valid_from, 'infinity',
            $now, $actor, $event_id, false);

    COMMIT;
  post_tx:
    - Queue async embedding for new version
  edge_cases:
    - expected_version omitted: last-write-wins (D-004). No conflict check.
    - Entity was soft-deleted (is_deleted=true): returns E002 NOT_FOUND.
      To "un-delete" → create a new entity with same type (gets a new node_id).
    - Data is identical to current: still creates new version (for audit trail).
      Optimization: gen1-impl MAY skip if data + epistemic are identical, but
      this is an implementation choice, not a spec requirement.
  errors:
    - E002 if entity not found or already deleted
    - E004 if expected_version mismatch or concurrent close
    - E006 if data fails schema validation

# ─── entity_removed ───────────────────────────────────────────

entity_removed:
  trigger: remove_entity
  pre_validate:
    - Fetch current version:
        SELECT node_id, tenant_id, type_node_id, data, epistemic
        FROM nodes
        WHERE node_id = $entity_id AND is_deleted = false AND valid_to = 'infinity'
        LIMIT 1;
      If not found → E002 NOT_FOUND
    - Check scope: write on $tenant
  pseudocode: |
    $event_id = uuid_generate_v7()

    BEGIN;

    INSERT INTO events (event_id, tenant_id, intent_type, payload, stream_id,
                        node_ids, occurred_at, recorded_at, created_by)
    VALUES ($event_id, $tenant, 'entity_removed',
            jsonb_build_object(
              'entity_id', $entity_id::text,
              'entity_type', $current.type_name,
              'final_data', $current.data        -- snapshot for audit
            ),
            $entity_id,
            ARRAY[$entity_id],
            $now, $now, $actor);

    UPDATE nodes
    SET    is_deleted = true, valid_to = $now
    WHERE  node_id = $entity_id
      AND  valid_to = 'infinity'
      AND  is_deleted = false;

    COMMIT;
  edge_cases:
    - Already deleted: E002 NOT_FOUND (WHERE clause filters is_deleted=false).
    - Connected edges: NOT automatically removed. They become dangling. explore_graph
      skips edges where either endpoint is_deleted=true. get_timeline still shows them.
    - Entity is a type_node: allowed (orphans entities of that type, but does not
      cascade-delete them — they retain their type_node_id FK for historical accuracy).
  errors:
    - E002 if not found or already deleted
    - E003 if scope denied

# ─── edge_created ─────────────────────────────────────────────

edge_created:
  trigger: connect_entities
  pre_validate:
    - Resolve source node:
        SELECT node_id, tenant_id, type_node_id FROM nodes
        WHERE node_id = $source_id AND is_deleted = false AND valid_to = 'infinity';
      If not found → E002 NOT_FOUND("source entity '$source_id'")
    - Resolve target node: same query for $target_id.
    - Resolve edge type_node_id:
        Same type resolution as entity_created, but kind='edge_type'.
        Search system tenant + caller tenant.
      If not found → E001 VALIDATION_ERROR("Invalid edge_type: '$edge_type'")
    - Cross-tenant check:
        $is_cross_tenant = (source.tenant_id != target.tenant_id)
        If $is_cross_tenant:
          - Check scope: read on target.tenant_id
          - Check grants table:
              SELECT 1 FROM grants
              WHERE subject_tenant_id = $tenant
                AND object_node_id = $target_id
                AND capability IN ('TRAVERSE', 'WRITE')
                AND is_deleted = false AND valid_to > $now
              LIMIT 1;
            If not found → E005 CROSS_TENANT_DENIED
    - Check scope: write on source.tenant_id
  pseudocode: |
    $edge_id  = uuid_generate_v7()
    $event_id = uuid_generate_v7()
    $edge_tenant = $source.tenant_id     -- edge owned by source tenant

    BEGIN;

    INSERT INTO events (event_id, tenant_id, intent_type, payload, stream_id,
                        node_ids, edge_ids, occurred_at, recorded_at, created_by)
    VALUES ($event_id, $edge_tenant, 'edge_created',
            jsonb_build_object(
              'edge_type', $edge_type,
              'edge_type_node_id', $edge_type_node_id::text,
              'source_id', $source_id::text,
              'target_id', $target_id::text,
              'source_tenant_id', $source.tenant_id::text,
              'target_tenant_id', $target.tenant_id::text,
              'data', COALESCE($data, '{}'::jsonb),
              'is_cross_tenant', $is_cross_tenant
            ),
            $source_id,            -- stream_id = source entity
            ARRAY[$source_id, $target_id],
            ARRAY[$edge_id],
            $now, $now, $actor);

    INSERT INTO edges (edge_id, tenant_id, type_node_id, source_id, target_id,
                       data, valid_from, valid_to, recorded_at,
                       created_by, created_by_event, is_deleted)
    VALUES ($edge_id, $edge_tenant, $edge_type_node_id,
            $source_id, $target_id,
            COALESCE($data, '{}'::jsonb),
            $now, 'infinity', $now,
            $actor, $event_id, false);

    COMMIT;
  edge_cases:
    - Duplicate edge (same source, target, type): allowed. Multiple edges of the same
      type between the same pair is valid (e.g., multiple "contacted_via" with different
      data like different phone numbers). Deduplication is the caller's responsibility.
    - Self-edges (source_id = target_id): allowed. Example: "entity references itself."
  errors:
    - E002 if source or target not found
    - E001 if edge_type not found
    - E005 if cross-tenant grant missing
    - E003 if scope denied

# ─── edge_removed ─────────────────────────────────────────────

edge_removed:
  trigger: (future tool or side effect)
  note: |
    Gen1 has no dedicated remove_edge tool. Edge removal happens via propose_event
    or future tool. This projection is defined for completeness and for propose_event
    to use when intent_type="edge_removed" is submitted.
  pre_validate:
    - Fetch edge:
        SELECT edge_id, tenant_id, source_id, target_id FROM edges
        WHERE edge_id = $edge_id AND is_deleted = false AND valid_to = 'infinity';
      If not found → E002 NOT_FOUND
    - Check scope: write on edge.tenant_id
  pseudocode: |
    $event_id = uuid_generate_v7()

    BEGIN;

    INSERT INTO events (event_id, tenant_id, intent_type, payload, stream_id,
                        edge_ids, occurred_at, recorded_at, created_by)
    VALUES ($event_id, $edge.tenant_id, 'edge_removed',
            jsonb_build_object(
              'edge_id', $edge_id::text,
              'source_id', $edge.source_id::text,
              'target_id', $edge.target_id::text
            ),
            $edge.source_id,       -- stream_id = source entity
            ARRAY[$edge_id],
            $now, $now, $actor);

    UPDATE edges
    SET    is_deleted = true, valid_to = $now
    WHERE  edge_id = $edge_id
      AND  is_deleted = false
      AND  valid_to = 'infinity';

    COMMIT;

# ─── epistemic_change ─────────────────────────────────────────

epistemic_change:
  trigger: store_entity with changed epistemic status on existing entity
  note: |
    Emitted INSTEAD OF entity_updated when epistemic status changes.
    If BOTH data and epistemic change in the same call, this intent_type is used
    (epistemic change is the more significant event for audit).
  pre_validate:
    - Same as entity_updated (fetch current, check scope, check expected_version)
    - Additionally validate epistemic transition:
        Valid transitions: hypothesis→asserted, hypothesis→confirmed, asserted→confirmed.
        Invalid: confirmed→anything, asserted→hypothesis, any→hypothesis (except capture_thought).
      If invalid → E001 VALIDATION_ERROR("Invalid epistemic transition: '$old' → '$new'")
  pseudocode: |
    $event_id   = uuid_generate_v7()
    $valid_from = $now

    BEGIN;

    INSERT INTO events (event_id, tenant_id, intent_type, payload, stream_id,
                        node_ids, occurred_at, recorded_at, created_by)
    VALUES ($event_id, $tenant, 'epistemic_change',
            jsonb_build_object(
              'entity_id', $entity_id::text,
              'old_epistemic', $current.epistemic,
              'new_epistemic', $params.epistemic,
              'old_data', $current.data,
              'new_data', $data                -- may be same as old_data if only epistemic changed
            ),
            $entity_id,
            ARRAY[$entity_id],
            $valid_from, $now, $actor);

    -- Close current version
    UPDATE nodes
    SET    valid_to = $valid_from
    WHERE  node_id = $entity_id AND valid_to = 'infinity' AND is_deleted = false;

    -- Open new version with new epistemic status
    INSERT INTO nodes (node_id, tenant_id, type_node_id, data, embedding,
                       epistemic, valid_from, valid_to,
                       recorded_at, created_by, created_by_event, is_deleted)
    VALUES ($entity_id, $tenant, $current.type_node_id,
            $data,              -- may include data changes too
            NULL,               -- embedding regenerated async
            $params.epistemic,  -- the new status
            $valid_from, 'infinity',
            $now, $actor, $event_id, false);

    COMMIT;
  post_tx:
    - Queue async embedding (data may have changed)

# ─── thought_captured ─────────────────────────────────────────

thought_captured:
  trigger: capture_thought
  pre_tx: |
    These steps happen BEFORE the database transaction:

    1. Fetch tenant type nodes (for LLM prompt context):
        SELECT node_id, data FROM nodes
        WHERE tenant_id = $tenant
          AND type_node_id = '00000000-0000-7000-0000-000000000001'
          AND is_deleted = false AND valid_to = 'infinity';

    2. Call LLM with complete prompt (§3.2 prompt template):
        $extraction = await call_extraction_llm($content, $type_nodes)
        If LLM fails or returns invalid JSON → E007 EXTRACTION_FAILED

    3. For each entity in $extraction.mentioned_entities, run deduplication:
        $dedup_result = await deduplicate(entity, $tenant)
        -- See §3.2 deduplication_strategy for the 3-tier algorithm.
        -- Result: { match: node | null, confidence: number, method: string }

    4. Build execution plan:
        $plan = {
          primary: { type_node_id, data, is_new: true },
          mentioned: [
            { type_node_id, data, is_new: !dedup.match, existing_id: dedup.match?.node_id }
          ],
          edges: [ { type_node_id, source_idx, target_idx_or_id } ]
        }

  pseudocode: |
    $event_id  = uuid_generate_v7()
    $primary_id = uuid_generate_v7()
    $new_node_ids = [$primary_id]
    $all_edge_ids = []

    BEGIN;

    -- 1. Create the thought_captured event
    INSERT INTO events (event_id, tenant_id, intent_type, payload, stream_id,
                        node_ids, occurred_at, recorded_at, created_by)
    VALUES ($event_id, $tenant, 'thought_captured',
            jsonb_build_object(
              'content', $content,
              'source', $source,
              'extraction_result', $extraction,    -- full LLM output for audit
              'dedup_results', $dedup_results       -- dedup decisions for audit
            ),
            $primary_id,        -- stream_id = primary entity
            $all_node_ids,      -- populated after inserts (set via UPDATE at end)
            $now, $now, $actor);

    -- 2. Create primary entity node
    INSERT INTO nodes (node_id, tenant_id, type_node_id, data, embedding,
                       epistemic, valid_from, valid_to,
                       recorded_at, created_by, created_by_event, is_deleted)
    VALUES ($primary_id, $tenant, $primary.type_node_id,
            $extraction.primary_entity.data,
            NULL,
            'hypothesis',       -- always hypothesis for capture_thought
            $now, 'infinity', $now, $actor, $event_id, false);

    -- 3. For each mentioned entity:
    FOR $mentioned IN $plan.mentioned:
      IF $mentioned.is_new:
        $mentioned_id = uuid_generate_v7()
        INSERT INTO nodes (node_id, tenant_id, type_node_id, data, embedding,
                           epistemic, valid_from, valid_to,
                           recorded_at, created_by, created_by_event, is_deleted)
        VALUES ($mentioned_id, $tenant, $mentioned.type_node_id,
                $mentioned.data, NULL,
                'hypothesis', $now, 'infinity', $now, $actor, $event_id, false);
        $new_node_ids.push($mentioned_id)
      ELSE:
        $mentioned_id = $mentioned.existing_id   -- link to existing entity

    -- 4. Create edges between entities
    FOR $rel IN $plan.edges:
      $edge_id = uuid_generate_v7()
      $source = resolve_idx($rel.source)   -- "primary" → $primary_id, index → $mentioned_ids[i]
      $target = resolve_idx($rel.target)   -- same, or "existing:UUID" → UUID
      INSERT INTO edges (edge_id, tenant_id, type_node_id, source_id, target_id,
                         data, valid_from, valid_to, recorded_at,
                         created_by, created_by_event, is_deleted)
      VALUES ($edge_id, $tenant, $rel.type_node_id,
              $source, $target,
              COALESCE($rel.data, '{}'::jsonb),
              $now, 'infinity', $now, $actor, $event_id, false);
      $all_edge_ids.push($edge_id)

    -- 5. Update event with final node_ids and edge_ids
    UPDATE events
    SET    node_ids = $new_node_ids, edge_ids = $all_edge_ids
    WHERE  event_id = $event_id;

    COMMIT;
  post_tx:
    - Queue async embeddings for all $new_node_ids
  edge_cases:
    - LLM returns zero entities: E007 EXTRACTION_FAILED("No entity type classified")
    - LLM returns entity type not in tenant's type_nodes: create as generic "note" type
      (must exist as a fallback type_node; seed data includes it).
    - Deduplication finds multiple strong matches (>0.95): use the highest similarity.
    - All mentioned entities are existing (no new nodes): still create primary + edges.
  errors:
    - E007 if LLM call fails or returns unparseable output
    - E003 if scope denied
    - Any INSERT failure → full ROLLBACK (INV-ATOMIC)

# ─── grant_created ────────────────────────────────────────────

grant_created:
  trigger: admin action (no dedicated MCP tool in gen1; use propose_event or direct API)
  pre_validate:
    - Verify caller has admin scope for issuing tenant.
    - Verify object_node_id exists and belongs to issuing tenant.
    - Verify subject_tenant_id exists.
  pseudocode: |
    $grant_id = uuid_generate_v7()
    $event_id = uuid_generate_v7()

    BEGIN;

    INSERT INTO events (event_id, tenant_id, intent_type, payload, stream_id,
                        node_ids, occurred_at, recorded_at, created_by)
    VALUES ($event_id, $tenant, 'grant_created',
            jsonb_build_object(
              'grant_id', $grant_id::text,
              'subject_tenant_id', $subject_tenant_id::text,
              'object_node_id', $object_node_id::text,
              'capability', $capability,
              'valid_from', $valid_from::text,
              'valid_to', COALESCE($valid_to, 'infinity')::text
            ),
            $object_node_id,    -- stream_id = the granted node
            ARRAY[$object_node_id],
            $now, $now, $actor);

    INSERT INTO grants (grant_id, tenant_id, subject_tenant_id, object_node_id,
                        capability, valid_from, valid_to, is_deleted,
                        created_by, created_by_event)
    VALUES ($grant_id, $tenant, $subject_tenant_id, $object_node_id,
            $capability,
            COALESCE($valid_from, $now), COALESCE($valid_to, 'infinity'),
            false, $actor, $event_id);

    COMMIT;

# ─── grant_revoked ────────────────────────────────────────────

grant_revoked:
  trigger: admin action (no dedicated MCP tool in gen1; use propose_event or direct API)
  pre_validate:
    - Fetch grant:
        SELECT * FROM grants
        WHERE grant_id = $grant_id AND is_deleted = false AND valid_to > $now;
      If not found → E002 NOT_FOUND
    - Verify caller has admin scope for grant.tenant_id
  pseudocode: |
    $event_id = uuid_generate_v7()

    BEGIN;

    INSERT INTO events (event_id, tenant_id, intent_type, payload, stream_id,
                        node_ids, occurred_at, recorded_at, created_by)
    VALUES ($event_id, $grant.tenant_id, 'grant_revoked',
            jsonb_build_object(
              'grant_id', $grant_id::text,
              'subject_tenant_id', $grant.subject_tenant_id::text,
              'object_node_id', $grant.object_node_id::text,
              'capability', $grant.capability,
              'reason', COALESCE($reason, 'admin_revocation')
            ),
            $grant.object_node_id,
            ARRAY[$grant.object_node_id],
            $now, $now, $actor);

    UPDATE grants
    SET    is_deleted = true, valid_to = $now
    WHERE  grant_id = $grant_id
      AND  is_deleted = false;

    COMMIT;

# ─── blob_stored ──────────────────────────────────────────────

blob_stored:
  trigger: store_blob
  pre_validate:
    - Decode $data_base64, compute $size_bytes.
    - If $related_entity_id provided:
        Verify entity exists and caller has write scope on its tenant.
    - Upload binary to object storage → receive $storage_ref.
      If upload fails → E009 INTERNAL_ERROR
    - Check scope: write on $tenant
  pseudocode: |
    $blob_id  = uuid_generate_v7()
    $event_id = uuid_generate_v7()

    BEGIN;

    INSERT INTO events (event_id, tenant_id, intent_type, payload, stream_id,
                        node_ids, occurred_at, recorded_at, created_by)
    VALUES ($event_id, $tenant, 'blob_stored',
            jsonb_build_object(
              'blob_id', $blob_id::text,
              'content_type', $content_type,
              'size_bytes', $size_bytes,
              'storage_ref', $storage_ref,
              'related_entity_id', $related_entity_id::text  -- nullable
            ),
            $related_entity_id,       -- stream_id = related entity (or NULL)
            CASE WHEN $related_entity_id IS NOT NULL
                 THEN ARRAY[$related_entity_id] ELSE '{}'::uuid[] END,
            $now, $now, $actor);

    INSERT INTO blobs (blob_id, tenant_id, content_type, storage_ref,
                       size_bytes, node_id, created_at, created_by)
    VALUES ($blob_id, $tenant, $content_type, $storage_ref,
            $size_bytes, $related_entity_id, $now, $actor);

    COMMIT;
  edge_cases:
    - Object storage upload succeeds but DB transaction fails: orphaned blob in storage.
      Mitigation: gen1-impl SHOULD implement a cleanup job or accept the orphan (storage
      is cheap). The upload happens pre-tx because it's an external side effect.
    - No related_entity_id: blob is standalone (e.g., a template document).
  errors:
    - E002 if related_entity_id provided but not found
    - E003 if scope denied
    - E009 if object storage upload fails
```

**Completeness check:** Every `intent_type` in the events table CHECK constraint (§2.1) has a matching projection entry above: `entity_created`, `entity_updated`, `entity_removed`, `edge_created`, `edge_removed`, `epistemic_change`, `thought_captured`, `grant_created`, `grant_revoked`, `blob_stored`. [gen1.1: completeness verified]

### 3.5 Transaction boundaries and error model

This section defines how mutations maintain consistency between the event layer and the fact layer.

```yaml
# ── TRANSACTION MODEL ──

principle: |
  Every mutation follows the same pattern:
  1. VALIDATE — check permissions, resolve types, verify constraints (outside tx)
  2. EXTERNAL CALLS — LLM extraction, entity search for deduplication (outside tx, capture_thought only)
  3. BEGIN TRANSACTION
  4. WRITE EVENT — INSERT into events table
  5. PROJECT — INSERT/UPDATE fact tables (nodes, edges, grants)
  6. COMMIT TRANSACTION
  7. ASYNC SIDE EFFECTS — queue embedding generation, notifications (outside tx)

  If step 5 fails, step 4 is rolled back. No orphaned events exist. (See INV-ATOMIC.)
  If step 7 fails, the entity exists but may lack embeddings temporarily. This is acceptable.

failure_modes:
  validation_failure:
    description: "Input doesn't pass schema/auth checks"
    behavior: "Return error immediately, no event created, no DB writes"
    examples: ["invalid entity_type", "missing required field", "auth denied"]

  projection_failure:
    description: "Event is valid but fact table write fails (constraint violation, FK error)"
    behavior: "ROLLBACK entire transaction including event. Return error with details."
    examples: ["EXCLUDE constraint on overlapping valid ranges", "FK to nonexistent type_node"]

  concurrency_conflict:
    description: "expected_version does not match current version"
    behavior: "Return CONFLICT error with current version info. No event created."
    examples: ["Two agents updating same lead simultaneously"]

  external_call_failure:
    description: "LLM or embedding API call fails"
    behavior: |
      - capture_thought LLM failure: return error, no entities created
      - Embedding generation failure: entity is created without embedding, embedding is retried via async queue
      - Embedding failures MUST NOT prevent entity creation (embeddings are eventual)

  partial_capture_thought:
    description: "LLM extraction succeeds but one of multiple entity/edge inserts fails"
    behavior: "ROLLBACK entire transaction. All-or-nothing for the capture. Return error with extraction result so agent can retry or store manually."

# ── ERROR CODE REGISTRY (gen1 — exact codes) ──

error_codes:
  E001_VALIDATION_ERROR:
    code: "VALIDATION_ERROR"
    http_status: 400
    template: "Validation failed: {details}"
    details_examples:
      - "Missing required parameter: entity_type"
      - "Invalid entity_type: 'foo' is not a recognized type node in tenant"
      - "Parameter 'limit' must be between 1 and 100"

  E002_NOT_FOUND:
    code: "NOT_FOUND"
    http_status: 404
    template: "Not found: {resource_type} with id '{id}'"
    details_examples:
      - "Not found: entity with id '30000000-...'"
      - "Not found: type_node with name 'nonexistent'"

  E003_AUTH_DENIED:
    code: "AUTH_DENIED"
    http_status: 403
    template: "Access denied: {reason}"
    details_examples:
      - "Access denied: token lacks scope 'tenant:T1:nodes:lead:write'"
      - "Access denied: token does not cover tenant 'T2'"

  E004_CONFLICT:
    code: "CONFLICT"
    http_status: 409
    template: "Conflict: entity '{entity_id}' was modified. Expected version {expected}, current version {actual}."
    details: "Returned when expected_version is provided and doesn't match current node's valid_from."

  E005_CROSS_TENANT_DENIED:
    code: "CROSS_TENANT_DENIED"
    http_status: 403
    template: "Cross-tenant access denied: no grant for {capability} on node '{node_id}' in tenant '{tenant_id}'"
    details: "Returned when cross-tenant edge creation or traversal fails grants check."

  E006_SCHEMA_VIOLATION:
    code: "SCHEMA_VIOLATION"
    http_status: 400
    template: "Schema violation for type '{entity_type}': {validation_errors}"
    details: "Returned when entity data doesn't match type_node's label_schema. Includes JSON Schema validation errors."

  E007_EXTRACTION_FAILED:
    code: "EXTRACTION_FAILED"
    http_status: 422
    template: "Entity extraction failed: {reason}"
    details_examples:
      - "LLM returned invalid JSON"
      - "LLM extraction timed out after 30s"
      - "No entity type could be classified from input"

  E008_RATE_LIMITED:
    code: "RATE_LIMITED"
    http_status: 429
    template: "Rate limited. Retry after {retry_after_seconds} seconds."
    details: "Returned when too many requests from same token. Includes Retry-After header."

  E009_INTERNAL_ERROR:
    code: "INTERNAL_ERROR"
    http_status: 500
    template: "Internal error: {message}. Reference: {error_ref}"
    details: "Includes a unique error reference for debugging. Never exposes internal details to the caller."

# Error response shape (all tools):
# {
#   "error": {
#     "code": "E003_AUTH_DENIED",
#     "message": "Access denied: token lacks scope 'tenant:T1:nodes:lead:write'",
#     "details": { ... }  // optional additional context
#   }
# }
```

### 3.6 Resources (read-only context)

```yaml
resonansia://tenants:
  description: "Tenants accessible to current token"
  returns: [{ tenant_id, name, role }]

resonansia://schema/{tenant_id}:
  description: "Data model of a tenant (same as get_schema tool)"

resonansia://entity/{entity_id}:
  description: "Full entity with connections and recent events"

resonansia://stats/{tenant_id}:
  description: "Dashboard statistics (same as get_stats tool)"

resonansia://stream/{entity_id}:
  description: "Full event stream for an entity (same as get_timeline tool)"
```

### 3.7 Prompts (reusable agent instructions)

```yaml
analyze_entity:
  description: "Summarize entity state, identify patterns, suggest next actions"
  args: { entity_id: string }

find_path:
  description: "Find and explain the shortest path between two entities"
  args: { from_id: string, to_id: string }

temporal_diff:
  description: "Compare entity state at two points in time, explain what changed"
  args: { entity_id: string, time_a: datetime, time_b: datetime }

verify_trust:
  description: "Assess data quality: check event lineage integrity, epistemic status distribution, and actor diversity for an entity"
  args: { entity_id: string }
```

---
