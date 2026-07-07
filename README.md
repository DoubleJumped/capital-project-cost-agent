# Capital Project Cost Agent — Research Base & Architecture Plan

An engineer or PM describes a new capital project's scope in natural language. The agent reasons over **~200 historical capital projects** (DBMs, scope documents, lessons learned, final cost reports — heterogeneous formats, some gaps) to find the genuinely comparable ones, adjust their normalized actual costs for the differences — size, greenfield vs. brownfield, summer vs. winter construction, complexity, era — and return a **cost range with named comparables, explicit adjustments, risk flags, and citations back to source documents**.

This repo is the complete research base and end-to-end plan for building that system. It is written to be handed to a build team (or a future agent) and executed.

## Start here

**[`plan/00-executive-summary.md`](plan/00-executive-summary.md)** — the whole design in two pages.

## The plan (`plan/`)

| Doc | What it decides |
|---|---|
| [00 Executive summary](plan/00-executive-summary.md) | The design in three moves; why this shape; the path |
| [01 Problem frame](plan/01-problem-frame.md) | Users, the "is / is not" scope contract, data reality & hazards, success criteria |
| [02 Corpus pipeline](plan/02-corpus-pipeline.md) | Messy folders → parsed artifacts → schema-first LLM extraction → normalization (BCPI / Handy-Whitman) → human validation → dossiers. Provenance on every field |
| [03 Knowledge base](plan/03-knowledge-base.md) | The Karpathy-style "LLM wiki": structured DB (L1) + per-project dossiers (L2) + synthesis pages (L3) + raw hybrid search (L4), git-versioned; why not pure RAG / GraphRAG / fine-tuning |
| [04 Agent flow](plan/04-agent-flow.md) | Fixed six-stage estimation flow: intake → retrieve → reason → compute → challenge → estimate basis memo. LLM reasons, code computes |
| [05 Evaluation & learning](plan/05-evaluation-and-learning.md) | Leave-one-out backtesting with leakage discipline; conformal (jackknife+/Mondrian/ACI) range calibration; the reviewed learning loop |
| [06 Delivery roadmap](plan/06-delivery-roadmap.md) | Phases with exit criteria, reference stack, the 10-decision tradeoff ledger, risk register, team envelope |
| [07 Phase 0 playbook](plan/07-phase-0-playbook.md) | Day-one execution: access requests, inventory script + registry schema, the four gating measurements, parser bake-off, estimator interview guide, SEERai demo, reality-report template |

## The research (`research/`)

Seven deep-dive chapters, each web-researched, adversarially critiqued, and revised (2024–2026 SOTA, sources cited inline):

1. [Agent memory architectures & the LLM wiki idea](research/01-agent-memory-architectures.md) — Karpathy's wiki, MemGPT/Letta, mem0, episodic/semantic/procedural splits, why curation beats runtime memory here
2. [RAG/GraphRAG & extraction over heterogeneous engineering corpora](research/02-rag-over-heterogeneous-corpora.md) — parsing bake-offs, hybrid retrieval + reranking numbers, why GraphRAG isn't warranted at n≈200
3. [LLM + case-based reasoning for analogous estimating](research/03-llm-case-based-reasoning.md) — the CBR cycle as the system's spine; evidence LLMs must not be the regression engine; the hybrid "statistical baseline + constrained LLM adjustment" template
4. [Capital cost estimation practice](research/04-cost-estimation-practice.md) — AACE classes 1–5, reference class forecasting, escalation indices for prairie Canada, winter premiums, contingency methods
5. [Project similarity modeling](research/05-similarity-modeling.md) — Gower + text embeddings, hard-filter-then-rank, distance-weighted k-NN as operationalized RCF
6. [Uncertainty, calibration & evaluation](research/06-uncertainty-calibration-eval.md) — conformal prediction at small n, LLM overconfidence evidence, PICP/pinball metrics, leakage discipline
7. [Prior art landscape](research/07-prior-art-landscape.md) — SEERai/Galorath, nPlan, takeoff tools, utility case studies, what ML accuracy is realistic at n≈200, build-vs-buy

## The design in one paragraph

Curate, don't just index: distill the 200 project folders — via an LLM pipeline with human validation on the fields that matter — into a git-versioned project knowledge base (structured records for math, dossiers and synthesis pages for reasoning, raw search as evidence backstop). Run estimation as a fixed six-stage agentic workflow where the LLM elicits scope, curates comparables, argues adjustments, and writes the memo, while deterministic tools do all arithmetic and statistics. Prove it by leave-one-out backtesting against the organization's own history with conformal-calibrated ranges, and compound it by flowing every newly completed project back through the same reviewed pipeline. Full argument: [`plan/00`](plan/00-executive-summary.md).

---

*Built with a multi-agent research workflow (7 threads × research/critique/revise), then revised after an end-to-end review pass (2026-07-06): retrieval filter/caliper design tightened, an estimate-growth double-count removed from range assembly, a raw-RAG baseline added to test the core curation hypothesis, backtest compute mitigations added, and the Phase 0 playbook (`plan/07`) written. `HANDOFF.md` records working state and how to resume or extend.*
