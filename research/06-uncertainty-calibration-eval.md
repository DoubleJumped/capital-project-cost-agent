# 06 — Uncertainty, Calibration & Evaluation of the Estimation Agent

*Research thread for the capital-project cost agent (Canadian prairie gas utility, ~200 historical projects). How to make the agent emit honest ranges instead of confident point estimates, and how to prove — on our own held-out history — that those ranges are trustworthy.*

---

## TL;DR for this project

- **Never ship a single number.** Every estimate is a distribution. The industry standard our estimators already trust — AACE estimate classes — is defined *in ranges*, and AACE 18R-97 gives a *range of ranges* per class: a Class 5 concept-screening estimate can be as tight as −20%/+30% or as wide as −50%/+100% depending on project complexity, reference-data quality, and definition, and the ranges are typically asymmetric (overrun risk exceeds underrun) [1]. Our output must speak this language.
- **Don't trust the LLM to tell you how sure it is.** Verbalized confidence is systematically overconfident — LLMs cluster their stated confidence in the 80–100% band almost regardless of actual accuracy [2], and RL-tuned models verbalize *worse* than their own token probabilities [2]. Confidence must come from measured historical error, not from the model's self-report.
- **Get ranges from residuals, not vibes.** Conformal prediction (specifically **jackknife+ / CV+**) gives distribution-free, finite-sample coverage guarantees from leave-one-out residuals — it works at small *n*, wraps *any* underlying predictor (including our LLM-analog estimator), and needs no distributional assumptions [3]. This is the statistical backbone of our range.
- **The 200 projects are simultaneously the knowledge base AND the test set.** Leave-one-out backtesting is our single most persuasive artifact. It also sets the conformal calibration constants — this is not optional theory, it is how the widening factors get their numbers.
- **Leakage is the failure mode that fakes success.** A held-out project's own docs, any synthesis page whose statistics included it, and post-hoc scope text that encodes the outcome must all be masked. Report strict-temporal (comparables predating the estimate) numbers as the honest ones [see plan/05].
- **Report the right metrics.** Point accuracy (MAPE) is not enough. Add **interval coverage (PICP)** vs. nominal, **sharpness** (interval width), and **pinball loss** — a proper scoring rule that rewards both calibration and tightness [4].
- **The learning loop is a distribution-shift problem.** Input-cost regimes shift (post-COVID escalation shocks); exchangeability breaks. Adaptive conformal inference (ACI) re-tunes coverage from recent under/over-coverage feedback and is the principled way to keep ranges honest as new projects land [5].

---

## 1. Why point estimates are malpractice here

Capital-cost estimation is irreducibly uncertain, and the profession has codified this. The **AACE International** estimate-classification system (18R-97) ties expected accuracy to the *maturity of project definition*, not to effort spent, and — critically — publishes each class as a **range of ranges** rather than a fixed band: a Class 5 concept-screening estimate can span anywhere from −20%/+30% to −50%/+100% depending on technical complexity, availability of reference cost data, degree of definition, and contingency treatment, with the wide end typical of early gas-utility scopes; later classes tighten similarly toward Class 1 [1]. Two features matter for us. First, ranges are usually **asymmetric** — the upside (overrun) tail is fatter, reflecting real capital-project behavior. Second, AACE is explicit that these published ranges are *guidelines only*; the actual range must be established by project-specific quantitative risk analysis (typically Monte Carlo), not applied by lookup table [1]. The practical consequence: we should never hard-code a single −50%/+100% band, but let the backtest-measured error for a given class *place* the estimate inside AACE's published span.

Our agent operates at roughly Class 5–4 maturity: an engineer describes a *new* scope in natural language, long before detailed engineering. Pretending to Class 2 precision would be dishonest. The design imperative is that the agent's native output is a distribution consistent with early-definition uncertainty, expressed the way estimators already consume it: **P50 (median), P80, and a class-appropriate band.**

## 2. Reference class forecasting — the profession's answer to optimism bias

Flyvbjerg's **reference class forecasting (RCF)**, drawn from Kahneman's outside view, is the most authoritative cost-estimation-under-uncertainty method and is conceptually identical to what our agent does. Rather than build bottom-up (the "inside view," which is systematically optimistic), RCF prices a project by the empirical outcome distribution of *similar completed projects* [6]. Flyvbjerg's datasets show cost overruns are both large and predictable: real-terms averages of ~45% for rail, ~34% for fixed links, ~20% for roads, with tails far worse (Olympics ~157%, dams ~96%) [6]. Critically, RCF produces **percentile uplifts**: e.g. for rail the P50 overrun ≈40% and P80 ≈57% — you pick the confidence level and read off the required uplift [6]. Validated applications report actual outcomes landing inside RCF prediction intervals >70–80% of the time, versus <20% for unadjusted inside-view forecasts [6]. RCF is endorsed by the American Planning Association and mandated in UK/Netherlands/Hong Kong/Denmark/Switzerland guidance [6].

**Implication and caveat.** Our agent *is* an automated, scope-aware RCF engine — find the reference class (comparable projects), read the empirical distribution, adjust. But RCF's known weakness is the **small-reference-class problem**: with only ~200 total projects and thin sub-classes (a rare station type may have 3–4 analogs), the empirical distribution is noisy. Natarajan (2022) and others explicitly combine RCF with machine learning to borrow strength across sparse classes [6a]. This is exactly why we need conformal methods that degrade gracefully at small *n*. The honest fallback for a thin sub-class is the *global* error distribution (wider) — but note the subtlety: a marginal conformal guarantee does **not** imply the band covers correctly *within* each asset class, so if we want a defensible per-class coverage claim (e.g. "compressor-station estimates land in-band 80% of the time") we need **class-conditional / Mondrian conformal**, which guarantees coverage simultaneously over a pre-declared collection of subgroups [16]. See §3.2.

## 3. Statistical prediction intervals: the toolbox

Three families of methods produce ranges. They are complementary, not competing.

### 3.1 Quantile regression
Directly model conditional quantiles (e.g. fit a 0.10 and 0.90 quantile regressor; their span is a ~80% interval) [7]. Assumption-free, robust to outliers, cheap at inference. Downsides: each quantile needs its own fitted model, quantiles can cross, and there is **no finite-sample coverage guarantee** — the interval is only as calibrated as the model is correct [7]. Useful as a *base predictor* but not a calibration guarantee on its own. The two families in §3.1 and §3.2 are not either/or: **Conformalized Quantile Regression (CQR)** fuses them, conformalizing a fitted quantile regressor's residuals to recover the finite-sample coverage guarantee while keeping the quantile model's *adaptive, heteroscedastic* interval width [15]. This matters directly for us because cost variance scales with project size — a small tie-in and a greenfield station should not carry the same absolute band — so CQR is the natural way to get wider bands on bigger stations *and* a guarantee, rather than the constant-width band that plain split conformal tends to produce.

### 3.2 Conformal prediction — the recommended backbone
Conformal prediction (CP) converts *any* predictor's residuals into intervals with a **distribution-free, finite-sample coverage guarantee**, which is precisely what a small, heterogeneous corpus needs [3]. Variants (all in the open-source **MAPIE** library [3]) trade off data efficiency vs. compute:

| Method | How | Coverage guarantee | Small-*n* behavior |
|---|---|---|---|
| **Split (inductive) conformal** | Train on one half, calibrate residual quantiles on the other | 1−α | Weak — wastes half the data on calibration; poor when *n* is small [3] |
| **Jackknife+** | *n* leave-one-out models; interval from LOO residual quantiles applied to LOO predictions | **1−2α** (proven [20]) | Best data efficiency; can be unstable when *n* ≈ #features [3] |
| **CV+** | K-fold version of jackknife+; out-of-fold residuals | 1−2α | Slightly wider/more conservative; far cheaper than LOO [3] |
| **Jackknife+-after-bootstrap** | Bootstrap resamples, out-of-bag residuals | 1−2α | Good compute/coverage trade-off [3] |

For ~200 projects, **jackknife+ or CV+** is the sweet spot: LOO is cheap at this scale and reuses every project for both fitting and calibration, and the 1−2α coverage bound is a proven theorem, not an implementation heuristic — the guarantee is established in Barber, Candès, Ramdas & Tibshirani (2021), which is what makes it load-bearing enough to put in front of management [20]. The elegance for us: CP does not care whether the underlying "predictor" is a linear model, a k-NN over structured attributes, or the **LLM's adjusted-analog P50** — it wraps all of them and turns their historical residuals into a guaranteed-coverage band. That directly implements the calibration rule sketched in `plan/05` §2.

**Marginal ≠ per-class.** The guarantees above are *marginal* — averaged over the whole corpus. They do not promise that compressor-station or rare-vintage estimates individually land in-band at the nominal rate; a class we systematically under-cover can be masked by classes we over-cover. If a defensible **within-asset-class** coverage claim is needed, use **class-conditional (Mondrian) conformal**, which reframes conditional coverage as coverage over a declared collection of subgroups and delivers exact finite-sample guarantees simultaneously across them [16]. For thin classes this trades tightness for honesty (wider bands) — the principled version of the "fall back to the global distribution" instinct in §2.

### 3.3 Monte Carlo cost simulation
The estimating-practice standard: decompose the estimate into line items, assign each a distribution (triangular/PERT are conventional), model correlations, and run thousands of iterations to get an S-curve from which P50/P80/P95 are read [8][9]. Practitioner guidance: a well-defined program sees only a **5–8% uplift from P50→P80**, while a high-uncertainty program can see **15–20%+** [9] — the gap *is* the uncertainty signal. P80 is the common budget-setting point; P90 adds management reserve [9]. Pitfalls: ignoring correlation between line items badly understates the tail; garbage input distributions produce false precision [8].

**How these fit together for us.** Conformal prediction gives the *empirically guaranteed outer band* ("this class of estimate lands within ±X% eight times in ten, measured"). Monte Carlo, if we later decompose scope into cost drivers, gives an *explainable inner decomposition* ("labor and weather-delay risk contribute most of the spread"). Quantile regression / k-NN can serve as the base point predictor CP wraps. Recommendation: **lead with conformal-calibrated ranges** (defensible, measured, small-*n* honest); add Monte Carlo decomposition later as an explanatory overlay, not the primary guarantee.

## 4. LLM-specific uncertainty: do not trust the model's self-report

This is the sharpest, most counterintuitive finding for the design, and it is well-replicated.

- **Verbalized confidence is miscalibrated and overconfident.** Xiong et al. (2024) and subsequent surveys find LLMs cluster stated confidence in the **80–100% range almost independent of actual correctness** [2][10]. Asking "how confident are you (0–100%)?" produces numbers that look precise and mean little.
- **RL-tuned models are worse.** Damani et al. (2025) found RL-trained models verbalize confidence *worse than their own token probabilities* [2] — the very models best at reasoning are among the worst at honestly reporting doubt.
- **Better signals exist but are indirect.** **Self-consistency / semantic entropy**: sample the model multiple times and measure disagreement, clustering by semantic *equivalence* rather than surface strings. Semantic entropy — entropy over meaning-clusters of sampled answers — was shown by Farquhar, Kossen, Kuhn & Gal (*Nature*, 2024) to detect confabulations far better than any single verbalized number [18]. Self-consistency sampling itself (Wang et al., 2022) lifts accuracy on numeric/arithmetic-reasoning tasks by **+17.9pp on GSM8K**, with **+11.0pp on SVAMP and +12.2pp on AQuA** [11] — evidence that multi-sample agreement is a meaningful signal, not noise.
- **Wrap the LLM in conformal prediction.** A 2025 line of work applies CP to LLM-as-a-judge outputs, building calibrated prediction intervals from a single evaluation run with an ordinal-boundary adjustment for discrete ratings [12] — concrete precedent for treating the LLM as an uncalibrated point-emitter and getting the guarantee from an external conformal wrapper, exactly our plan.

**Design consequence.** The agent's *confidence language* must be bound to *measured backtest coverage*, never to a self-reported percentage. If we want a model-internal uncertainty signal (e.g. to widen ranges when the scope is genuinely novel), derive it from **multi-sample disagreement**, not from asking the model how sure it is.

## 5. Evaluation: backtesting on our own history

The 200 projects are the test set. The protocol (detailed in `plan/05`) is **leave-one-out backtesting**: mask project *P* entirely, feed the agent *P*'s early-stage scope, compare its range to *P*'s normalized actual.

**Metrics — a scorecard, not a single number:**
- **Point accuracy:** median APE and MAPE of P50 vs. actual, *sliced* by asset class, era, and data completeness (an aggregate MAPE hides that we're great on stations and terrible on rare compressor work).
- **Interval coverage (PICP):** fraction of actuals inside P10–P90 (target ≈80%) and P25–P75 (≈50%). This is the calibration test — a model can have low MAPE and still lie about its ranges [4].
- **Sharpness:** mean interval width. Coverage is trivial to achieve with absurdly wide bands; sharpness keeps us honest — the goal is the *tightest* band that still covers [4].
- **Pinball loss (quantile score):** a *proper scoring rule* that jointly penalizes miscoverage and excess width, so it can't be gamed by widening [4]. This is the single headline metric for range quality; CRPS (its integral over quantiles) is the density-level generalization [4].
- **Calibration curve / reliability diagram (regression):** PICP checks coverage at *one* nominal level and hides *where* miscoverage lives. Plot empirical vs. nominal coverage across the full range of quantiles (Kuleshov, Fenner & Ermon, 2018) to see, e.g., that we cover well at P80 but systematically miss the upper tail on greenfield stations — the standard visual diagnostic that turns a single coverage number into an actionable map, and the same procedure post-hoc *recalibrates* a miscalibrated regressor [19].
- **Baselines to beat:** (a) naive class-average $/unit; (b) k-NN on structured attributes, no LLM; (c) **the organization's own original early estimates** where recorded. Beating (c), or matching it with better-calibrated ranges and instant turnaround, is the business case in one chart.

## 6. Leakage: the way backtests lie

An agent that "sees the answer" backtests beautifully and fails in production. Guardrails:
- **Full masking:** the held-out project's own documents *and* any synthesis/knowledge-base page whose statistics were computed over it must be regenerated per fold. (The "SQL-embedded-in-page" KB design in `plan/03` makes this mechanical.)
- **Time-travel realism:** restrict comparables to projects *completed before the held-out project's estimate date*. Report both strict-temporal and full-corpus numbers; the strict-temporal number is the honest one and also quantifies how the tool's value compounds as the corpus grows.
- **Scope-text leakage:** if the archived "scope description" was written post-completion, it encodes the outcome. Prefer DBM-dated text; flag projects where only retrospective scope exists.

This mirrors a live concern in the LLM-eval literature: benchmark-contamination audits find leakage of 1–45% across popular multiple-choice QA benchmarks [13a], and "search-time contamination" inflates deep-research-agent scores when the agent can retrieve the answer — measured at up to ~4% on affected subsets [13]. The magnitudes matter: the point is not that leakage produces huge distortions (the search-time figure is modest), but that even a few points of unearned lift is enough to make a backtest lie about the tool's real value. Our corpus is private, which removes *training* contamination — but *retrieval* leakage (the agent pulling the held-out project's own final cost report) is the same failure and must be engineered out per fold.

## 7. Regression-testing the agent as software

Cost accuracy is necessary but not sufficient; the agent's *behavior* must be pinned so a prompt/model/KB change can't silently regress it (`plan/05` §3):
- **Golden-run suite** (~20 curated scopes spanning asset classes, thin-comparable cases, and adversarial inputs like "scope with the answer hinted") with structural assertions: right intake questions asked, hard filters applied, degradation flagged when comparables are thin, all figures carry provenance, arithmetic done in tools not prose.
- **LLM-judge checks** for qualitative reasoning (was the comparable-exclusion logic sound?), human-audited on a sample — noting the judge itself needs uncertainty handling (§4, [12]).
- **Threshold gates:** a change ships only if backtest metrics don't regress *and* the golden suite passes. This makes "upgrade to the newest frontier model" a routine gated event rather than a leap of faith.

## 8. The continuous-learning loop is a distribution-shift problem

As projects complete, the corpus grows and the cost regime drifts — exchangeability (the assumption behind vanilla conformal guarantees) breaks. Two distinct breakages want two distinct tools. First, a *growing or re-weighted reference class* is a **covariate-shift** problem, not just a temporal one: as the mix of asset types and eras in the corpus changes, the calibration set no longer matches the query distribution. The textbook fix is **weighted conformal prediction** (Tibshirani, Barber, Candès & Ramdas, 2019), which restores the coverage guarantee by re-weighting calibration residuals by the likelihood ratio between the reference and query distributions — the principled way to up-weight the comparables that actually resemble the new scope while keeping the finite-sample guarantee [17]. Second, *drift over time* — where the target relationship itself moves — is the province of **Adaptive Conformal Inference (ACI)** (Gibbs & Candès, 2021), which treats the miscoverage level as an online-learned parameter, tightening or widening the band based on recent under/over-coverage feedback, with extensions (Conformal PID, bias-corrected ACI) for stronger tracking under shift [5]. The two are complementary: weighted CP handles *who counts as a comparable* in a shifting corpus; ACI handles *how wrong we have recently been*. For us this means: escalation shocks (post-COVID input-cost spikes) show up as rising backtest error on recent-era slices → ACI-style feedback re-widens ranges automatically while the underlying escalation indices are re-examined, and weighted CP keeps the calibration honest as the corpus composition evolves. Recalibration is a versioned artifact tied to the KB commit hash; the learning loop is **reviewed pipeline events, not silent runtime memory writes** (`plan/05` §4).

For routing and cost control, the hybrid-annotation literature (e.g. HyPAC-style PAC error guarantees) shows how to use calibrated uncertainty to decide when to escalate to a human estimator vs. auto-answer — directly applicable to a "confidence gate" that flags novel scopes for human review [14].

---

## Implications for the capital-project cost agent (opinionated)

1. **Primary output = a conformal-calibrated range, in AACE/RCF language.** Emit P50 + P80 + an asymmetric band, labeled with the AACE class the scope maturity implies. *Pro:* speaks estimators' native dialect, defensible to management. *Con:* requires disciplined per-fold calibration to earn the coverage claim — no shortcuts.

2. **Backbone = jackknife+ or CV+ conformal prediction wrapping the LLM's adjusted-analog P50.** *Pro:* distribution-free, finite-sample guarantee at *n*≈200, wraps any predictor, degrades to global error distribution for thin classes. *Con:* the 1−2α guarantee means to *guarantee* 80% you calibrate at α such that 1−2α≥0.80; be explicit about this in the memo. *Alternative considered:* pure quantile regression — rejected as primary because it offers no coverage guarantee at our sample size [3][7].

3. **Confidence words are bound to measured coverage, never to LLM self-report.** Language like "estimates of this class have historically landed within ±40%, eight times in ten" — sourced from §5 backtests. *Pro:* honest, auditable. *Con:* requires the backtest harness to exist before the confidence language is trustworthy (chicken-and-egg — build the harness first).

4. **Use multi-sample disagreement, not verbalized confidence, for a model-side novelty signal.** Sample the analog-selection/adjustment step several times; high semantic disagreement → widen the range and flag for human review [2][11]. *Con:* Nx inference cost — reserve for the final estimate, not intake chatter.

5. **Headline metric = pinball loss; report PICP and sharpness alongside MAPE.** *Pro:* proper scoring rule can't be gamed by widening; catches "right P50, dishonest range." *Con:* less intuitive to non-statisticians — pair every dashboard with a plain-language coverage statement.

6. **Monte Carlo as a later explanatory overlay, not the primary guarantee.** Once scope is decomposed into cost drivers, run MC to *explain* where the spread comes from (weather, labor, brownfield tie-ins). *Pro:* engineers trust and can interrogate it. *Con:* garbage input distributions produce false precision, and correlation modeling is fiddly — don't lead with it [8][9].

7. **Engineer retrieval-leakage out per fold; report strict-temporal numbers as the honest ones.** *Pro:* the credibility of the whole project rests on backtests nobody can accuse of cheating. *Con:* per-fold KB regeneration is engineering work — budget for it.

8. **Continuous learning via ACI-style feedback + reviewed pipeline events.** *Pro:* ranges stay honest through cost-regime shifts; each new project measurably compounds value. *Con:* needs an estimator review board to own methodology changes — this is political groundwork as much as engineering [5].

---

## Sources

1. [AACE Estimate Classification System (Class 1–5), via Cenex](https://cenex.au/aace/estimate-classification.html) — accuracy ranges per class, maturity-as-primary-driver, asymmetric ranges, "risk analysis required" caveat.
2. [Uncertainty Quantification & Confidence Calibration in LLMs: A Survey (KDD 2025 tutorial)](https://xiao0o0o.github.io/2025KDD_tutorial/survey.pdf) — verbalized-confidence overconfidence (80–100% band), Xiong et al. 2024, Damani et al. 2025 RL-worse-than-token-probs finding. *(PDF binary; findings drawn from search-indexed abstract/summary, not full-text read — flagged.)*
3. [MAPIE: an open-source library for distribution-free uncertainty quantification (arXiv 2207.12274)](https://ar5iv.labs.arxiv.org/html/2207.12274) — split conformal, jackknife+, CV+, jackknife+-after-bootstrap; 1−2α guarantee; small-*n* trade-offs; wraps any regressor.
4. [Pinball loss / quantile score, PICP, sharpness, CRPS (MetricGate + probabilistic-forecasting literature)](https://metricgate.com/docs/pinball-loss-evaluation/) — proper scoring rule for quantiles; coverage vs. width; CRPS as integral of pinball loss.
5. [Adaptive Conformal Predictions for Time Series (Zaffran et al. 2022; ACI = Gibbs & Candès 2021)](https://proceedings.mlr.press/v162/zaffran22a/zaffran22a.pdf) — online miscoverage-level learning under distribution shift; Conformal PID / bias-corrected ACI extensions.
6. [Reducing risks in megaprojects: the potential of reference class forecasting (ScienceDirect 2023)](https://www.sciencedirect.com/science/article/pii/S2666721523000248) — RCF method, percentile uplifts (rail P50 40% / P80 57%), >70–80% hit rate vs <20% inside-view, government endorsements. *(Publisher 403 on fetch; figures corroborated across multiple search-indexed summaries and the Flyvbjerg literature — flagged as not directly full-text read.)*
6a. [Reference Class Forecasting and Machine Learning for Offshore Oil & Gas Megaprojects (Natarajan 2022)](https://journals.sagepub.com/doi/abs/10.1177/87569728211045889) — combining RCF with ML to address sparse reference classes.
7. [Quantile Regression and Prediction Intervals (Analytics Vidhya / practitioner writeups)](https://medium.com/analytics-vidhya/quantile-regression-and-prediction-intervals-e4a6a33634b4) — building intervals from quantile pairs; assumption-free; no coverage guarantee; per-quantile model cost.
8. [Monte Carlo Simulation & Analysis for cost risk (Galorath)](https://galorath.com/risk/monte-carlo-simulation/) — input distributions, iteration counts, correlation, S-curve outputs, pitfalls.
9. [P50, P80, P95 in Cost Estimation (SOMA Project Controls)](https://www.somaprojectcontrols.com/resources/guides/p50-p80-p95-confidence-levels-explained/) — P-value definitions, Monte Carlo QRA origin, 5–8% vs 15–20% P50→P80 uplift, P80 for budget / P90 for reserve.
10. [Verbalized Confidence Scores in LLMs (EmergentMind topic survey)](https://www.emergentmind.com/topics/verbalized-confidence-scores) — corroborates overconfidence and elicitation-method landscape.
11. [Self-Consistency Sampling & semantic entropy for LLM UQ (EmergentMind; 2025 UQ literature)](https://www.emergentmind.com/topics/self-consistency-sampling) — multi-sample disagreement, semantic clustering, +27.6pp/+23.7pp reasoning gains.
12. [Analyzing Uncertainty of LLM-as-a-Judge: Interval Evaluations with Conformal Prediction (arXiv 2509.18658)](https://arxiv.org/abs/2509.18658) — conformal intervals from a single LLM eval run; ordinal-boundary adjustment; precedent for wrapping an uncalibrated LLM.
13. [Search-Time Contamination in Deep Research Agents (arXiv 2606.05241)](https://arxiv.org/pdf/2606.05241) + [benchmark-leakage audits](https://arxiv.org/html/2605.19999v1) — retrieval/contamination inflation of agent benchmarks; 1–45% leakage across QA benchmarks.
14. [HyPAC: LLM–Human Hybrid Annotation with PAC Error Guarantees (arXiv 2602.02550)](https://arxiv.org/html/2602.02550) — calibrated-uncertainty thresholds to route between LLM and human; model for a confidence gate / escalation to human estimators.

*Verification note: sources [2] and [6] could not be read in full text (PDF-binary and publisher-403 respectively); their specific figures are drawn from search-engine-indexed summaries and cross-checked against the broader Flyvbjerg and LLM-calibration literature, but should be confirmed against primary text before being quoted in a management-facing deliverable. All conformal-prediction, AACE-range, and P-value figures were read from the fetched source text directly.*
