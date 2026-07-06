# 07 — Prior Art Landscape: Commercial Tools & Published Deployments of AI Cost Estimation

*Research chapter for the capital-project cost agent. Scope of this thread: who has already built
AI/ML systems for capital cost (and schedule) estimation from historicals — commercial vendors,
utility/energy case studies, government cost-estimating doctrine, and the academic ML-for-cost
literature — and what lessons transfer to our ~200-project gas-utility estimating agent.*

---

## TL;DR for this project

- **Nobody has publicly shipped exactly what we are proposing** — an *analogical, evidence-cited,
  LLM-legible* whole-project estimator over a utility's own heterogeneous corpus. The market splits
  into (a) computer-vision *takeoff* tools (Togal, Kreo, STACK) that count doors and walls off
  drawings, and (b) statistical/ML *parametric* estimators (Cleopatra, InEight, academic RF/XGBoost
  models). Our niche — reasoning over messy DBMs/lessons-learned to name comparables — is genuinely
  open. That is an opportunity and a warning (no template to copy).
- **The strongest conceptual precedent is Flyvbjerg's Reference Class Forecasting (RCF)** [12][13]:
  find similar past projects, build an empirical distribution, adjust. This *is* our method with an
  LLM doing the similarity matching. RCF has a documented accuracy track record and is the
  defensibility story we should lean on — frame the agent as "automated RCF," not "AI black box."
- **At n≈200, ensembles beat neural nets.** The ML-for-cost literature is consistent: Random Forest /
  XGBoost / LightGBM give the best accuracy-stability balance on small tabular datasets; deep
  learning (LSTM/Transformer) is repeatedly noted as *unusable* at these sample sizes [7][8]. Any
  numeric model we build should be a gradient-boosted ensemble, not a neural net.
- **Realistic accuracy is ~10–15% error early-stage, and that already matches AACE expectations.**
  Class 5/4 concept estimates carry −50%/+100% to −30%/+50% *by definition* [14]. We should not
  promise point precision; we should promise well-calibrated *ranges* and cited comparables.
- **Explainability is the adoption gate, not accuracy.** Only ~12% of estimators report strong
  confidence in estimation tools, and black-box outputs *erode* credibility in regulated
  industries [18][19]. Our cite-the-evidence design is not a nice-to-have — it is the single biggest
  differentiator versus every incumbent.
- **Data quality kills ~40% of these projects** [19]. Our heterogeneous, partly-missing corpus is the
  central risk. Budget most effort for the ingestion/normalization layer (the "wiki" distillation),
  not the model.
- **Build-vs-buy leans build for the reasoning layer, buy for commodity pieces.** No vendor ingests
  *your* DBMs and cites them. But don't rebuild takeoff CV or a parametric benchmarking DB from
  scratch if a station-cost regression already suffices.

---

## 1. The commercial landscape, and why most of it isn't what we need

### 1.1 Computer-vision takeoff tools (the loud, well-funded corner)

The most visible "AI construction estimating" startups are really **quantity-takeoff** engines: they
read PDF/BIM drawings and auto-count rooms, walls, doors, and fixtures.

- **Togal.AI** — claims ~97–98% accuracy on *space/floor-plan detection*; independent testing had it
  complete an architectural takeoff in ~12 minutes; Growth plan ~$299/mo [3][4].
- **Kreo** — "Caddie" agent reads drawings and produces a bill of quantities; got testers "95% of the
  way" on BIM workflows; from ~$35/mo [4][6].
- **Handoff AI** — trained on 100,000+ residential projects and 60M+ SKUs; vendor claims residential
  estimates within ~$100 of manual bids; from ~$149/mo [5]. (Residential-only, vendor-reported.)
- **STACK, Procore Estimating, Beam AI** — similar CV-assist takeoff plus assemblies.

A February 2026 independent test of six tools (InEight, Togal, Procore, STACK, Kreo, Beam) is the most
useful evidence here [4]: **InEight Estimate hit 1.8% error** against a ground-truth baseline (best of
the six); Procore landed within ~4%, STACK within ~3%, Beam had the second-lowest miss rate. Crucially:
**every tool required human validation, and all degraded on hand-sketched plans, heavy-civil
cross-sections, and non-standard symbols.**

**Relevance to us: low-to-moderate.** These tools answer "how many linear metres of wall are on this
drawing," not "what will this station cost given 30 years of similar builds." They presuppose a
finished design; we operate at *concept/scope* stage where no drawings exist. The transferable lesson
is the failure mode: **CV/automation collapses exactly on the messy, non-standard, older artifacts
that dominate our corpus.** Our value has to come from reasoning over prose (DBMs, lessons-learned),
which these tools don't touch.

### 1.2 Incumbent cost-engineering suites (the parametric, enterprise corner)

These are the tools a utility cost engineer already knows:

- **Cleopatra Enterprise** (Cost Engineering / Hexagon) — parametric estimating + benchmarking for
  capital-intensive industries (oil & gas, energy, infrastructure); centralizes estimates
  feasibility→execution [17].
- **InEight Estimate** (ex-Hard Dollar) — probabilistic analysis, benchmarking, historical-DB-driven
  estimating with BIM/quantity integration [17].
- **CostOS** (Nomitech) — 5D BIM conceptual-to-detailed estimating and benchmarking [17].
- **RIB Candy** — estimating, QTO, planning, cash-flow for civil/build.

Their "AI" today is mostly **outlier flagging and benchmarking against a normalized historical
database** — valuable but conventional. None of them, as far as public materials show, ingest
free-text DBMs/lessons-learned and produce a natural-language, cited estimate. *(Vendor AI-feature
claims move fast and are marketing-heavy; I could not verify a specific "generative estimate from
scope prose" feature in any of them — flag as unverified.)*

**Relevance to us: high as a benchmark and integration target.** These define what "defensible" looks
like to a utility (normalized cost databases, benchmarking, probabilistic ranges). If SaskEnergy
already licenses one, the agent should *feed* it or sit beside it, not replace it. Build-vs-buy note:
the parametric-benchmarking *storage* layer is a solved commodity; the *reasoning-over-prose* layer is
not.

### 1.3 nPlan — the closest "learn from historicals" precedent (but schedule, not cost)

**nPlan** trains deep-learning **graph neural networks** on the largest known corpus of construction
schedules — **750,000+ historical schedules, >2 million years of construction time** — to forecast
activity-level durations and risk, and to flag where a schedule will "snowball" [1][2]. It explicitly
positions itself *against* manual Reference Class Forecasting: automate the outside view at activity
granularity. Deployed on £11B+ infrastructure portfolios.

Important caveats: nPlan forecasts **schedule/delay, not cost**; its accuracy is **vendor-claimed with
no public third-party benchmark** [1]; and its power comes from a **massive cross-client dataset** we
do not have (it trains on customer schedules "with permission").

**Relevance to us: high — conceptually the template, cautionary on data.** nPlan proves the "learn the
outside view from historicals and forecast a distribution" thesis is commercially real. But it also
shows the thesis usually needs *hundreds of thousands* of examples. We have ~200. That gap is the
crux of our design: we cannot brute-force a GNN; we must compensate with (a) an LLM that reasons
qualitatively over rich per-project prose, and (b) explicit analogical retrieval — which is exactly
what RCF does with far fewer samples.

---

## 2. The academic evidence: what actually works at n≈200

The ML-for-construction-cost literature is large and directly informative, because **sample sizes of
a few hundred are the norm** there, not the exception.

### 2.1 Model choice — ensembles win on small tabular data

A 2026 scientometric review of ML/AI in construction cost prediction [7] gives a clear ordering:

- **Artificial Neural Networks (ANN)** were the early standard — "high" accuracy but "low"
  interpretability.
- **Ensemble methods (Random Forest, XGBoost, LightGBM)** now dominate and "achieve a relatively good
  balance between accuracy and stability."
- **SVM/SVR** shows "strong generalization" specifically on *small-sample* scenarios.
- **Deep learning (LSTM, Transformer)** is repeatedly flagged as **not yet usable** because datasets
  are too small — the review explicitly says studies "have not yet employed" them for this reason.

Reported accuracy: early-stage models typically control error to **10–15%**; advanced ensembles can
cut error **>50% versus traditional methods**; a green-building study reported **XGBoost R²≈0.96** [7].

### 2.2 Energy-sector case studies (most transferable)

- **Power-plant construction cost, Random Forest + SHAP** [8]: RF beat SVR and KNN, reaching
  **R²=0.956** and lowest RMSE (29.27) on test, with SHAP explaining the contribution of size,
  material, and labor drivers. *(Paper's exact sample size not stated in the sources I could access —
  flag as unverified; the MDPI full text returned 403.)* The pattern — **RF + SHAP for an
  explainable, energy-sector cost model** — is almost exactly a numeric backbone we could adopt.
- **Engineered-to-order gas-turbine parts, oil & gas** [10]: **gradient-boosted trees** were the best
  algorithm; the ML parametric model achieved **~7% error vs ~9%** for the prior method, in **minutes
  vs hours**. Confirms GBTs + parametric framing for energy hardware. *(ScienceDirect full text
  403'd; details from the abstract/search snippet.)*
- **Pipeline regression (directly gas-utility relevant)** [11]: classic regression on **180 pipelines
  and 136 compressor stations** produced cost equations by diameter, length, region, and year;
  average cost shares were **material 50.6% / labor 27.2% / misc 21.5% / ROW 0.8%** for pipelines,
  with **material dominant (~50%) for compressor stations**. Interstate-pipeline work (1980–2017)
  built capital-cost equations as f(diameter, length, U.S. region) split into four cost components.
  This is the kind of **structured feature set and cost-component decomposition** our knowledge base
  should capture per project.

### 2.3 The honest caveats the literature repeats

- Datasets are small and regional (e.g., 320 samples), so models **generalize poorly across regions/
  project types** [7]. Our single-utility, single-province corpus is actually a *benefit* here —
  homogeneous context reduces the generalization gap.
- Construction data typically carries **~7% missing values and ~5% outliers** [7] — matches our
  "some artifacts missing, formats vary across eras" reality. Robustness to missingness is mandatory.
- **Interpretability is a named limitation**: only ~23% of professionals claim to understand DNN
  logic [7]. This is a recurring theme (see §4).

---

## 3. Government & institutional doctrine: the defensibility playbook

For a regulated utility, the estimate has to survive scrutiny. Public-sector cost-estimating doctrine
is the gold standard for *how to be defensible*, and it maps cleanly onto our design.

- **AACE 18R-97 estimate classes** [14] define accuracy by project definition:
  Class 5 (ROM, 0–2% defined) **−50%/+100%**; Class 4 (feasibility, 1–15%) **−30%/+50%**; Class 3
  (budget/control, 10–40%) **−20%/+30%**. FEL 1/2/3 ≈ Class 5/4/3. **Design implication: the agent
  must output a class, and its range must widen/narrow with how much scope the user provided.** A
  point estimate at concept stage is malpractice.
- **GAO Cost Estimating and Assessment Guide (GAO-20-195G)** [15] and the **DOD/DOE/NASA** guides
  codify four methods — **analogy, build-up (engineering), parametric, extrapolation-from-actuals** —
  and 12 steps for a credible estimate. Our agent is fundamentally an **analogy + parametric** engine.
- **Cost Estimating Relationships (CERs)** [16]: statistical equations linking cost to technical/
  programmatic drivers, built by NCCA/Tecolote-style handbooks. These are the formal name for what
  our numeric backbone produces (cost = f(station size, greenfield/brownfield, season, length…)).
- **Flyvbjerg's Reference Class Forecasting** [12][13] is the crown jewel of precedent. Cost overruns
  on major projects are **fat-tailed** — 50–100% overruns "common," >100% "not uncommon" — and the
  root cause is **behavioral (optimism bias + strategic misrepresentation)**, not analytical [12]. The
  cure is the **outside view**: (1) identify a reference class of similar past projects, (2) build the
  empirical outcome distribution, (3) adjust the current estimate by an uplift or relative-risk
  judgment [13]. First real use: Edinburgh Tram 2004, £320M → £400M at the 80% confidence level.

**This is our system, described 20 years early.** The agent's pitch to a utility should be: *"an
automated, auditable Reference Class Forecaster over your own project history."* That framing
inherits RCF's documented credibility and sidesteps the "AI black box" objection.

---

## 4. Why these systems succeed or fail — and the adoption barrier

The failure literature is remarkably consistent, and it is *organizational*, not algorithmic.

- **The adoption paradox**: 87% of contractors think AI will meaningfully affect the industry, but
  only ~4% use it [19]. Tools that estimators don't trust "sit unused and never pay back."
- **Data quality derails ~40% of construction AI implementations** [19]. Galorath's postmortem [18]
  is blunt: "AI applied to fragmented data simply automates flawed assumptions." 56% of orgs run
  partially-integrated systems, 47% cite inaccurate forecast data as a top challenge.
- **Explainability is the trust gate**: only ~12% of professionals report strong confidence in
  estimation accuracy; **"black-box AI erodes rather than builds credibility"** and regulated
  industries "require auditable, defensible estimates… traceable assumptions and audit-ready
  outputs" [18]. This is the single most important lesson for us.
- **Ownership & integration**: pilots die when they sit isolated from ERP/financial systems and no one
  owns them [18]. Continuous-modeling orgs are 2.5x more likely to stay within budget [18] — static
  one-shot models rot.
- **Estimator resistance**: fear of job displacement and accountability when the model is wrong [19].
  Mitigation is positioning the agent as a *drafting/first-pass* tool the estimator overrides, not a
  replacement.

---

## 5. Implications for the capital-project cost agent (opinionated)

**5.1 Frame it as automated Reference Class Forecasting, not "AI estimating."**
Pros: inherits AACE/Flyvbjerg credibility, matches how utility estimators already think, defensible in
a rate case. Cons: sets an expectation of rigorous comparable-selection we must actually deliver. Do
this anyway — it is the whole defensibility story [12][13][15].

**5.2 The retrieval/similarity layer is the LLM's job; the numeric backbone should be a
gradient-boosted ensemble.**
The literature is unanimous that RF/XGBoost/LightGBM beat neural nets at n≈200 [7][8][10]. Design:
LLM reasons over prose to (a) select the reference class and (b) extract structured features
(station size, greenfield/brownfield, season, length, weather-delay flags); a small GBT/RF then
produces the point-and-range CER, with **SHAP for per-driver attribution** [8].
- *Alternative considered:* pure-LLM numeric estimation (no ML model). Pro: simplest, no training.
  Con: LLMs are weak, un-calibrated numeric predictors and can't give defensible intervals. Use the
  LLM for reasoning + retrieval, the ensemble for the number.
- *Alternative considered:* nPlan-style GNN. Con: needs 10³–10⁵ examples we don't have. Rejected.

**5.3 Invest most engineering in the ingestion/distillation layer (the "LLM wiki").**
Data quality is the #1 failure cause [18][19] and our corpus is the worst case (heterogeneous, era-
varying, partly missing). Distill each project into a normalized, LLM-legible record — scope summary,
cost breakdown by component (mirror the pipeline material/labor/misc/ROW split [11]), complexity tags,
season, weather-delay narrative, lessons-learned — *before* any estimation. This is the Karpathy
"curated agent memory" idea and it is where the value is, not in the model.

**5.4 Never output a bare number — output a calibrated range tied to an AACE class.**
Bind the width to scope-definition level [14] and to the empirical spread of the reference class
(fat-tailed, so report P50/P80, not mean ± symmetric σ) [12]. Widening ranges when comparables are
few or dissimilar is a feature, not a weakness.

**5.5 Cite everything; make the evidence chain the product.**
Every estimate must name its comparable projects and quote the source artifacts. This directly answers
the #1 adoption blocker — auditability [18] — and is what no incumbent does. It is our moat.

**5.6 Position as estimator-assist with human override, and design for continuous refresh.**
Counters resistance and the "static model rots" failure [18][19]. Log every override to improve the
knowledge base — a feedback loop, not a one-shot deploy.

**5.7 Build-vs-buy checklist.**
- *Buy/borrow:* parametric benchmarking storage (Cleopatra/InEight-style) if already licensed; any
  existing station-cost regressions; standard SHAP/GBT libraries.
- *Build:* the prose-ingestion "wiki" distiller, the LLM analogical-retrieval + reasoning layer, and
  the cited-evidence estimate report. **No vendor does these over *your* DBMs** — confirmed gap.
- *Check before building:* (1) Does an incumbent suite already hold normalized historicals we can read
  instead of re-parsing PDFs? (2) Can the numeric backbone be a documented CER an estimator can audit
  by hand? (3) Is there an integration path into the tool estimators already use (else it "sits
  unused" [19])? (4) Do we have a labeled outcome (final actual cost) for enough of the 200 to train
  and *validate* — without it, no calibration and no credibility.

---

## Sources

1. [nPlan Review — DataDrivenAEC](https://datadrivenaec.com/tools/nplan) — nPlan uses deep learning + GNNs on 750k+ schedules; forecasts schedule/delay not cost; accuracy vendor-claimed, no third-party benchmark; trains on client data with permission.
2. [nPlan official site](https://www.nplan.io/) and [AEC Magazine AI directory](https://aidirectory.aecmag.com/entry/nplan/) — 750,000+ schedules / >2M construction-years; activity-level risk forecasting; positioned vs reference class forecasting; £11B+ portfolios.
3. [Best AI Construction Estimating Software 2026 — Dan Cumberland Labs](https://dancumberlandlabs.com/blog/ai-construction-estimating-software/) and [Construction Placements 2026 list](https://www.constructionplacements.com/best-ai-estimating-software-construction/) — Togal/Kreo/Handoff/STACK feature and pricing landscape; CV takeoff focus.
4. [6 AI estimating tools tested on complex projects (Feb 2026) — Robotics & Automation News](https://roboticsandautomationnews.com/2026/02/19/6-ai-construction-estimating-software-tested-on-complex-project-accuracy/98967/) — InEight 1.8% error (best), Procore ~4%, STACK ~3%, Beam second; all needed human validation; all failed on hand-sketched/heavy-civil/non-standard.
5. [Handoff AI — 6 best AI estimating tools](https://handoff.ai/blog/6-best-ai-construction-estimating-software-2026-picks-compared) — Handoff trained on 100k+ residential projects + 60M SKUs, claims within $100 on residential (vendor-reported, residential-only).
6. [Kreo Software](https://www.kreo.net/) — "Caddie" autonomous takeoff agent; ~$35/mo; strongest on BIM; ~95% auto-takeoff in independent testing.
7. [ML/AI in construction cost prediction: scientometric review, Frontiers 2026](https://www.frontiersin.org/journals/built-environment/articles/10.3389/fbuil.2026.1867673/full) — ensembles (RF/XGBoost/LightGBM) best on small data; deep learning unusable at small n; SVM strong on small samples; 10–15% typical early error; ~7% missing / ~5% outliers; XGBoost R²≈0.96 green building.
8. [Explainable ML for power-plant construction cost, Random Forest + SHAP — MDPI](https://www.mdpi.com/2673-4109/6/2/21) — RF beat SVR/KNN, R²=0.956, RMSE 29.27; SHAP for driver attribution. (Full text 403'd; sample size unverified.)
9. [Transparent & reliable construction cost prediction with explainable AI — ScienceDirect](https://www.sciencedirect.com/science/article/pii/S2215098625002149) — advanced ML + explainable AI for auditable cost prediction. (Full text 403'd; cited from search abstract only.)
10. [ML cost modelling for engineered-to-order products — ScienceDirect](https://www.sciencedirect.com/science/article/pii/S0952197624011151) — gradient-boosted trees best; ~7% vs ~9% error, minutes vs hours; oil & gas parametric CERs. (Full text 403'd; from abstract/snippet.)
11. [Regression models estimate pipeline construction costs — Oil & Gas Journal / ResearchGate](https://www.ogj.com/refining-processing/gas-processing/new-plants/article/17213121/regression-models-estimate-pipeline-construction-costs) and [NG/H2 pipeline capital-cost equations — ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0360319922034048) — 180 pipelines + 136 compressor stations; cost shares material 50.6%/labor 27.2%/misc 21.5%/ROW 0.8%; equations f(diameter, length, region).
12. [Five Things You Should Know About Cost Overrun — Bent Flyvbjerg (Medium)](https://bentflyvbjerg.medium.com/five-things-you-should-know-about-cost-overrun-d2bce69d6f51) — overruns fat-tailed; root cause behavioral bias → scope underestimation; RCF as the cure with documented accuracy.
13. [Reference class forecasting — Wikipedia](https://en.wikipedia.org/wiki/Reference_class_forecasting) and [PMI on RCF](https://www.pmi.org/learning/library/nobel-project-management-reference-class-forecasting-8068) — three-step outside-view method; Edinburgh Tram 2004 £320M→£400M at 80% confidence.
14. [AACE Cost Estimate Classification (18R-97) — Cenex](https://cenex.au/aace/estimate-classification.html) and [EPCLand guide](https://epcland.com/cost-estimate-classification-system-for-process-industries/) — Class 5 −50/+100%, Class 4 −30/+50%, Class 3 −20/+30%; ranges set by risk analysis, tied to definition level.
15. [GAO Cost Estimating and Assessment Guide GAO-20-195G](https://www.gao.gov/assets/gao-20-195g.pdf) — four methods (analogy/build-up/parametric/extrapolation), 12 steps for credible estimates.
16. [Joint Agency CER Development Handbook (NCCA/Tecolote)](https://www.asafm.army.mil/Portals/72/Documents/Offices/CE/CER%20Developement%20Handbook.pdf) and [DOD Cost Estimating Guide](https://www.govinfo.gov/content/pkg/GOVPUB-D-PURL-gpo156710/pdf/GOVPUB-D-PURL-gpo156710.pdf) — CERs as statistical cost-vs-driver equations; parametric/analogous doctrine.
17. [CostOS vs Cleopatra benchmarking — Nomitech](https://www.nomitech.com/benchmarking/best-construction-cost-benchmarking-software-enterprise) and [Cleopatra Enterprise — Software Advice](https://www.softwareadvice.com/construction/cleopatra-enterprise-profile/) — incumbent parametric/benchmarking suites for capital-intensive/energy projects; AI features limited to outlier/risk flagging (generative-from-prose feature unverified).
18. [Five AI Adoption Mistakes Holding Back Cost Estimation — Galorath](https://galorath.com/blog/ai-mistakes-in-cost-estimation/) — fragmented data automates flawed assumptions; only 12% confident in estimates; regulated industries need auditable/traceable outputs; continuous modeling 2.5x more likely within budget.
19. [AI in Construction Estimating adoption barriers — Profound Estimates](https://profoundestimates.com/future/ai-in-construction-estimating-how-artificial-intelligence-will-reshape-cost-forecasting-by-2030) and [McCormick Systems](https://www.mccormicksys.com/blog/should-construction-estimators-be-concerned-about-ai-in-estimating/) — 87% believe AI matters / only ~4% use it; data quality derails ~40%; estimator resistance and black-box accountability as key barriers.
