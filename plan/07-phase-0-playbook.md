# 07 — Phase 0 playbook (data reconnaissance, 2-3 weeks)

> `plan/06` §1 says *what* Phase 0 must produce — coverage numbers, the original-estimate recording rate, a parser bake-off, and a kill/reshape decision leadership has seen. This doc says *how*, day by day, for the single builder standing inside the utility with a fresh SharePoint login. The governing spirit: Phase 0 is cheap and its only job is to convert every assumption in `plan/01` §4 into a measured fact before anyone commits months. Nothing here is architecture — it is reconnaissance that de-risks architecture.

## 0. The two things that actually kill this phase

1. **Access wrangling.** You cannot inventory folders you cannot open, and you cannot call an LLM on cost data until IT has blessed an endpoint. Both are other people's calendars. Fire all access/governance requests on **day 1, hour 1** (§2) and treat them as the critical path everything else schedules around.
2. **Measuring the wrong thing thoroughly.** The exit criteria (`plan/06` §1) reduce to four numbers: how many projects have *scope + final actuals*, the *original-early-estimate recording rate*, lessons-learned coverage, and whether the ugliest cost reports are machine-parseable. Everything below is instrumentation for those four numbers. Do not gold-plate the crawler; do not start the real pipeline. Measure, report, decide.

## 1. Week-by-week schedule

| Week | Theme | Concrete deliverables (end of week) |
|---|---|---|
| **1** | Access + first light | All access/governance requests filed (§2); read access to ≥1 project-folder tree confirmed; inventory script v0 crawling a **10-15 project pilot slice**; file-type census for that slice; first estimator meeting booked; SEERai demo requested (§7). |
| **2** | Full crawl + bake-off + interviews | Inventory script run over the **entire** corpus → `registry.csv` populated; artifact-type LLM classification pass done; completeness scores computed; parser bake-off on the 5 ugliest cost reports scored (§5); golden extraction mini-set started (~5 projects, ≥2 hand-verified); 2-3 estimator interviews done (§6). |
| **3** | Measure, decide, report | The four gating numbers finalized (§4); SEERai demo watched and scored (§7); **inventory reality report** written (§8) and walked through with leadership; kill/reshape decision framed and taken; exit checklist (§9) green or explicitly waived. |

If access slips (it will), weeks 1-2 telescope: run the bake-off and interviews on whatever slice you *can* open, and let the full crawl trail the access grant. The report in week 3 is the immovable milestone — ship it with caveats rather than late.

## 2. Access & governance groundwork — file all of this on day 1

Access is the schedule killer (`plan/02` §5 budgets "1-2 weeks incl. access wrangling" for inventory alone). Requests are asynchronous; submit them before writing a line of code.

- **Read access to the project archive.** SharePoint site(s) and/or network drive paths holding the ~200 project folders. Ask specifically for **read-only recursive** access and the *canonical* location — utilities routinely have a "real" archive plus scattered team copies. Get the name of whoever owns that archive; you will need them again for gap-filling.
- **A representative sample pulled by a human, now.** Ask a PM or records custodian for **3-5 complete project folders** (ideally spanning eras and asset classes) hand-picked as "typical." This unblocks the bake-off and schema thinking on day 2 even if broad access is still pending, and it shows you what "complete" looks like before the crawler judges completeness.
- **The IT/security conversation about LLM endpoints.** The blocker behind the blocker. Confirm, in writing: (a) which **approved enterprise LLM endpoint** exists or can be stood up (Bedrock/Vertex/Azure-hosted frontier model per `plan/06` §2); (b) a **no-training / no-retention guarantee** on prompts and completions — cost data is commercially sensitive (`plan/01` §6); (c) whether cost documents may leave the tenancy at all, which decides self-hosted parsing (Docling) vs. managed (Azure DI) in §5. If no endpoint exists, the inventory's cheap classification pass can run on a small local/open model — flag this as a governance dependency, not a technical one.
- **Records-management / privacy sign-off** to *read and process* the archive for this purpose, and confirmation of where derived artifacts (registry, parsed text) may live. Cheap to ask now, expensive to discover you skipped.

Log every request with date sent and owner in the report's appendix — the audit trail doubles as a schedule-slip diagnosis when week 3 arrives.

## 3. The inventory script

Purpose: a **census, not a pipeline.** It walks the folders, records what exists, and cheaply guesses each artifact's type — nothing more. Keep it a single re-runnable Python script writing a flat `registry.csv` (SQLite optional); idempotent so re-runs after access expands are free. Two passes: a **cheap deterministic crawl** (filesystem metadata, no LLM) then a **cheap LLM classification pass** over filename + first pages only (`plan/02` §2.1) — never the whole document.

**Route by original file type first (`research/02` §1).** Cost breakdowns are very likely native `.xlsx`/`.xls`/`.csv`. Do **not** round-trip spreadsheets through PDF/OCR — read cells, sheet names, and formulas directly (openpyxl/pandas) and record the native path. The census only needs to *flag* `native_spreadsheet` so the eventual pipeline routes it correctly and the bake-off doesn't waste a slot OCR-ing an Excel export.

**Registry schema (draft — one row per artifact; a per-project view aggregates):**

| Column | Meaning |
|---|---|
| `project_id` | Stable id (existing project code if one exists; else assigned) |
| `project_name` | Human name |
| `year` | Approved / in-service year if inferable from folder or filename |
| `folder_path` | Canonical source path (SharePoint URL or UNC) |
| `file_name` | Artifact filename |
| `ext` | Extension, lowercased |
| `native_kind` | `pdf_born_digital` / `pdf_scanned` / `native_spreadsheet` / `docx` / `image_drawing` / `other` (heuristic: extension + text-layer probe) |
| `size_bytes`, `mtime` | Filesystem metadata |
| `artifact_type` | LLM guess: `DBM` / `scope` / `cost_report` / `lessons_learned` / `schedule` / `change_order` / `contract` / `drawing` / `unknown` |
| `type_confidence` | LLM confidence 0-1 (routes spot-checks) |
| `has_cost_signal` | Cheap flag: filename/first-page hints of dollar tables |
| `page_count` | For PDFs |
| `notes` | Free text (e.g. "password-protected", "duplicate of…") |

Per-project rollup (derived, drives sequencing and the §4 numbers): `has_scope_source`, `has_final_actuals`, `has_original_estimate`, `has_lessons`, `completeness_score`.

**Pseudocode sketch:**

```
for project_dir in crawl(archive_roots):          # recursive walk
    pid = derive_project_id(project_dir)
    for f in files(project_dir):
        row = filesystem_metadata(f)              # ext, size, mtime, path
        row.native_kind = sniff_kind(f)           # extension + PDF text-layer probe
        head = cheap_preview(f)                    # filename + first 1-2 pages / sheet names
        row.artifact_type, row.type_confidence = llm_classify(head)   # cheap model, JSON out
        row.has_cost_signal = looks_like_costs(head)
        write(registry, row)
# rollup
for pid, rows in group_by_project(registry):
    p.has_scope_source  = any(r.artifact_type in {DBM, scope} for r in rows)
    p.has_final_actuals = any(r.artifact_type == cost_report and r.has_cost_signal for r in rows)
    p.has_lessons       = any(r.artifact_type == lessons_learned for r in rows)
    p.completeness_score = weighted(p.has_scope_source, p.has_final_actuals, p.has_lessons, ...)
    write(project_rollup, p)
```

Human spot-check the classification on ~20 rows and every `type_confidence < 0.6`; the census is worthless if `cost_report` is being missed. `completeness_score` weighting is a **suggested starting point, not a validated formula** — e.g. scope 0.3 / actuals 0.5 / lessons 0.2, tuned once you see the distribution. The score only needs to *rank* projects for sequencing and bucket them for the report, not to be precise.

## 4. The measured fields that gate everything

Four numbers. Each is a query over the rollup plus, where the census can't tell, a human skim. Report each as a **count and a fraction of ~200**, sliced by asset class where the data allows.

- **(a) Scope + final actuals.** `count(has_scope_source AND has_final_actuals)`. This is the population that can ever enter cost statistics and backtesting (`plan/05` §1). The kill/reshape threshold lives here: **<100 usable actuals ⇒ tilt qualitative** (`plan/06` §1). "Usable" is stricter than "a cost file exists" — the file must plausibly contain a *final total*, not just a budget line; confirm on a sample by eye.
- **(b) Original-early-estimate recording rate.** The one Phase 0 was arguably created to measure (`plan/06` §1). For each project with actuals, is the organization's **own early estimate — value, AACE class if noted, and date** — actually on record (in the DBM, business case, or funding submission)? The headline business-case chart (`plan/05` §1 baseline (c): agent vs. actuals vs. the org's own early estimate) exists **only** if this field does. Measuring it needs eyes inside DBMs/business cases, not just filenames — sample ~20-30 projects, extend if the rate is borderline. If it's recorded for only a small fraction, say so plainly: the pitch shifts to baselines (a)/(b)/(d) and this is a **week-3 framing decision, not a month-6 surprise**.
- **(c) Lessons-learned coverage.** `count(has_lessons)` and a quick read on quality (a real post-mortem vs. a one-line closeout). Lessons feed adjustment reasoning and risk flags (`plan/02` §2.3); `plan/01` §4 predicts this is *most likely missing*, so confirm the damage.

Record all three with their measurement method (query vs. sampled-by-eye) and sample size — honesty about how a number was obtained is the report's credibility.

## 5. Parser bake-off protocol

Goal (`research/02` §1, §8): decide the PDF parser on **your** cost reports, not vendor leaderboards, and stand up eval before the pipeline exists.

- **Pick the 5 ugliest cost reports** from the census — old scans, merged cells, subtotal rows, multi-page financial-system exports. Deliberately adversarial; the crown-jewel parses are the hardest (`plan/02` §2.2). Exclude native spreadsheets — those bypass parsing entirely (§3).
- **Candidates:** **Docling** (open, self-host, strong table default), **Azure Document Intelligence** (managed, on-Azure compliance story), **vision-LLM fallback** (Gemini/GPT/Claude) for the worst scans (`research/02` §1). Governance (§2) may pre-decide this — if cost data can't leave the tenancy, Azure DI's managed path may be out and Docling/self-host wins by default; note that as a *finding*, not just a benchmark result.
- **Scoring rubric — table-cell accuracy against hand-verified ground truth.** For each report, hand-transcribe the key cost table(s) into a truth CSV, then score each parser on **cell-value accuracy** (correct numbers in correct cells), **structure fidelity** (rows/cols/merges preserved, subtotals not double-counted), and **failure mode** (silent wrong number = disqualifying; visible garble = recoverable). A small **suggested** scoring table (accuracy %, structure pass/fail, notes) per report × parser; pick the winner on worst-case cost-table fidelity, not average.
- **Golden extraction mini-set — start it now (`research/02` §8).** Take ~5 projects and hand-verify a slice of the eventual attribute schema — final total, capacity metric, asset type, site type — each with its source location and `null` where genuinely absent. This is not the full golden set; it is the seed that lets eval exist *before* the pipeline, so every later parser/model choice is decided by numbers. Two fully-verified projects by end of week 2 beats five half-done.

## 6. Estimator engagement

The senior estimators are both the domain oracle and the adoption gate (`plan/01` §3, `plan/02` §2.5). Meet **1-2 in week 1, the rest early week 2.** Goal is not schema design yet (that's Phase 1's workshop) — it's to extract the cost drivers, the comparable-finding heuristics, the canonical comparables (which become retrieval test cases), and where the early estimates hide.

**Interview guide (~10 questions):**

1. When you size a new project of *[asset class]* early, which 3-5 attributes drive the cost most? (per asset class — repeat)
2. How do you find comparable past projects today — memory, a spreadsheet, asking someone specific?
3. Name 2-3 past projects you'd reach for as *canonical* comparables for a typical new one, and why those. *(These become retrieval-quality test cases — `plan/05` §1.)*
4. Tell me about a project whose cost surprised you. What did the early estimate miss?
5. Where does the *original early estimate* physically live — DBM, business case, a funding memo, a system? Is its AACE class or date written down?
6. What normalizes a 2011 cost to today in your head — a rule of thumb, an index, gut?
7. What makes two superficially-similar projects cost very differently (brownfield, season, site, era)?
8. Which cost reports do you trust, and which formats are a mess to read?
9. What would make you *distrust* a tool like this on sight?
10. If this worked, what's the first question you'd ask it?

Capture the named canonical comparables verbatim into the golden set — they are the cheapest retrieval ground truth you will ever get, and they buy estimator buy-in by making their expertise the yardstick.

## 7. SEERai calibration demo (`plan/06` ledger #9)

SEERai (Galorath, Oct 2025) is the closest commercial analog — agentic NL→audit-ready estimates over validated historicals (`research/07` §5.5). Phase 0 books a demo as a **calibration point, not a purchase evaluation.** We are building (ledger #9); the demo is to learn, not to buy.

- **Ask Galorath for:** a live walkthrough on a *utility/energy/infrastructure* example if they have one; specifically how "Instant RAG" grounds outputs and shows source traceability; how a user overrides comparable selection; and — the load-bearing question — **what corpus it needs and how it ingests heterogeneous prose (DBMs, lessons learned)** vs. structured historicals.
- **Score it on:** evidence-chain quality (does every number cite a real source?), comparable transparency and override, how it handles *missing* data and *our-corpus* ingestion, and range/calibration honesty.
- **What we're looking to learn:** where the audit-ready bar actually sits now (it's table stakes, not a moat — `research/07` §5.5), UI patterns worth stealing, and confirmation of our real edge: **depth on this utility's own DBM/lessons-learned corpus**, which no vendor ingests (`research/07` §5.7). If SEERai already does something we planned to build, that's a finding for the report, not a failure.

## 8. The inventory reality report — output template

The Phase-0 deliverable (`plan/02` §2.1, `plan/06` §1). It **replaces the assumptions table in `plan/01` §4 with measured fact** and frames the kill/reshape decision for leadership. Keep it tight; the four numbers and the decision are the point.

1. **Executive summary** — the four gating numbers and the go / reshape / kill recommendation, on one page.
2. **Corpus census** — project count, file-type distribution, artifact-type coverage, native-spreadsheet share (§3).
3. **Completeness distribution** — histogram of completeness scores; the count with scope + actuals (§4a) against the **<100 kill line**.
4. **Original-early-estimate recording rate** (§4b) — the number, its measurement method, and the business-case-chart implication.
5. **Lessons-learned coverage & quality** (§4c).
6. **Parser bake-off results** (§5) — winner, per-report cell-accuracy table, governance constraint noted.
7. **Estimator findings** — cost drivers per asset class, how comparables are found today, the named canonical comparables (→ golden set).
8. **SEERai calibration notes** (§7).
9. **Access & governance status** — what's granted, what's pending, endpoint/no-training confirmation.
10. **Kill / reshape decision** — the explicit call: full build as planned, or, if **<100 usable actuals**, tilt toward qualitative comparables with the statistical layers shrunk (`plan/06` §1) — decided *here*, not in month six.
11. **Appendices** — registry schema, access-request log with dates, golden-set seed.

## 9. Exit checklist (1:1 with `plan/06` §1 exit criteria)

- [ ] **We know how many projects have (scope + final cost)** — number reported, sliced by asset class, measured against the <100 kill line. *(→ §4a)*
- [ ] **The original-estimate recording rate is measured** — value + method + sample size; business-case-chart viability called. *(→ §4b)*
- [ ] **Leadership has seen the real coverage numbers** — reality report walked through, kill/reshape decision taken and recorded. *(→ §8)*
- [ ] **Kill/reshape signal evaluated** — if <100 usable actuals, the qualitative-tilt reshape is explicit in the report, not implied. *(→ §8 #10)*

Supporting artifacts that should also exist leaving Phase 0 (enablers for the above, not separate gates): `registry.csv` over the full corpus; the parser bake-off winner chosen on cost-table fidelity; the golden extraction mini-set seeded (≥2 verified); the approved-endpoint / no-training guarantee confirmed in writing; the named canonical comparables captured as retrieval test cases.
