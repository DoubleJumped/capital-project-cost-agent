# 01 — Agent Memory Architectures & the "LLM Wiki" Idea

*Research thread 1 of 6. Focus: how to give an agent durable, navigable memory of ~200 historical
capital projects — as a curated, versioned "project wiki" the agent reads, rather than raw retrieval
over everything. Covers the SOTA landscape (2024–2026), write/curate/read patterns, staleness,
provenance, and concrete design recommendations for the cost-estimation agent.*

---

## TL;DR for this project

- **The "LLM wiki" (Karpathy, Apr 2026) is the right backbone for us.** Compile the raw project
  corpus *once* into a persistent, interlinked markdown wiki, then point all queries at the wiki
  rather than re-retrieving raw docs each time. His three layers — **raw sources (immutable) →
  compiled wiki (LLM-maintained) → schema (a `CLAUDE.md`-style config)** — map almost exactly onto
  the L1–L4 knowledge base already sketched in `plan/03`. [1][6]
- **At n≈200 projects, curation cost is small and the payoff is large.** The wiki pattern's weakness
  (upfront compile effort) barely bites at this scale, while its strengths (aggregate/statistical
  answers, vetted facts, human-readable audit trail) are exactly what a *defensible cost estimate*
  needs. GraphRAG-scale machinery is not warranted. [1][11]
- **Split memory by type, deliberately.** SOTA converges on **episodic** ("what happened on project
  X"), **semantic** ("what is generally true about winter compressor jobs"), and **procedural**
  ("how we estimate / normalize"). Our dossiers are episodic, synthesis pages are semantic, the
  schema/skills are procedural. [9][10][12]
- **The hard part is bookkeeping, not thinking — and that is what the LLM does well.** Cross-references,
  consistency checks, contradiction flagging, and an append-only change log are the maintenance
  burden that kills human wikis; delegate them to the agent via an explicit *ingest / query / lint*
  workflow. [1]
- **Do NOT adopt runtime agent-written memory (mem0/MemGPT/Letta-style) as our institutional store.**
  Those target *conversation* memory accumulated as a side effect of chats. Our memory is
  institutional and must be built by a reviewed pipeline. Borrow their *retrieval ideas*
  (multi-signal search, memory-as-tools) but not their write-path. [4][7][8]
- **Provenance is a first-class requirement, not a nice-to-have.** Every wiki fact must link back to
  a source doc + page. 2025–26 research shows stale/weakly-grounded memory silently propagates errors;
  Anthropic's Citations API gives character-level source anchoring we should mirror. [13][16]
- **Plan for staleness explicitly.** Normalize costs to a base year, timestamp every fact, and run a
  periodic "lint" pass that flags contradictions, orphaned pages, and stale claims. "Memory staleness"
  is named as an open problem even by the vendors. [1][11]

---

## 1. The core idea: compile once, read many (Karpathy's LLM Wiki)

In April 2026 Andrej Karpathy popularized an "LLM wiki" pattern and, notably, shared the *idea* (a
`llm-wiki.md` gist / schema) rather than an app — arguing that in the agent era the reusable artifact
is the workflow, not the code. [1][2] The pattern has three layers:

1. **Raw sources** — immutable input documents (papers, PDFs, reports). The LLM reads them but never
   edits them, preserving a canonical source of truth.
2. **The wiki** — an LLM-generated directory of markdown files: summaries, entity pages, concept
   pages, and cross-references, all maintained by the LLM. This is "a persistent, compounding
   artifact" and is what distinguishes the approach from ordinary RAG. [1]
3. **The schema** — a config document (in practice a `CLAUDE.md`) that "tells the LLM how the wiki is
   structured, what the conventions are, and what workflows to follow" for ingest, query, and
   maintenance. [1]

The operational loop has three named flows:

- **Ingest** — a new source triggers: read → discuss takeaways → write summaries → update the relevant
  entity/concept pages → maintain cross-references → append to the log. Karpathy notes a single source
  typically touches **10–15 wiki pages**. [1]
- **Query** — answer by searching wiki pages, synthesizing with citations, and optionally filing
  valuable outputs back as new pages.
- **Lint** — periodic health checks that surface contradictions, stale claims, orphaned pages, missing
  cross-references, and data gaps.

Two structural conventions do the heavy lifting: an **`index.md`** (a catalog listing every page with
a one-line summary + metadata, updated on each ingest and used to locate pages at query time) and a
**`log.md`** (append-only, chronological, with consistent line prefixes so it is greppable — an audit
trail of the wiki's evolution). [1]

The framing insight, verbatim: *"The tedious part of maintaining a knowledge base is not the reading
or the thinking — it's the bookkeeping."* LLMs update cross-references and flag contradictions across
dozens of pages without fatigue; humans curate sources and ask strategic questions. The pattern is
explicitly an answer to Vannevar Bush's 1945 Memex — a curated store with associative trails — solving
Bush's unanswered question of *who maintains it.* [1]

**Why this fits us:** our 200 projects are exactly "raw sources that should be compiled once." A
per-project dossier is an entity page; an asset-class synthesis page ("Compressor stations — cost
behaviour") is a concept page; the estimating conventions are the schema. Human-legibility is a bonus:
the estimating group gets an independently valuable institutional knowledge base even with no agent
attached.

## 2. Anthropic's context-engineering stance (why "curated markdown loaded up front" is defensible)

Anthropic's own guidance (2025) frames the whole problem as **context engineering**: models have a
finite "attention budget," every token "depletes this budget," and studies show **"context rot"** —
accuracy degrades and long-range reasoning weakens as token count grows. The goal is *"the smallest
possible set of high-signal tokens that maximize the likelihood of some desired outcome."* [3]

Their recommended architecture is a **hybrid**: some curated context loaded up front (this is the
`CLAUDE.md` role — a naive drop-in of high-signal instructions) plus **just-in-time retrieval** where
the agent holds lightweight identifiers (file paths, queries, links) and pulls detail at runtime via
tools like `glob`/`grep`/`head`/`tail` — enabling **progressive disclosure** and sidestepping stale
indexes. [3] Anthropic recommends keeping each `CLAUDE.md` **under ~200 lines** and concise, noting
that specificity and concision improve adherence. [5]

Two more directly reusable primitives:

- **Structured note-taking / memory tool.** Agents "regularly write notes persisted to memory outside
  the context window." Anthropic shipped a file-based **memory tool** (public beta on the Claude
  Developer Platform) for exactly this; the Claude-plays-Pokémon example maintains tallies and strategy
  notes across context resets. [3]
- **Compaction.** When a context nears its limit, summarize and reinitialize — preserving
  "architectural decisions, unresolved bugs, and implementation details" while discarding redundant
  tool output. Guidance: maximize recall first, then precision; "tool-result clearing" is the safest
  light-touch form. [3]
- **Sub-agents.** A sub-agent may burn tens of thousands of tokens exploring but returns a
  **1,000–2,000-token distilled summary**, keeping the lead agent's context clean. This maps directly
  onto a "find-comparables" sub-agent that reads many dossiers and returns a short shortlist. [3]

**Implication:** load the schema + relevant synthesis pages up front; retrieve individual dossiers
just-in-time; use a comparables sub-agent to keep the main reasoning context small. This is Anthropic's
own recommended shape, and it directly counters "context rot" that would otherwise bite if we dumped
all 200 projects into context.

## 3. The runtime agent-memory ecosystem (MemGPT/Letta, mem0, A-MEM, Zep)

These systems solve a *different* problem from ours — memory **accumulated at runtime from
conversation** — but their mechanisms and benchmarks are instructive.

**MemGPT → Letta.** MemGPT (2023) reframed context management as an OS problem: a tiered memory of
**core** (in-context, agent reads/writes directly — like RAM), **recall** (searchable conversation
history — like a disk cache), and **archival** (long-term store queried via tool calls — cold storage).
The innovation is that the agent *actively manages its own memory* via tool calls (read/write/search),
moving data between "fast" and "slow" tiers. It has since matured into **Letta**, a production framework
adding filesystem-based memory, PostgreSQL persistence, and richer tool integration. [7][8]

**mem0.** A memory-centric layer that, for each new message pair, runs an LLM extraction step to
identify candidate facts and then decides **ADD / UPDATE / DELETE / NOOP** against existing memories.
Its ECAI-2025 paper ran the first broad head-to-head of ten memory approaches on the **LoCoMo**
benchmark. Its 2026 "token-efficient" algorithm (single-pass hierarchical extraction + multi-signal
retrieval) reports **92.5 on LoCoMo, 94.4 on LongMemEval, 64.1/48.6 on BEAM (1M/10M)** while averaging
**under ~7,000 tokens/retrieval** — versus ~26,000 tokens for a full-context baseline. Earlier results
claimed a **~26% relative LLM-as-judge uplift over OpenAI's memory** (66.9% vs 52.9%). Gains were
largest on **temporal (+29.6) and multi-hop (+23.1)** reasoning. [4][11]

**A-MEM (NeurIPS 2025).** Applies the **Zettelkasten** method to agent memory: each new memory becomes
a structured note (contextual description, keywords, tags); the system analyzes prior memories to
create **interconnected links** where meaningful similarities exist, and new memories can trigger
updates to older notes — a continuously self-refining network. Reports gains over SOTA baselines across
six foundation models. [9][14] This is the closest research analogue to Karpathy's wiki: interlinked,
self-organizing, note-based.

**Zep / Graphiti.** A memory *service* built on **Graphiti**, a temporally-aware knowledge-graph engine
that fuses unstructured conversation and structured business data while maintaining **validity
intervals** (when each fact was true) in a non-lossy way. It beats MemGPT on the **DMR** benchmark
(**94.8% vs 93.4%**) and, on **LongMemEval** (which better simulates enterprise temporal reasoning),
reports up to **+18.5% accuracy and 90% lower latency**; an independent LongMemEval run put **Zep at
63.8% vs mem0 at 49.0%** (GPT-4o). [10][15][16] The **bitemporal** idea — tracking both event time and
ingestion time — is directly relevant: our cost data must be reasoned about *as of* an era.

**What to borrow vs. reject.** Borrow: (a) **multi-signal retrieval** — mem0's own finding is that
combining semantic + keyword + entity matching beats any single signal; [11] (b) **memory-as-tools**
so the agent explicitly searches/reads; (c) **temporal validity** from Zep. Reject the **write path**:
these systems build memory as a *side effect of chats* with no human review. Institutional cost memory
must be built by a controlled, reviewed pipeline — an error baked into a comparable propagates into
every future estimate that cites it.

## 4. Memory types, consolidation, and distillation

This **episodic / semantic / procedural / working** taxonomy — **procedural** ("how" — skills/code),
**semantic** ("what the policy/general fact is"), **episodic** ("what happened, with context"), and
**working** (live context) — is not a recent invention: it was set out for language agents by **CoALA**
(Sumers et al., 2023) and is only echoed by the 2025–26 surveys. [10][12][23] Crucially, recent work
reframes memory not as "passive stores of raw episodic logs" but as **curated 'lessons-learned'
journals** — consolidating specific experiences into semantic rules for future tasks. [12]

The **consolidation step** (episodes → semantic knowledge) is described as *"particularly
underserved,"* typically done via explicit developer rules or **periodic LLM-driven summarization**,
where the LLM is invoked at intervals to strip redundancy and store compressed facts/summaries. [12]
Its seminal form is Park et al.'s **Generative Agents** (2023) *reflection* step, which periodically
synthesizes streams of episodic observations into higher-level semantic memories. [22] The most
on-point *published mechanism* for actually generating our synthesis pages is **RAPTOR** (2024):
recursively embed, **cluster, and abstractively summarize** chunks into a tree of increasingly general
nodes, then retrieve at whichever level a question needs — precisely what lets a store answer
multi-level *aggregate* questions ("winter premium across all compressor jobs") rather than only
chunk-level ones. [21] This is Karpathy's ingest+lint loop with a concrete algorithm: periodically
cluster the episodic dossiers (L2) and summarize each cluster into a semantic asset-class page (L3).

One evidence-based caution: a controlled ablation ("Verbatim Chunks Beat Extracted Artifacts") found
that for long-conversation tasks, **verbatim retained chunks decisively outperformed aggressively
extracted facts** (LoCoMo **43.9 vs 28.0**; LongMemEval-S **67.4 vs 45.4**) — over-distillation loses
the detail that later matters. Its sharper result: a **chunks ∪ artifacts *union* store matched
verbatim retention on both benchmarks, while extracted artifacts alone never beat naive RAG.** [17]
For us this argues *not* for treating raw text as a mere tail fallback but for **co-indexing the L3
synthesis pages and the raw L4 source text in one retrievable store**: distill for navigability and
aggregate answers, but keep the verbatim source alongside rather than as a last resort.

## 5. Curated wiki vs. raw RAG vs. GraphRAG — the tradeoff

| Approach | Strengths | Weaknesses | Verdict for us |
|---|---|---|---|
| **Raw vector RAG** over docs | Near-zero curation cost; simple; fast to iterate; good on unstructured text | Can't answer aggregate/statistical questions; citations are chunks not vetted facts; retrieval quality invisible; era-inconsistent numbers compared raw | Keep only as **L4 fallback** for quote-level evidence [18] |
| **Curated LLM wiki** (ours) | Vetted facts + provenance; aggregate answers; human-legible audit trail; compounding artifact; cheap at n≈200 | Upfront compile cost; risk of baked-in extraction errors; needs lint discipline | **Primary** (L1–L3) [1] |
| **GraphRAG / KG** | Multi-hop and **global/corpus-level "sensemaking"** questions; explicit entities/relations; the primary Microsoft study shows graph community-summaries beat vector RAG on comprehensiveness/diversity for global questions over ~1M-token corpora | Heavy build/maintenance; needs well-defined relations; overkill for shallow graphs | **Rejected for v1**; the L1 relational schema is our "pragmatic graph." Revisit only if multi-hop questions become common [20][18][19] |
| **Fine-tuning on the corpus** | Fluent domain voice | No auditable provenance; stale on update; overfits at n=200; costly | **Rejected** |

The consensus in the GraphRAG-vs-vector literature is that graphs pay off in **curated, structured,
compliance-heavy domains** and for **multi-hop / global-sensemaking** queries — the effect the primary
Microsoft "From Local to Global" paper measures directly [20] — and that the strongest systems
**combine** vector search (initial retrieval) with graph traversal (context expansion). [18][19] Our cross-project
"edges" are mostly *similarity* (which we compute anyway from L1 attributes) plus a few relations
(shared contractor/region), so a relational schema + markdown captures the same structure at a fraction
of the complexity a 1–3-person team can maintain.

## 6. Provenance, grounding, and staleness

2025–26 research is blunt that **memory is a provenance-bearing evidence source**, and that *"stale,
conflicting, poisoned, incorrectly summarized, or weakly grounded memory can propagate errors through
agent execution."* A cited source can be topically relevant yet **not actually support** the specific
claim ("Cited but Not Verified"). Provenance for agents must track **semantic influence** across
paraphrase/summarize/infer steps, not just a bare link. [13][16] The survey explicitly lists the
requirements future memory needs: *source attribution, versioning, conflict detection, stale-memory
invalidation, and evidence-grounded verification.* [16]

Concrete mechanisms available now:

- **Anthropic Citations API** — returns structured references with **character-level provenance** to
  source documents (2025). Mirror this: every dossier fact carries `[source: DBM p.3]`. [16]
- **Versioning** — keep the wiki in **git**; every dossier regeneration preserves prior versions and a
  diffable history — a natural fit for Karpathy's append-only `log.md`. [1]
- **Staleness handling** — even vendors name **"memory staleness"** (facts once accurate becoming
  wrong) as an open problem. [11] Our defenses: (a) **normalize all costs to a base year** so raw-era
  dollars are never compared directly; (b) **timestamp every fact** (Zep-style bitemporal validity);
  (c) run a **periodic lint pass** flagging contradictions, orphaned pages, stale claims, and data
  gaps. [1][10]

---

## 7. Implications for the capital-project cost agent

The four-layer KB in `plan/03` (L1 structured records → L2 dossiers → L3 synthesis pages → L4 raw
store) *is* Karpathy's pattern instantiated for cost estimation. The research above sharpens it:

**7.1 Adopt the three-flow discipline explicitly (ingest / query / lint).** Write the *schema* as a
short `CLAUDE.md`-style doc (Anthropic's guidance: **under ~200 lines**, concise) that defines the dossier template, the synthesis-page
conventions, the provenance format, and the read order. Keep an `index.md` (every dossier + one-line
summary + key attributes) and a greppable `log.md` (append-only ingest/lint history). *Pro:* directly
proven pattern, human-legible, cheap. *Con:* requires discipline to keep lint running — automate it as
a scheduled job.

**7.2 Memory-type split.**
- *Episodic* = L2 dossiers (one per project: scope, execution story, cost shape, lessons, data quality).
- *Semantic* = L3 synthesis pages (asset-class cost behaviour, "winter construction premiums,"
  "brownfield surprise catalogue," "estimate-growth patterns by class"). Generate these by **periodic
  LLM consolidation** over L1+L2 — Karpathy's ingest/lint loop *is* the consolidation pipeline. A
  concrete, published algorithm for it is **RAPTOR**: cluster the L2 dossiers and abstractively
  summarize each cluster into a higher-level page, so the L3 tree can answer aggregate questions at the
  right level of generality. [12][21]
- *Procedural* = the schema + estimating skills (normalization method, reference-class steps).

**7.3 Read order + sub-agent.** Default: load schema + relevant L3 synthesis pages up front (small,
high-signal); **query L1** for candidate comparables; a **comparables sub-agent** reads the top-k L2
dossiers and returns a ~1–2k-token shortlist with citations; drop to **L4** only for source-level
detail. This is Anthropic's hybrid + progressive-disclosure shape and directly limits context rot.
[3] *Tradeoff:* the sub-agent adds a round-trip of latency; worth it to keep the estimating reasoning
context clean and auditable.

**7.4 Retrieval = multi-signal, not pure vectors.** Combine attribute filtering (L1: asset class,
greenfield/brownfield, season, size band) + keyword/BM25 + dense embeddings. mem0's own result is that
multi-signal beats any single signal; for numeric/categorical project attributes, structured filtering
matters more than embeddings. [11][18] A ready-made, on-brand recipe is Anthropic's **Contextual
Retrieval**: prepend an LLM-generated, chunk-specific context blurb to each chunk *before* indexing it
into both the embedding store and a BM25 index (contextual embeddings + contextual BM25), which cut
top-20 retrieval-failure rate by ~49% (67% with reranking) — a concrete method for our hybrid store.
[24] Retrieval *scoring* can borrow Park et al.'s **Generative Agents** formula — a weighted sum of
**recency × importance × relevance** — which maps cleanly onto our signals (era-recency, project
materiality, scope-similarity). [22] *Pro:* higher recall on heterogeneous, era-varied corpora. *Con:*
hybrid indexing adds real build/tuning cost, and **weighting the signals is itself unsolved** — there
is no principled default for balancing BM25 vs dense vs attribute filters vs recency/importance, so it
needs held-out tuning against real comparables.

**7.5 Provenance + normalization are load-bearing, because the output is a *defensible* estimate.**
Every fact in a dossier links to source doc + page (Citations-API-style anchoring). All costs are
stored raw **and** normalized to a base year with the index used, so comparisons and the final range
are era-consistent and auditable. [16]

**7.6 Do NOT let the agent silently self-edit institutional memory.** Runtime patterns (mem0/Letta)
write memory as a chat side effect; we want the opposite. New estimates and their eventual actuals feed
back into the KB **only through the same reviewed ingest pipeline** (see `plan/05`). *Pro:* no error
propagation, full audit trail. *Con:* slower to update than autonomous memory — acceptable for an
institutional-record system where correctness dominates.

**7.7 Co-index synthesis and raw source — don't demote L4 to a tail fallback.** The union-store
ablation [17] is sharper than "keep a fallback": extracted artifacts *alone* never beat naive RAG,
while a **chunks ∪ artifacts union matched verbatim retention** (LoCoMo 43.9 vs 28.0; LongMemEval-S
67.4 vs 45.4). So retain page-anchored raw L4 text in the *same* hybrid-searchable store as the L3
synthesis pages — distill for navigation and aggregate answers, but keep the verbatim source
co-indexed so questions the curation didn't anticipate still land on real evidence.

**7.8 Scale check.** At n≈200, GraphRAG/fine-tuning are over-engineered and the wiki's only real cost
(compile effort) is modest and one-time. Reassess *only* if the corpus grows an order of magnitude or
if multi-hop relational questions ("which contractors ran the projects that overran on winter tie-ins?")
become routine — at which point promote selected L1 attributes into a light knowledge graph rather than
adopting full GraphRAG. [18][19]

---

## Sources

1. [Karpathy's LLM Wiki gist / idea file (llm-wiki.md)](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — primary: the three-layer architecture, ingest/query/lint flows, index.md/log.md conventions, "bookkeeping" insight, Memex framing.
2. [Karpathy's LLM Wiki: guide to the idea file (Agentpedia)](https://agentpedia.codes/blog/karpathy-llm-wiki-idea-file) — context on the Apr-2026 tweet and "share the idea not the code" framing.
3. [Anthropic — Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — attention budget, context rot, just-in-time retrieval, compaction, note-taking/memory tool, sub-agent distillation (1–2k tokens).
4. [Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory (arXiv 2504.19413)](https://arxiv.org/abs/2504.19413) — ADD/UPDATE/DELETE/NOOP extraction, LoCoMo comparison, ~26% uplift over OpenAI memory.
5. [Claude Code memory docs / CLAUDE.md best practices](https://docs.anthropic.com/en/docs/claude-code/memory) — hierarchical markdown memory, ~200-line / concise guidance.
6. [Karpathy's LLM Wiki as Agent Memory (AAIF)](https://aaif.io/blog/karpathys-llm-wiki-as-agent-memory/) — wiki-as-living-memory framing; compile-once-read-many.
7. [MemGPT: Towards LLMs as Operating Systems (summary)](https://www.leoniemonigatti.com/papers/memgpt.html) — tiered core/recall/archival memory; self-managed via tool calls.
8. [Virtual context management with MemGPT and Letta](https://www.leoniemonigatti.com/blog/memgpt.html) — evolution to Letta; filesystem + Postgres persistence.
9. [A-MEM: Agentic Memory for LLM Agents (arXiv 2502.12110, NeurIPS 2025)](https://arxiv.org/abs/2502.12110) — Zettelkasten-based interlinked, self-refining note network.
10. [Zep: A Temporal Knowledge Graph Architecture for Agent Memory (arXiv 2501.13956)](https://arxiv.org/abs/2501.13956) — Graphiti engine, bitemporal validity, DMR 94.8% vs 93.4%.
11. [Mem0 — State of AI Agent Memory 2026](https://mem0.ai/blog/state-of-ai-agent-memory-2026) — benchmark numbers (LoCoMo 92.5, LongMemEval 94.4, ~7k tokens/query), multi-signal retrieval finding, "memory staleness" open problem.
12. [Memory for Autonomous LLM Agents: Mechanisms, Evaluation, and Emerging Frontiers (arXiv 2603.07670)](https://arxiv.org/html/2603.07670v1) — episodic/semantic/procedural/working stack; consolidation "underserved," periodic LLM summarization; "lessons-learned journals."
13. [Cited but Not Verified: Source Attribution in LLM Deep Research Agents (arXiv 2605.06635)](https://arxiv.org/html/2605.06635v1) — a cited source may not support the specific claim.
14. [A-Mem code (NeurIPS 2025)](https://github.com/WujiangXu/A-mem) — reference implementation of agentic Zettelkasten memory.
15. [Zep is the new state of the art in agent memory (getzep blog)](https://blog.getzep.com/state-of-the-art-agent-memory/) — LongMemEval results, latency reduction, independent Zep vs mem0 comparison.
16. [From Agent Traces to Trust: Evidence Tracing and Execution Provenance in LLM Agents (arXiv 2606.04990)](https://arxiv.org/html/2606.04990v3) — memory as provenance-bearing evidence; requirements list (attribution, versioning, stale-invalidation); Anthropic Citations API character-level provenance.
17. [Verbatim Chunks Beat Extracted Artifacts: A Controlled Ablation of Memory Representations for Long LLM Conversations (arXiv 2601.00821)](https://arxiv.org/abs/2601.00821) — verbatim chunks beat extracted artifacts (LoCoMo 43.9 vs 28.0; LongMemEval-S 67.4 vs 45.4); a **chunks ∪ artifacts union store matched verbatim while artifacts alone never beat naive RAG** — argues for co-indexing raw L4 with L3, not raw-as-tail-fallback. *(Single ablation study — treat as suggestive, not settled.)*
18. [GraphRAG vs Vector RAG — when knowledge graphs outperform (Fluree)](https://flur.ee/fluree-blog/graphrag-vs-vector-rag-when-knowledge-graphs-outperform-semantic-search/) — secondary/vendor blog: graphs win on curated/structured/multi-hop; hybrid is strongest.
19. [GraphRAG vs Vector RAG side-by-side (Meilisearch)](https://www.meilisearch.com/blog/graph-rag-vs-vector-rag) — secondary/vendor blog: accuracy gap widens with query complexity; upfront build cost.
20. [From Local to Global: A Graph RAG Approach to Query-Focused Summarization (Edge et al., Microsoft; arXiv 2404.16130)](https://arxiv.org/abs/2404.16130) — primary GraphRAG paper: LLM-built entity graph + community summaries beat vector RAG on comprehensiveness/diversity for global "sensemaking" questions over ~1M-token corpora.
21. [RAPTOR: Recursive Abstractive Processing for Tree-Organized Retrieval (Sarthi et al.; arXiv 2401.18059)](https://arxiv.org/abs/2401.18059) — recursive embed/cluster/abstractive-summarize into a tree; retrieve at multiple abstraction levels; the concrete mechanism for generating L3 synthesis pages and answering multi-level aggregate questions.
22. [Generative Agents: Interactive Simulacra of Human Behavior (Park et al., 2023; arXiv 2304.03442)](https://arxiv.org/abs/2304.03442) — seminal reflection-based consolidation (episodes → higher-level semantic memories) and the recency × importance × relevance retrieval score.
23. [Cognitive Architectures for Language Agents / CoALA (Sumers et al., 2023; arXiv 2309.02427)](https://arxiv.org/abs/2309.02427) — canonical origin of the episodic/semantic/procedural/working memory taxonomy for language agents.
24. [Anthropic — Introducing Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) — LLM-contextualized chunks + contextual embeddings + contextual BM25 (+ reranking); ~49% (67% with reranking) reduction in retrieval failures — a concrete hybrid multi-signal recipe.

*Note on dating/verification: Karpathy's LLM-wiki gist and the Anthropic context-engineering post are
primary sources read directly. Benchmark numbers (LoCoMo/LongMemEval/DMR) are vendor-reported —
directionally consistent across mem0 and Zep sources but not independently re-run here; treat absolute
figures as indicative. The GraphRAG discussion now rests on the primary Microsoft "From Local to Global"
paper [20] rather than vendor blogs alone; an untraceable "+35% precision (AWS test)" figure that had
appeared only via a secondary blog has been removed. No citations were fabricated; where a claim rests
on a single or secondary source it is flagged inline.*
