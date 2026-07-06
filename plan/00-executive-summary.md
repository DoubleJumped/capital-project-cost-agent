# 00 — Executive summary

## What we're building

An **estimating assistant with institutional memory**. An engineer or PM describes a new capital project in plain language; the agent finds the genuinely comparable projects among ~200 historicals, adjusts their normalized actual costs for the differences (size, greenfield/brownfield, construction season, complexity, era), cross-checks against population statistics, and returns a **cost range with named comparables, explicit adjustments, risk flags from lessons learned, and citations back to source documents** — an estimate basis memo, produced in minutes instead of days.

## The core idea in three moves

1. **Curate, don't just index** (`plan/02`, `plan/03`). The 200 project folders are distilled — by an LLM pipeline with human validation on the fields that matter — into a *project knowledge base*: a structured database (for math) plus a git-versioned wiki of per-project dossiers and cross-project synthesis pages (for reasoning). This is Karpathy's "LLM wiki" instinct applied to institutional memory: an LLM-legible, human-auditable knowledge layer, with raw-document search kept only as a fallback. At 200 projects, curation is affordable and beats web-scale RAG patterns on every axis that matters here: aggregate statistics, missing-data honesty, auditability.

2. **LLM reasons, code computes** (`plan/04`). The agent runs a fixed six-stage flow — intake → retrieve comparables → read & reason → compute → self-challenge → memo. The LLM elicits scope, curates the comparable set, argues the adjustments, and writes the memo; deterministic tools do every piece of arithmetic, similarity scoring, and statistics. No LLM mental math, no free-roaming chat methodology, no number without a source.

3. **Prove it on our own history** (`plan/05`). Leave-one-out backtesting over the historicals — with strict temporal and leakage discipline — measures point accuracy and range calibration against naive baselines *and against the organization's own past early estimates*. Ranges are calibrated from measured backtest error (conformal-style), never from LLM-verbalized confidence. New completed projects flow back through the same pipeline: the corpus, and the tool, compound.

## Why this shape (the short version of the tradeoff ledger, `plan/06` §3)

- **Not fine-tuning, not GraphRAG, not a vector-store-only RAG bot**: at n=200, a reviewed curation pass costs weeks and yields an asset that is auditable and independently valuable; the rejected options are each weaker on provenance, aggregates, or maintainability (full arguments in `plan/03` §2).
- **Not a pure ML cost model**: 200 projects, sliced by asset class, is small-data; published construction-ML results at this n support ensembles over structured features at best (`research/07`) — which is exactly our parametric *cross-check*, not the headline method. The analog method with transparent adjustment matches how estimators already think, which is the adoption battle half-won.
- **Not autonomous**: the output is a document a human owns. The tool's job is to make the best estimator-style argument the corpus supports, visibly.

## The path (full detail `plan/06`)

**Phase 0 (weeks):** inventory the real corpus — coverage, formats, completeness; kill assumptions early. **Phases 1-2 (~4-5 months to credibility):** one asset class end-to-end — pipeline, KB, agent, backtest — and put the accuracy chart in front of the estimating group. **Phase 3:** the rest of the corpus, calibration per class, hardening. **Phase 4:** operate; every completed project makes it smarter; expansions (schedule, risk registers, scope checking) only once the core is trusted.

## Reading order

- `plan/01` problem frame → `plan/02` corpus pipeline → `plan/03` knowledge base → `plan/04` agent flow → `plan/05` eval & learning → `plan/06` roadmap/stack/risks.
- `research/01-07` — the SOTA research base behind these choices (agent memory, RAG/extraction, LLM+CBR, cost-engineering practice, similarity, uncertainty, prior art), each with cited sources.
