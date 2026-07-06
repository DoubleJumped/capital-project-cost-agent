# 04 — Capital Cost Estimation Practice: AACE, Reference Class Forecasting, and Normalization

*Research thread for the Capital Project Cost Agent — a prairie gas utility assistant that reasons over ~200 historical capital projects (DBMs, scope docs, lessons-learned, final cost reports) to produce defensible, cited cost estimates for new projects described in natural language.*

---

## TL;DR for this project

- **Every estimate the agent produces must self-declare an AACE estimate class (1–5) and carry the accuracy range that class implies.** A natural-language scope from an engineer is almost always Class 5 or Class 4 (0–15% project definition), so honest output is a wide asymmetric range like −30%/+50%, not a single number. Pretending to Class 3 precision from a paragraph of scope is the single biggest credibility failure mode. [1][7]
- **Our agent is, architecturally, a reference-class-forecasting (RCF) engine.** Flyvbjerg's method — find ≥10 comparable completed projects, anchor on their *actual* cost distribution, then adjust for project-specific differences — is exactly the "named comparables + range + evidence" output we want. The Karpathy-style curated knowledge base is the mechanism that makes the "outside view" cheap to compute. [2][5][9]
- **Optimism bias is the enemy we exist to counter.** Only ~0.5% of 16,000+ megaprojects hit budget/schedule/benefits; cost overrun distributions are *fat-tailed* (a minority of projects blow out catastrophically). The agent should anchor on **historical actuals, never on the new project's optimistic self-estimate**, and surface tail risk explicitly. [8][9]
- **Normalization is non-negotiable and must be explicit metadata.** Every historical cost must be normalized to a common basis before comparison: time (escalation via StatCan BCPI — which covers Saskatoon, Calgary, Edmonton, Winnipeg — or ENR CCI), location factor, scope/capacity, and currency. Store both the raw and normalized figure plus the index/factor used, so estimates are auditable. [4][6][11]
- **Contingency should be derived, not defaulted.** Prefer parametric/risk-driver contingency (AACE 42R-08) and Monte Carlo range estimating (AACE 41R-08) over a flat "add 15%." For a knowledge-base agent, the empirical spread of comparable projects' actual-vs-estimate ratios *is* a defensible contingency basis. [10][12]
- **Encode the domain's real cost drivers as first-class features:** compressor horsepower (with strong economies of scale — 2,000 hp costs ~3.2× per-hp of 30,000 hp), greenfield vs brownfield, station size, pipe diameter/length, and **season of construction**. Winter construction in Canada adds a commonly cited ~5–15% direct-cost premium / 15–20% productivity loss — but the literature is nuanced (well-planned winter work can offset this). [13][14][15]
- **Missing artifacts are the norm, not the exception.** The class/normalization framework degrades gracefully: a project with only a final cost report can still serve as a reference-class anchor even without a DBM. Track *data completeness* per project and let it widen the output range.

---

## 1. AACE estimate classification — the accuracy contract

AACE International's **18R-97** (process industries) and **17R-97** (generic) define a five-class system for cost estimates. The **sole primary characteristic is the maturity of project definition** (roughly, % of engineering complete); end usage, methodology, effort, and accuracy are *secondary* characteristics that follow from it. [1][7]

| Class | Project definition (maturity) | Typical end use | Primary methodology | Typical accuracy range (low / high) |
|-------|------------------------------|-----------------|---------------------|-------------------------------------|
| **5** | 0–2% | Concept screening / feasibility | Capacity-factored, parametric, judgment | −20%…−50% / +30%…+100% |
| **4** | 1–15% | Feasibility, concept evaluation | Equipment-factored, parametric | −15%…−30% / +20%…+50% |
| **3** | 10–40% | Budget authorization / funding | Semi-detailed unit costs, line items | −10%…−20% / +10%…+30% |
| **2** | 30–75% | Control baseline, bid evaluation | Detailed unit cost, forced quantities | −5%…−15% / +5%…+20% |
| **1** | 65–100% | Check estimate, bid/tender | Detailed take-off | −3%…−10% / +3%…+15% |

Two points matter enormously for us. First, **the ranges are asymmetric** — the upside (overrun) tail is always larger than the downside, which mirrors Flyvbjerg's empirical fat tails (§3). Second, AACE is explicit that **the published ranges are guideline indices only; a proper range for a specific estimate must come from quantitative risk analysis (Monte Carlo)**, not from reading a number off the table. [1][7]

An engineer typing a scope paragraph is providing Class 5 / Class 4 maturity. The agent's job is to squeeze the best possible Class 4/5 estimate out of a rich *historical* corpus — never to claim a class its inputs don't support.

## 2. Estimating methods: analogous, parametric, bottom-up

Three canonical methods map cleanly onto project maturity: [3]

- **Analogous (top-down):** find a completed project of similar scope/complexity/environment, take its actual cost, and adjust for known differences. Fast, cheap, low-precision; the natural technique for conceptual/early phases. **This is the agent's default mode.**
- **Parametric:** apply statistical cost-estimating relationships (CERs) — e.g. $/inch-mile of pipe, $/horsepower of compression, $/m² — derived from historical data. More rigorous than analogous when you have enough data points to fit a relationship; faster than bottom-up. **This is the agent's second gear once enough comparables exist to fit CERs.**
- **Bottom-up (detailed):** build the estimate from work-package quantities × unit rates and sum. Highest accuracy, highest effort, needs mature scope. **Out of scope for an early-stage NL agent** — but the agent should recognize when a historical project's final cost report *was* bottom-up and weight it as a higher-quality data point.

Best practice: analogous/parametric estimates should be **validated or replaced** by more detailed methods as definition matures. Our agent lives permanently in the analogous/parametric regime and should say so. [3]

## 3. Reference class forecasting & optimism bias (Flyvbjerg)

Reference class forecasting (RCF) is the intellectual core of this project. Developed from Kahneman & Tversky's "outside view" and operationalized by **Bent Flyvbjerg**, it has three steps: [2][5]

1. Identify a **reference class** of past, similar projects — broad enough to be statistically meaningful, narrow enough to be genuinely comparable.
2. Build a **probability distribution** of outcomes (e.g. cost-overrun ratios) from credible empirical data for that class.
3. Position the new project within that distribution and adjust its estimate accordingly (an empirical "uplift").

The evidence that conventional estimating fails is overwhelming. In Flyvbjerg's database of **16,000+ projects, only ~0.5% hit budget, schedule, and benefits**; only ~47.9% meet budget or better; only ~8.5% meet budget *and* schedule. [8] Overruns are **fat-tailed** — mean overruns and, crucially, the size of the upper tail vary sharply by type: [9]

| Project type | Mean cost overrun | % in fat tail | Fat-tail mean overrun |
|---|---|---|---|
| Solar | +1% | 2% | +50% |
| Wind | +13% | 7% | +97% |
| Tunnels | +37% | 28% | +103% |
| Rail | +39% | 28% | +116% |
| Buildings | +62% | 39% | +206% |
| IT | +73% | 18% | +447% |
| Nuclear | +120% | 55% | +204% |
| Olympics | +157% | 76% | +200% |

The lesson for cost *type* selection: **standardized, modular, repeatable projects (solar, wind) have thin tails; bespoke megaprojects have fat ones.** A prairie gas utility's routine station/pipe work is closer to the modular end — but brownfield tie-ins, regulatory surprises, and winter weather can create local fat tails. RCF's remedy is to **anchor on actuals and apply an empirical uplift for the specific reference class**, rather than trusting a bottom-up estimate that systematically ignores historical distributions. [2][5][9] RCF was first applied in practice on the 2004 Edinburgh Tram (a reference class of 46 rail projects) and is now embedded in UK, Danish, Australian, Dutch, Norwegian, Swiss and other government appraisal guidance. [2]

Caveat to record honestly: RCF has critics who argue the planning fallacy is not the *sole* driver of overruns (strategic misrepresentation, scope change, and principal–agent problems also matter), and that reference-class construction is itself subjective. Our agent's transparency (named comparables + cited evidence) is exactly the mitigation these critics ask for. [2][5]

## 4. Cost normalization — the unglamorous foundation

Comparing a 2009 project cost to a 2026 estimate without normalization is malpractice. Four axes: [4][6]

**(a) Time / escalation.** Restate historical costs to a common date using a construction cost index.
- **Statistics Canada Building Construction Price Index (BCPI):** quarterly, base 2017=100 (2015 for non-residential/high-rise), covering **11 CMAs including the prairie cities Saskatoon, Calgary, Edmonton and Winnipeg**. Explicitly intended for "making adjustments to project costs for escalation." Best default for Canadian building-type work. [11]
- **ENR Construction Cost Index (CCI) / Building Cost Index (BCI):** a labor+materials weighted index published monthly; covers 20 US cities plus **Montreal and Toronto** in Canada. Useful as a cross-check and for US-sourced equipment, but not prairie-specific. [4][6]
- **Gap flag:** StatCan's BCPI is *building*-oriented. Linear infrastructure (pipelines, stations) is better matched by upstream/midstream capital cost indices (e.g. the IHS/S&P Global *UCCI/IPIC* pipeline construction indices) — **I could not verify a free, prairie-specific pipeline construction index in this pass; treat this as an open data-sourcing task.**

**(b) Location factor.** An overall multiplier translating all cost elements (labor productivity/rates, freight, taxes, engineering) from one geography to another. AACE **28R-03** covers deriving location factors by factoring. Note: a location-factored estimate is itself considered Class 5 (±35% at best) — location factors normalize, they don't add precision. For our single-utility, single-region corpus, inter-project location variation is small but **remoteness/access** matters. [16]

**(c) Scope / capacity normalization.** Reduce projects to comparable units before comparing — $/hp, $/inch-mile, $/station of a given size — so a 5,000 hp and 30,000 hp station can inform each other via the CER rather than raw totals. Beware **economies of scale**: unit cost is strongly non-linear (see §6). [17]

**(d) Currency / basis.** Record currency, whether cost is overnight vs including AFUDC/escalation/owner's costs, and the estimate's contingency treatment. Comparing "all-in" to "construction-only" costs silently corrupts a reference class.

**Design consequence:** normalization must be *stored metadata on every historical cost*, not computed on the fly — persist raw value, index used, index values at source/target dates, location factor, capacity basis, and normalized value, so any estimate is fully auditable.

## 5. Contingency methods

AACE recognizes several contingency approaches of increasing sophistication: [10][12]

1. **Deterministic (predetermined %):** flat add-on (e.g. 15%). Simple, defensible only as a fallback; ignores project-specific risk.
2. **Expert judgment / risk-register additive:** sum of line-item risk allowances.
3. **Parametric / systemic (AACE 42R-08):** regress historical contingency draw against systemic risk drivers (scope definition level, technology novelty, team experience). Good for early-phase, data-driven contingency.
4. **Range estimating + Monte Carlo (AACE 41R-08):** replace point values on the **critical few** cost items with three-point (min/most-likely/max) or PERT/triangular distributions, model correlations between them, and simulate to get a total-cost probability distribution. Contingency = (P50 or P70/P85 target) − base estimate. [12]

For our agent, the **most defensible and native contingency signal is empirical**: the distribution of *actual ÷ estimate* (or actual-vs-normalized-comparable) ratios across the matched reference class **is** a Monte-Carlo-free, RCF-consistent contingency basis. Where enough comparables exist, we can also fit a parametric contingency model on risk drivers (greenfield/brownfield, season, scope maturity). Monte Carlo is worth offering as an explainable overlay, but the reference-class spread is the honest core.

## 6. Gas-utility / pipeline / compressor-station cost drivers

Domain-specific benchmarking (patchier public data, but usable): [17][18]

- **Compressor stations** are commonly modeled parametrically on **horsepower**, with pronounced **economies of scale**: the study behind OGJ's regression models found a 2,000 hp station costs **~3.2× per-hp** what a 30,000 hp station does, and unit total cost for a 5,000 hp station ranged ~$2,097/hp (Central US) to ~$2,820/hp (Western US) — regional spread ~34%. Model form: nonlinear regression on capacity + capacity² with regional dummies. [18]
- **Cost components** for compressor stations: material, labor, miscellaneous (survey, engineering, supervision, interest, admin, contingency, freight, taxes, regulatory), and land. A 220-project study found average overruns of ~+3% material, **+60% labor**, +2% misc, −14% land, **+11% total** — i.e., **labor is the dominant overrun risk**, which points straight at productivity/weather drivers. [17]
- **Rule-of-thumb benchmarks (verify before use):** onshore compressor & equipment ~US$1,500/demand-hp; a full onshore compressor station ~US$25M for site/buildings/non-compression equipment. These are order-of-magnitude anchors, not prairie-calibrated. [19]
- **Pipelines** are classically parametrized as **$/inch-mile** (or $/diameter-inch-metre); I could **not verify current prairie $/inch-mile figures** in this pass — flag as data to mine from our own corpus, which is exactly the kind of CER the knowledge base should learn.
- **Named benchmarking practice:** Solomon Associates runs a *Natural Gas Transmission Pipeline Systems Benchmarking Study* — evidence that peer benchmarking is standard midstream practice and a model for what our internal knowledge base does. [20]

## 7. Winter construction premiums (Canada)

Season of construction is a first-class cost driver on the prairies. The evidence is real but nuanced: [13][14][15]

- **NRC Canadian Building Digest (CBD-7):** the **usual extra-cost figure is 5–10%**, from tarpaulins, heaters, fuel, snow removal, insulation. But it stresses that **well-planned winter work can recover this** through uninterrupted schedules and avoided spring rain — winter is not automatically a penalty. It even notes spring labor can run $3–4k higher per $100k of work. [13]
- **Regional practitioner guidance (Calgary):** commonly assumes **15–20% productivity loss** in winter and recommends a **10–15% weather contingency**. [14]
- **Net:** encode season as a feature and derive its premium *from our own corpus* where possible (the disagreement between 5–10% and 15–20% across sources means a learned, project-specific factor beats any hard-coded constant). Winter *ground* conditions (frozen soil can aid access on muskeg) cut both ways, reinforcing that this must be data-driven, not a fixed uplift.

## 8. AI/LLM angle (2024–2026)

The agent design aligns with active research: NLP ensembles aligning quantity take-offs to cost indexes (2025), RAG + fine-tuning outperforming baseline GPT-4 for structured construction risk/knowledge tasks, and LLMs shown to beat classical ML baselines for early-stage estimation *especially because they tolerate missing features* — directly relevant to our heterogeneous, gap-ridden corpus. The consistent finding: **retrieval over a curated corpus beats naive generation**, validating the Karpathy "curated knowledge base the agent reads" thesis over raw RAG-over-everything. [21][22]

---

## Implications for the capital-project cost agent

1. **Make AACE class a mandatory output field.** Every estimate declares its class, the project-definition maturity it inferred, and the asymmetric accuracy range that follows. Default new NL scopes to Class 5/4. *Pro:* instant credibility with cost engineers and auditors. *Con:* users wanting a single number may find wide ranges unsatisfying — mitigate by also reporting a point (P50) plus the range.

2. **Architect the core as an RCF engine, not a generative guesser.** Pipeline: parse scope → match a reference class of ≥5–10 normalized historical projects → anchor on their actual-cost distribution → apply drivers (greenfield/brownfield, season, size, hp, diameter) → output range + named comparables + citations. *Pro:* directly counters optimism bias, matches the deliverable. *Con:* thin reference classes (rare project types) degrade it — detect and warn when <5 good comparables exist, and widen the range.

3. **Persist normalization as metadata on every cost.** Store {raw cost, currency, cost date, index used + values, location factor, capacity basis, normalized cost, cost-basis flags}. *Pro:* auditable, defensible, re-runnable when indices update. *Con:* upfront ingestion effort and requires choosing indices — default to StatCan BCPI (has Saskatoon/Regina-region prairie coverage) with ENR CCI cross-check; flag pipeline-specific index as an open sourcing task.

4. **Derive contingency from the reference class, offer Monte Carlo as an overlay.** Primary contingency = empirical spread of actual/estimate ratios in the matched class (RCF-native). Secondary = parametric model on risk drivers (AACE 42R-08 style). Tertiary = explainable Monte Carlo range estimate (AACE 41R-08) on the critical-few items. *Pro:* every number traces to evidence. *Con:* small samples make the empirical spread noisy — blend with a parametric prior and report sample size.

5. **Encode domain cost drivers as structured features with learned CERs.** Compressor hp (non-linear, economies of scale), pipe diameter × length ($/inch-mile), station size, greenfield/brownfield, and season. Mine our own corpus for the CERs rather than importing generic $/hp rules (which vary 34%+ regionally and aren't prairie-calibrated). *Pro:* self-calibrating to the utility. *Con:* ~200 projects is modest for stable regressions — prefer robust/median methods and always fall back to nearest-analog when a CER is under-supported.

6. **Treat season and labor as the top overrun risks.** Compressor-station data shows labor overran ~+60% on average while material was flat — so the agent should weight **labor-productivity and weather drivers heavily** and surface a season-specific premium derived from our data (not a hard-coded 10%). *Pro:* targets the real fat-tail source. *Con:* requires season/weather metadata on historicals, which older DBMs may lack — infer from construction dates where missing.

7. **Degrade gracefully on missing artifacts.** A project with only a final cost report is still a valid RCF anchor; one with DBM + lessons-learned is a richer, higher-weight anchor. Carry a per-project *data-completeness score* that both weights its influence and widens the output range when the matched class is thin or poorly documented.

8. **Surface tail risk, never hide it.** Because overruns are fat-tailed, report P50 *and* P80/P90, and name which comparable projects sit in the tail and why (weather, brownfield surprise, regulatory delay) — this is the "cited evidence" the deliverable promises and the exact discipline RCF and AACE 41R-08 demand. [9][12]

---

## Sources

1. [Cenex — AACE Cost Estimate Classification (Class 1–5)](https://cenex.au/aace/estimate-classification.html) — full 18R-97 class table: maturity %, methodology, asymmetric accuracy ranges; ranges are guidelines requiring project-specific risk analysis.
2. [Flyvbjerg — Curbing Optimism Bias… Reference Class Forecasting in Practice (ResearchGate)](https://www.researchgate.net/publication/233258056_Curbing_Optimism_Bias_and_Strategic_Misrepresentation_in_Planning_Reference_Class_Forecasting_in_Practice) — RCF three-step method, Edinburgh Tram 2004 first application, government adoption.
3. [Cleopatra Enterprise / Galorath / PM Vidya — analogous vs parametric vs bottom-up](https://cleopatraenterprise.com/blog/which-is-better-parametric-or-analogous-estimating/) — method definitions, maturity mapping, accuracy/effort tradeoffs.
4. [Engineering News-Record — Construction Economics / CCI](https://www.enr.com/economics) — ENR CCI/BCI as labor+materials escalation indices; 20 US cities plus Montreal & Toronto.
5. [Reference class forecasting: promises, problems, and a research agenda (Cranfield/T&F 2025)](https://dspace.lib.cranfield.ac.uk/server/api/core/bitstreams/076d9c29-659d-4bfd-9b1b-e0019ea9ad20/content) — modern review of RCF strengths and critiques.
6. [ENR — Using ENR Indexes (FAQ)](https://www.enr.com/economics/faq) — how CCI/BCI are composed and used to escalate/normalize between years.
7. [AACE 18R-97 TOC (aacei.org)](https://web.aacei.org/docs/default-source/toc/toc_18r-97.pdf) — authoritative source that maturity of project definition is the sole primary characteristic of estimate class.
8. [BudgetOverrun.com — Flyvbjerg Megaproject Database](https://budgetoverrun.com/studies/flyvbjerg-megaproject-database) — 16,000+ projects; only ~0.5% hit budget/schedule/benefits; overrun-by-type table with fat-tail percentages.
9. [Flyvbjerg — Five Things You Should Know About Cost Overrun (Medium)](https://bentflyvbjerg.medium.com/five-things-you-should-know-about-cost-overrun-d2bce69d6f51) — fat-tailed overrun distributions, behavioral-bias root cause, RCF remedy.
10. [AACE — RP 118R-21 announcement (Source)](https://source.aacei.org/2022/11/08/new-aace-recommended-practice-fills-gap-in-quantitative-risk-analysis-maturity-modelnew-rp-118r-21-cost-risk-analysis-and-contingency-determination-using-estimate-ranging-for-inherent-risks/) — the four AACE contingency methods; 41R-08 vs 42R-08 sophistication.
11. [Statistics Canada — Technical Guide for the Building Construction Price Index](https://www150.statcan.gc.ca/n1/pub/62f0014m/62f0014m2022005-eng.htm) — BCPI base 2017=100, 11 CMAs incl. Saskatoon/Calgary/Edmonton/Winnipeg, used for cost escalation.
12. [AACE 41R-08 — Risk Analysis & Contingency via Range Estimating (WSDOT PDF)](https://wsdot.wa.gov/sites/default/files/2021-12/risk-analysis-contingency-RangeEstimating.pdf) — Monte Carlo on critical-few, three-point distributions, correlations, P50/P75/P85 contingency.
13. [NRC Canadian Building Digest CBD-7 — Winter Construction](https://web.mit.edu/parmstr/Public/NRCan/CanBldgDigests/cbd007_e.html) — usual 5–10% extra cost; nuance that well-planned winter work offsets it.
14. [Good Earth Builders — Winter Construction in Calgary](https://goodearthbuilders.ca/winter-construction-in-calgary-how-to-avoid-delays-cost-overruns/) — 15–20% winter productivity loss, 10–15% weather contingency (practitioner guidance).
15. [FTI Consulting — Winter Construction: Recover Delays and Cost Overruns](https://www.fticonsulting.com/insights/articles/winter-construction-strategies-recover-delays-cost-overruns) — winter delay/overrun mechanisms and mitigation.
16. [AACE 28R-03 TOC — Developing Location Factors by Factoring](https://web.aacei.org/docs/default-source/toc/toc_28r-03.pdf) — location factor definition; a location-factored estimate is Class 5 (±35% at best).
17. [Pipeline compressor station construction cost analysis (ResearchGate)](https://www.researchgate.net/publication/275590240_Pipeline_compressor_station_construction_cost_analysis) — 220-project dataset; five cost components; +60% avg labor overrun, +11% total.
18. [Oil & Gas Journal — Regressions for compressor cost estimation models](https://www.ogj.com/home/article/17298392/regressions-allow-development-of-compressor-cost-estimation-models) — nonlinear regression on hp; economies of scale (2,000 hp ≈ 3.2× per-hp of 30,000 hp); $2,097–2,820/hp regional spread.
19. [JM Campbell — Onshore/Offshore Gas Pipeline Capital Cost Comparisons](http://www.jmcampbell.com/tip-of-the-month/2013/03/onshore-natural-gas-pipeline-transportation-alternatives-capital-cost-comparisons/) — rule-of-thumb ~US$1,500/hp and ~US$25M/station (order-of-magnitude anchors).
20. [Solomon Associates — Natural Gas Transmission Pipeline Systems Benchmarking Study](https://www.solomoninsight.com/industries/midstream/benchmarking/natural-gas/natural-gas-transmission-systems-study) — evidence that peer benchmarking is standard midstream practice.
21. [AI-augmented construction cost estimation: ensemble NLP aligning QTOs with cost indexes (T&F 2025)](https://www.tandfonline.com/doi/full/10.1080/15623599.2025.2558070) — recent NLP method linking take-offs to cost indexes.
22. [LLMs for predicting cost and duration in engineering projects (arXiv 2024)](https://arxiv.org/pdf/2409.09617) — LLMs beat classical ML baselines for early estimation, robust to missing features.

*Verification flags: pipeline-specific prairie escalation index and current $/inch-mile benchmarks were NOT verified in this pass — treat as corpus-mining / data-sourcing tasks. The $1,500/hp and $25M/station figures are dated (2013) US order-of-magnitude anchors, not prairie-calibrated. AACE per-class accuracy numbers vary slightly between secondary summaries; the 18R-97/17R-97 primary documents are paywalled and should be obtained for the authoritative table.*
