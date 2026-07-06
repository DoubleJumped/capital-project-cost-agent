# RAG / GraphRAG & Structured Extraction over Heterogeneous Engineering Corpora

*Research thread 02 of the Capital Project Cost Agent research base. Context: a Canadian prairie gas
utility with ~200 historical capital projects, each backed by a messy, era-varying bundle of DBMs,
scope docs, lessons-learned reports, and final cost reports — some artifacts missing. The system must
let an engineer describe a new project in natural language and get a defensible, cited cost estimate
with ranges and named comparables. This thread covers how we get raw documents into an LLM-legible,
retrievable form.*

---

## TL;DR for this project

- **Our corpus is small (~200 projects, plausibly 1,000–5,000 documents).** This is *below* every
  practitioner threshold where GraphRAG earns its complexity. GraphRAG/knowledge-graph indexing is
  designed for 100k+ document, global-synthesis workloads; at our scale it adds cost, latency, and a
  brittle construction step for marginal gain [1][6][9]. **Recommendation: do not build a full
  Microsoft-style GraphRAG. Build hybrid retrieval + a curated per-project knowledge base.**
- **The highest-leverage retrieval upgrade is boring and cheap:** BM25 (keyword) + dense vectors +
  a cross-encoder reranker. The most rigorous measurement is Anthropic's Contextual Retrieval:
  hybrid contextual embeddings + BM25 cut top-20 retrieval failures by **49%**, and **67% with reranking**
  added [14]. Blog-reported enterprise cascades claim larger but less-controlled gains (**+17–48%** recall/accuracy
  over dense-only [4]); treat those as directional. Either way it needs no graph infrastructure. Do this first.
- **A curated-document / "table-of-contents navigation" layer (PageIndex-style) fits our Karpathy
  "LLM wiki" thesis better than vector RAG for the reasoning step** — the agent navigates a structured
  index of 200 project summaries and decides which to open, rather than similarity-matching chunks [7].
  With only 200 projects this is tractable and more explainable/citable.
- **For parsing, Docling is the strongest open default** (~94–98% table accuracy, structure-preserving,
  linear scaling), with a **vision-LLM fallback (Gemini 2.5 Pro / GPT-4.1 / Claude)** for scanned or
  drawing-heavy legacy artifacts that layout models mangle [3][8]. Reserve Azure Document Intelligence
  (~$10/1,000 pages, 99%+ text, 95%+ enterprise) if we want a managed, on-Azure path [10].
- **Extraction should be schema-first with source grounding.** Use LLM structured extraction (JSON
  schema / function calling; LangExtract-style with span offsets back to source) to populate a fixed
  project schema — cost line items, station size, greenfield/brownfield, season, weather delays [2][5].
- **Missing/inconsistent data is the norm, not the exception.** Model every field as explicitly
  nullable with a provenance and confidence tag. Route low-confidence and null-critical extractions to a
  human validation queue — studies show humans confirm the LLM ~87% of the time on low-confidence cases,
  so review effort concentrates where it pays [12].
- **Net architecture:** parse → schema extraction with provenance → curated per-project "project card"
  (the LLM-legible unit) → hybrid retrieval + reranking over cards and their source chunks → agentic
  navigation for the estimate. Graph edges (only if ever) should be a thin, hand-designed layer over the
  200 project cards, not an auto-extracted entity graph.

---

## 1. The parsing problem: turning heterogeneous PDFs into structured text

Our corpus spans eras and formats: born-digital PDFs, scanned reports, cost tables in Excel-exported
PDFs, and possibly CAD-derived station drawings. Parsing quality caps everything downstream — a garbled
cost table poisons the estimate. The 2025 field has consolidated around three open/commercial options
plus vision-LLM parsing.

**Docling (IBM, open source).** Uses DocLayNet for layout analysis and TableFormer for table structure
recognition. In independent 2025 benchmarks it reached **~97.9% accuracy on complex tables**, ~94%+ on
most numerical/textual tables, 100% on dense paragraphs, with best-in-class structural preservation and
**linear scaling** (~6.3s for 1 page, ~65s for 50) [3][8]. Weaknesses: minor data omissions, occasional
formatting loss. It is the strongest open default for our cost reports.

**Unstructured.io.** A broad enterprise ingestion platform. Strong basic OCR and hierarchy detection —
**100% on simple tables but ~75% cell accuracy on complex tables**, and slow (51–141s for 1–50 pages) [3].
Its own benchmarks claim a leading overall table score (~0.844) [also vendor-reported, treat with
caution]. Good if we want one platform handling many formats; weaker on the complex merged-cell cost
tables that matter most to us.

**LlamaParse (LlamaIndex).** Fast and consistent (~6s regardless of size), excellent on simple docs,
but **fails on complex multi-column layouts and merged tables** with column misalignment [3]. In 2025 it
added GPT-4.1 and Gemini 2.5/3.1 Pro backends, automatic skew/orientation correction, and LLM/LVM/agentic
parse modes — so its ceiling now depends on which vision model you attach [8].

**Vision-LLM parsing (Gemini 2.5 Pro, GPT-4.1/4o, Claude, or libraries like Vision Parse).** Multimodal
LLMs are **the only models that can parse engineering drawings** — dimensions, tolerances, layouts — but
general models lack drafting-convention knowledge and misread projection/tolerance semantics [MechVQA,
businesswaretech benchmark]. Counterintuitively, benchmarks found **GPT-4o-mini beating full GPT-4o and
Gemini Flash occasionally beating Pro** on drawings, while **Gemini 2.5 Pro hit near-perfect extraction**
at higher latency/cost [also that benchmark]. Use vision-LLMs as a *fallback* for scans and drawings,
not the default for clean tables.

**Azure Document Intelligence (now Azure Content Understanding).** Managed layout model: **99%+ text/table
accuracy on supported types, ~95%+ on typical enterprise deployments, ~$10 per 1,000 pages** [10]. For a
SaskEnergy-like shop already on Azure, this is a credible managed path that avoids self-hosting Docling.
Forrester notes table extraction is *the* differentiator: AI parsers hit 90%+ vs ~60% for template OCR [10].

**Design stance:** default **Docling** (or Azure DI if we standardize on Azure), with a **vision-LLM
fallback** triggered when Docling confidence is low or the artifact is a scan/drawing. Always keep the
parsed markdown *and* page/coordinate anchors so downstream extraction can cite exact source locations.

## 2. Structured extraction: from parsed text to a project schema

The cost agent does not reason over prose; it reasons over *comparable structured attributes* (station
size, cost per line item, greenfield/brownfield, season, weather-delay days, complexity). So extraction —
not retrieval — is arguably the core value-creation step. The 2024–2026 pattern is **schema-first LLM
extraction with source grounding**.

- **Schema enforcement via function calling / JSON schema.** Modern LLM tooling (e.g. the `llm` library's
  schema support, OpenAI/Anthropic structured outputs) constrains generation to a typed spec, turning an
  article/PDF/screenshot into typed JSON reliably [2]. Define one canonical **project schema** and extract
  every project into it.
- **LangExtract (Google, open source)** is the reference pattern worth emulating: few-shot,
  example-driven extraction with **source grounding — every extracted value maps to its exact span in the
  source** — plus schema enforcement, long-document chunking with parallel multi-pass extraction, and
  multi-model support (Gemini/OpenAI/Ollama) [5]. Source grounding is the killer feature for us: an
  estimate that cites "$1.2M mainline steel, from Project X final cost report p.14" is *defensible*; a
  number with no provenance is not.
- **Schema optimization research (PARSE, 2025)** shows LLM-driven schema refinement improves entity
  extraction reliability — worth knowing if our first hand-designed schema underperforms, but likely
  overkill for a fixed, well-understood cost schema.

**Handling missing / inconsistent data (this is central for us).** Some projects lack a lessons-learned
report; older cost reports use different line-item taxonomies; "station size" may be recorded as MMSCFD,
inlet pressure, or physical footprint depending on era. Recommendations:

1. **Every schema field is explicitly nullable** and carries `{value, source_span, confidence,
   extractor}`. Never silently impute; a missing weather-delay field must read `null`, not `0`.
2. **Normalize taxonomies at extraction time** via a controlled vocabulary / mapping layer (map each era's
   cost line items onto a canonical WBS). This is where most of the messy-corpus work actually lives.
3. **Confidence-routed human validation.** Extract with confidence scoring (self-consistency: run
   extraction multiple times and score agreement; or multi-model cross-validation). High-agreement →
   auto-accept; middling → human queue; low/contradictory → mandatory expert review [12]. In systematic-
   review studies, **humans confirmed the LLM's low-confidence extraction ~87% of the time**, and
   two-tier (multi-model → expert) routing concentrates scarce engineer attention on genuinely ambiguous
   cases [12]. For 200 projects this is a one-time curation effort, entirely feasible.

## 3. Chunking: not the main event, but get it right

For the *source-evidence* retrieval layer (as opposed to the curated project cards), chunking still
matters for long DBMs and cost reports.

- **Semantic chunking** (split on topic-shift via embedding similarity) delivers up to **~70% accuracy
  lift over naive fixed-size chunking** on some benchmarks [chunking guide]. It respects table and section
  boundaries better than fixed windows.
- **Anthropic's Contextual Retrieval (Sept 2024):** prepend an LLM-generated blurb situating each chunk in
  its parent document before embedding. This preserves cross-chunk context (e.g. a cost line that only
  makes sense given the project it belongs to) at the cost of extra LLM calls at index time [chunking
  guide]. Very relevant for us — a bare "$430,000" chunk is useless without "…for Project Y horizontal
  directional drilling."
- **Late chunking** (embed the whole doc with a long-context embedder, then pool per-chunk) is cheaper and
  keeps long-distance dependencies but tends to sacrifice some relevance/completeness [Late Chunking paper].

**Stance:** chunk source docs *structurally* (by section/table), enrich with contextual-retrieval blurbs
naming the parent project. But recognize that for our use case the **curated project card (§5) is the
primary retrieval unit**, so chunking of raw source is a secondary, evidence-citation layer.

## 4. Retrieval: hybrid + rerank is the workhorse

The single most reliable, best-documented upgrade in 2024–2026 RAG is **hybrid retrieval + cross-encoder
reranking**, and it needs no graph.

- **Hybrid = BM25 (sparse/keyword) ∪ dense (vector), fused (RRF), then reranked by a cross-encoder**, top-k
  to the LLM [4]. Sparse catches exact terms (station IDs, part numbers, WBS codes) that embeddings miss;
  dense catches paraphrase/semantic scope similarity; the reranker fixes ordering.
- **Measured gains:** a production cascade (BM25 + FAISS + cross-encoder) reported **+48% accuracy**;
  a two-stage hybrid→neural-rerank hit **Recall@5 0.816 vs 0.695 for RRF-only (+17%), 0.644 BM25 (+27%),
  0.587 dense (+39%)**; on WANDS, tuned hybrid gave **+7.4% NDCG** over either method alone [4]. As one practitioner writeup
  puts it, *"a two-week reranking project saved them a six-month migration"* [13].
- **Metadata filtering is free precision.** Because we control extraction, we can hard-filter candidates by
  structured attributes *before* semantic ranking — e.g. only greenfield stations, only winter builds, only
  >X MMSCFD. This is often more valuable than any fancier retrieval, and directly serves the
  comparable-selection logic the estimate needs.

## 5. GraphRAG, knowledge graphs, and curated navigation — what our scale actually calls for

This is the crux decision for the thread. Three families of "beyond-plain-RAG" exist:

**(a) Microsoft GraphRAG** — LLM-extract an entity/relationship graph from the whole corpus, detect
communities, pre-summarize them, then answer global queries by aggregating community summaries.

- **When it wins:** multi-hop, reasoning-intensive, and corpus-level-synthesis queries ("what themes recur
  across all projects?"). In systematic evaluation, GraphRAG beat RAG on **comparison (60.6% vs 57.6%) and
  especially temporal (50.6% vs 30.7%)** queries, and **13.6% of MultiHop-RAG queries were answerable only
  by GraphRAG** [6]. Combining both improved accuracy **+6.4%** [6].
- **When it loses / the costs:** it is **worse on single-hop, factual, and null queries** (RAG 96.0% vs
  GraphRAG 80.1% on null queries) [6]. Construction is brutally expensive: in the same study GraphRAG
  variants took **5,560–7,700s to construct vs 135s for RAG** [6]. Quality is **highly sensitive to
  graph-construction LLM quality** (GPT-4o 75% vs GPT-4o-mini 71%) and to graph *completeness* — in one KG,
  only **65.8% of answer entities even existed in the graph**, capping accuracy [6]. Practitioner thresholds
  put GraphRAG's payoff at **100k+ documents with regular multi-hop/global needs** [1][9].
- **Cost trend:** the ecosystem has attacked the cost. **LazyGraphRAG (Microsoft; public preview,
  integrated into Azure Local + Microsoft Discovery as of mid-2025 — not yet GA)** defers all
  LLM summarization to query time, giving **indexing cost identical to vector RAG (0.1% of full GraphRAG)**
  and comparable global-query quality at **~700× lower query cost / 4% of GraphRAG's global query cost**
  [11]. **LightRAG (EMNLP 2025)** claims near-GraphRAG quality at ~2 orders of magnitude lower cost [1]. So
  if we ever wanted graph-style *global* reasoning, LazyGraphRAG is the cost-sane door.

**(b) Plain hybrid RAG** (§4) — cheap, robust, best for the factual/single-hop retrieval that dominates
"find me comparable projects and their cost lines."

**(c) Curated-document / reasoning-based navigation (PageIndex-style, "LLM wiki")** — build a hierarchical
table-of-contents index (a tree of nodes with titles, descriptions, metadata) and let the LLM **navigate
and reason about which sections to open**, rather than similarity-matching chunks [7]. This is *vectorless*
retrieval: no embedding DB, retrieval is LLM reasoning over structure. It claims strong results on
professional-document analysis and directly mirrors how a human estimator flips through past project files
[7]. **This is the closest existing technique to the Karpathy "LLM wiki / curated agent memory" thesis
that frames this whole project.**

**Our verdict for ~200 projects.** We are two-plus orders of magnitude below GraphRAG's break-even. The
right architecture is **(b) + (c)**: a **curated per-project "project card"** (a distilled, LLM-legible
1–2 page structured summary of each project — scope, size, greenfield/brownfield, season, weather delays,
normalized cost breakdown, lessons, links to source evidence) forming a navigable index the agent reasons
over, **backed by hybrid+rerank retrieval into raw source chunks for citation**. A full auto-extracted
entity knowledge graph is not justified: construction cost/latency, sensitivity to construction quality,
and graph incompleteness all bite, for gains our metadata-filtered hybrid retrieval already captures.

**If we ever add a graph,** make it **thin and hand-designed** — nodes = the 200 projects, edges =
deliberate relations (same station, same region, same contractor, brownfield-successor-of) with structured
attributes for typed traversal. That is a small, curated, high-precision graph, not an LLM-hallucinated
one, and it directly powers "find comparables" without GraphRAG's machinery.

## 6. Agentic retrieval: let the agent decide what to read next

Static "retrieve-once, stuff-context" RAG underperforms for the multi-step reasoning our estimate needs
(find candidates → compare attributes → pull cost detail on the top 3 → normalize → range). **Agentic RAG**
lets the agent plan retrieval: decompose the query, choose sources/tools, re-query, and decide when it has
enough [others]. Planning agents that decompose before retrieving hit **94.5% on HotpotQA / 89.7% on
2WikiMultiHop using comparable or fewer tokens** than static retrieval [others].

For us this maps cleanly: the agent (1) filters project cards by structured attributes, (2) reads the top-N
cards, (3) decides which raw cost reports to open for line-item detail, (4) reasons about
normalization/escalation, (5) cites evidence. The curated project-card index (§5) is exactly the
"LLM-legible memory" that makes agentic navigation cheap and explainable — the agent reads distilled cards,
not thousands of raw chunks.

## 7. Implications for the capital-project cost agent

**Recommended pipeline (opinionated):**

1. **Ingest & parse:** Docling default (or Azure Document Intelligence if we standardize on Azure), with a
   **vision-LLM fallback** (Gemini 2.5 Pro / GPT-4.1 / Claude) auto-triggered on scans, drawings, or
   low-confidence tables. Preserve markdown + page anchors. *Tradeoff:* self-hosted Docling = no per-page
   cost, more ops; Azure DI = ~$10/1k pages, managed, on-Azure compliance story.
2. **Extract to schema with provenance:** one canonical project schema; LangExtract-style source-grounded
   extraction; every field nullable with `{value, source_span, confidence, extractor}`; era-taxonomy
   normalization onto a canonical WBS. *Tradeoff:* schema-first is more upfront design work but yields
   comparable, citable attributes — the whole point of the system.
3. **Curate project cards (the "LLM wiki"):** distill each project into a compact structured card; this is
   the primary retrieval/reasoning unit and the artifact humans validate. *Tradeoff:* one-time curation
   effort for 200 projects vs. durable, explainable, low-token retrieval forever after.
4. **Retrieval:** hybrid (BM25 + dense) + cross-encoder rerank + **metadata pre-filtering** on structured
   attributes, over both cards and raw-source chunks. Skip GraphRAG. *Tradeoff:* forgo global-synthesis
   graph queries — acceptable, since our task is comparable-finding, not corpus-wide theme mining. If
   global synthesis is ever needed, add **LazyGraphRAG** (vector-RAG-level index cost) rather than full
   GraphRAG.
5. **Navigation & reasoning:** agentic retrieval over the card index (PageIndex-style reasoning where it
   helps explainability), deciding which raw reports to open for cost detail. *Tradeoff:* more LLM calls /
   latency per estimate vs. markedly better multi-step reasoning and a defensible audit trail.
6. **Human validation loop:** confidence-routed queue at extraction time (self-consistency + multi-model
   agreement; auto-accept high, expert-review low), and a lightweight "was this comparable/estimate right?"
   feedback capture at query time to improve cards over time [12].
7. **Optional thin graph:** hand-designed 200-node project graph with typed edges (same-station,
   same-region, brownfield-successor) to power comparable discovery — curated, not auto-extracted.

**What to explicitly *not* build:** full Microsoft GraphRAG entity-graph indexing (wrong scale), pure
vector-only RAG without reranking (leaves 15–48% accuracy on the table), and any pipeline that silently
imputes missing fields (destroys defensibility). 

**Key risks / unverified items to flag:** vendor self-benchmarks (Unstructured's 0.844 table score;
PageIndex's professional-doc accuracy claims) are vendor-reported and should be re-benchmarked on *our*
cost reports before commitment. The Docling 97.9% figure comes from a third-party benchmark on
sustainability reports, not gas-utility cost reports — validate on our own sample. The specific engineering-
drawing model rankings (GPT-4o-mini > GPT-4o) come from one benchmark and may not generalize.

---

## Sources

1. [GraphRAG vs. RAG: When Knowledge Graphs Earn Their Complexity](https://aloknecessary.github.io/blogs/graph-rag-vs-rag/) and search-synthesized practitioner consensus — thresholds for GraphRAG (100k+ docs), the +15–25% hybrid+rerank gain, LightRAG/LazyGraphRAG cost points, "two-week reranking vs six-month migration."
2. [Structured data extraction from unstructured content using LLM schemas — Simon Willison](https://simonwillison.net/2025/Feb/28/llm-schemas/) — schema/function-calling enforced structured output turning PDFs/screenshots into typed JSON.
3. [PDF Data Extraction Benchmark 2025: Docling vs Unstructured vs LlamaParse — Procycons](https://procycons.com/en/blogs/pdf-data-extraction-benchmark/) — table accuracy (Docling 97.9%/94%+, Unstructured 100% simple / 75% complex, LlamaParse fails complex), speed numbers, per-tool pros/cons and recommendations.
4. [Hybrid Retrieval and Reranking in RAG — Genzeon](https://www.genzeon.com/hybrid-retrieval-deranking-in-rag-recall-precision/) plus [Better RAG Accuracy with Hybrid BM25 + Dense — Medium](https://medium.com/@pbronck/better-rag-accuracy-with-hybrid-bm25-dense-vector-search-ea99d48cba93) — hybrid+cross-encoder gains (+17–48%, Recall@5 and NDCG figures), standard production pipeline.
5. [google/langextract (GitHub)](https://github.com/google/langextract) and [Extracting Structured Data with LangExtract — Towards Data Science](https://towardsdatascience.com/extracting-structured-data-with-langextract-a-deep-dive-into-llm-orchestrated-workflows/) — source grounding (span mapping), schema enforcement, long-doc multi-pass extraction, multi-model support.
6. [RAG vs. GraphRAG: A Systematic Evaluation and Key Insights (arXiv 2502.11371)](https://arxiv.org/html/2502.11371v3) — query-type wins/losses, construction cost (5,560–7,700s vs 135s), GraphRAG-only 13.6% of queries, +6.4% from combining, sensitivity to construction-LLM quality and graph incompleteness (65.8% entity coverage).
7. [PageIndex: Next-Generation Vectorless, Reasoning-based RAG](https://pageindex.ai/blog/pageindex-intro) and [PageIndex intro](https://www.buildfastwithai.com/blogs/vectorless-rag-pageindex-guide) — table-of-contents tree index, LLM reasoning-based navigation, the curated-doc alternative closest to the "LLM wiki" thesis.
8. [Best LLM Document Parsers 2025 — LlamaIndex](https://www.llamaindex.ai/insights/best-llm-document-parser-2025) and [parsemypdf (GitHub)](https://github.com/genieincodebottle/parsemypdf) — vision-LLM parsing options, LlamaParse 2025 model backends (GPT-4.1, Gemini 2.5/3.1 Pro), skew correction, parse modes.
9. [When to use Graphs in RAG: A Comprehensive Analysis (arXiv 2506.05690)](https://arxiv.org/html/2506.05690v3) — analysis of when graph retrieval helps vs. plain retrieval.
10. [Azure AI Document Intelligence](https://azure.microsoft.com/en-us/products/ai-foundry/tools/document-intelligence) and [pricing](https://azure.microsoft.com/en-us/pricing/details/document-intelligence/) — layout/table model, 99%+ text/table accuracy, ~$10/1,000 pages, Forrester 90% AI vs 60% template-OCR context.
11. [LazyGraphRAG sets a new standard for quality and cost — Microsoft Research](https://www.microsoft.com/en-us/research/blog/lazygraphrag-setting-a-new-standard-for-quality-and-cost/) — indexing cost = vector RAG (0.1% of full GraphRAG), ~700× lower global query cost, defers LLM summarization to query time. Status: public preview, integrated into Azure Local and Microsoft Discovery (per the blog's June 2025 editor's note) — not general availability.
12. [Large Language Models with Human-In-The-Loop Validation for Systematic Review Data Extraction (ResearchGate)](https://www.researchgate.net/publication/388316685_Large_Language_Models_with_Human-In-The-Loop_Validation_for_Systematic_Review_Data_Extraction) plus search-synthesized confidence-routing patterns — uncertainty sampling, self-consistency confidence scoring, ~87% human-confirms-LLM on low-confidence cases, two-tier multi-model→expert routing.

*Additional sources consulted (context, not primary claims):* [Late Chunking (arXiv 2409.04701)](https://arxiv.org/pdf/2409.04701); [Reconstructing Context: chunking strategies (arXiv 2504.19754)](https://arxiv.org/abs/2504.19754); [Agentic RAG survey (arXiv 2501.09136)](https://arxiv.org/html/2501.09136v4); [LlamaIndex PropertyGraphIndex](https://www.llamaindex.ai/blog/introducing-the-property-graph-index-a-powerful-new-way-to-build-knowledge-graphs-with-llms); [MechVQA: multimodal LLMs on mechanical drawings (arXiv 2605.30794)](https://arxiv.org/html/2605.30794v1); [We Benchmarked AI on Tables and Engineering Drawings — Businessware Tech](https://www.businesswaretech.com/blog/benchmarking-ai-on-tables-and-engineering-drawings-results-findings).
