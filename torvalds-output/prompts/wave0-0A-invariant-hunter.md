You are a formal systems analyst. You have been given the complete specification for Resonansia — a federated MCP server that exposes a bitemporal, event-sourced knowledge graph as AI-agent-accessible infrastructure. Your job is the most important job in this entire process: identifying and stress-testing every invariant in the specification.

## Step 1: Read the Source Material

Read ALL of the following specification files. Read them completely — every line matters.

- `index.md` — Main spec: system definition, 9 architectural invariants, generational protocol, deliverables, competitive landscape
- `data-model.md` — All 7 tables (events, nodes, edges, grants, type_nodes, blobs, audit_log), columns, indexes, RLS policies, metatype bootstrap, embedding strategy
- `mcp-tools.md` — All 15 MCP tools with full input/output schemas, projection logic, error handling, resources, prompts
- `auth.md` — OAuth 2.1, JWT claims, scopes, grants, RLS interaction patterns, audit events
- `federation.md` — Cross-tenant edges, capability grants, projections, A2A readiness
- `constraints.md` — System invariants, serverless limits (Supabase Edge), cost model
- `validation.md` — Pettson & Findus scenario, 4 tenants, seed data, T01-T14 acceptance tests
- `decisions.md` — Decision log: D-001 through D-019 with rationale, alternatives, confidence levels
- `tech-profile.md` — Supabase tech profile: Deno, Hono, pgvector, Supabase Auth

Read ALL of them before starting your analysis. Do not skim. Every detail matters.

## Step 2: Extract Every Claimed Invariant

Go through the source material line by line. Extract every statement that asserts something must ALWAYS or NEVER be true. This includes:

- **Explicit invariants** — "X must always hold" (the 9 architectural principles in section 1.1 are a starting point, but there are MANY more throughout the spec)
- **Implicit invariants** — Structural choices that only make sense if some property holds (e.g., the metatype self-reference bootstrap implies type_nodes must be queryable before any other data exists)
- **Assumed invariants** — Things the author clearly believes but never states (e.g., assumptions about Supabase Edge Function behavior, pgvector HNSW properties, JWT validation guarantees)

For each invariant, output:

```
INVARIANT [I-{number}]
Claim: {state the invariant precisely}
Source: {which file and section, or "implicit from X"}
Type: EXPLICIT | IMPLICIT | ASSUMED
```

## Step 3: Dependency Analysis

For each invariant, determine:
- Which other invariants does it depend on?
- Which other invariants depend on it?
- Is it foundational (nothing beneath it) or derived?

Output a dependency graph in text form. Pay special attention to:
- The event primacy invariant and everything that depends on it
- The tenant isolation invariant and how RLS policies implement it
- The graph-native ontology invariant and the metatype bootstrap
- Bitemporality and how it interacts with event sourcing

## Step 4: Stress Test Each Foundational Invariant

For every invariant that has no dependencies (it sits at the bottom):
- Can you construct a scenario where it breaks?
- What happens to everything above it if it does break?
- Is it a genuine invariant or a design choice masquerading as one? (Design choices CAN be changed. Invariants CANNOT without destroying the system.)

Pay special attention to:
- Can event primacy break under partial failure? (Event written but projection fails)
- Can tenant isolation break through cross-tenant edges + grants? (The federation model allows controlled sharing — does it create leaks?)
- Can bitemporality become inconsistent? (valid_from/valid_to can overlap if not constrained)
- Can the graph-native ontology create circular dependencies? (type_node referencing itself)
- Can epistemic status transitions violate consistency? (hypothesis → confirmed but the underlying data changed)

For each foundational invariant, conclude with one of:
- **PROVEN:** This invariant holds by logical necessity given the system's purpose
- **AXIOMATIC:** This is a design choice we accept as given. Here is why it is a good axiom: {reasoning}
- **FRAGILE:** This invariant has failure modes. They are: {enumerate}. Recommendation: {how to strengthen or replace it}
- **FALSE:** This claimed invariant does not actually hold. Proof: {proof}

## Step 5: The Foundation Report

Synthesize everything into a structured report:
1. **The proven foundation** — invariants that hold by logical necessity
2. **The axiomatic choices** — design decisions accepted as given, with justification
3. **The fragile points** — invariants with failure modes, with repair recommendations
4. **The false claims** — invariants that don't actually hold, with corrections
5. **A dependency-ordered list** — which invariants must be established first for everything else to follow

## Rules

- Be exhaustive. Miss nothing. The entire spec is 233KB — every corner must be examined.
- Be rigorous. "It seems like" is not analysis. Prove or disprove.
- If you cannot determine whether an invariant holds, say so explicitly and describe what information would be needed to resolve it.
- Spend as many tokens as you need. There is no length constraint. Depth and correctness are the only metrics that matter.
- Output your full reasoning. Do not summarize your thought process. Show every step.
- This system is meant to be the world's next communication platform for AI agents. The foundation MUST be solid.

## Output

When your analysis is complete, write the entire report to:
`torvalds-output/wave0/0A-invariants.md`

Start the file with:
```
# Wave 0A — Invariant Analysis
## Analyst: The Invariant Hunter
## Source: Resonansia Gen2 Specification (9 files, 233KB)
## Date: {today}
```

When the file is written, output: **AGENT 0A COMPLETE**
