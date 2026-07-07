# 06 — Delivery roadmap, stack, risks, and the tradeoff ledger

> How to actually ship this: phases with exit criteria, a deliberately boring reference stack, the consolidated tradeoff table, and the risk register. Written so a team of 1-3 can execute it.

## 1. Phasing — each phase ships standalone value

### Phase 0 — Data reconnaissance (2-3 weeks)
Inventory the 200 projects (`plan/02` §2.1). Output: the *inventory reality report* — artifact coverage, format census, completeness distribution, parser bake-off on the 5 ugliest cost reports, and — critically — the **original-early-estimate recording rate**: for how many projects is the organization's own early estimate (and its class/date) actually on record? The headline business-case chart (`plan/05` §1 baseline (c): agent vs. actuals vs. the org's own early estimates) depends entirely on this field existing; if it's recorded for only a small fraction of projects, that comparison shrinks to an anecdote and the pitch must lean on baselines (a)/(b)/(d) instead — a framing decision to make in week 3, not month 6.
**Exit criteria:** we know how many projects have (scope + final cost), the original-estimate recording rate is measured, and leadership has seen the real coverage numbers. **Kill/reshape signal:** if <100 projects have usable actuals, the statistical layers shrink and the design tilts further toward qualitative comparables — decided here, not discovered in month six.
Execution detail — the day-by-day schedule, day-1 access requests, registry schema, parser bake-off rubric, estimator interview guide, and reality-report template — is in **`plan/07` (Phase 0 playbook)**.

### Phase 1 — The spine, thin (4-6 weeks)
Pick **one asset class** with the best data density (likely stations or a pipeline program). Run the full pipeline for those ~30-60 projects: parse → extract → normalize → validate → dossiers + first L3 asset-class page. Build the L1 database and the five KB access tools (`plan/03` §5).
**Exit criteria:** an estimator can be shown 5 dossiers and the asset-class page and says "yes, that's what happened on those jobs." A scripted (non-agentic) comparable query returns sane neighbours.

### Phase 2 — First estimating agent + backtest (4-6 weeks, overlaps 1)
Implement the `plan/04` flow for that one asset class. Immediately run the leave-one-out backtest (`plan/05` §1) on it.
**Exit criteria:** backtest report exists with honest numbers; golden-run suite passes; 3 pilot users produce real memos on live early-stage projects.
This is the **credibility milestone**: one asset class, end-to-end, with measured accuracy — worth more than every class half-done.

### Phase 3 — Corpus completion + hardening (6-10 weeks)
Remaining asset classes through the pipeline; cross-cutting L3 pages (winter premium, brownfield adders, growth-by-class); calibration fit per class; challenge pass tuned; memo template iterated with the estimating group; access control + run logging productionized.

### Phase 4 — Operate & compound (steady state)
New actuals loop (`plan/05` §4), quarterly review board, accuracy dashboard, model-upgrade gate. Possible expansions *only after* the core is trusted: schedule/duration estimating, risk-register drafting, scope-completeness checking (each reuses the same KB).

## 2. Reference stack (boring on purpose)

| Concern | Choice | Rationale / alternative |
|---|---|---|
| Raw docs | Object store / existing SharePoint + snapshot | Don't migrate source systems; snapshot for reproducibility |
| Parsing | Docling or Azure Document Intelligence (bake-off winner); vision-LLM fallback | See `research/02`; table fidelity on cost reports is the deciding test |
| Extraction & dossier LLM | Frontier model via approved enterprise endpoint (Claude on Bedrock/Vertex-class arrangement per IT policy) | Accuracy dominates at n=200; swappable behind one interface |
| L1 store | Postgres (SQLite in dev) | Plain SQL; every BI tool can read it |
| L2/L3 wiki | Markdown in a git repo | Versioning, diffs, PR review for free; humans can read it in the repo browser today |
| L4 search | pgvector + Postgres FTS (or OpenSearch if IT prefers) | Hybrid BM25+dense with a reranker; no dedicated vector DB needed at this scale |
| Orchestration (pipeline) | Python + Prefect (or Make, honestly) | 200 projects ≠ big data |
| Agent runtime | Thin custom orchestrator on the model provider's native tool-use API (Anthropic-style tools / MCP for the KB interfaces) | The `plan/04` flow is a fixed 6-stage workflow — owning ~1k lines beats inheriting a framework's churn; MCP keeps KB tools reusable from other clients (incl. desktop agents) |
| UI | Chat + memo viewer: Streamlit/Next.js thin client; memos as markdown/PDF | The memo is the product; the UI is a delivery vehicle |
| Eval | pytest-style golden suite + backtest scripts in the same repo; results as versioned artifacts | Eval is code, versioned with methodology |
| External sanity anchors | FERC Form 2 (regulated gas plant costs), EIA pipeline construction-cost data; RSMeans/Gordian for commodity line items | Cheap outside-baseline cross-checks so the corpus's own unit costs are never the only reference (`research/07` §5.7); not a substitute for the internal CERs |

## 3. Consolidated tradeoff ledger

Decisions argued in earlier docs, gathered with their one-line verdicts:

| # | Decision | Options | Verdict | Where argued |
|---|---|---|---|---|
| 1 | Memory substrate | Curated wiki + DB **vs** raw RAG **vs** GraphRAG **vs** fine-tune | Curated wiki + structured DB; RAG demoted to fallback; GraphRAG & fine-tune rejected at this n | `plan/03` §2 |
| 2 | Estimation core | LLM-direct guess **vs** pure ML regression **vs** analog+parametric hybrid with LLM adjustment | Hybrid: code does math, LLM selects/adjusts/explains. Deliberate deviation from `research/07`'s GBT+SHAP backbone: at per-class n of 10–30, gradient-boosted ensembles overfit; robust unit-cost regression/quantiles is the defensible parametric layer (`research/04` impl. 5). Revisit GBTs only if pooled cross-class CERs prove useful | `plan/04` §0,§4 |
| 3 | Agent freedom | Free chat agent **vs** fixed-skeleton workflow with agentic stages | Fixed skeleton — consistency & testability beat flexibility for institutional estimates | `plan/04` §0 |
| 4 | Similarity | Learned embedding-only **vs** hand-weighted mixed-attribute + text embedding + agent curation | The latter; weights backtest-tuned; no opaque single score | `plan/04` §2, `research/05` |
| 5 | Ranges | LLM-verbalized confidence **vs** backtest-calibrated (conformal-style) intervals | Calibrated intervals; LLM never states unmeasured confidence | `plan/05` §2, `research/06` |
| 6 | Learning | Runtime agent memory writes **vs** reviewed pipeline events | Reviewed pipeline only; agent proposes, humans merge | `plan/03` §6, `plan/05` §4 |
| 7 | Build order | All classes broad-first **vs** one class deep-first | Deep-first (credibility milestone) | this doc §1 |
| 8 | Framework | LangChain/agent-framework **vs** thin custom on native tool-use | Thin custom + MCP-shaped KB tools | this doc §2 |
| 9 | Buy vs build | Commercial AI estimating tools **vs** build on own corpus | Build — but eyes open: Galorath's **SEERai** (Oct 2025) is the closest commercial analog (agentic NL→audit-ready estimates over validated historicals); our edge is depth on our own DBM/lessons-learned corpus, not the concept. Demo SEERai during Phase 0 as a calibration point | `research/07` |
| 10 | Extraction QA | Full human review **vs** confidence-sampled review | 100% on cost/capacity/type fields, sampled elsewhere | `plan/02` §2.5 |

Each verdict is revisitable; the ledger exists so future debates start from the recorded reasoning instead of re-litigating from zero.

## 4. Risk register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Data worse than assumed (missing actuals, unparseable cost reports) | Medium-high | High | Phase 0 exists precisely to measure this before commitments; design degrades to qualitative-comparables mode |
| Extraction errors poison the KB | Medium | High | Two-pass numeric verification, 100% human review of driver fields, provenance on everything, git-versioned corrections |
| Estimator rejection ("black box pricing my job") | Medium | Fatal | Estimators co-own schema & validation (`plan/02` §2.5), memo shows all arithmetic, backtest vs. their own historical estimates, quarterly review board |
| Backtest disappoints (no better than naive baseline) | Low-medium | High | Still ship comparable-retrieval + memo drafting (already valuable); investigate whether similarity weights or normalization are the weak link — the layered design localizes failure |
| Leakage inflates backtest, then production disappoints | Medium | High | Strict-temporal protocol + per-fold L3 regeneration (`plan/05` §1) from day one |
| Scope creep into a general PM copilot | High | Medium | `plan/01` §2 "is not" list; expansions gated on Phase 4 trust |
| Model/vendor churn | Medium | Low | All domain knowledge in the KB, not the model; regression gate makes upgrades routine |
| Key-person dependency (1 builder) | High | Medium | Everything-as-code in one repo, this plan as the onboarding doc, HANDOFF discipline |
| Cost data confidentiality incident | Low | High | Private tenancy, no-training endpoints, access control on KB and logs |

## 5. Team & effort envelope

Realistic minimum: **1 senior AI/software builder** (pipeline, KB, agent, eval) + **estimator time** (~0.2 FTE through Phases 0-2 for schema, validation, review board) + IT support for endpoints/access. Calendar to credibility milestone (end Phase 2): **~4-5 months**. A second builder compresses calendar mostly in Phase 3.

## 6. The one-paragraph recommended path

Run Phase 0 immediately — it is cheap and converts every assumption in this plan into measured fact. Then build the spine deep on one asset class: pipeline → validated structured records → dossiers → asset-class page → fixed-skeleton agent → leave-one-out backtest, and put the backtest chart (agent range vs. actuals vs. the org's own historical early estimates) in front of the estimating group. Everything after that — full corpus, cross-cutting pattern pages, calibration per class, the operate-and-compound loop — is scaling a thing that has already proven itself on the organization's own history.
