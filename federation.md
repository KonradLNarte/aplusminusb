# RESONANSIA MCP SERVER — GEN 2 SPEC

> Part of [Gen2 Spec](index.md)

## 5. FEDERATION

### 5.1 Core concept

Federation = controlled data sharing across tenant boundaries.

- **NOT** copying data between databases
- **IS** cross-tenant edges + capability grants + scoped tokens that allow queries to traverse boundaries

### 5.2 Primitives

```yaml
cross_tenant_edge:
  what: Edge where source and target belong to different tenants
  requires: Write scope on both tenants
  consent: Platform admin creates grants manually (gen1). Consent protocol (gen2+).

grants:
  what: Explicit capability grants stored in the grants table
  how: Grant(subject_tenant=T2, object_node=N1, capability=TRAVERSE, valid_range)
  constraint: Grants are temporal — they expire. Grants emit events when created.

virtual_endpoint:
  what: Single MCP URL that combines multiple tenants into one logical view
  how: Token with multiple tenant_ids → queries span all
  constraint: RLS + grants still enforce per-entity permissions

partner_endpoint:
  what: Time-limited MCP access for external partners
  scopes: Read-only, specific entity types
  logging: All access logged in events table
  expiry: Token exp claim, non-renewable
```

### 5.3 Federation scope per generation

```yaml
gen1: [cross_tenant_edge: YES, grants_table: YES, virtual_endpoint: YES, partner_token: YES, consent_protocol: NO]
gen2: [consent_protocol: YES, federated_search: YES, grant_delegation: YES, a2a_agent_card: YES, a2a_server_endpoint: YES]
gen3: [remote_query_forwarding: YES, cryptographic_event_chain: YES, federated_graph_merge: YES, a2a_client: YES, a2a_federated_mesh: YES]
    # NOTE ON CRYPTOGRAPHIC EVENT CHAINS:
    # Gen3 introduces hash-linked events (SHA-256 chain where each event references
    # the hash of the previous event in its stream) and Ed25519 signatures per actor.
    # This enables trustless federation — a remote Resonansia instance can verify that
    # an event stream has not been tampered with without trusting the source database.
    # Gen1-2 use UUID event_id with no hash linking. The verify_lineage tool checks
    # FK integrity only. Gen3 upgrades verify_lineage to check cryptographic chain integrity.
    # Design reference: content-addressable event_hash as PK, JCS canonicalization (RFC 8785)
    # for deterministic hashing, prev_hash column linking to previous event in stream.
```

### 5.4 A2A agent interoperability

Resonansia is an MCP server — agents access its tools via MCP. But in a multi-agent world, agents also need to discover and collaborate with *each other*. The A2A (Agent2Agent) Protocol addresses this layer.

**Relationship between MCP and A2A:**
- **MCP** = how an agent connects to tools and data (Resonansia's primary interface)
- **A2A** = how agents discover each other's capabilities, delegate tasks, and exchange results
- A Resonansia tenant could be *both*: an MCP tool server that agents connect to, AND an A2A-discoverable agent that other agents can delegate tasks to

**Why this matters for Resonansia:**
The federation model (section 5.1-5.3) currently assumes all agents connect directly to Resonansia via MCP. But in practice, an agent running in one organization may want to *discover* that a Resonansia tenant exists and what it can do — without having a preconfigured MCP connection. A2A's AgentCard provides this discovery mechanism.

```yaml
# ── A2A INTEGRATION ROADMAP ──

gen1:
  # No A2A implementation. Research completed, awareness documented.

  # [RESOLVED:RESEARCH:gen1] A2A Protocol v0.3 specification
  # Findings (verified Feb 2026):
  # - A2A v0.3 released July 2025. Draft v1.0 in development.
  # - 150+ backing organizations (Google, Salesforce, SAP, LangChain)
  # - AgentCard format: JSON at /.well-known/agent-card.json
  #   Required fields: name, url, version, capabilities, skills
  #   Auth via securitySchemes (OAuth2, OIDC, ApiKey, mTLS)
  # - Task lifecycle: submitted → working → completed/failed/canceled/rejected
  #   Also: input-required, auth-required states
  # - Three transport options: JSON-RPC, HTTP+JSON/REST, gRPC (all equal status)
  # - Relationship to MCP: COMPLEMENTARY. MCP = agent-to-tools. A2A = agent-to-agent.
  # - Signed AgentCards supported (not enforced) for integrity verification
  # References:
  #   https://a2a-protocol.org/latest/specification/
  #   https://a2a-protocol.org/v0.3.0/specification/

  # [RESOLVED:RESEARCH:gen1] A2A AgentCard auto-generation from MCP tools
  # Findings:
  # - Multiple bridge implementations exist (A2ABridge, A2A-MCP-Server, MCPAdapt proposal)
  # - Mapping MCP tools → A2A skills is feasible but lossy:
  #   - MCP Tool.name → AgentSkill.id
  #   - MCP Tool.description → AgentSkill.description
  #   - MCP Tool.inputSchema → AgentSkill.inputModes (with transformation)
  # - A single A2A skill may map to multiple MCP tools (semantic grouping)
  # - Auto-generated AgentCards are valid but "impoverished" — need manual augmentation
  #   for tags, examples, human-readable descriptions
  # Recommendation for gen2: auto-generate base AgentCard from MCP tool metadata,
  # augment with hand-authored tenant-specific metadata, group related tools into skills.
  # References:
  #   https://github.com/a2aproject/A2A/issues/134
  #   https://lobehub.com/mcp/darrelmiller-a2abridge

  - Design constraint: gen1 architecture VERIFIED as A2A-ready:
    - Tool descriptions are machine-readable (MCP tool schemas with Zod) ✓
    - Tenant capabilities are introspectable (get_schema returns type nodes) ✓
    - Auth model is compatible (OAuth 2.1 aligns with A2A's securitySchemes) ✓
    - Streaming support via Streamable HTTP transport aligns with A2A SSE ✓

gen2:
  # [RESEARCH:gen2] A2A Protocol version stability check
  # [gen1.1] This marker is MANDATORY before any gen2 A2A work begins.
  # A2A v0.3 was current as of Feb 2026. The protocol is pre-1.0 and may
  # have breaking changes to AgentCard format, task lifecycle, or auth.
  # Gen2-spec MUST:
  #   1. Check current A2A version (may be v0.4, v1.0, or deprecated)
  #   2. Compare AgentCard schema against gen1 research findings above
  #   3. If breaking changes: revise gen2 A2A DECIDE markers below
  #   4. If v1.0 released: treat as stable, remove experimental caveats
  #   5. If deprecated/abandoned: remove A2A from gen2 scope entirely
  # question_this_if: "A2A v1.0 released with breaking changes from v0.3"

  - "[DECIDE:gen2] A2A AgentCard generation"
    description: "Each Resonansia tenant publishes an AgentCard at /.well-known/agent-card.json"
    content: |
      The AgentCard would advertise:
        - name: tenant name
        - description: from tenant config
        - skills: derived from tenant's type nodes and available MCP tools
        - authentication: OAuth 2.1 (same as MCP auth)
        - supportedModes: ["text"] (gen2), ["text", "data"] (gen3)
    question_this_if: "A2A protocol changes significantly before gen2 implementation"
    depends_on: "[RESEARCH:gen2] A2A Protocol version stability check (above)"

  - "[DECIDE:gen2] A2A server endpoint"
    description: "Resonansia exposes A2A message/send endpoint alongside MCP"
    options:
      A: "Separate A2A endpoint that translates A2A tasks to MCP tool calls internally"
      B: "Unified endpoint that speaks both MCP and A2A based on content negotiation"
      C: "A2A handled by a thin proxy layer in front of the MCP server"
    recommendation: "Option A — cleanest separation, A2A adapter wraps MCP tools"

gen3:
  - A2A client capability: Resonansia agents can discover and delegate tasks to external A2A agents
  - Federated agent mesh: Resonansia tenants across different instances discover each other via A2A
  - AgentCard signing for trust verification between federated instances
```

**References:**
- A2A Protocol Specification: https://a2a-protocol.org/latest/specification/
- A2A GitHub: https://github.com/a2aproject/A2A
- A2A Security Analysis: https://semgrep.dev/blog/2025/a-security-engineers-guide-to-the-a2a-protocol/
- MCP + A2A relationship: https://codelabs.developers.google.com/intro-a2a-purchasing-concierge

---
