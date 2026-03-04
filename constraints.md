# RESONANSIA MCP SERVER — GEN 2 SPEC

> Part of [Gen2 Spec](index.md)

## 6. CONSTRAINTS

Real-world constraints that bound all design decisions. **Research these values before implementation — they may have changed.**

### 6.1 Platform constraints (VERIFIED March 2026)

```yaml
# [RESOLVED:RESEARCH:gen1] Supabase Edge Functions — verified limits
# Source: https://supabase.com/docs/guides/functions/limits
serverless_cpu_per_request: "2 seconds"          # HARD LIMIT, all plans. Async I/O excluded.
serverless_wall_clock: "150 seconds"              # Request idle timeout (was 400s in gen0 estimate)
serverless_memory_heap: "150 MB"                  # JavaScript heap memory
serverless_memory_external: "150 MB"              # Array buffers, WASM (separate from heap)
serverless_memory_total: "~300 MB"                # Combined
function_bundle_size: "20 MB"                     # After bundling with Supabase CLI
function_source_size: "10 MB"                     # Pre-bundle

# IMPORTANT: capture_thought does LLM call + entity search + node creation + edge creation.
# [RESOLVED:gen1 D-016] capture_thought: SYNCHRONOUS (Option A).
# Analysis: LLM call (~1-3s wall, <50ms CPU) + dedup search (~100ms wall, ~20ms CPU)
# + DB transaction (~100ms wall, ~30ms CPU) = ~2-5s wall, <200ms CPU.
# Supabase limits: 2s CPU, 150s wall-clock. Fits comfortably because LLM/DB calls
# are async I/O (don't count against CPU). Async adds complexity not justified at gen1 scale.
# question_this_if: "capture_thought p95 latency exceeds 10s or CPU exceeds 1.5s"
```

### 6.2 Vector search constraints

```yaml
# pgvector HNSW performance benchmarks (1536 dimensions, March 2026):
# 15K vectors:  480 QPS, 16ms p95  (1 GB RAM)
# 100K vectors: 240 QPS, 126ms p95 (4 GB RAM)
# 1M vectors:   560 QPS, 58ms p95  (32 GB RAM)
# Source: https://supabase.com/docs/guides/ai/choosing-compute-addon
#
# Implication: At gen1 scale (<100K nodes), vector search is fine on minimal infra.
embedding_latency_target: "<100ms p95"
search_latency_target: "<200ms p95 including RLS filtering"
```

### 6.3 MCP protocol constraints

```yaml
# [RESOLVED:RESEARCH:gen1] Verified against MCP spec 2025-11-25
protocol_version: "2025-11-25"                    # Latest stable
transport: "Streamable HTTP (JSON-RPC 2.0)"       # Required by spec
auth: "OAuth 2.1 — Resource Server only"           # June 2025 + Nov 2025 revisions
sdk_version: "@modelcontextprotocol/sdk@^1.27.1"  # Latest stable; v2 not yet stable
hono_mcp_version: "@hono/mcp@^0.2.3"              # Hono MCP middleware
multi_tenant: "NOT standardized — solved at application level via JWT claims"
# Key spec features available:
#   - Streamable HTTP transport (replaces deprecated SSE transport)
#   - Tool Annotations (metadata on tools)
#   - Protected Resource Metadata (RFC 9728)
#   - Resource Indicators (RFC 8707)
#   - OpenID Connect Discovery
# Source: https://modelcontextprotocol.io/specification/2025-11-25
```

### 6.4 Cost constraints

```yaml
# Gen1-impl MUST produce cost estimates covering:
# - Database hosting plan requirements for Pettson scenario
# - Embedding API cost per 1K / 10K / 100K nodes
# - LLM extraction cost per 1K capture_thought calls
# - Expected monthly cost for Pettson-scale usage (4 tenants, ~1000 nodes, ~100 capture_thought/month)
```

---
