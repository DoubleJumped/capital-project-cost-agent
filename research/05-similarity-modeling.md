# 05 — Project Similarity Modeling over Mixed Attributes

*Research chapter for the capital-project cost-estimation agent. Focus: how to compute "which past
projects are most like this new one" over mixed categorical / numeric / free-text features, and how
to turn those neighbors into a defensible cost estimate with ranges and named comparables.*

---

## TL;DR for this project

- **Two matching layers, not one.** Use a cheap, transparent **structured-attribute distance** (Gower)
  as the backbone, and a **text-embedding semantic layer** over scope prose as a complement. They
  answer different questions ("is this the same *kind* of project?" vs "does this scope *read* like
  that one?") and their disagreements are diagnostically useful. [1][6][14]
- **Hard filter first, then rank.** Adopt the standard two-stage pattern: **hard filters** on
  non-negotiable attributes (asset type, greenfield/brownfield) to define the candidate set, then a
  **similarity ranking** inside it. Filtering before scoring improves both precision and defensibility
  ("we only compared to like assets"). [8][10]
- **With ~200 projects, favor interpretable methods.** Learned distance-metric / embedding fine-tuning
  needs more labeled pairs than we have. Hand-crafted **Gower with expert-set (or AHP-derived) feature
  weights** is the right default; treat learned weighting as a later optimization. [3][12][15]
- **Predict cost with distance-weighted k-NN over the neighbors**, not a single "best match." Weight
  neighbors by inverse distance so closer analogues dominate, and expose the neighbor set + their
  actual costs as the evidence. This *is* essentially **reference-class forecasting** operationalized. [11][13][17]
- **The estimate is a distribution, not a point.** The spread of neighbor costs (after normalization
  for size/era/inflation) becomes the **range**; report P10/P50/P80 in the spirit of Flyvbjerg's uplift
  method rather than a single number. [13][17]
- **Use an LLM only where it earns its keep:** (a) to **extract structured attributes** from the new
  project's natural-language scope so structured matching can run, and (b) as an optional **listwise
  reranker / similarity judge** over the top handful of candidates — never as the primary retriever. [4][7][9]
- **This is a curated-memory play, not raw RAG.** Following the Karpathy "LLM wiki" framing, distill
  each historical project into a compact structured **project card** (attributes + normalized costs +
  lessons) that the agent reads directly; keep raw DBMs/cost reports as backing evidence to cite. [16]

---

## 1. The problem shape and why it is unusual

We have ~200 heterogeneous projects. Each carries a mix of **categorical** attributes (asset/station
type, greenfield vs brownfield, region, construction season), **numeric** attributes (station size /
throughput, pipe diameter, length, duration, weather-delay days, final cost), and **free text**
(scope docs, DBM narrative, lessons-learned). Some artifacts are missing per project. The task —
"find the most similar past projects to a described new one and estimate its cost" — is a textbook
**case-based reasoning (CBR)** / **k-nearest-neighbor regression** problem, dressed in modern
retrieval clothing. [2][11]

Two structural facts drive every design choice:

1. **Small n.** 200 rows is tiny for machine learning. Methods that need thousands of labeled
   similarity pairs (deep metric learning, embedding fine-tuning) will overfit; interpretable,
   low-parameter methods (Gower distance, weighted k-NN, CBR) are both more accurate here and more
   *defensible* to a cost-review board. [12][15]
2. **Mixed types + missing values.** A distance function must gracefully combine a categorical match,
   a numeric gap, and a text similarity, and must tolerate a project missing its lessons-learned
   report. Gower distance was literally designed for exactly this. [1][6]

---

## 2. Similarity over mixed categorical / numeric / text: Gower distance

**Gower's distance** (Gower 1971) is the canonical similarity for mixed-type data. It computes a
per-feature similarity, then takes a weighted average across features, producing a value in [0,1]. [1][6]

The overall similarity between records *i* and *j* is:

```
S_ij = ( Σ_k  w_ijk · s_ijk ) / ( Σ_k  w_ijk )
```

- **Numeric features:** `s_ijk = 1 − |x_ik − x_jk| / R_k`, where `R_k` is the observed range
  (max − min) of feature *k*. The range normalization is what lets a $10M cost gap and a 3-metre
  diameter gap live on the same 0–1 scale — no single large-magnitude feature dominates. [6]
- **Binary / categorical features:** `s_ijk = 1` if the categories match, else `0`. Ordinal features
  (e.g., a small/medium/large station-size band) can use a rank-based form
  `s_ijk = 1 − |r_i − r_j| / (max r − min r)` (Podani extension) so "small vs large" counts as more
  distant than "small vs medium." [6]
- **Weights `w_ijk`** default to 1 (equal importance) but are the main tuning lever — set them by
  domain judgment or an analytic method (Section 6). [6]
- **Missing values** are handled naturally: set `w_ijk = 0` for any feature missing in either record,
  so the average is taken only over co-observed features. This is exactly the property we need for
  projects with missing artifacts. [1]

*Verification note:* the Wikipedia formula and per-type similarities above are directly confirmed;
the missing-value zero-weight convention is the standard `daisy`/`gower` package behavior and is
described in the mixed-variable-distance literature, though the Wikipedia page itself did not spell
it out. [1][6]

**Implementations:** R `cluster::daisy(metric="gower")`, R `gower` package, Python `gower` package.
All are O(n·features) and trivially fast at n=200.

**Recent refinements (2024–2025)** address Gower's known weakness — arbitrary equal weighting.
"Gower's Similarity Coefficients with Automatic Weight Selection" and "Unbiased mixed variables
distance" (arXiv 2411.00429) propose data-driven weightings that correct for the fact that
categorical and continuous features have different natural variances, which otherwise biases the
average. Worth piloting once a baseline works. [3]

---

## 3. Text-embedding similarity for scope descriptions

The scope prose ("new 20 MMscf/d town border station, brownfield tie-in adjacent to existing
regulator station, winter construction") carries signal no structured field captures. Modern
**dense text-embedding models** turn each scope into a vector; cosine similarity ranks scopes by
semantic closeness. [7][14]

**SOTA landscape (2025–2026), from MTEB leaderboards** — treat exact scores cautiously (MTEB v2 is
not comparable to v1, and vendor-reported numbers vary): [7]

| Model | MTEB (approx) | Dim | Notes |
|---|---|---|---|
| Gemini Embedding 2 (Google) | ~68 (0.677 retrieval) | 3072 | Strong retrieval; multimodal; ~$0.20/M tok [7] |
| Qwen3-Embedding-8B | ~70 | 4096 | Apache-2.0, self-hostable [7] |
| Voyage 4 Large | ~66.8 | 2048 | $0.12/M tok; shared-space MoE [7] |
| Cohere Embed v4 | ~65 | 1536 | 128K context — fits whole DBMs [7] |
| OpenAI text-embedding-3-large | ~64.6 | 3072 | Stable, ubiquitous, $0.13/M tok [7] |
| BGE-M3 | ~63 | 1024 | MIT, free self-host, budget option [7] |

*Flag:* the "Harrier-OSS-v1-27B (74.3)" and other early-2026 entries appear on the vendor blog
leaderboard but I could **not** independently verify those scores against an authoritative MTEB
snapshot; treat the leaderboard ordering as indicative, not gospel. [7]

**Practical guidance that matters more than the leaderboard:** [7]

- **Benchmark on our own data.** Generic MTEB rank rarely survives contact with a narrow domain like
  prairie-gas capital works. A ~66-MTEB model may beat a ~70 model on *our* scopes.
- **Fine-tuning yields +10–30% in specialized domains** — but needs labeled pairs we mostly lack;
  park it. [7]
- **Context length matters:** Cohere v4 (128K) or long-context models can embed an entire DBM;
  otherwise chunk and pool.
- **Matryoshka embeddings** let us truncate dimensions (e.g., 3072→512) with minor quality loss —
  useful given tiny corpus and cheap storage. [7]

**Hand-crafted vs learned representation — the honest tradeoff:**

- *Hand-crafted structured vector (Gower):* interpretable, defensible, works at n=200, encodes
  engineering knowledge directly. But it only "knows" the attributes we chose to extract.
- *Learned text embedding:* captures nuance we did not think to encode; zero feature engineering.
  But it is opaque ("why is Project 143 similar?"), can latch onto stylistic era-differences in
  document formatting rather than engineering substance, and can't be *audited* by a cost engineer.

**Recommendation: use both, in different roles** — structured Gower for the hard, auditable backbone;
embeddings as a semantic tie-breaker / recall booster. See Section 7.

---

## 4. Two-stage retrieval: hard filters, then semantic ranking

The dominant modern retrieval pattern is **decoupled candidate selection then ranking**. [8][10]
Mapped to our problem:

- **Stage 0 — hard metadata filter.** Constrain the candidate set by non-negotiable attributes:
  asset/station type, greenfield vs brownfield, maybe region. "A lightweight metadata filtering
  layer scopes retrieval to a relevant portion of the knowledge base, improving precision and
  latency by excluding items clearly outside the query's target category." [8] For us the payoff is
  as much **defensibility** as precision — the estimate should never be justified by a comparator
  that is a different class of asset.
- **Stage 1 — similarity ranking** inside the filtered set, via Gower + embedding score.
- **Stage 2 — (optional) reranking** of the top ~10 by a stronger model (cross-encoder or LLM judge;
  Section 5).

**Hybrid retrieval note:** where scope text is the query, combining **BM25 (sparse) + dense
embeddings**, fused by **Reciprocal Rank Fusion** (`score = Σ 1/(k+rank)`, k typically 60), reliably
beats either alone (e.g. WANDS benchmark: RRF 0.707 vs BM25 0.698 vs kNN 0.695, tuned hybrid 0.750).
BM25 catches exact tokens (station codes, "20 MMscf/d", regulator model numbers) that embeddings
blur. **Small-corpus caveat: the default k=60 can make hybrid *worse* than dense-only on tiny
corpora** — one practitioner debugged exactly this on an 80-doc set. With ~200 projects, tune k
(try 10–30) or lean dense-only for the semantic layer. [18]

---

## 5. LLM-as-reranker / similarity-judge

After narrowing to a handful of candidates, an LLM can reorder them by holistic similarity ("which of
these 8 past projects is genuinely most analogous, considering brownfield complexity and winter
construction?"). Three flavors: **pointwise** (score each alone), **pairwise** (O(n²) comparisons),
**listwise** (rank all in one prompt). [4][9]

Evidence (2024–2025 empirical analyses): [4][9]

- **Listwise wins on quality.** RankGPT-style listwise reranking with a strong model posts the top
  nDCG@10 (e.g., RankGPT-GPT-4 ~75.6, Zephyr-7B listwise ~62.7 on one benchmark); it models
  inter-document relationships that pointwise cannot. [4]
- **Cross-encoder rerankers** (Cohere Rerank, Voyage rerank, BGE-reranker) give large gains for small
  latency: studies report **+33–40% retrieval accuracy for ~+120ms**, and reranking commonly moves
  truly-relevant items from rank 14/23 to the top. Cohere Rerank 3.5 ≈ 170ms on small payloads. [5][10]
- **Sobering caveats:** LLM rerankers show a **consistent 5–15% performance drop on genuinely novel
  queries** (post-training-cutoff topics), undercutting "they generalize" claims; some models
  (ListT5) that shine on benchmarks **collapse to ~9.7 nDCG@10** on novel queries. [4] For us this
  means: use the LLM judge as a *re-ranker over already-good candidates*, never as the retriever, and
  always keep the transparent Gower score as the fallback of record.

**Our recommendation:** a listwise **LLM similarity-judge over the top ~8** is attractive because it
can weigh engineering nuance and *explain its ranking in prose we can cite*. But gate it: the
structured Gower ranking is the auditable primary; the LLM reorders and, crucially, **writes the
"why these are comparable" narrative**.

---

## 6. Feature weighting: how much does each attribute matter?

Equal weighting is rarely right — station size and greenfield/brownfield status likely matter more
for cost than region. Options, roughly in order of how much data they need: [12][15]

- **Expert / AHP weights.** Elicit pairwise importance from cost engineers via the Analytic Hierarchy
  Process; standard in the CBR construction-cost literature. Zero data needed, fully defensible.
  Default choice. [2]
- **Genetic-algorithm-optimized weights.** CBR cost-estimation studies use a GA to search feature
  weights that minimize retrieval error on held-out past projects; reported to "reduce errors
  consistently." Feasible at n=200 with leave-one-out CV. [2]
- **Gradient-descent / learned weighted metrics.** Learn per-feature (or per-class) weights minimizing
  a k-NN error surrogate. Powerful but data-hungry and less interpretable — a later optimization. [15]
- **Automatic Gower weight selection** (2024 methods) to de-bias mixed-type averaging. [3]

*Tradeoff:* every step down this list buys accuracy with opacity. Given a cost-review audience, start
with AHP weights, then validate/tune with leave-one-out GA, and only reach for learned metrics if
LOO error is unacceptable.

---

## 7. From neighbors to a cost estimate: distance-weighted k-NN regression

Once neighbors are ranked, predict cost with **distance-weighted k-NN regression**: the estimate is a
weighted average of neighbor costs, where closer neighbors count more. [11] Standard forms weight by
inverse distance, `w_i = 1/d_i` (or `1/d_i²`, or a Gaussian kernel `exp(−d²/σ²)`):

```
ĉost_new = Σ_i  w_i · cost_i,norm  /  Σ_i  w_i        over the k nearest projects
```

Design choices, with our recommendations:

- **Normalize neighbor costs before averaging.** Escalate historical final costs to today's dollars
  (published construction-cost indices) and, where possible, normalize by a size driver (cost per
  MMscf/d, per metre, per station) so a 2011 small station and a 2023 large one are comparable. This
  mirrors the CBR literature's use of cost indices and characteristic-difference adjustments. [2]
- **Choose k by leave-one-out CV** on the 200 projects (minimize MAPE). Small corpora usually favor
  small k (3–7); too-large k drags in dissimilar projects.
- **Distance weighting > uniform.** Confirmed to improve k-NN regression accuracy by damping far
  neighbors — important here because after a hard filter the "k-th" neighbor may still be quite
  different. [11]
- **Output a distribution, not a point.** The spread of the (normalized) neighbor costs is the honest
  uncertainty. Report **P10/P50/P80** and name each neighbor with its actual cost. This operationally
  *is* **reference-class forecasting**: take the outside view from a class of analogues rather than a
  bottom-up inside view. [13][17]

*Accuracy expectation:* CBR construction-cost studies report meaningful error reduction over
regression baselines, but I could **not** extract a single authoritative MAPE figure from the
open-access sources (several were paywalled); typical early-estimate CBR MAPEs in the literature sit
in the ~10–20% range but treat that as unverified here. [2]

---

## 8. Matching estimators from causal inference — as inspiration

Causal-inference **matching estimators** solve a structurally identical problem: find, for each unit,
the most-similar unit(s) on observed covariates. Transferable ideas: [12]

- **Nearest-neighbor matching** on covariates = our k-NN over attributes; the machinery of choosing
  1:1 vs 1:many maps to choosing k. [12]
- **Caliper matching:** refuse matches beyond a distance threshold. For us: **if no neighbor is within
  a Gower-distance caliper, flag "no good comparable exists" rather than forcing a bad estimate.**
  This is a genuinely valuable honesty mechanism for a novel/greenfield project. [12]
- **Propensity-score dimension reduction:** collapse many covariates into one score to match on.
  Analogue: a learned "cost-driver score." Probably overkill at n=200, but conceptually clean. [12]
- **Balance checking:** verify matched sets resemble the query on key covariates — a built-in QA that
  our neighbor set isn't skewed (e.g., all summer builds when the query is winter). [12]

The lesson to steal: matching literature is obsessive about *when a match is too poor to trust* —
exactly the discipline a cost-estimate agent needs.

---

## 9. Eliciting structured attributes from natural-language scope

Structured matching (Gower, filters) only works if the new project's free-text scope is parsed into
the same attribute schema as the historicals. Use an LLM for **schema-constrained structured
extraction**: [9]

- **Define a fixed JSON schema** (asset_type enum, station_size numeric, greenfield/brownfield enum,
  construction_season enum, pipe_diameter, length_m, expected_weather_exposure, …) and have the LLM
  return values conforming to it. Concise field-list or full JSON-Schema both work; libraries
  (Anthropic/OpenAI/Gemini structured outputs, Pydantic, the `llm` CLI) support this. [9]
- **Constrained decoding** (grammar/schema-constrained token generation) guarantees *valid* JSON;
  tools like XGrammar make it cheap. Validity ≠ correctness, though. [9]
- **Grounding & hallucination:** even top models show non-trivial invalid/incorrect rates on complex
  extraction (one benchmark cites ~12% invalid responses for GPT-4-class models on hard schemas);
  mitigate by (a) asking the model to cite the source span for each extracted value, (b) allowing
  `null`/`unknown` rather than forcing a guess, and (c) a human-in-the-loop confirm step for the
  handful of driver attributes that most swing cost. [4][9]
- **Apply the same extractor to the 200 historicals** so the new project and the corpus share one
  schema — this is the single most important consistency requirement for matching to work.

---

## 10. The curated-memory framing (Karpathy "LLM wiki")

Rather than raw RAG over every DBM and cost report, distill each historical project into a compact,
LLM-legible **project card**: structured attributes + normalized costs + a short lessons/complexity
summary, with links back to raw artifacts as citable evidence. This is the Karpathy "LLM wiki as
agent memory" pattern: **raw sources → compiled wiki → schema**, where the agent reads the
*compiled* knowledge and consults raw material only to verify/cite. [16]

Why this fits us:

- The agent starts from **summarized, comparable** cards instead of re-parsing heterogeneous
  documents every query — faster, and formats-across-eras are normalized once, up front.
- The cards carry **semantic memory** (per-project attributes/lessons), and a running **episodic
  log** can capture estimate-vs-actual outcomes to improve future weighting. [16]
- **Governance:** an agent-maintained draft vault vs a human-reviewed canonical vault matches a
  utility's need for sign-off before a card influences an estimate. [16]

Tradeoff vs raw RAG: distillation risks **information loss** (a subtlety in a lessons-learned report
gets summarized away). Mitigate by keeping raw docs one hop away and letting the LLM judge/citation
step pull the original text when a comparator is decisive.

---

## 11. Recommended architecture for the cost agent

1. **Ingest:** LLM extracts each historical project into the fixed attribute schema + normalized
   (escalated, size-adjusted) cost + short summary card. Human review of cards. [9][16]
2. **New project:** same extractor parses the engineer's NL scope into the schema; ambiguous driver
   attributes confirmed with the user. [9]
3. **Stage 0 — hard filter** on asset type / greenfield-brownfield (and a distance **caliper** to
   detect "no comparable"). [8][12]
4. **Stage 1 — rank** survivors by **Gower distance** (AHP/GA-weighted) as the auditable primary,
   with a **dense-embedding cosine** over scope text as a parallel signal / recall booster
   (RRF-fused, k tuned for small corpus). [1][7][18]
5. **Stage 2 — optional listwise LLM similarity-judge** over the top ~8 to reorder on engineering
   nuance and *write the "why comparable" narrative*. [4]
6. **Estimate:** distance-weighted k-NN regression over the top k (k by LOO-CV) → **P10/P50/P80
   range**, named comparables, each with its normalized actual cost and citations. [11][13]
7. **Learn:** log estimate-vs-actual to retune feature weights and k over time. [16]

---

## Sources

1. [Gower's distance — Wikipedia](https://en.wikipedia.org/wiki/Gower's_distance) — canonical formula,
   per-type similarities, [0,1] range; basis for mixed-type + missing-value handling.
2. [Cost estimation model for building projects using CBR (SNU) / CBR cost-estimation literature](https://www.researchgate.net/publication/237189236_Cost_estimation_model_for_building_projects_using_case-based_reasoning)
   — CBR retrieval, attribute weighting via AHP/GA, cost-index normalization (full PDF was
   inaccessible; drawn from abstracts/search summaries).
3. [Unbiased mixed variables distance (arXiv 2411.00429)](https://arxiv.org/pdf/2411.00429) and
   [Gower's Similarity Coefficients with Automatic Weight Selection](https://www.researchgate.net/publication/377814622_Gower's_Similarity_Coefficients_with_Automatic_Weight_Selection)
   — 2024 data-driven weightings correcting Gower's equal-weight bias.
4. [How Good are LLM-based Rerankers? An Empirical Analysis (arXiv 2508.16757)](https://arxiv.org/html/2508.16757v1)
   — pointwise/pairwise/listwise nDCG numbers; 5–15% drop on novel queries; ListT5 generalization failure.
5. [Cross-Encoder reranking study (Ailog)](https://app.ailog.fr/en/blog/news/reranking-cross-encoders-study)
   and [Cohere vs zerank latency benchmark (ZeroEntropy)](https://zeroentropy.dev/articles/lightning-fast-reranking-with-zerank-1/)
   — +33–40% accuracy for ~+120ms; Cohere Rerank 3.5 ~170ms latency figures.
6. [Gower's distance for mixed data (Towards Data Science)](https://towardsdatascience.com/gowers-distance-for-mixed-categorical-and-numerical-data-799fedd1080c/)
   — worked per-feature similarity computation and range normalization.
7. [Best Embedding Models 2025: MTEB Scores & Leaderboard (Ailog)](https://app.ailog.fr/en/blog/guides/choosing-embedding-models)
   — model names/dims/costs/MTEB scores; "test on your data"; fine-tune +10–30%; Matryoshka. Some
   early-2026 scores unverified.
8. [Metadata-driven RAG for financial QA (arXiv 2510.24402)](https://arxiv.org/pdf/2510.24402) and
   [Metadata Filtering and Reranking (DeepWiki)](https://deepwiki.com/AyushDhimann/RAGging/5.3-metadata-filtering-and-reranking)
   — hard metadata filter before ranking improves precision/latency.
9. [Structured data extraction with LLM schemas (Simon Willison)](https://simonwillison.net/2025/Feb/28/llm-schemas/)
   and [PARSE: LLM-driven schema optimization (arXiv 2510.08623)](https://arxiv.org/html/2510.08623v1)
   — JSON-schema extraction, constrained decoding, ~12% invalid-rate on hard schemas.
10. [Rerankers and Two-Stage Retrieval (Pinecone)](https://www.pinecone.io/learn/series/rag/rerankers/)
    — bi-encoder info loss, cross-encoder mechanics, top_k→top_n workflow, latency contrast.
11. [Understanding Weighted k-NN (Medium)](https://medium.com/@lakshmiteja.ip/understanding-weighted-k-nearest-neighbors-k-nn-algorithm-3485001611ce)
    and [Regression Conformal Prediction with Nearest Neighbours (arXiv 1401.3880)](https://arxiv.org/pdf/1401.3880)
    — distance-weighted k-NN regression as weighted average of neighbor labels.
12. [Matching Methods chapter (Causal ML book)](https://www.causalmlbook.com/matching-methods.html)
    and [KNN matching / propensity primer (TDS)](https://towardsdatascience.com/causal-inference-with-python-a-guide-to-propensity-score-matching-b3470080c84f/)
    — nearest-neighbor/caliper matching, balance checks; the "when is a match too poor" discipline.
13. [Reference class forecasting — Wikipedia](https://en.wikipedia.org/wiki/Reference_class_forecasting)
    — outside-view method steps, reference-class selection, uplift/percentile (Edinburgh Tram) example.
14. [Top embedding models on the MTEB leaderboard (Modal)](https://modal.com/blog/mteb-leaderboard-article)
    — corroborating MTEB retrieval leaders and benchmark caveats.
15. [Learning weighted similarity functions to improve NN (ScienceDirect)](https://www.sciencedirect.com/science/article/abs/pii/S0020025509001832)
    — learned per-feature/per-class distance weights via gradient descent.
16. [Karpathy's LLM Wiki as Agent Memory (AAIF)](https://aaif.io/blog/karpathys-llm-wiki-as-agent-memory/)
    and [Karpathy LLM knowledge base bypassing RAG (VentureBeat)](https://venturebeat.com/data/karpathy-shares-llm-knowledge-base-architecture-that-bypasses-rag-with-an)
    — raw→wiki→schema; semantic/episodic/summary memory; agent-vault vs human-vault governance.
17. [Practical application of reference class forecasting: properties of similarity (ScienceDirect 2021)](https://www.sciencedirect.com/science/article/abs/pii/S0377221721002939)
    — similarity properties for building a reference class (abstract-level; full text paywalled).
18. [Hybrid Search: BM25, Vector & RRF reference](https://www.digitalapplied.com/blog/hybrid-search-bm25-vector-reranking-reference-2026)
    and [Sparse vs Dense Retrieval (ML Journey)](https://mljourney.com/sparse-vs-dense-retrieval-for-rag-bm25-embeddings-and-hybrid-search/)
    — RRF formula `1/(k+rank)`, WANDS numbers, small-corpus k-tuning caveat.
