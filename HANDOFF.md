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
- [ ] **Phase 1 — Deep research** (in flight): fan-out web research on 6 threads:
  1. Agent memory architectures (Karpathy LLM wiki, MemGPT/Letta, mem0, A-MEM, knowledge distillation into curated docs)
  2. RAG / GraphRAG over heterogeneous document corpora; document parsing & structured extraction pipelines
  3. LLM + case-based reasoning / analogous estimating; retrieval of comparable cases
  4. Construction cost estimation practice: AACE estimate classes, reference class forecasting, escalation & normalization, parametric estimating
  5. Similarity modeling across project attributes (categorical + numeric + text); hybrid retrieval
  6. Uncertainty, ranges, calibration for cost predictions; eval of estimation agents
- [ ] **Phase 2 — Write research docs**: one markdown file per thread in `research/`, with sources cited. Commit after each.
- [ ] **Phase 3 — Architecture plan** in `plan/`: corpus curation pipeline, project knowledge base ("project wiki") design, retrieval + reasoning flow, estimation methodology, UX flow, eval & calibration loop, phased delivery roadmap, tradeoff tables, recommended end-to-end path.
- [ ] **Phase 4 — Final README** tying it together; final push.

## Conventions

- Commit + push after every completed doc (`git push origin main`).
- All docs are self-contained markdown; cite sources with URLs.
- Domain context: Canadian gas utility (SaskEnergy-like), prairie climate (winter construction is a real constraint), stations + pipelines as typical assets.
