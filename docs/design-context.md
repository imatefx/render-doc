# QuarterMaster — Design Context (from architecture review, 6 Nov 2026)

Baseline for the clean architecture of QuarterMaster. Anything in the current code is "what exists now"; what is written here is "what we are designing toward". Companion artifacts: [Architecture Decision Log](architecture-decision-log.md) and [Target Architecture Reference](target-architecture-reference.md). Source docs read: End-to-End Flow, Architecture Standard (2026-11-05), Database Schema (head d3a58b7c1e94), test-2 Sonnet spend report (Oct 26–30).

## 1. What Render and QuarterMaster are

- **Render** is a task-management / project-management platform for telecom construction (Jira-like). Clients send engineering design files (AutoCAD DWG/DXF, shapefiles in ZIP, KML/KMZ, GeoJSON, PDF). Render staff (NDMs) read them, work out tasks and dependencies, and create tasks with forms that crews fill in the field. Today this is mostly manual plus scripts.
- **QuarterMaster** automates that flow ("blueprinting"). It is a menu inside the Render web app, scoped to the tenant the user signed into. Tenants contain projects. Output is a blueprint of Render tasks written into Render's task tables.
- **RDL** (Render Domain Language) is a small DSL. An RDL program plus one or more GeoJSON designs → tasks, quantities, asset IDs, dependencies. Compile is deterministic, no AI.

## 2. Target user flow (screen by screen)

1. **Start** – pick/create QuarterMaster project (checked against Render project ACL).
2. **Upload** – choose job code (unit code) list for the project; upload design files. Front end validates format (ZIP is a ZIP; shapefile sets complete; DWG/KML/GeoJSON valid). Files go browser → S3 via presigned POST. `complete` queues extraction.
3. **Reading design** – extraction on Fargate (GDAL, ezdxf, KML parser, LibreDWG): verify → segregate ZIP → extract per source → GeoJSON with stable feature IDs, `properties.layer` on every feature. GeoJSON stored content-addressed in S3 as the immutable baseline. Layer summary (name, count, geometry kinds) saved on `source_file`.
4. **Choose layers** – user unticks layers that carry no work. **Only kept layers go any further** (profiling, AI). This happens *before* any AI call.
5. **Task flow** – pick a Render-default or tenant template of **activities** (see §4) or edit it. Join is gated on this.
6. **Design analysis** – per-layer draft calls run in parallel; join writes the RDL; question planner picks relevant questions from the ~112; user answers one by one; stats shown ("we identified this much").
7. **Preparing** – apply answers → validate (incl. task-flow conformance) → fix loop → compile → Project Knowledge → unit-code mapping. Run becomes `ready`.
8. **Review & validate** – map, asset packages, feature table, confidence per layer (<75% = needs review, ≥75–80% = complete but still editable). Edits go to a change set in Postgres, not to the GeoJSON. **Ask AI** chat panel: select a layer, ask for a change → RDL and Project Knowledge updated together → rerun → diff → merge.
9. **Configure** – job codes (many-to-many with tasks) → planned quantity / BoQ report; forms per task (see §6); recap. Grouped task list and dependency graph in a React Flow UI.
10. **Blueprint** – pre-flight, BoQ, Create → finalised run; tasks written to Render in batches (idempotent via `integration_metadata.quartermaster.task_key`).

## 3. Drafting pipeline (the core decision)

```
extract → choose layers → profile kept layers (engine, no AI)
  → draft_layer × N  (one small LLM call per kept layer → small JSON)
  → [task flow confirmed]
  → join            (one LLM call: all layer JSONs + knowledge slice + activities → RDL TEXT)
  → parse RDL → AST (engine, code)
  → questions       (LLM picks relevant questions from the AST)
  → apply answers   (edits on the AST, code) → serialize AST → RDL text
  → validate (+ task-flow conformance) → fix (LLM, ≤2–3 attempts per invocation, chained)
  → compile (code) → project knowledge (LLM) → unit code mapping (LLM where rules can't)
```

- **Per-layer output is small JSON, not free text**: layer name, attribute keys, field mappings, Render asset-type guess (customer's splitter/ONT/OLT → Render default asset e.g. equipment/closure), confidence. Structured so the join writes exact match blocks; validate before spending the join call.
- **Asset mapping is folded into the per-layer call.** No separate `map_assets` step.
- **Join writes RDL text**, the compact DSL the model is good at. No whole-program draft; initial run and revision use the same path. Revision: only uncovered layers run `draft_layer`; join merges with the base run's AST.
- **Engine parses RDL → JSON AST and serializes AST → RDL in code.** LLM never reads/writes JSON AST (2–4× tokens). Questions, overlay edits, task-flow conformance, Project Knowledge and diff view all operate on the AST. **OPEN: confirm the engine has parse + serialize, that the round trip is lossless, and that rule IDs are stable.**
- **Questions run after the join**, because many questions depend on which Render asset types were identified. Only **submitted** answers are stored on the project (`project_setting.discover`); defaults are never copied. Each run pins the questions knowledge version (`run.knowledge_version_ids`) so "answered X, defaulted Y from version N" is reconstructable and new defaults reach new runs.
- Edits on Review are a change set: append-only `overlay_version` / `tag_overlay` / `feature_exception`, merged with the baseline GeoJSON at compile time.
- Runs are immutable; every change creates a new run and moves `project.current_run_id`. Project Knowledge (human-readable markdown of what the RDL does) is always written in sync with the RDL.

## 4. Task flow = contract; unit = activity

- The task flow is the **source of truth** for which tasks exist in a project and their coarse order. The RDL must conform: every task type in the RDL exists in the flow; every RDL dependency respects flow order. Engine validation rule, no AI. Task type outside the flow → error through fix. Feature type with no activity → "we don't know what to do with this" notice.
- **Activity** = one task type placed at one position in the flow, with `id`, `task_type`, `milestone`, `crew_days` (flat for v1; rate × quantity later), `predecessor activity ids`. The same task type may appear on several activities (e.g. pole inspection in pre-construction and close-out). Task flow template = ordered list of activities. Every compiled task belongs to exactly one activity (`staged_task.placement_id`, becomes a real compile output).
- Milestone and crew days live on the activity, **not in the RDL** (removes the unknown-milestone fix loop). Configure › Dependencies mostly collapses into the task-flow page.
- Task type on two activities → join needs a discriminating condition; if none, surfaced as an open question.
- Editing the task flow after the RDL exists → targeted edit → diff → merge (same path as a knowledge change). Task-flow hash is part of the replay key.
- Confirm the word "activity" with Render (alternative: "work step"). Avoid "stage", "step", "placement", "card".

## 5. AI ownership and hosting

- **Retire `quartermaster-agents`** (LangGraph: asset mapping, whole-program draft, chat).
- **RDL engine (TypeScript, Node 22)** owns every LLM call that touches RDL: `draft-layer`, `draft-join`, `fix`, `edit`, `diff-explain`, `project-knowledge`, chat `apply-change`. Non-AI: `profile`, `validate`, `compile`, `inspect`, `parse`, `serialize`.
- **Python backend (render-services)** owns every non-RDL LLM call as plain LLM calls: question planner, unit-code selection, job-code suggestions, form matching, conversational chat. Backend stays the orchestrator (stage rows, calls the engine).
- **Engine runs as a Lambda container image**, invoked directly by the stage worker (IAM Invoke, no HTTP/API key). 15-min timeout. API Lambda (29 s) never calls engine or LLM inline → review recompiles become a job (ADR-07 mandatory). Design cache is per warm container only.
- **Time budget rule:** no invocation does more than one LLM call + one deterministic step. Fix attempts capped at 2–3 per invocation, chained by the backend, attempt count on the run. Extraction stays on Fargate (only step that can exceed 15 min).
- Engine reads designs by reference `{sha256, bytes, presigned URL, excluded_layers}` (ADR-04). Presigned vs read-only IAM needs Render's call.

## 6. Forms

- Buttons today: change task status, create follow-up task, attach photo, flag issue. No arbitrary logic.
- **Option 3 chosen:** QuarterMaster owns template, viewer and logic; Render's crew app opens a QuarterMaster link with a short-lived signed token (task, user, tenant); viewer posts to Render's submission API and task API. Render's own form builder/viewer is limited and support for extending its schema is uncertain.
- Closed vocabulary: field types; condition operators (equals, not empty, greater than…); button actions mapping 1:1 to Render API calls. Templates versioned per tenant; project-level edits are new versions attached to the project. Baseline default form per task; editable later.
- Forms should attach to an **activity**, not a task type (to confirm with Render/NDMs).
- Needs own tables (`form_template`, `form_template_version`, `form_mapping`) instead of the `forms` key in `project_setting`.
- To define with Render: submission payload shape (flat vs structured, photos/signatures, template version per submission), task API for buttons, token handoff.

## 7. Knowledge

Kinds (code): `system, domain, project, questions, forms, task_types, telecom, blueprint, customer`. Stored as versioned markdown in S3 (`knowledge/<kind>-knowledge.<token>.md`) with `knowledge_version` index rows; design conversations said Databricks volume. **Decision: one store with versions; S3 + index is the source of truth, Databricks reads from it if needed (RA-08).**

Hierarchy: **Render domain knowledge (global)** → **tenant/customer knowledge** → **project knowledge** (generated per project, kept in sync with its RDL).

### Spend findings (test-2, Oct 26–30, Sonnet)

- 10,300 DBU ≈ $720 list; 4,907 LLM calls; **87% input tokens**; **79% from `rdl_run`**.
- Each RDL call ≈ 130k input: whole `knowledge/global/` (~260k chars ≈ 65k tok) + ~280k-char system prompt. ~12 RDL calls per run (max 27) → knowledge resent 12+ times per run. 90–99% of input is fixed text; 53% of prompts exact repeats.
- Knowledge sizes: form_mapping 35.6k tok; question_and_answer_tree 24k; RDL instructions + task types ~5k; element-task-matching 3.1k; design_extraction 2.4k. **85% of what every RDL call carried (forms + questions) is never used by RDL drafting.**
- Other leaks: `ask_questions` resends a 78k-char design brief 4× per run; `chat_agent` ~160k input per call.
- Databricks rates: 28.571 DBU/1M input, 142.857 output, **1.429 cache read (20× cheaper)**. Report estimate: $3.80 → $1.50 per run with caching alone.

### Knowledge strategy

1. **Slice by consumer**, not just topic: per-layer ← design_extraction + element_task_matching (~6k tok); join/fix/edit ← those + RDL instructions + task types (~8–10k); planner ← questions; form matching ← forms. Per-layer input drops ~130k → <10k.
2. **Prompt caching**: knowledge slice as a byte-identical prefix, pinned per run, layer profile after it. Ten per-layer calls share one cached prefix.
3. Retrieval of candidate asset types by layer name/attribute match only if vocabulary is still too large after 1–2.
4. Ask Jude: reorganise the knowledge folder by consumer now, or slice at prompt-assembly time? (First is simpler for versioning and the replay key.)

### Knowledge promotion flow (new, from voice note — to design)

- Every project generates Project Knowledge, so customer-specific learnings accumulate there.
- Need an NDM flow (e.g. monthly): review all Project Knowledge, find what is new, judge validity, and **promote** items either to that tenant's customer knowledge or to global Render domain knowledge.
- Structure of the current knowledge folder (global vs tenant) to be provided as a screenshot. Flow not yet designed: needs a review queue, diff vs existing knowledge, promote/reject actions, new knowledge version, and "changes pending" on affected projects (existing targeted-edit path).

## 8. Pipeline mechanics kept from current code

- One `stage` row per unit of work, unique idempotency key, enqueued in the same transaction as the state change, claimed by conditional UPDATE, exactly once; sweeper Lambda re-drives stale rows. Counters (`files_total/registered`, `sources_finished`, `layers_total/done`, `join_enqueued`) via single `UPDATE … RETURNING`.
- Determinism: temperature 0, pinned model, `llm_cache` by input hash; parse/serialize/validate/compile in code. Replay key = project ID, GeoJSON hashes, overlay version, exceptions version, knowledge version IDs, vocabulary hash, **task flow hash**, answers, pinned asset IDs, model version.
- Tenant isolation: `tenant_id` on every row, composite FKs, S3 keys under `<prefix>/<tenant_id>/`.
- Fragment library (`rdl/fragment-library/<hash>.json`) and `extraction_key` reuse — keep both.
- `require_editable()` refuses writes once blueprinted; finalised runs are read-only.

## 9. Open items

| ID | Item |
|---|---|
| OPEN-01 | RDL parse/serialize round trip, losslessness, stable rule IDs |
| OPEN-02 | Exact per-layer JSON schema and how the join consumes it |
| OPEN-03 | Form spec v1 (fields, condition grammar, button actions, versioning, submission payload) |
| OPEN-04 | Which of the 112 questions edit which AST fields |
| OPEN-05 | Crew days flat vs rate × quantity |
| OPEN-06 | What to keep from agents service before retiring (validator, retry loop, chat graph) |
| OPEN-07 | Fragment library key still valid after per-layer output → JSON |
| OPEN-08 | LibreDWG/`dwg2dxf` missing from Fargate image |
| OPEN-09 | PostGIS vs GeoJSON in S3 |
| OPEN-10 | Vector tiles for review map |
| OPEN-11 | UI sends no `expected_versions` on Configure saves |
| NEW | Knowledge promotion flow (§7) |
| NEW | Knowledge slicing by consumer + caching (§7) |

## 10. Needs Render approval

RA-01 retire agents · RA-02 engine as Lambda in Render account, own repo, pipeline owner · RA-03 presigned URLs vs IAM · RA-04 task flow as contract, "activity" vocabulary, milestone/crew days on activity · RA-05 forms attach to activity · RA-06 form viewer link + submission/task API shapes + token handoff · RA-07 review recompiles as job (UX pending state) · RA-08 S3 as knowledge source of truth · RA-09 per-tenant feature flag · RA-10 malware scanning.

## 11. Vocabulary

- **Blueprinting** – converting a project's design into tasks and pushing to Render.
- **Activity** – a task type at one position in the task flow (+ milestone, crew days, predecessors).
- **Task type** – Render's vocabulary (e.g. "install aerial strand"); unchanged.
- **Unit code / job code** – Render quantity codes; many-to-many with tasks; drive the planned quantity (BoQ) report.
- **Run** – one immutable pipeline execution; project points at `current_run_id`.
- **Overlay / change set** – user edits stored in Postgres and merged with the baseline GeoJSON at compile.
- **Project Knowledge** – readable markdown kept in sync with the project's RDL.
- **Join** – the single LLM call that writes RDL text from per-layer JSONs.
- **AST** – the RDL program as JSON, produced by the engine's parser.
