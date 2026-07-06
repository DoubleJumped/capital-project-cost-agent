# 03 — LLM + Case-Based Reasoning for Analogous Cost Estimating

*Research thread 3 of 6 for the Capital Project Cost Agent. How to make an LLM agent
price a new prairie-gas capital project by retrieving, adapting, and rolling up
evidence from ~200 heterogeneous historical projects — grounded in current (2024–2026)
case-based-reasoning + LLM literature and in cost-engineering practice.*

---

## TL;DR for this project

- **Case-based reasoning (CBR) is the right spine for this system.** The classic CBR
  cycle — Retrieve → Reuse → Revise → Retain — maps almost 1:1 onto our problem:
  retrieve comparable projects, reuse their costs, adjust for scope/size/season/greenfield
  differences, and retain the new project once actuals land. Modern "CBR-augmented LLM"
  papers ([1][2][4]) formalize exactly this and show it beats vanilla RAG on
  knowledge-intensive, evidence-requiring tasks — which is precisely a defensible cost estimate.
- **Do NOT ask the LLM to be the regression engine.** Multiple 2025 studies find LLMs are
  unreliable numeric predictors on tabular data: negative R² vs classical models above
  0.95, and >80% swings in error from trivial prompt changes ([6][7]). The LLM's job is
  *qualitative adjustment and reasoning over retrieved cases*, not raw number-crunching.
- **The winning pattern is hybrid: statistical/parametric baseline + LLM narrative
  adjustment on top.** The "last-mile forecasting" architecture ([8]) — an immutable
  statistical baseline that an LLM agent revises through *constrained, logged, evidence-cited
  actions* — is the single best structural template we found. It cut forecast MAE 80–88%
  while staying fully auditable. We should copy it.
- **Represent each historical project as a structured case tuple `(P, S, O, M)`** — Problem
  (scope/attributes), Solution (how it was executed + cost breakdown), Outcome (final cost,
  variance, what went wrong), Metadata (era, region, source docs). This is the consensus
  representation across the CBR-LLM literature ([4]) and it is what makes the estimate cite-able.
- **Adaptation must be explicit and quantitative.** Use engineering adaptation rules where
  they exist (capacity-factor scaling with exponent ~0.5–0.85, seasonal/winter uplift,
  escalation to today's dollars) and reserve the LLM for the residual qualitative delta.
  "Nearest neighbour + an explicit statement of the differences" produced the *highest user
  trust* in CBR-LLM triage studies ([4]).
- **This is Reference Class Forecasting in agent form.** Flyvbjerg's RCF ([5]) — take the
  outside view from a distribution of similar past projects to counter optimism bias — is the
  authoritative cost-engineering justification for what we're building. Frame the whole system
  as "automated RCF with case-level traceability."
- **Anchor accuracy expectations to AACE estimate classes.** With NL scope only, we live in
  AACE Class 5→4 territory: realistic accuracy is **−50%/+100% down to −30%/+50%** ([9]).
  Our ranges must be *asymmetric* (overruns more likely than underruns) and the UI must say
  which class the estimate is.

---

## 1. What CBR is, and why LLMs revived it

Case-Based Reasoning is a decades-old AI paradigm: solve a new problem by retrieving similar
past problems and adapting their solutions, rather than reasoning from first principles or
fitting a global model. Its canonical loop is the **4-R cycle**: **Retrieve** similar cases,
**Reuse** their solutions, **Revise** (adapt) to fit the new problem, **Retain** the solved
case for future use ([4]). The core premise — *"similar problems have similar solutions"* — is
exactly the premise a PM makes when they say "this new town-border station is a lot like the
2019 Rosetown job."

CBR fell out of fashion during the deep-learning era but has been sharply revived in 2024–2026
as a way to give LLM agents **explicit, inspectable memory** and to fight three well-known LLM
failure modes: hallucination, lack of contextual memory across interactions, and brittleness on
structured tasks ([1][4]). The 2025 *Review of Case-Based Reasoning for LLM Agents* ([4])
formalizes a case as a tuple:

> **c = (P, S, O, M)** — Problem features, Solution components, Outcome metrics, Metadata.

Retrieval is a weighted similarity over features:
**sim(q, Pᵢ) = Σⱼ wⱼ · simⱼ(qⱼ, pᵢⱼ)** — i.e. per-attribute similarity, weighted by importance,
then summed. This is important for us: it means we can mix a *numeric* similarity on station
size with a *categorical* match on greenfield/brownfield and a *semantic* embedding similarity
on the scope text, each with its own weight. (Attribute-level hybrid similarity is the subject
of research thread 5.)

The review identifies three families of **adaptation** ([4]), all of which we will use:
1. **Transformational** — modify a retrieved solution by substitution/deletion/insertion
   (e.g. "same as Rosetown but delete the cathodic-protection package").
2. **Compositional** — combine pieces of *several* retrieved cases (price the pipeline from
   one analog, the metering skid from another).
3. **Generative** — let the LLM synthesize a novel solution *guided by* the retrieved cases.

**Retention** is gated: a solved case is added to the library only if its utility (novelty ×
effectiveness × generalizability) exceeds a threshold ([4]). For us, retention = feeding the
project back in once final actuals exist, which turns the corpus into a living, improving asset —
the "curated agent memory" idea from Karpathy's LLM-wiki framing (thread 1).

**Cognitive extras worth stealing:** the same review describes *introspection* — an agent that
diagnoses whether a bad answer came from a **retrieval failure** (wrong analogs) or an
**adaptation failure** (right analogs, wrong adjustment) and tunes accordingly ([4]). That is a
concrete, buildable eval/diagnostic loop for us (see thread 6).

## 2. CBR-RAG: structured retrieval beats naive RAG

The most directly relevant paper is **CBR-RAG** (Wiratunga et al., ICCBR 2024) ([1][2]). Its
thesis: instead of dumping top-k semantically similar chunks into the prompt (vanilla RAG),
structure the retrieval using the CBR cycle's indexing vocabulary and similarity knowledge, so
the context that reaches the LLM is *case-aligned*. On legal question-answering they compared:
- **Representations:** general-purpose embeddings vs domain-specific embeddings.
- **Similarity methods:** *inter* (query-to-case), *intra* (component-to-component), and *hybrid*.

Their finding: "CBR's case reuse enforces similarity between relevant components of the questions
and the evidence base, leading to significant improvements in the quality of generated answers"
over baseline RAG ([2]). The lesson for us is direct: **decompose the case and match component
to component** (scope-to-scope, cost-line-to-cost-line) rather than matching one big blob.
Domain-specific embeddings mattered — a generic embedding of "station" won't distinguish a
town-border station from a compressor station; we likely need domain-tuned or attribute-augmented
embeddings (thread 5).

## 3. Evidence CBR-augmented LLM agents actually work

- **DS-Agent** (Guo et al., ICML 2024) ([3]) is the strongest quantitative proof point. It wraps
  an LLM in the CBR cycle for automated data-science/ML engineering, drawing "cases" from Kaggle
  solutions. With GPT-4 it hit a **100% success rate in the development stage** and a **36% average
  improvement in one-pass rate** across weaker LLMs in a low-resource deployment stage — at
  **$1.60 and $0.13 per run** respectively. Two takeaways for us: (a) CBR let *weaker/cheaper*
  models perform well by leaning on retrieved exemplars — relevant for cost control; (b) a
  two-stage design (rich "development" reasoning vs cheap "deployment" reuse) is a real pattern.
- **CBR-LLM triage / classification** studies found that "providing case knowledge improved both
  user trust and LLM accuracy," and specifically that giving the **nearest-neighbour case plus an
  explicit statement of differences** elicited the *highest* user-trust scores ([4]). For a
  defensible utility cost estimate that an engineer has to sign, this "here's the comparable, here's
  exactly how yours differs" pattern is the whole game.
- **Logical-fallacy detection** and **legal QA** both improved in accuracy *and generalizability*
  with CBR integration ([4][1]). The generalizability point matters because our corpus is small
  (~200) and heterogeneous — CBR degrades gracefully into "here are the closest analogs" when it
  has no exact match, unlike a global model that extrapolates blindly.
- **Persistent case memory** is an active 2026 subfield (CBR-augmented R&D agents with local small
  LMs, Online Boundary-Aware Memory / OBAM that learns decision boundaries from a stream of
  contrasting cases) ([4]). Signals that the "case base as long-term agent memory" pattern is
  maturing, not speculative.

## 4. The hard truth: LLMs are weak *numeric* estimators

This thread's most important negative result. Several 2025 studies converge:

- **"Robustness is Important: Limitations of LLMs for Data Fitting"** (Liu, Yang & Adomavicius,
  2025) ([6]): LLM tabular-regression predictions are *highly* prompt-sensitive — trivial changes
  to label wording, feature order, or formatting caused error deltas exceeding **80%**. Their
  recommendation: prioritize classical ML for structured prediction where robustness matters, and
  treat LLMs as *complementary*, not as the estimator.
- **"LLMs as Universal Predictors on Small Tabular Datasets"** (2025) ([7]): classical models
  (XGBoost, linear regression) generally beat LLMs on small tabular data, *especially in the
  low-data regime we're in*. LLMs "require substantially more data to achieve comparable results."
  Conclusion in the title's negative: LLMs are **not** universal predictors for small tabular sets.
- General benchmark synthesis: high-performing classical models reach R² > 0.95 while LLMs
  frequently post **negative R²** (worse than predicting the mean) on continuous outputs, and are
  markedly better at classification than at regression. Prediction-interval benchmarks like
  QuantSightBench exist precisely because raw point-estimate quality is shaky. *(These general
  figures came from a search-result synthesis rather than a single fetched paper — treat the exact
  numbers as directional, not citable-to-one-source.)*

**Implication:** never let the LLM freehand a dollar figure. It should (a) select and weight
analogs, (b) apply *explicit, coded* adjustment rules, and (c) narrate/justify — while the actual
arithmetic (scaling, escalation, roll-up, distribution) runs in deterministic code or a simple
statistical model. This is not a limitation to apologize for; it's the correct division of labour.

## 5. The hybrid pattern to copy: statistical baseline + LLM "last-mile" revision

The single most transferable architecture we found is **"Bridging the Last Mile of Time Series
Forecasting with LLM Agents"** (2026) ([8]). Even though it's time-series, its structure is a
near-perfect template for defensible estimation:

- A **statistical foundation model (TimesFM) produces an immutable baseline** forecast.
- A **unified "forecast workspace"** holds, in one aligned table: historical actuals, the
  *immutable* baseline, and an *editable* final column — so observations, baseline, and the
  agent's edits never get conflated.
- The LLM agent may only change the estimate through **constrained, validated revision actions**
  (range multiply/add/clip, point overrides, merges) — not free-form output. **Every action is
  logged with its supporting evidence and a confidence score.**
- Before revising, the agent **retrieves context via tools** (calendar/holiday lookup, historical
  retrieval, memory queries) and packages it into structured "revision proposals."
- **Long horizons are decomposed map-reduce style** — local reasoners handle each event window,
  then results merge.
- A **memory bank** stores post-hoc reflections (realized vs baseline vs revised) for reuse.

Results: holiday-aware revision cut **MAE by 80.0% vs Prophet and 88.2% vs TimesFM**, and improved
MAPE from 155–263% down to ~33%; enabling the memory bank improved MAE to 60.10 from 92.94 (cross-
session learning) ([8]).

**Mapped onto our agent:** the "baseline" = a parametric/analog estimate (capacity-factored median
of the reference class); the "revision actions" = seasonal-winter uplift, greenfield premium,
brownfield tie-in complexity, escalation, weather-delay risk, each an auditable coded operation the
LLM *chooses and justifies*; the "workspace" = a structured estimate object where the base analog
cost and every adjustment is a separate, cited line; the "memory bank" = the retained case library.
This gives us exactly the auditability a utility needs while keeping the LLM out of the arithmetic.

## 6. Cost-engineering practice this must respect

The AI has to answer to established estimating doctrine, or engineers won't trust it.

- **Reference Class Forecasting (RCF)** — Flyvbjerg, from Kahneman & Tversky ([5]). Steps:
  (1) identify a reference class of similar past projects; (2) build a probability distribution of
  their outcomes; (3) position the new project in that distribution. RCF exists to counter chronic
  **optimism bias / strategic misrepresentation** — the reason megaprojects run over. **Our system
  is essentially automated, case-traceable RCF**, and should be pitched that way to give it
  institutional legitimacy. Known limitations to flag: defining the right reference class is hard,
  small samples make the distribution noisy (acute for us at ~200 projects, fewer per class), and it
  corrects *systemic* bias better than it nails any single project.
- **AACE estimate classes** ([9]) set the accuracy we can honestly promise. Estimate quality is
  governed by **project-definition maturity, not by effort or tooling**. With natural-language scope
  only we're at Class 5→4:

  | Class | Project definition | Typical methods | Accuracy range |
  |------|------|------|------|
  | 5 | 0–2% | capacity-factored, parametric, judgement | **−50% / +100%** |
  | 4 | 1–15% | equipment-factored, parametric | **−30% / +50%** |
  | 3 | 10–40% | semi-detailed unit costs | −20% / +30% |
  | 2 | 30–75% | detailed unit costs, forced quantities | −15% / +20% |
  | 1 | 65–100% | detailed take-offs | −10% / +15% |

  Ranges are **asymmetric** (overruns likelier than underruns) and should be set by project-specific
  risk analysis, not by pasting the table. Our UI must state the class and never imply Class-2
  precision from a Class-5 input.
- **Capacity factoring / scaling exponent** ([9]): adapt a comparable's cost across sizes with
  `Cost₂ = Cost₁ × (Capacity₂/Capacity₁)ⁿ`, where **n ≈ 0.5–0.85** (the "six-tenths rule");
  n<1 encodes economies of scale. This is a ready-made, defensible *adaptation rule* for the Revise
  step — e.g. scaling a known station cost to a differently-sized new station.
- **Prior CBR-in-construction accuracy** gives a realistic bar: conceptual-stage models report
  **MAPE ~13–15%** (GRNN 13%, R² 0.959; regression+ANN 15.09%), which sits inside AACE's accepted
  ±10–30% conceptual band. Recent work adds *exponential-smoothing-weighted case adaptation* for
  data-scarce settings — directly relevant to our small corpus (search-level source, not deep-read).

## 7. Agentic decompose-price-rollup workflows

The 2025 agentic-workflow literature ([10] and the broader survey results) favours **modular task
decomposition** with plan-and-solve / least-to-most strategies and inspector agents that reassign
subtasks. For estimating, this licenses a **bottom-up compositional** flow: decompose the new scope
into components (pipeline, station/metering skid, electrical/SCADA, civil, cathodic protection,
commissioning), price *each component* from its own best analog (compositional adaptation, §1), then
**roll up** with deterministic arithmetic and propagate uncertainty. This is more defensible than a
single whole-project analogy because each line carries its own citation and its own range, and it
degrades gracefully when only *some* components have good analogs. Tradeoff: decomposition multiplies
retrieval calls and can double-count or miss integration costs — so keep a **top-down whole-project
RCF check in parallel** and reconcile the two (hybrid RCF is a known practice).

---

## 8. Implications for the capital-project cost agent (opinionated)

**8.1 Architecture: CBR spine, hybrid estimation, LLM as reasoner-not-calculator.**
Adopt the 4-R cycle as the top-level control flow. Compute the *number* with deterministic
code/parametric rules; use the LLM for analog selection, adjustment choice, and justification.
This directly follows the LLM-numeric-weakness evidence (§4) and the last-mile hybrid (§5).
- *Pro:* auditable, robust to prompt noise, cheap arithmetic, engineer-trustable.
- *Con:* more engineering than "just prompt GPT with the docs"; requires building the adaptation-rule
  library and the structured case store.

**8.2 Case representation.** Store each historical project as `(P, S, O, M)`:
- **P (Problem):** structured attributes — asset type, station size/capacity, greenfield vs brownfield,
  region, construction season, terrain, plus the raw + LLM-summarized scope text.
- **S (Solution):** cost breakdown by component, execution approach, contractor model.
- **O (Outcome):** final cost, estimate-vs-actual variance, schedule, lessons-learned / what drove
  overruns (weather delays, scope creep).
- **M (Metadata):** era (for escalation), source-document provenance (for citation), data-completeness
  flags (some artifacts missing).
This is the thread-2 corpus-curation and thread-3 knowledge-base output; the structure is what makes
answers cite-able.

**8.3 Retrieval: attribute-weighted hybrid similarity, decomposed.** Match component-to-component
(CBR-RAG §2), combining numeric similarity (size), categorical match (greenfield/season), and semantic
embedding on scope — weighted. Return a *reference class* (a set), not a single "best match," so we can
form a distribution (RCF §6). Detail lives in thread 5.

**8.4 Adaptation: explicit rules first, LLM for the residual.** Apply coded adjustments in order —
(1) escalate all analog costs to today's dollars; (2) capacity-factor scale for size (n≈0.6 default,
tunable per asset class); (3) seasonal/winter-construction uplift; (4) greenfield/brownfield delta;
(5) known weather/complexity risk. Each is a logged line with a value and rationale. The LLM proposes
*which* rules apply and estimates any residual qualitative delta, but never overwrites the arithmetic.
- *Pro:* every dollar of the estimate is traceable to a rule or a cited analog.
- *Con:* rule library needs domain SME calibration; risk of false precision if rules are guessed.

**8.5 Output: asymmetric ranges + named comparables + AACE class.** Present a point estimate *and* a
range derived from the reference-class distribution, tagged with the AACE class implied by input
maturity (§6). Always show the top comparables with "here's how yours differs" (the highest-trust
pattern, §3). Never present Class-5 input as a tight number.

**8.6 Retention + calibration loop.** When actuals land, retain the project as a new case and record
retrieval-vs-adaptation error attribution (introspection, §1). Over time this both grows the corpus and
tunes similarity weights and adjustment factors — the living-memory payoff.

**8.7 Two model tiers (cost control).** Following DS-Agent (§3), use a strong model for the rich
reasoning/adaptation step and a cheaper model for routine reuse/formatting. CBR's exemplar-grounding is
what lets the cheaper tier stay accurate.

**8.8 What to explicitly NOT do.** Don't fine-tune an LLM to output costs from features (§4 evidence:
brittle, data-hungry, worse than XGBoost on small tabular). Don't do pure vanilla RAG over raw docs
(§2: component-aligned CBR retrieval beats it). Don't hide uncertainty behind a single number (§6:
violates AACE doctrine and RCF's whole point).

---

## Sources

1. [CBR-RAG: Case-Based Reasoning for Retrieval Augmented Generation in LLMs for Legal QA (arXiv 2404.04302)](https://arxiv.org/abs/2404.04302) — the foundational 2024 paper showing CBR-structured retrieval (inter/intra/hybrid similarity, general vs domain embeddings) significantly beats vanilla RAG.
2. [CBR-RAG, Springer/ICCBR 2024 chapter](https://link.springer.com/chapter/10.1007/978-3-031-63646-2_29) — published venue; confirms case reuse enforces component-level similarity between query and evidence.
3. [DS-Agent: Automated Data Science by Empowering LLMs with Case-Based Reasoning (ICML 2024 / arXiv 2402.17453)](https://arxiv.org/abs/2402.17453) — quantitative proof: 100% dev-stage success with GPT-4, 36% one-pass improvement across LLMs, $1.60/$0.13 per run; two-stage CBR design.
4. [Review of Case-Based Reasoning for LLM Agents (arXiv 2504.06943)](https://arxiv.org/html/2504.06943v1) — the reference survey: case tuple (P,S,O,M), weighted similarity, three adaptation types, gated retention, introspection, and the nearest-neighbour-plus-differences trust finding.
5. [Reference class forecasting — Wikipedia / Flyvbjerg overview](https://en.wikipedia.org/wiki/Reference_class_forecasting) — RCF method, steps, and its role countering optimism bias in capital projects (Flyvbjerg, Kahneman & Tversky).
6. [Robustness is Important: Limitations of LLMs for Data Fitting (arXiv 2508.19563)](https://arxiv.org/pdf/2508.19563) — LLM tabular regression is prompt-fragile (>80% error swings); recommends classical ML for numeric prediction, LLMs as complementary.
7. [LLMs as Universal Predictors? Empirical Study on Small Tabular Datasets (arXiv 2508.17391)](https://arxiv.org/pdf/2508.17391) — classical models (XGBoost) beat LLMs on small/low-data tabular; LLMs are not universal predictors, especially in the low-data regime.
8. [Bridging the Last Mile of Time Series Forecasting with LLM Agents (arXiv 2606.02497)](https://arxiv.org/html/2606.02497) — the hybrid template: immutable statistical baseline + LLM constrained/logged revision actions + memory bank; 80–88% MAE reduction while staying auditable.
9. [AACE Cost Estimate Classification (Classes 1–5) explainer](https://cenex.au/aace/estimate-classification.html) — definition-maturity-driven accuracy ranges (Class 5 −50/+100% … Class 1 −10/+15%), capacity-factor scaling exponent (~0.5–0.85), asymmetric ranges.
10. [Agentic Workflows guide (Vellum) and 2025 agentic-workflow survey results](https://www.vellum.ai/blog/agentic-workflows-emerging-architectures-and-design-patterns) — modular decomposition, plan-and-solve, inspector/roll-up patterns supporting bottom-up component pricing.

*Not independently deep-verified (search-synthesis only; treat as directional):* the general "classical R²>0.95 vs LLM negative R²" figures in §4; the CBR-in-construction MAPE 13–15% and the ES-CBR data-scarce adaptation result in §6 (ScienceDirect/ResearchGate sources returned 403 on fetch and were read only via search snippets).
