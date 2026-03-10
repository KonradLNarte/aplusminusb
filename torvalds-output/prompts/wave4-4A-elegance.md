You are reviewing the Resonansia specification for elegance, simplicity, and conceptual clarity. Your standard: the specification should have the quality of the best systems engineering you have ever seen. Think of UNIX, Internet protocols, Git — systems where the core abstractions were so well-chosen that they remained valid for decades.

## Step 1: Read the Complete Specification

Read the unified specification:
- `torvalds-output/wave3/unified-spec.md`

This is a large document. Read it completely. Every section. Every detail.

Also read the original system definition for context:
- `index.md` — section 1.1 (architectural principles) and section 1.2 (competitive landscape)

## Step 2: Abstraction Quality

### Core abstractions in Resonansia:
1. **Events as truth, facts as projections** — Is this the right abstraction? Is it well-executed?
2. **Graph-native ontology (type nodes in the graph)** — Does this carve nature at its joints?
3. **Bitemporality (valid time + system time)** — Is this necessary complexity or essential capability?
4. **Epistemic status (hypothesis/asserted/confirmed)** — Is this a genuine abstraction or bolted-on metadata?
5. **Capability grants for federation** — Is this the right model for cross-tenant sharing?
6. **MCP as the interface layer** — Does the MCP tool model fit the knowledge graph domain?
7. **UUIDv7 for natural ordering** — Is this a good choice or an unnecessary dependency on ID format?

For each:
- Is it well-chosen? Does it simplify or complicate?
- Could anything be removed without loss of capability?
- Is there a simpler way to achieve the same guarantees?
- Does the design have COMPOSABILITY — can the parts be combined in unexpected ways?
- Is the abstraction leaky? (Does the implementation detail bleed through?)

### Unnecessary complexity:
- Are there features that add complexity but provide marginal value for gen1?
- Is the bitemporality model pulling its weight, or would simple versioning suffice?
- Is the event-sourcing architecture justified for the expected scale?
- Is the deduplication algorithm (3-tier) over-engineered for gen1?
- Are there simpler alternatives that would work for the Pettson scenario?

## Step 3: Naming Quality

Review every name in the specification:
- Table names: events, nodes, edges, grants, type_nodes, blobs, audit_log
- Column names: every column in every table
- Tool names: all 15 MCP tools
- Event intent_types: entity_stored, edge_created, etc.
- Scope names: the scope model
- Error codes: every error code
- Architectural terms: "projection," "fact table," "metatype," "epistemic status"

For each:
- Does the name communicate clearly what the thing IS?
- Would a developer unfamiliar with the codebase understand it?
- Is it consistent with industry conventions?
- Could it be confused with something else?

Specific concerns to evaluate:
- "type_node" vs "TypeNode" vs "entity_type" — is the naming consistent?
- "projection" — is this the right term? (event sourcing uses it, but it's not universal)
- "metatype" — is this clear or obscure?
- "capture_thought" — does this name communicate what the tool does?
- "epistemic status" — will AI agent developers understand this term?

## Step 4: Conceptual Integrity

This specification was produced by 14 independent agents across 5 waves. Find where the seams show:

- Does the specification feel like it was designed by ONE mind?
- Are the same problems solved the same way throughout?
- Is error handling consistent? (same pattern everywhere, or ad-hoc per section?)
- Is the authorization model applied uniformly? (same scope-checking pattern for all tools?)
- Are the data model conventions consistent? (naming, types, constraints)
- Are there conflicting philosophies embedded in different parts?

## Step 5: Longevity

- Will this design age well?
- What assumptions could become invalid?
  - MCP protocol (2025-11-25) — will it change significantly?
  - Supabase Edge Functions — will the platform persist?
  - pgvector — will it remain the vector search standard for PostgreSQL?
  - OpenAI text-embedding-3-small — will the model be deprecated?
- Is the design coupled to specific technologies that may not exist in 10 years?
- Does the tech-agnosticism principle (Invariant 7) actually hold in practice?

## Step 6: The Verdict

Grade each dimension:

| Dimension | Grade | Justification |
|-----------|-------|---------------|
| Abstraction Quality | EXEMPLARY / SOLID / ADEQUATE / DEFICIENT | ... |
| Naming Quality | EXEMPLARY / SOLID / ADEQUATE / DEFICIENT | ... |
| Conceptual Integrity | EXEMPLARY / SOLID / ADEQUATE / DEFICIENT | ... |
| Longevity | EXEMPLARY / SOLID / ADEQUATE / DEFICIENT | ... |
| Overall Elegance | EXEMPLARY / SOLID / ADEQUATE / DEFICIENT | ... |

For anything below EXEMPLARY: provide specific, actionable improvements. Not "make it better" — exact changes with exact reasoning.

## Rules

- "It's fine" is not a review. Everything can be improved. Find how.
- Be specific. "The naming could be better" is useless. "'NodeRelation' should be 'Edge' because it describes a directed relationship in a graph, and 'edge' is the universally understood term" is useful.
- Compare against the best systems you know. Would Linus Torvalds approve of this design? Would the IETF accept this as an RFC? Would it survive a decade of use?
- Spend as many tokens as needed.

## Output

Write the complete elegance review to:
`torvalds-output/wave4/4A-elegance.md`

Start with:
```
# Wave 4A — Elegance Review
## Reviewer: The Elegance Critic
## Input: Unified Specification (wave3/unified-spec.md)
## Date: {today}
```

When the file is written, output: **AGENT 4A COMPLETE**
