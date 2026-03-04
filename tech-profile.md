# RESONANSIA MCP SERVER — GEN 2 SPEC

> Part of [Gen2 Spec](index.md)

## 10. TECH PROFILE

This section is injected by the human when starting a generation. The spec above is tech-agnostic. The tech profile binds it to a specific stack.

### 10.1 Profile format

```yaml
# === TECH PROFILE — copy and customize for your generation ===
runtime: ""              # e.g. "deno" | "bun" | "node"
framework: ""            # e.g. "hono" | "express" | "fastify"
mcp_adapter: ""          # e.g. "@hono/mcp" | "@modelcontextprotocol/sdk direct"
database: ""             # e.g. "supabase postgres" | "neon" | "self-hosted postgres"
vector_extension: ""     # e.g. "pgvector" | "pgembedding" | "external (pinecone)"
embedding_dimensions: "" # e.g. "1536" | "768" — overrides DECIDE if set
auth_provider: ""        # e.g. "supabase auth" | "cognito" | "auth0" | "custom jwt"
embedding_api: ""        # e.g. "openai via openrouter" | "openai direct" | "voyage"
extraction_llm: ""       # e.g. "gpt-4o-mini via openrouter" | "claude-haiku"
object_storage: ""       # e.g. "supabase storage" | "s3" | "r2"
deployment: ""           # e.g. "supabase edge functions" | "fly.io" | "railway" | "docker"
test_runner: ""          # e.g. "deno test" | "bun test" | "vitest"
schema_validation: ""    # e.g. "zod" | "typebox" | "arktype"
```

### 10.2 Bound profile: Supabase (gen1 target)

```yaml
# This is the BOUND tech profile for gen1. Not an example — this is what gen1-impl builds against.
runtime: "deno"
framework: "hono"
mcp_adapter: "@hono/mcp@^0.2.3"
mcp_sdk: "@modelcontextprotocol/sdk@^1.27.1"
mcp_protocol: "2025-11-25"
database: "supabase postgres (PG 15+) + pgvector"
vector_extension: "pgvector (HNSW, 1536d)"
embedding_dimensions: 1536
auth_provider: "supabase auth + custom JWT signing for agents"
embedding_api: "text-embedding-3-small via OpenAI (or OpenRouter)"
extraction_llm: "gpt-4o-mini via OpenAI (or OpenRouter)"
object_storage: "supabase storage"
deployment: "supabase edge functions"
test_runner: "deno test"
schema_validation: "zod"
```

### 10.3 Example: Bun standalone profile

```yaml
runtime: "bun"
framework: "hono"
mcp_adapter: "@hono/mcp"
database: "postgres (any) + pgvector"
vector_extension: "pgvector"
auth_provider: "custom JWT"
embedding_api: "text-embedding-3-small via openai"
extraction_llm: "gpt-4o-mini via openai"
object_storage: "s3-compatible"
deployment: "docker on fly.io"
test_runner: "bun test"
schema_validation: "zod"
```

---
