# 03 — The project knowledge base: a curated wiki the agent reads

> The memory architecture. This is where the Karpathy "LLM wiki" idea becomes concrete: instead of an opaque vector store, the institution's project history becomes a **versioned, human-readable, LLM-legible knowledge base** that both people and agents can navigate — with a structured database underneath for arithmetic.

## 1. Four layers, one knowledge base

| Layer | Form | Serves | Written by |
|---|---|---|---|
| **L1 Structured records** | `projects` tables (SQLite/Postgres): attribute schema of `plan/02` §2.3 | Filtering, similarity math, unit-cost statistics, backtesting | Extraction pipeline + human validation |
| **L2 Project dossiers** | One markdown page per project, git-versioned | Agent (and human) reading of individual comparables; qualitative adjustment reasoning | LLM from L1 + parsed docs; template-driven |
| **L3 Index & synthesis pages** | Markdown: per-asset-class pages ("Compressor stations — cost behaviour"), cross-cutting pages ("Winter construction: observed premiums and failure modes", "Brownfield surprise catalogue", "Estimate growth patterns by class") | Agent priors and checklists; onboarding humans; the distilled *wisdom* layer | LLM aggregation over L1+L2, human-reviewed — the highest-curation layer |
| **L4 Raw store + embeddings** | Parsed artifacts (page-anchored markdown/tables) with hybrid search (BM25 + dense) | Fallback: quote-level evidence, questions the curation didn't anticipate | Pipeline |

The agent's default reading order is L3 → L1 (query) → L2 (read top-k dossiers) → L4 (source-level detail). One refinement from the research (`research/01`, the memory-benchmark evidence): don't treat L4 as a rarely-touched fallback — **co-index L3/L2 pages and raw L4 chunks in a single hybrid search store**, so retrieval can return a synthesis page and its underlying source snippets together; benchmarked memory systems that searched the union of distilled artifacts *and* raw history beat artifacts-alone configurations decisively. Navigation (PageIndex-style table-of-contents reading, `research/02`) remains the default for reasoning; the union index is what search-shaped queries hit. Humans can read every layer directly — that is a feature, not a side effect: the KB is independently valuable to the estimating group even with no agent attached.

## 2. Why this instead of alternatives

**vs. pure RAG over raw documents**
- Pro-RAG: near-zero curation cost; no extraction-error risk baked in.
- Contra: cannot answer aggregate/statistical questions; retrieval quality invisible; no explicit handling of missing data; citations are chunks, not vetted facts; era-inconsistent numbers get compared raw. At n=200 the curation cost that RAG avoids is *small*, so RAG's one advantage is weak here. → RAG kept, but demoted to L4 fallback.

**vs. knowledge graph / GraphRAG**
- Pro-KG: explicit entities/relations; multi-hop questions; impressive on huge heterogeneous corpora.
- Contra: heavy build/maintenance; the "graph" here is shallow (project → attributes → documents; few meaningful cross-project *edges* beyond similarity, which we compute anyway); a relational schema + markdown pages captures the same structure at a fraction of the complexity, and a 1-3 person team must maintain it. → Full GraphRAG rejected for v1; the L1 schema *is* the pragmatic graph. Revisit only if multi-hop questions ("which contractors were involved in projects that overran due to X?") become common — and even then, columns/tags in L1 likely suffice.

**vs. fine-tuning a model on the corpus**
- Contra on every axis at n=200: no auditable provenance, stale on update, small-data overfitting, cost. Rejected without much regret. (Research thread 3/7 checks whether any published result contradicts this.)

**vs. agent-written episodic memory (mem0/MemGPT-style, accumulated at runtime)**
- That pattern targets *conversation* memory. Our memory is *institutional* and should be built by a controlled pipeline with review — not accumulated as a side effect of chats. Runtime learning enters instead through the feedback loop in `plan/05` (new estimates and their eventual actuals become new KB entries through the same reviewed pipeline).

## 3. Dossier design (L2)

Template (target 1-2 pages; consistency beats prose flair — agents navigate templates well):

```markdown
# P-2019-041 — Rosetown compressor station brownfield upgrade
**Facts:** asset: compressor station · type: brownfield upgrade · capacity: +1,500 HP ·
in-service: 2020 · season: summer civil / winter tie-in · region: SW ·
final actual: $14.2M raw / $17.1M @2026$ · estimate class at sanction: 3 · growth: +18%

## Scope
...3-6 bullets, inclusions/exclusions...
## Execution story
...site conditions, season, what happened, delays with causes...
## Cost shape
...normalized table by bucket; unit costs; what drove the big buckets...
## Lessons & warnings
...distilled lessons-learned, tagged (winter-work, brownfield-unknowns, vendor)...
## Data quality
completeness: 0.9 · missing: original estimate breakdown · sources: [DBM p.3][cost report p.1-4]...
```

Every dossier fact carries a provenance link (§`plan/02` §3). Dossiers are regenerated from L1 + sources when inputs change; git history preserves every prior version.

## 4. Index & synthesis pages (L3) — the highest-value curation

These pages are the difference between "a search tool" and "an experienced estimator's head":

- **Asset-class pages**: population statistics (n, unit-cost distributions by era/site-type with the *spread* shown, not just means), what drives cost in this class, which attributes matter for similarity, known regime breaks ("post-2021 compression costs shifted due to X").
- **Cross-cutting pattern pages**: winter construction premium (observed, with the project list behind it); brownfield adder distributions; weather-delay base rates; estimate-growth-by-class (the in-house reference-class-forecasting table).
- **Navigation index**: the wiki homepage the agent loads first — what pages exist, what the schema is, how to query L1, methodological rules ("always use normalized costs", "never average fewer than 5 comparables without flagging").

L3 pages must show their derivation (generated from L1 queries whose SQL is embedded in the page) so they can be *regenerated* when the corpus grows — synthesis pages that silently go stale are worse than none.

## 5. Access interfaces for the agent

- `query_projects(sql | structured filter)` → L1 rows (the agent composes filters, gets tables back).
- `find_comparables(new_project_attrs, k, filters)` → ranked candidates with similarity breakdown (`plan/04` §2).
- `read_page(path)` → dossier or index page.
- `search_raw(query, project_ids?)` → L4 hybrid search, page-anchored snippets.
- `cost_stats(filter, metric)` → distributions/quantiles of normalized unit costs (so statistics are computed by code, not by LLM arithmetic).

Tool-shaped, deliberately: the agent *navigates* the wiki like Claude Code navigates a repo — read the index, query, open pages — rather than receiving one stuffed context. Small corpus + strong model means a long-context "read many dossiers" step is affordable where needed.

## 6. Versioning & governance

- The whole KB (dossiers, index pages, schema, mapping tables) lives in a git repo; L1 database is rebuilt from versioned inputs. Every agent answer records the KB commit hash it read — estimates are reproducible.
- Write access: pipeline + reviewed PRs only. The agent proposes KB edits (e.g., "this lessons-learned tag looks wrong") as suggestions, never direct writes.
- New completed projects enter via the `plan/02` pipeline; L3 pages regenerate; diffs are human-reviewed like code.
