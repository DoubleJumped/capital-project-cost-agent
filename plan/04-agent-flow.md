# 04 — The estimation agent: end-to-end flow

> What actually happens between "here's my new project" and a defensible estimate basis memo. The core design principle: **the LLM reasons and explains; code does arithmetic and statistics.** Numbers are computed by tools over L1; the agent decides *which* numbers, *which* comparables, *which* adjustments — and writes the argument.

## 0. Flow at a glance

```
User scope description (free text + optional fields)
   │ 1. INTAKE — structured elicitation
   ▼
New-project attribute record (same schema as historicals) + open questions
   │ 2. RETRIEVE — hard filters → similarity ranking → agent curation
   ▼
Comparable set (k≈5-10) + population stats for the asset class
   │ 3. READ & REASON — dossiers + L3 priors; qualitative deltas per comparable
   ▼
Adjustment plan (explicit, per-comparable, per-driver)
   │ 4. COMPUTE — normalized unit costs, adjusted analogs, statistical baseline,
   │             range assembly (code, not LLM arithmetic)
   ▼
Draft range (P10/P50/P90) + drivers + risk flags
   │ 5. CHALLENGE — self-critique pass + reference-class sanity check
   ▼
6. ESTIMATE BASIS MEMO — document out, fully cited, with confidence class
   │ 7. FEEDBACK — user corrections; eventual actuals loop back (plan/05)
```

Steps 2-5 are agentic (tool calls against `plan/03` §5 interfaces), but the *skeleton is fixed*: a workflow with defined stages, inside which the agent has freedom. A fixed skeleton makes behaviour testable and consistent across users — a free-roaming chat agent would produce different methodology per conversation, which is fatal for institutional trust.

## 1. Intake

- User describes the project in natural language; optional quick-form for knowns (asset type, capacity, site, target season/year).
- Agent maps the description onto the **attribute schema** (same one as historicals — this symmetry is what makes matching possible), then asks *only the questions that matter*: it knows which attributes drive retrieval and cost for this asset class (from L3), so it asks 3-6 targeted questions ("brownfield? live station tie-ins? winter civil work?") rather than a 40-field form.
- Unknowns stay `unknown` and propagate honestly: they widen the final range and are listed in the memo as "assumptions to firm up" — this is exactly what a good senior estimator does, and it's also the AACE-class discipline (scope maturity ⇒ range width).
- Output artifact: the new-project record + assumption list, shown to the user for confirmation. *The user confirms the inputs before any number is produced* — cheap trust win.

## 2. Retrieve comparables

Two-stage, then agent-curated:

1. **Hard filters** (code): asset type, completeness (`has_actuals`), any user-imposed constraints — *and nothing else*. At n≈200, hard-ANDing more attributes (era, site type, region) collapses the candidate pool to single digits and destabilizes everything downstream (`research/05` §4). Era in particular must not be a hard filter: its price-level component is already handled by normalization to a reference basis, and recency is a weighted similarity feature — so hard-filtering on era starves the pool while adding nothing those two levers don't capture. Any *residual* era effect beyond index escalation — a genuine cost-regime break (e.g. a post-2021 compression-cost shift, per the L3 regime-breaks page) — enters as an explicit, cited adjustment in stage 3, never as a filter.
2. **Similarity ranking** (code): weighted mixed-attribute similarity (Gower-style over numeric capacity/size + categorical site/season/execution + embedding similarity of scope text), with the would-be filters (greenfield/brownfield, era, region) expressed as **soft distance penalties** — a mismatched project ranks lower but can still surface when nothing better exists. Weights per asset class, set initially with estimators, tuned later by backtest (`plan/05`). A **distance caliper** guards the other end: if no candidate falls within the caliper, the answer is "no good comparable exists," never a forced match (`research/05` §8). Returns candidates *with per-attribute similarity breakdown* — no opaque single score.
3. **Agent curation** (LLM): reads the candidates' fact blocks, drops false friends ("similar HP but it was a greenfield sweetening plant — different animal"), pulls in edge cases worth reading anyway ("include P-2017-003: closest winter-brownfield precedent"), documents *why* each kept comparable is comparable. Target k ≈ 5-10 kept + reasons for notable exclusions.

If the comparable set is thin (n < 3 close matches — will happen at 200 projects), the agent says so and switches mode: decompose into sub-scopes and find comparables per component (station modification comps + pipeline-lateral comps), or lean on the parametric baseline with an explicitly wider range. **Degrading loudly, never silently.** In decomposition mode, the component ranges roll up via **Monte Carlo simulation over the per-component distributions** (AACE 41R-08 range estimating; `research/03` §7) — summing per-line ranges by hand understates the tails and produces a falsely symmetric band, when in reality one or two high-variance components (weather-exposed civil, brownfield tie-ins) should drive the tail. These rolled-up ranges carry the class's *global* conformal widening (`plan/05` §2) as a floor, since the adjusted-analog calibration doesn't cover the decomposition path.

## 3. Read & reason

- Agent reads the kept dossiers (L2) and the relevant L3 pages (asset-class cost behaviour, winter premium, brownfield adders).
- For each comparable, it produces a **delta table**: dimensions where new ≠ comp (size, site, season, cost-regime/era shifts *beyond escalation*, scope inclusions), direction of cost impact, and the evidence basis for an adjustment (L3 observed distributions where available; flagged judgment where not). The era delta is strictly the residual regime effect — index escalation is already applied at normalization and is never re-adjusted here (§2).
- Lessons-learned surface here as **risk flags**: "2 of 5 comparables hit unplanned ground-condition costs; none of your assumptions cover geotech" — this is where the corpus's qualitative gold pays out.

## 4. Compute

All arithmetic in tools:
- **Analog method** (primary): per comparable, normalized cost → scale by capacity ratio with class-appropriate scaling (linear $/unit or capacity-exponent rule per L3 — economies of scale are large and real: `research/04` cites ~3.2× per-HP cost difference between 2,000 HP and 30,000 HP compression) → apply the adjustment plan's quantified deltas (each with a source: observed distribution / L3 prior / flagged judgment) → k adjusted analog estimates.
- **Parametric baseline** (independent check): unit-cost regression/quantiles for the asset class from L1 population (works even when close comps are thin).
- **Range assembly**: spread of adjusted analogs + baseline ⇒ P10/P50/P90, with the widening factors fit on measured backtest error (`plan/05` §2). Exact assembly rule is a `plan/05` calibration decision (candidates: quantiles of pooled adjusted analogs widened by conformal calibration; or Monte Carlo over adjustment uncertainties). The rule is *fixed methodology, versioned* — not improvised per run by the LLM.
- **What deliberately does NOT enter the range: a reference-class estimate-growth uplift.** The adjusted analogs are anchored on comparables' *final actuals*, so cost growth and optimism bias are already embedded in those numbers — layering the historical estimate-growth distribution on top would double-count the correction (Flyvbjerg's uplift corrects a bottom-up *early estimate*, which is not what we produce; `research/05` §7). The estimate-growth-by-class table (an L3 page) still has two legitimate jobs: (a) the challenge pass uses it to score a *user-supplied* bottom-up estimate against class base rates, and (b) a narrowly-scoped **scope-maturity correction** where measurable — if the new project is matched on immature DBM-stage scope while comparables' attributes reflect as-built records, the class's observed early-scope→as-built drift may be applied, explicitly and as its own cited line, never as a blanket uplift.

Two methods disagreeing badly is signal, not embarrassment: the agent must surface it ("analogs say $12M, population parametric says $18M — likely because your comps are all pre-2022; here's both") rather than average it away.

## 5. Challenge pass

A second agent pass (fresh context, critic prompt) attacks the draft: comps that don't belong, adjustments double-counted (winter premium *and* schedule-delay adder may overlap), unit costs outside historical population bands — and, where an external anchor exists for the class, outside FERC Form 2 / EIA / RSMeans sanity ranges (`plan/06` §2) — missing standard scope items (checklist from L3 asset-class page), optimism-bias check against the estimate-growth base rates. Material findings loop back to step 3/4; the memo records what the critic raised and how it was resolved. This is cheap (one extra model call) and directly targets the known failure mode of analog estimating: motivated comparable-shopping.

## 6. The estimate basis memo

Markdown/PDF out:
1. Scope as understood (the confirmed intake record) + assumptions & exclusions
2. Range: P10/P50/P90 in reference-year and target-year dollars; implied AACE-class framing ("treat as Class 5/4")
3. Comparables table: each with normalized actuals, similarity rationale, adjustments applied (numbers + basis)
4. Parametric cross-check and reconciliation
5. Risk flags & lessons from the corpus (cited to source projects/pages)
6. What would tighten this estimate (the unknowns that matter most)
7. Provenance appendix: KB commit hash, every figure's source links

The memo is the product. It reads like a competent estimator's basis-of-estimate note, and every sentence is checkable.

## 7. Conversation after the memo

The user pushes back naturally: "we'll do the tie-ins in summer actually" → agent re-runs the affected adjustments and diffs the range; "why is P-2018-11 in the set?" → shows its similarity breakdown; "show me the winter premium evidence" → opens the L3 page and underlying projects. Because arithmetic lives in tools and the memo is assembled from a structured result object, updates are recomputations, not vibes.

## 8. Implementation shape

- Orchestration: a typed workflow (steps 1-6 as explicit stages) built on the chosen agent framework (final call in `plan/06`; the safe default is a thin custom orchestrator + the model provider's tool-use API — frameworks churn, and this flow is simple enough to own).
- Model: strongest available frontier model for steps 1, 2c, 3, 5, 6; no fine-tuning; all domain knowledge lives in the KB, which is exactly what makes the system inspectable and model-upgradable.
- Every run logged end-to-end (inputs, tool calls, comps, adjustments, output) — this trace corpus feeds evaluation (`plan/05`) and is the audit record.
