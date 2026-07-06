# 02 — Corpus curation & extraction pipeline

> How 200 messy project folders become a clean, queryable, LLM-legible knowledge base. This is the highest-effort, highest-value phase; everything the agent does is bounded by the quality of what this pipeline produces.
>
> Draft written before research threads completed; cross-references to `research/` are firmed up in the final pass.

## 0. Design stance

Two competing instincts exist for this corpus:

- **"Just RAG it"** — chunk every PDF, embed, retrieve at question time.
- **"Curate it"** — run a one-time (then incremental) distillation that turns each project's folder into a *structured record + written dossier*, and have the agent work primarily from those.

**We take the curation stance** (the Karpathy-wiki instinct), with raw RAG demoted to a fallback layer. Reasons:

1. *The corpus is small and static.* 200 projects is a curation-sized corpus, not a web-scale one. A few weeks of pipeline + review effort covers it, and only a handful of projects are added per year.
2. *The questions are aggregative.* "What do brownfield station upgrades cost per unit?" needs normalized numbers across many projects — chunk retrieval is the wrong shape for that; a table is.
3. *Auditability.* A human-reviewed dossier with provenance links is defensible; "the vector database returned these chunks" is not.
4. *Missing data must be explicit.* Curation forces a completeness assessment per project; RAG silently returns nothing.

Tradeoff acknowledged: curation front-loads cost and can bake in extraction errors. Mitigations: human validation loop (§5), provenance on every field (§3), raw-document fallback retrieval so the agent can always drop to source (§6).

## 1. Pipeline overview

```
Project folders (SharePoint/network drives)
        │  1. Inventory & registry
        ▼
Document registry (per-project manifest, types, completeness score)
        │  2. Conversion & parsing (PDF/DOCX/XLSX → clean text + tables)
        ▼
Parsed artifact store (markdown + table JSON, page-anchored)
        │  3. Schema-first LLM extraction (per artifact type)
        ▼
Structured extraction candidates (with confidence + source anchors)
        │  4. Normalization (escalation, units, taxonomy mapping)
        ▼
        │  5. Human validation (review UI / spreadsheet loop, prioritized by impact)
        ▼
┌────────────────────────────────────────────────┐
│  PROJECT KNOWLEDGE BASE (see plan/03)          │
│  a. projects.db — attribute schema, SQL        │
│  b. dossiers/ — one markdown dossier/project   │
│  c. index pages — cross-project synthesis      │
│  d. raw store + embeddings — fallback RAG      │
└────────────────────────────────────────────────┘
```

Each stage writes durable artifacts; the pipeline is re-runnable per-project and per-stage (idempotent, incremental).

## 2. Stage details

### 2.1 Inventory & registry
- Crawl the source locations; build `registry.csv`/table: project id, name, year, folder path, artifact list with detected types (DBM / scope / lessons / cost report / other), file dates, sizes.
- Classify artifact type with a cheap LLM pass over filename + first pages; human spot-check.
- Compute a **completeness score** per project (has scope source? has final cost? has lessons?). This drives sequencing: projects with cost + scope go first; scope-only projects still enter the KB flagged `no-actuals` (useful as comparables for scope, excluded from cost statistics).
- Deliverable in week 1-2: *the inventory reality report* — replaces the assumptions table in `plan/01` §4. Cheap, and de-risks everything.

### 2.2 Conversion & parsing
- Born-digital DOCX/XLSX: direct conversion (pandoc / openpyxl) — lossless and cheap.
- PDFs incl. scans: a document-parsing service or library with strong table support (Docling, Azure Document Intelligence, LlamaParse — pick after a bake-off on 5 gnarly real cost reports; see `research/02`). Vision-LLM parsing as the fallback for the worst scans.
- Output convention: one markdown file per artifact + one JSON per extracted table, every block carrying `{doc_id, page}` anchors so later citations can point to *page-level* provenance.
- Cost reports are the crown jewels and the hardest parses (financial-system exports, merged cells, subtotal rows). Budget disproportionate effort here; a human-in-the-loop table-fixing pass on cost tables is acceptable and bounded (200 projects × ~1-3 cost tables).

### 2.3 Schema-first structured extraction
- Define the **attribute schema v1** *with estimators, before extraction*, ~40-80 fields across:
  - Identity: id, name, year approved / in-service, region, program.
  - Asset & scope: asset type (station / pipeline / integrity / facility...), sub-type, capacity metrics (e.g., flow, HP, km, diameter, pressure class), scope bullet summary, inclusions/exclusions.
  - Site & execution: greenfield / brownfield / expansion, site access, construction season(s), execution mode (contract/self-perform), schedule (planned vs actual).
  - Costs: original estimate (+ class if known), sanctioned budget, final actual, breakdown by standard buckets (materials, construction labour, engineering, land/ROW, owner's costs, contingency spent), change-order total.
  - Outcomes: overrun %, delay, top delay/growth causes (controlled vocabulary + free text), lessons-learned digest.
  - Meta: per-field `source` (doc + page), `confidence`, `status` (extracted / reviewed / imputed / unknown).
- Extraction = LLM with JSON-schema-constrained output, one prompt per artifact type, fed the parsed artifact. Fields it cannot find are returned as `unknown` — **never guessed** (prompt + schema enforce this).
- Two-pass for numerics: extract, then a verifier prompt re-locates each number in the source and checks arithmetic (buckets sum to total). Disagreement → flag for human review.

### 2.4 Normalization
- **Escalation**: restate all costs to reference-year dollars using a documented index (Canadian construction/utility cost index — selection documented in `research/04`); store both raw and normalized, plus the index snapshot used (reproducibility).
- **Units**: single unit system; derived unit-cost fields ($/HP, $/km, $/m³ per day...) computed here, not ad hoc later.
- **Taxonomy mapping**: era-specific terminology mapped to the controlled vocabularies (asset types, cost buckets, delay causes). The mapping table is itself a reviewed artifact.
- Optional (phase 2): location factors within the service territory; likely minor within one prairie region — validate rather than assume.

### 2.5 Human validation loop
- Prioritize by leverage: 100% human review of *total costs, capacities, asset type, site type* (the fields that drive retrieval and $/unit math); sampled review of the long tail.
- Mechanics: keep it boring — a review spreadsheet or a lightweight web table (e.g., a Streamlit grid) showing field, extracted value, source snippet link, confidence; reviewer accepts/edits. Every edit is logged (trains later extraction prompts, and proves diligence).
- Estimator involvement here is *deliberate adoption strategy*: the people who must trust the tool personally sign off its foundation.

### 2.6 Dossier generation
- For each project, an LLM writes the **project dossier** (1-2 page markdown) from the validated structured record + parsed artifacts: narrative scope, what made it expensive/cheap, execution story (season, weather, surprises), lessons, cost table (normalized), links to sources. Template-driven; regenerated whenever inputs change; human spot-review of a sample.
- Dossiers are the agent-facing memory pages (`plan/03`); the structured DB is the math-facing layer. Same facts, two projections.

## 3. Provenance model (non-negotiable)

Every structured field and every dossier claim carries `{project_id, doc_id, page, snippet_hash, extractor_version, review_status}`. This is what makes the eventual agent's citations real instead of decorative, and it is cheap only if built in from stage 1.

## 4. Tooling choices (held loosely; final call in `plan/06`)

- Pipeline orchestration: plain Python + a workflow runner (Prefect/Dagster) or even Make — 200 projects does not need big-data infrastructure.
- Storage: files in git-versioned repo (dossiers, parsed markdown) + SQLite/Postgres (structured records) + object store for raw docs. Git versioning of dossiers gives history/diff/rollback for free — the "wiki" property.
- Models: strongest available frontier model for extraction of cost tables and dossier writing (accuracy >> cost at n=200); cheap model for classification/inventory.

## 5. Effort estimate & sequencing

| Step | Effort (order of magnitude) |
|---|---|
| Inventory + registry | 1-2 weeks incl. access wrangling |
| Parser bake-off + parsing run | 1-2 weeks |
| Schema workshop with estimators | 2-3 sessions, calendar-bound |
| Extraction prompts + run + verifier | 2-3 weeks |
| Human validation (core fields, 200 projects) | ~2-4 min/project/field-group ⇒ a few reviewer-days, spread over weeks |
| Normalization + dossier generation | 1-2 weeks |

The pipeline is incremental by construction: new completed projects flow through the same stages, which is also the agent's long-term learning loop (`plan/05`).
