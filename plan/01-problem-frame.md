# 01 — Problem frame, users, and data reality

This document pins down *what we are actually building and for whom*, before any architecture. Everything downstream (corpus pipeline, knowledge base design, agent flow, evaluation) traces back to decisions made here.

## 1. The problem in one paragraph

Engineers and project managers starting a new capital project (a new or modified station, a pipeline segment, a system upgrade) need an early, defensible cost picture long before a bottom-up estimate exists. Today that knowledge lives in ~200 historical projects' documents — DBMs, scope docs, lessons learned, final cost reports — and, critically, in the heads of a few senior estimators who "remember" the comparables. The goal is to turn that corpus into an **agentic estimating assistant**: the user describes the new project's scope in natural language; the agent finds and reasons over comparable historicals, adjusts for the differences (size, site conditions, greenfield vs. brownfield, season, complexity, era), and returns a cost range with named comparables, explicit adjustments, and citations back to source documents.

## 2. What the agent is — and is not

**Is:**
- A *reasoning layer over institutional memory*: retrieval of comparable projects + transparent analogous/parametric adjustment + honest ranges.
- A *Class 5/4 estimating aid* (AACE terminology — see `research/04-cost-estimation-practice.md`): screening and feasibility-grade estimates, concept ranges, sanity checks on other estimates.
- An *explainer*: every number traceable to named projects and documents. "Your project resembles P-2019-041 and P-2021-007; here's the normalized unit cost from each and the adjustments applied, and here's the lessons-learned warning about winter tie-ins from P-2021-007."

**Is not (at least initially):**
- A replacement for Class 3-1 bottom-up estimates produced by estimators for sanctioning. It feeds and challenges those; it does not sign them.
- A general project-management copilot. Scope discipline matters for trust: cost estimation with evidence, plus the lessons-learned surfacing that naturally rides along with the comparables.
- Autonomous. A human reads and owns every output; the deliverable is a *estimate basis memo*, not a number in a database.

This framing matters for adoption: estimators will (rightly) attack a black box that emits numbers. They will use a tool that finds comparables faster than they can and shows its arithmetic.

## 3. Users and their asks

| User | Moment of use | What they need |
|---|---|---|
| Project engineer | Writing a DBM / early scoping | "What did similar stations cost? What am I forgetting to scope?" |
| Project manager | Business case / budget cycle | Defensible range + confidence class + key risk drivers for the funding ask |
| Estimator | Building or reviewing an estimate | Fast comparable retrieval, normalized unit costs, a challenge function ("your $/HP is 2σ below the historical population — why?") |
| Portfolio / management | Reviewing many asks | Consistency: are all early estimates built on the same historical basis? |

Design consequence: the primary interface is **conversational with a structured intake**, and the primary output is a **document** (estimate basis memo) — not a dashboard.

## 4. The data reality (assumptions to validate on day 1)

Assumed corpus per project, with expected coverage — **the first project task is to replace this table with a measured inventory**:

| Artifact | Contents of value | Expected reality |
|---|---|---|
| DBM (Design Basis Memorandum) | Scope definition, design parameters (capacity, pressure class, materials, site), assumptions, constraints | Prose + tables; format drift across eras; the single richest scope source |
| Scope documents / project charters | Business driver, boundaries, inclusions/exclusions | Overlaps DBM; sometimes the only scope record for small projects |
| Final cost reports / cost breakdowns | Actual total, breakdown by WBS/discipline (materials, labour, contracts, land, overheads), possibly original estimate vs. actual | The ground truth for training/backtesting; formats vary the most (exports from different financial systems across 15+ years) |
| Lessons learned | Delay causes, weather impacts, brownfield surprises, vendor issues, what the estimate missed | Free text; gold for *adjustment* reasoning and risk registers; most likely to be missing |
| Schedules, change orders, contracts | Duration, seasonality actually experienced, growth from baseline | Optional enrichment; probably spotty |

Known hazards, each addressed later in the plan:
- **Missing artifacts** → per-project *completeness score*; the knowledge base must represent "unknown" distinctly from "zero/none".
- **Format heterogeneity** → extraction is schema-first with human validation, not blind parsing (see `plan/02`).
- **Era effects** → all costs normalized to a common basis (escalation + scope normalization) at ingestion; raw numbers never compared directly.
- **Small n** → 200 projects is small-data territory. Per relevant strata (e.g., "compressor station brownfield upgrades") it may be 10-30 cases. This kills any dream of training deep models and pushes the design toward retrieval + transparent adjustment + statistical baselines that work at small n (see `research/03`, `research/05`, `research/07`).
- **Confidentiality** — cost data is commercially sensitive; the architecture must assume private deployment of the corpus and controllable model endpoints.

## 5. What "good" looks like (success criteria)

1. **Backtest accuracy**: on held-out historical projects (leave-one-out), the agent's P10-P90 range contains the actual final cost at the promised rate (calibration), and median absolute error beats both naive class-average $/unit and the organization's *original* early estimates for those projects, where recorded (the full baseline set — including a no-LLM k-NN and a raw-RAG bot — is enumerated in `plan/05` §1). The comparison against the org's own early estimates is the persuasive one.
2. **Retrieval quality**: estimators agree the top-5 comparables are the right ones (spot-checked panel review) ≥80% of the time.
3. **Traceability**: 100% of quantitative claims in an output link to a source project/document.
4. **Time**: comparable search + first range in minutes, vs. days of asking around.
5. **Adoption**: used in real budget-cycle submissions within two cycles; estimators reference its outputs rather than fighting them.

## 6. Constraints and non-functionals

- **Auditability over cleverness**: regulator- and finance-facing numbers need a written basis. Every pipeline stage keeps provenance.
- **Data governance**: corpus and derived knowledge base live in the utility's tenancy (private cloud or on-prem-adjacent); LLM calls via an approved enterprise endpoint (e.g., cloud provider-hosted Claude/GPT with no-training guarantees) — decided by IT policy, held as a swappable dependency.
- **Maintainability by a small team**: this will realistically be built and run by 1-3 people. Prefer boring, inspectable components (markdown + SQL + a workflow engine) over exotic infrastructure. This constraint does heavy lifting in the tradeoff calls of `plan/03`-`plan/05`.
- **Incremental value**: the corpus curation must pay off even if the agent layer stalls — a clean, normalized project history database is independently valuable to the estimating group.

## 7. Vocabulary used throughout the plan

- **Project dossier**: the distilled, structured-plus-prose record of one historical project (the "wiki page" — see `plan/03`).
- **Attribute schema**: the fixed set of structured fields extracted for every project (asset type, capacity, site type, season, region, costs by bucket, outcome flags...).
- **Normalized cost basis**: all money restated to a reference year/location via documented indices.
- **Comparable set**: the k historical projects retrieved as most similar for a given new scope.
- **Estimate basis memo**: the agent's output document — range, comparables, adjustments, exclusions, confidence class, citations.
