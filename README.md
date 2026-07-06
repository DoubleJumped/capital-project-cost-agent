# Capital Project Cost Agent — Research & Architecture

Research base and end-to-end architecture plan for an **agentic cost-estimation assistant** built on a corpus of ~200 historical capital projects (DBMs, scope documents, lessons learned, final cost reports — heterogeneous formats, some gaps).

**The goal:** an engineer or PM describes a new project's scope in natural language, and the agent reasons over all past projects — similarity in scope, complexity, greenfield vs. brownfield, seasonal construction constraints, station size, weather delays — to produce a defensible cost estimate with ranges, comparables, and cited evidence.

## Repository layout

| Path | Contents |
|---|---|
| `research/` | Deep research on SOTA approaches — agent memory, RAG/GraphRAG, LLM + case-based reasoning, cost-estimation practice, similarity modeling, uncertainty |
| `plan/` | The end-to-end architecture and project plan, including tradeoff analyses |
| `HANDOFF.md` | Working state — what's done, what's in flight, what a new agent should do next |

## Status

🚧 In progress — research phase running. See `HANDOFF.md`.
