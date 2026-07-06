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
- *Baselines to beat*: (a) naive class-average $/unit, (b) k-NN on structured attributes with no LLM, (c) the organization's **original early estimates** for those same projects where recorded. Beating (c) — or even matching it with better-calibrated ranges and instant turnaround — is the business case in one chart.

**Leakage discipline (the failure mode that fakes success):**
- The held-out project's own documents must be *fully* absent — L1 row, dossier, L4 chunks, and any L3 synthesis page statistics that included it (L3 pages must be regenerated per fold, which the "SQL embedded in page" design of `plan/03` §4 makes mechanical).
- Time-travel realism: comparables restricted to projects completed *before* the held-out project's estimate date. Report both the strict-temporal and full-corpus numbers; strict-temporal is the honest one, and it also reveals how the tool's value grows with corpus size.
- Watch for scope-text leakage: if the archived "scope description" was written post-completion, it encodes the outcome. Prefer DBM-dated text; flag projects where only retrospective scope text exists.

## 2. Calibration: turning backtest error into honest ranges

The range-assembly rule of `plan/04` §4 is *fit on backtest results*, not assumed:
- Measure the empirical error distribution of adjusted-analog P50s per asset class; set the widening factors so coverage hits target (conformal-style: use held-out residual quantiles directly — simple, distribution-free, works at small n; details refined per `research/06`).
- Where the corpus is thin (rare asset classes), ranges inherit the *global* error distribution — wider, honestly.
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
