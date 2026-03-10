You are an expert developer who has been handed the Resonansia specification and told to implement it. Your job: find every place where the spec would leave you confused, uncertain, or forced to make assumptions. A great spec makes implementation boring. Your implementation experience should be boring.

## Step 1: Read the Complete Specification

Read the unified specification:
- `torvalds-output/wave3/unified-spec.md`

Also read the tech profile and implementation context:
- `tech-profile.md` — Supabase tech profile (Deno, Hono, pgvector)
- `index.md` — Section 8.5 (implementation brief), section 8.6 (gen2+ roadmap)

You are implementing this on:
- **Runtime:** Deno (Supabase Edge Functions)
- **Framework:** Hono
- **MCP adapter:** @hono/mcp@^0.2.3
- **MCP SDK:** @modelcontextprotocol/sdk@^1.27.1
- **Database:** Supabase PostgreSQL + pgvector
- **Auth:** Supabase Auth + custom JWT
- **Embedding:** text-embedding-3-small via OpenAI
- **LLM:** gpt-4o-mini via OpenAI
- **Test runner:** Deno test
- **Validation:** Zod

## Step 2: Implementation Walkthrough

Go through the spec as if you were implementing it, work package by work package, in the order specified.

For each work package:

### What you would build:
Describe the concrete files, functions, and modules you would create. Map spec sections to code artifacts.

### Where you would need to make a judgment call:
Flag every point where the spec doesn't give you enough information and you'd have to decide yourself. Examples:
- File organization (what goes in which file?)
- Error message wording (spec says "return an error" but what exact message?)
- Configuration values (connection pool size, timeout durations, retry counts)
- Import structure (circular dependency risks)

### Where you would need to ask a question:
Flag every point where you genuinely don't know what the spec intends. Examples:
- "The spec says 'handle gracefully' — does that mean return null, throw an error, or log and continue?"
- "The spec defines the embedding pipeline as 'async' in one place and 'synchronous' in another — which is it?"
- "The spec says store_entity should check for duplicates, but what's the exact matching logic when embeddings aren't available yet?"

### Where you would be unsure about expected behavior:
Flag every ambiguity in behavior specification. Examples:
- "If capture_thought's LLM returns entities that conflict with existing entities, does it merge them, create duplicates, or fail?"
- "If an edge is created to a soft-deleted node, does the operation succeed or fail?"

## Step 3: Technology Assessment

### Can this be built with the specified stack?

**Supabase Edge Functions:**
- 2s CPU limit, 150s wall-clock: Is capture_thought feasible? (LLM call + dedup + embedding + multiple DB writes)
- 300MB RAM: Is this enough for the MCP SDK + Hono + tool implementations?
- 20MB bundle size: Can all dependencies fit?
- Cold start latency: What's the expected first-request latency?
- Deno compatibility: Are @hono/mcp and @modelcontextprotocol/sdk Deno-compatible?

**PostgreSQL + pgvector:**
- HNSW index build time at 10K, 100K vectors with 1536 dimensions?
- Can the EXCLUDE constraint work efficiently on the nodes table?
- Can RLS policies with subqueries (grants table lookup) perform acceptably?

**External APIs:**
- OpenAI rate limits: What happens if we hit them during bulk operations?
- LLM response variability: How robust is the capture_thought prompt against different response formats?

### Are there operations that are impractical at the implied scale?
- Graph traversal (explore_graph) with large fan-out?
- Semantic search with complex filters?
- Bitemporal queries across large time ranges?

### Performance vs correctness conflicts:
- INV-ATOMIC requires transaction-level atomicity, but capture_thought involves external API calls (LLM, embedding) that can't be inside a database transaction
- Deduplication requires embedding comparison, but embeddings are generated asynchronously
- Graph traversal depth limits vs comprehensive exploration needs

## Step 4: Implementation Risks

### Hardest to implement:
Rank the top 5 most difficult implementation challenges and explain why.

### Most likely bugs:
Where will bugs hide? What are the trickiest edge cases?

### Hardest to test:
Which behaviors are hardest to test in an automated way? (LLM-dependent behavior, timing-dependent behavior, concurrent access)

### Hardest to debug in production:
What will be the most difficult issues to diagnose when things go wrong in production? (Silent data corruption, RLS policy interactions, event-fact inconsistency)

## Step 5: Developer Experience

### Would you want to work with this specification?
- Is it well-organized? Can you find what you need quickly?
- Is it overwhelming? Too much detail in some places, not enough in others?
- Does it make the implementation feel achievable?

### What would make it better FROM THE IMPLEMENTER'S PERSPECTIVE?
- Code examples? (Even pseudocode in a specific language is more useful than abstract pseudocode)
- Dependency diagram? (Visual representation of which modules depend on which)
- Configuration reference? (All environment variables in one place)
- Quick-start guide? ("To get from zero to running server, do these 10 things in order")

### Can a competent developer implement this in work package order without backtracking?
- Are there forward references? (WP3 needs something from WP5?)
- Are there hidden dependencies?
- Would you need to refactor earlier work as you learn more from later WPs?

## Step 6: Improvement Report

For each issue:

```
IMPLEMENTER ISSUE [{priority}]
Section: {which section of the spec}
What the spec says: {quoted or summarized}
What I'd need it to say: {specific additional information}
Proposed additional text: {exact text to add to the spec}
```

Priority:
- **P0 — BLOCKER:** Cannot implement without this information
- **P1 — SIGNIFICANT:** Would likely implement incorrectly without this
- **P2 — IMPROVEMENT:** Would make implementation smoother/faster
- **P3 — NICE-TO-HAVE:** Would improve developer experience

## Rules

- You are the specification's toughest customer. Do not be polite. Be precise.
- "This seems hard" is not useful. "This requires distributed consensus but the spec doesn't specify a consensus protocol; the options are X/Y/Z with tradeoffs A/B/C" is useful.
- Think about the actual Deno code you'd write. Think about the actual SQL queries. Think about the actual Zod schemas. If you can't translate a spec passage into code, it's not specified well enough.
- Consider the full development lifecycle: implementation, testing, deployment, monitoring, debugging, updating.
- Spend as many tokens as needed.

## Output

Write the complete implementer's review to:
`torvalds-output/wave4/4C-implementer.md`

Start with:
```
# Wave 4C — Implementer's Review
## Reviewer: The Implementer's Advocate
## Input: Unified Specification (wave3/unified-spec.md)
## Stack: Deno + Hono + Supabase + pgvector + @hono/mcp
## Date: {today}
```

When the file is written, output: **AGENT 4C COMPLETE**
