# 05 — Evaluation, calibration, and the learning loop

> An estimating tool that can't prove its accuracy is a liability with a UI. The 200 historical projects are not just the knowledge base — they are also the test set, and backtesting against them is the single most persuasive artifact this project can produce for skeptical estimators and management.

## 1. Backtesting: the core evaluation

**Protocol — leave-one-out over projects with actuals:**
For each held-out project *P*:
1. Rebuild the agent's world with *P* (and any project completed after *P*'s estimate date, for strict realism — see leakage rules below) removed/masked.
2. Feed the agent *P*'s scope as it was known early (DBM-stage text; not the as-built record).
3. Agent produces its range; compare to *P*'s normalized final actual.

**Metrics:**
- *Point accuracy*: median APE and MAPE of P50 vs. actual, sliced by asset class, era, completeness.
- *Calibration*: fraction of actuals inside P10-P90 (target ≈80%), inside P25-P75 (≈50%); pinball loss on the quantiles.
- *Retrieval quality*: estimator panel spot-check — are the chosen comparables the right ones? (This catches "right answer, wrong reasoning," which pure error metrics miss.)
- *Baselines to beat*: (a) naive class-average $/unit, (b) k-NN on structured attributes with no LLM, (c) the organization's **original early estimates** for those same projects where recorded, and (d) a **raw-RAG agent** — the same LLM given hybrid search over the parsed documents with no curated KB. Baseline (d) exists because "curation beats RAG at this n" is the central design hypothesis of the whole system (`research/04` §8 flags it as exactly that — a hypothesis, not a settled result); without this baseline, a winning backtest can't answer "would a two-week RAG bot have done the same?". Beating (c) — or even matching it with better-calibrated ranges and instant turnaround — is the business case in one chart.

**Compute reality (do not under-budget this):** leave-one-out with the LLM in the loop means ~200 full agent-pipeline runs, each against a per-fold-regenerated KB — genuinely expensive and slow if run naively on every methodology tweak (`research/06` implications §2). Mitigations, adopted as the working protocol: (a) **calibrate the conformal band on a cheap structured base predictor** (k-NN / quantile regressor over the L1 attributes) that tracks the LLM-analog P50, so recalibration is cheap — this surrogate is for fast iteration between gates only; the *shipped* band's coverage is confirmed (and re-derived if the two residual distributions diverge) on true LLM-analog LOO residuals at the release gate in (c), so the deployed guarantee still attaches to the §2 method; (b) **cache per-project agent outputs** keyed on (KB commit, methodology version) so unchanged folds are never re-run; (c) reserve **full LLM-in-the-loop LOO for periodic validation and release gates**, not every iteration; (d) per-fold L3 regeneration only needs the *statistics* recomputed (the embedded SQL re-run minus the held-out project) — the prose need not be rewritten per fold *provided L3 prose is written to reference classes and distributions, not named individual projects*; any project-named narrative line must be regenerated or masked per fold exactly like the statistics, or it leaks the held-out project's identity and outcome into its own fold.

**Leakage discipline (the failure mode that fakes success):**
- The held-out project's own documents must be *fully* absent — L1 row, dossier, L4 chunks, and any L3 synthesis page statistics that included it (L3 pages must be regenerated per fold, which the "SQL embedded in page" design of `plan/03` §4 makes mechanical).
- Time-travel realism: comparables restricted to projects completed *before* the held-out project's estimate date. Report both the strict-temporal and full-corpus numbers; strict-temporal is the honest one, and it also reveals how the tool's value grows with corpus size. Caveat to state up front: strict-temporal folds for the *oldest* projects have near-empty comparable pools, so the honest metric effectively runs on the recent two-thirds of the corpus — slice the report by era and don't let starved early folds drag the headline number.
- Watch for scope-text leakage: if the archived "scope description" was written post-completion, it encodes the outcome. Prefer DBM-dated text; flag projects where only retrospective scope text exists.

## 2. Calibration: turning backtest error into honest ranges

The range-assembly rule of `plan/04` §4 is *fit on backtest results*, not assumed:
- Measure the empirical error distribution of adjusted-analog P50s per asset class; set the widening factors so coverage hits target. `research/06` names the machinery: **jackknife+ / CV+ conformal prediction** — distribution-free, finite-sample coverage from leave-one-out residuals, wraps *any* underlying predictor including the LLM-analog method, and works at small n.
- Where the corpus is thin (rare asset classes), ranges inherit the *global* error distribution — wider, honestly. Caveat from `research/06`: marginal conformal coverage does **not** guarantee per-class coverage; if we want a defensible "station estimates land in-band 80% of the time" claim, use **Mondrian (class-conditional) conformal** over pre-declared asset classes. Set expectations accordingly: at per-class n of 10–30, Mondrian bands come from quantiles of 10–30 residuals and will be *honest but wide* — brief the estimating group that width is the price of a per-class guarantee at this corpus size, and that it narrows as projects accrue, before anyone reads width as failure.
- As regimes shift (escalation shocks), **adaptive conformal inference (ACI)** re-tunes coverage from recent under/over-coverage — the principled version of §5's drift response.
- Metrics vocabulary per `research/06`: interval coverage (PICP) vs. nominal, sharpness (width), and pinball loss as the proper scoring rule.
- Recalibrate whenever the KB or methodology version changes; calibration parameters are versioned artifacts alongside the KB commit hash.

Also calibrate the *words*: the memo's confidence language maps to measured coverage ("this class of estimate has historically landed within ±40% eight times out of ten"), never to LLM-verbalized confidence, which research consistently shows is miscalibrated (see `research/06`).

## 3. Regression evaluation of the agent itself

Beyond cost accuracy, the agent's *behaviour* needs tests that run on every prompt/model/KB change:
- **Golden-run suite**: ~20 curated scope descriptions (spanning asset classes, thin-comp cases, adversarial inputs like scope-with-the-answer-hinted) with assertions on structure: intake asked the right questions, hard filters applied, degradation flagged when comps are thin, memo sections present, all figures carry provenance, arithmetic in tools not prose.
- **LLM-judge checks** for the qualitative parts (was the comparable-exclusion reasoning sound?), human-audited on a sample.
- Threshold gates: a methodology change ships only if backtest metrics don't regress and the golden suite passes. Model upgrades go through the same gate — this is what makes "swap to the newest frontier model" a routine event instead of a leap of faith.

## 4. The learning loop

The system improves through **reviewed pipeline events**, not silent runtime memory writes:

1. **New actuals**: when a project completes, it flows through the `plan/02` pipeline → new L1 row + dossier → L3 pages regenerate → next calibration run includes it. The corpus compounds; strict-temporal backtesting (§1) quantifies how much each year of data is worth.
2. **Estimate outcomes**: every agent-produced memo is itself logged; when that project later gets a sanctioned estimate and eventually actuals, the agent's early range is scored against reality *in production*, not just backtest. This produces the ongoing accuracy dashboard.
3. **User corrections**: when an estimator overrides a comparable choice or an adjustment, the trace is tagged; recurring override patterns become issues against similarity weights, L3 priors, or extraction quality. Curated, not auto-applied.
4. **Lessons-learned closure**: risk flags the agent raised that then materialized (or didn't) refine the L3 pattern pages' base rates.

## 5. Ongoing quality watch

- Drift: input-cost regime changes (post-COVID-style escalation shocks) show up as rising backtest error on recent-era slices → triggers re-examination of escalation indices and era weighting, with the L3 "regime breaks" page updated.
- Usage analytics: which memo sections users read/ignore, where conversations push back — feeds UX iteration.
- Quarterly estimator review board: 1 hour, sample of recent memos, thumbs on comparables and adjustments. Keeps the human experts owning the methodology — which is both quality control and the political foundation the tool stands on.
