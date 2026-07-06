# HANDOFF — working state

**Purpose of this file:** if the session ends partway, a new agent (or Graeme at work) reads this to know exactly where things stand and what to do next.

## The assignment (verbatim intent)

Build a research base + project plan for an agentic system that:
- Ingests ~200 past capital projects (DBMs, scope docs, lessons learned, final costs; heterogeneous formats, some data missing)
- Lets engineers/PMs describe a new project scope
- Reasons over historicals (similarity, greenfield/brownfield, summer/winter construction, weather delays, station sizes, complexity) to price out the new project
- Inspired by Karpathy's LLM wiki / agent-memory ideas; grounded in current SOTA

Deliverables: exhaustive research in markdown (`research/`), plus an end-to-end project plan (`plan/`) covering the full flow, knowledge sources and how each is used, tradeoffs with pros/cons, and a single recommended path.

## Plan of work

- [x] Repo created, pushed public as `DoubleJumped/capital-project-cost-agent`
- [ ] **Phase 1 — Deep research** (in flight, workflow run `wf_af8d7aa8-0be`, all subagents pinned to Opus per Graeme's usage constraint — orchestrator stays Fable, subagents must be `model: 'opus'`): fan-out web research on 7 threads (7th = prior-art/commercial landscape). Each thread: research+write → adversarial critique → revise. A separate deep-research backbone workflow was deliberately dropped to conserve usage.
  1. Agent memory architectures (Karpathy LLM wiki, MemGPT/Letta, mem0, A-MEM, knowledge distillation into curated docs)
  2. RAG / GraphRAG over heterogeneous document corpora; document parsing & structured extraction pipelines
  3. LLM + case-based reasoning / analogous estimating; retrieval of comparable cases
  4. Construction cost estimation practice: AACE estimate classes, reference class forecasting, escalation & normalization, parametric estimating
  5. Similarity modeling across project attributes (categorical + numeric + text); hybrid retrieval
  6. Uncertainty, ranges, calibration for cost predictions; eval of estimation agents
  7. Prior art: commercial tools & published AI cost-estimation deployments
- [x] **Phase 2 — research docs written by workflow agents directly** into `research/01..07-*.md` (an auto-commit loop pushes snapshots every 3 min while the workflow runs)
- [~] **Phase 3 — Architecture plan** in `plan/`: drafted ahead of research completion — `01-problem-frame`, `02-corpus-pipeline`, `03-knowledge-base`, `04-agent-flow`, `05-evaluation-and-learning` done; still to write: `06-delivery-roadmap` (stack, phases, risks, tradeoff summary, recommended path) and `00-executive-summary`. After research lands: reconcile plan docs against research findings and fix cross-references.
- [ ] **Phase 4 — Final README** tying it together; final push.

## If resuming after interruption

1. Check `research/` — 7 docs expected. If the workflow died mid-run, resume with the scriptPath+resumeFromRunId noted above, or just spawn Opus agents per missing thread using the briefs in the workflow script.
2. Write `plan/06` + `plan/00` if missing; reconcile plan ↔ research cross-references.
3. Rewrite README as the entry point; remove wip commits noise is fine (history stays).

## Conventions

- Commit + push after every completed doc (`git push origin main`).
- All docs are self-contained markdown; cite sources with URLs.
- Domain context: Canadian gas utility (SaskEnergy-like), prairie climate (winter construction is a real constraint), stations + pipelines as typical assets.
