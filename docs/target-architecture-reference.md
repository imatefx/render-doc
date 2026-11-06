# QuarterMaster — Target Architecture Reference

Nov 6, 2026 · Stalin Pereira

The target architecture after the 6 November 2026 design review: the source for the overview, component, sequence and deployment diagrams. Decisions are referenced by their IDs in the Architecture Decision Log (QM-nn, RA-nn, OPEN-nn).

## 1. System context

QuarterMaster is a feature of the Render platform that turns an uploaded telecom network design into a blueprint of Render tasks ("blueprinting"). It lives inside the Render web app under a QuarterMaster menu, scoped to the tenant the user signed into, and writes its final output into Render's own task tables.

**Actors**

| Actor | Role | Enters through |
| --- | --- | --- |
| Uploader (customer project user) | Creates a QuarterMaster project, picks the job code list, uploads design files, chooses layers, confirms the task flow, answers questions, reviews and edits, maps job codes and forms, creates the blueprint | Render web app → QuarterMaster MFE |
| NDM (Network Deployment Manager, Render domain expert) | Everything the uploader does, plus maintains Render Domain Knowledge, task-flow templates, form templates, learned mappings; reruns old projects to test knowledge changes | Same |
| Render engineering | Maintains System Knowledge, configuration, feature flag | Same + infra |
| Crew member | Fills forms against tasks in the field | Render crew app → QuarterMaster form viewer link |

**External systems**

| System | Direction | Used for |
| --- | --- | --- |
| Render platform (respawned schema, REST, Okta) | Both | Tenants, projects, users, ACLs; job code lists; task types; task tables written by the blueprint; task status and submission APIs called by the form viewer |
| Render crew app | Inbound | Opens the QuarterMaster form viewer with a signed task/user/tenant token (RA-06) |
| Databricks model serving | Outbound | Every LLM call (two callers only: RDL engine and backend, QM-12) |
| Mapbox | Outbound (browser) | Map preview, tag editing, task map |
| Okta | Inbound | API Gateway authorizer |

**Boundary.** Inside: the QuarterMaster MFE, the QuarterMaster routers and services in render-services, the stage worker, the long-running worker, the RDL engine, the form viewer, the QuarterMaster Postgres tables and S3 areas. Outside: everything above. The quartermaster-agents service is removed (QM-12).

## 2. Components

Six runtime components plus two stores. Each has one job and is the single writer of its own data. Arrows for the component diagram: the "Calls" column.

| Component | Repo / location | Runtime | Responsibility | Interface exposed | Calls |
| --- | --- | --- | --- | --- | --- |
| **QuarterMaster MFE** | scale-respawned `src/features/quartermaster` | Static bundle, single-spa, React 18, MUI, SWR | Discover wizard (Project → Upload → Design layers → Task flow → Design analysis → Review), Configure (Job codes, Forms), Blueprint, Knowledge, Form builder. Polls progress. Records last screen for resume | Browser UI | API (HTTPS via API Gateway); S3 (presigned POST upload); Mapbox |
| **QuarterMaster API** | render-services `lambdas/scale/rest/routers/quartermaster` | Lambda (Mangum, FastAPI, Python 3.11), 29 s API Gateway limit | Projects, uploads, layers, task flow, progress, questions, review read models, overlays, chat threads, knowledge, job codes, forms, blueprint. Writes stage rows that start work. Never runs an LLM call or a compile inline (QM-13, ADR-07) | `/rest/v1/projects/{project}/quartermaster/…` (ProjectACL), `/rest/v1/tenants/{tenant}/quartermaster/…` (TenantACL) | Postgres; S3 (presigned URLs); SNS (publish stage) |
| **Stage worker** | render-services `worker/jobs/quartermaster` | SNS-triggered Lambda, 15 min | Runs every stage except extraction: `rdl_generation` chain (per-layer draft, join, questions, apply answers, validate\_fix, compile, project\_knowledge), `unit_code_mapping`, `recompile`, `edit`, `chat`, `blueprint`, `project_delete`. Backend LLM calls (question plan, unit code selection, job code suggestions, form matching, conversational chat) live here | Stage rows claimed by conditional UPDATE | Postgres; S3; RDL engine (direct Lambda invoke); Databricks; Render task API (blueprint) |
| **Long-running worker** | render-services `common/quartermaster/extract` | ECS Fargate, GDAL image (+ LibreDWG, OPEN-08) | Job `quartermaster_extract_upload`: verify → segregate → extract → layer summary. Only reader of raw uploads and only writer of extracted GeoJSON (ADR-01) | Stage rows with `runtime = lrw` | Postgres; S3 |
| **Sweeper** | render-services | Scheduled Lambda | Re-publishes stale or stuck stage rows | — | Postgres; SNS |
| **RDL engine** | rdl-engine (own repo, ADR-09) | Lambda container image, Node 22, 15 min, stateless | No AI: `profile`, `validate` (incl. task-flow conformance, QM-07), `compile`, `inspect`, `parse` (RDL → AST), `serialize` (AST → RDL) (QM-04). AI: `draft-layer`, `draft-join`, `fix`, `edit`, `diff-explain`, `project-knowledge`, `apply-change`. Also a CLI with the same actions | One action per invocation; design passed by reference `{sha256, bytes, presigned URL, excluded_layers}` (ADR-04) | S3 (presigned GET); Databricks |
| **Form viewer** | new, served by render-services or the MFE (OPEN-16) | Static bundle | Renders a form template against a task for the crew app, evaluates conditions and button actions (QM-11) | Link opened by the crew app with a signed token | API (template, context); Render submission API; Render task API |
| **Postgres** | RDS, schema `respawned`, prefix `quartermaster_` | — | All workflow state: 29 tables + forms tables (QM-11) | SQL | — |
| **S3** | one bucket, `<prefix>/<tenant_id>/<area>/<key>` | — | Uploads, content-addressed GeoJSON and profiles, knowledge versions, RDL programs and ASTs, task files, hub, tenant library | Presigned POST/GET | — |

**Removed from the current code:** quartermaster-agents (LangGraph; asset mapping, whole-program draft, chat) — its calls are redistributed per QM-12. **Not QuarterMaster's:** multi-tenancy, auth, job code and task type APIs, task ingestion, form filling against tasks (Render owns these).

## 3. Data ownership

Every piece of design data has exactly one writer. Content-addressed objects are immutable and never deleted.

| Data | Location | Written by | Read by |
| --- | --- | --- | --- |
| Raw upload | S3 `uploads/projects/<qm_project_id>/uploads/<upload_id>/<file_id>/<name>` | Browser (presigned POST) | Long-running worker |
| Extracted GeoJSON | S3 `geojson/<sha256>.geojson`; `source_file.geojson_key/hash/feature_count/layers` | Long-running worker | DesignLoader (API, stage worker), RDL engine (by reference), blueprint writer |
| Layer summary (name, count, geometry kinds) | `source_file.layers` | Long-running worker (ADR-06) | Design layers page |
| Left-out layers | `upload.excluded_layers` | API (layers confirm) | DesignLoader, RDL engine (applies identically, ADR-04) |
| Design profile | S3 `profiles/designs/<design_hash>.json`; `run.profile_key` | Stage worker at `rdl_generation/start` via engine `profile` (reused by hash, ADR-03) | Per-layer draft |
| Per-layer draft JSON | `run_fragment` (run, layer) + S3 `rdl/fragment-library/<tenant>/<sha256>.json` | Stage worker via engine `draft-layer` | Join |
| Task flow (activities) | `project_task_flow`, `task_flow_template`; S3 `rdl/task-flows/<hash>.json` per run | API (user), blueprint (template save) | Join, engine `validate`, `compile`; replay key (QM-10) |
| RDL program text + AST | `run.rdl_text`; S3 `rdl/runs/<run_id>/program.rdl`, `program.ast.json` | Stage worker via engine `draft-join` / `serialize` | Engine `validate`, `compile`, `edit`, `project-knowledge`; questions apply; review |
| Question plan | `run.question_plan` | Stage worker (backend LLM) | Design analysis page |
| Answers (submitted only, QM-06) | `project_setting` key `discover` | API | Apply-answers step |
| Knowledge (system, domain, project, questions, forms, task\_types, telecom, blueprint, customer) | S3 `knowledge/<kind>-knowledge.<token>.md`, `knowledge/projects/<id>/project-knowledge.<token>.md`; index `knowledge_version` | Knowledge service (S3 first, then index row); engine `project-knowledge` for project kind | Draft, join, questions, chat, each through its own slice (QM-16); `run.knowledge_version_ids` pins them (RA-08) |
| Overlay (tag overrides, exceptions) | `overlay_version`, `tag_overlay`, `feature_exception` (append-only) | API (review edits, chat proposals) | DesignLoader → engine `compile` |
| Compiled plan | `run_feature`, `staged_task` (with `placement_id` = activity, QM-07), `staged_dependency` | Stage worker `compile` persist | Review, mapping, blueprint |
| Unit / job code mapping | `feature_mapping`, `mapping_review`, `learned_mapping` (tenant-wide) | Stage worker `unit_code_mapping`; API (manual pick) | Review, BoQ, blueprint |
| Form templates and activity mappings | new tables `form_template`, `form_template_version`, `form_mapping` (QM-11) | API (form builder) | Form viewer, blueprint, recap |
| Form submissions | Render (its submission API) | Form viewer | Render |
| Chat | `chat_thread`, `chat_message` (pending → done/failed, `proposal_run_id`) | API, stage worker `chat` | Review Ask AI panel |
| LLM log and cache | `llm_call`, `llm_cache` (global `cache_key`) | Engine and backend LLM clients | Analytics, run cost totals |
| Stage rows and events | `stage`, `pipeline_event` | API (enqueue), workers (claim, complete) | Workers, sweeper, progress page |
| Blueprint push | `blueprint_push`; Render `respawned.tasks` with `integration_metadata.quartermaster.task_key` | Stage worker `blueprint` | Render; retry skips written tasks |
| Tenant library | S3 `library/tenant-library/<tenant>.json` | Blueprint create | New projects |

**Knowledge hierarchy and slices.** Render domain knowledge (global) → tenant (customer) knowledge → Project Knowledge, generated per project in sync with its RDL. Promoting Project Knowledge upward is OPEN-12. Each LLM caller reads one slice, sent as a byte-identical cached prefix pinned per run (QM-16, QM-17); sizes are from the test-2 spend report.

| Consumer | Knowledge slice | Size (tokens) |
| --- | --- | --- |
| Per-layer draft | design\_extraction + element\_task\_matching | \~6k |
| Join, fix, edit | per-layer slice + RDL instructions + task types | \~8–10k |
| Question planner | questions (question\_and\_answer\_tree) | \~24k |
| Form matching | forms (form\_mapping) | \~35.6k |

## 4. Pipeline stages

One stage row per unit of work, enqueued in the same transaction as the state change, claimed by conditional UPDATE, exactly once. Order on an initial run; `*` marks an LLM call. No invocation holds more than one LLM call plus one deterministic step (QM-14).

| # | Stage · step | Runtime | Trigger | Input | Output | Next |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | `extract · verify` | Fargate | Upload complete (one per file) | upload\_file | kind, hash, status | 2 or 3 |
| 2 | `extract · segregate` | Fargate | ZIP verified | ZIP | one source\_file per shapefile set | 3 (per set) |
| 3 | `extract · extract` | Fargate | Source registered | source file | `geojson/<hash>.geojson`, layer summary, counters; last source marks upload ready | — (user chooses layers) |
| 4 | `rdl_generation · start` | Lambda | Layers confirmed | GeoJSON refs, excluded layers | run row, design hash, design profile (engine `profile`, reused by hash) | 5 (one per kept layer) |
| 5 | `rdl_generation · draft_layer` \* | Lambda | start done | profile slice for one layer + per-layer knowledge slice as a cached prefix (QM-16, QM-17); the first layer is sent alone to warm the cache | per-layer JSON: asset type, attribute keys, field mappings, confidence (`run_fragment`); `layers_done` += 1 | 6 when `layers_done = layers_total` (`join_enqueued` guard) |
| 6 | `rdl_generation · join` \* | Lambda | All layers done AND task flow confirmed | all layer JSONs + join knowledge slice (QM-16) + task flow (activities) | RDL text + AST (engine `draft-join`, `parse`), assumptions | 7 |
| 7 | `rdl_generation · questions` \* | Lambda | join done | AST + questions knowledge | question plan on run; run status `waiting_for_answers` | — (user answers) |
| 8 | `rdl_generation · apply_answers` | Lambda | questions continue | AST + submitted answers + defaults from pinned questions version | edited AST, RDL text (engine `serialize`) | 9 |
| 9 | `rdl_generation · validate_fix` (\*) | Lambda | apply done | RDL, task flow, vocabulary | validation result; one fix attempt per invocation (engine `fix`), chained | 9 again or 10 |
| 10 | `rdl_generation · compile` | Lambda | validated | RDL + GeoJSON ref + overlay + task flow | tasks, dependencies, run features, `placement_id` per task, no-task notices (engine `compile`); persisted | 11 |
| 11 | `rdl_generation · project_knowledge` \* | Lambda | compiled | AST | Project Knowledge v1 (engine `project-knowledge`) | 12 |
| 12 | `unit_code_mapping` \* | Lambda | knowledge done | tasks grouped by (task type, descriptor); learned mappings; job code list | feature\_mapping, mapping\_review, planned quantities; run `ready` | — (user reviews) |
| 13 | `recompile` | Lambda | overlay edit or manual code pick | current RDL + new overlay | new run (parent set), re-mapped changed tasks | — |
| 14 | `edit` \* | Lambda | chat Apply or knowledge / task-flow change | current AST + changed knowledge or task flow | proposed run with RDL diff + explanation (engine `edit`, `diff-explain`); `proposed` until merged | 9 → 12 on merge |
| 15 | `chat` \* | Lambda | chat message | thread, focus, Project Knowledge | assistant message; optional proposal (→ 14) | — |
| 16 | `blueprint` | Lambda | Create blueprint | finalized run | task types, milestones, subsectors, tasks with geometry and quantities written to Render in batches; `blueprint_push` progress | — |

On a **revision** (new upload into a project with a program) only uncovered layers run step 5; the join merges with the carried-forward AST (`compile_summary.base_run_id`). On a **rerun** every layer runs from scratch.

## 5. Sequence: Discover to Blueprint

Lifelines for the sequence diagram: **Browser**, **API** (Lambda), **Stage worker** (Lambda), **LRW** (Fargate), **RDL engine** (Lambda), **Databricks**, **Postgres**, **S3**, **Render platform**. Numbers give the order within a screen; the screen polls wherever a job runs.

| Screen | # | From → To | Call | Effect |
| --- | --- | --- | --- | --- |
| 1 Start | 1 | Browser → API | `GET tenants/…/projects` · `POST projects/…/projects` | project row (checked against Render project ACL) |
| 2 Upload | 1 | Browser → API | `POST …/uploads` | upload + upload\_file rows; one presigned POST per file (max 500 MB) |
|  | 2 | Browser → S3 | presigned POST | raw file stored |
|  | 3 | Browser → API | `POST …/uploads/{id}/complete` (job code list) | files checked; one `extract/verify` stage per file (runtime lrw) |
| 3 Reading design | 1 | API → Postgres | stage rows published via SNS | — |
|  | 2 | LRW → S3 | download, verify, segregate, extract (GDAL) | `geojson/<hash>.geojson`; layer summary on source\_file; counters |
|  | 3 | Browser → API (poll) | `GET …/layers` | state + percent until all sources finished |
| 4 Choose layers | 1 | Browser → API | `GET …/layers`, `GET …/layers/preview` | list from `source_file.layers`; thinned GeoJSON for Mapbox |
|  | 2 | Browser → API | `POST …/layers/confirm` | `excluded_layers` saved; `rdl_generation/start` enqueued |
|  | 3 | Worker → Engine | `profile` (design by reference, kept layers) | profile reused if `designs/<hash>.json` exists |
|  | 4 | Worker → Postgres | run row; one `draft_layer` stage per kept layer | — |
|  | 5 | Worker → Engine → Databricks | `draft-layer` × N (parallel) | per-layer JSON in `run_fragment`; `layers_done` += 1 |
| 5 Task flow | 1 | Browser → API | `GET tenants/…/task-flow-templates` · `PUT …/task-flow` | activities saved (`project_task_flow`); join is gated on this and on all layers done |
| 6 Design analysis | 1 | Worker → Engine → Databricks | `draft-join` (layer JSONs + knowledge + activities) | RDL text; engine `parse` → AST; assumptions |
|  | 2 | Worker → Databricks | question plan over the AST | `run.question_plan`; run waits |
|  | 3 | Browser → API (poll) | `GET …/progress`, `GET …/discover` | stage progress, then questions one by one |
|  | 4 | Browser → API | `PUT …/discover answers` | submitted answers in `project_setting.discover` |
|  | 5 | Browser → API | `POST …/questions/continue` | `apply_answers` enqueued |
| 7 Preparing | 1 | Worker → Engine | `serialize` (AST + answers → RDL) | RDL stored |
|  | 2 | Worker → Engine (→ Databricks on fail) | `validate` incl. task-flow conformance; `fix` one attempt per invocation, chained up to the per-run cap | validation result, fix attempts |
|  | 3 | Worker → Engine | `compile` (RDL + GeoJSON ref + overlay + activities) | tasks, dependencies, features, `placement_id`, no-task notices; persisted |
|  | 4 | Worker → Engine → Databricks | `project-knowledge` | Project Knowledge v1 (S3 + index row) |
|  | 5 | Worker → Databricks | unit code selection per (task type, descriptor); learned mappings first | feature\_mapping, mapping\_review; run `ready` |
|  | 6 | Browser → API (poll) | `GET …/runs/{id}` | status and current step |
| 8 Review | 1 | Browser → API | `GET review, map, packages, features` | read models from run\_feature / staged\_task |
|  | 2 | Browser → API | `POST overlay` / asset type | new overlay version; `recompile` stage; Review shows pending until the new run is ready (ADR-07) |
|  | 3 | Browser → API | `POST chat` | message pending; `chat` stage; worker → Databricks; proposal → `edit` stage → Engine `edit` + `diff-explain` |
|  | 4 | Browser → API | approve proposal | run `merged`; validate → compile → mapping chain |
| 9 Configure | 1 | Browser → API | `GET/PUT` job codes, forms, recap | choices in `project_setting`; job code suggestions and form matching are backend LLM calls run as stages |
|  | 2 | Browser → API | form builder save, map template to activity | form\_template\_version, form\_mapping (QM-11, RA-05) |
| 10 Blueprint | 1 | Browser → API | `GET blueprint`, `GET boq` | pre-flight checks, bill of quantities |
|  | 2 | Browser → API | `POST …/blueprint` | run `finalized`, project `blueprinted`; task file to S3; `blueprint` stage |
|  | 3 | Worker → Render | batched writes: task types, milestones, subsectors, tasks with geometry and quantities | `blueprint_push` progress; `staged_task.render_task_id`; tenant library saved |
|  | 4 | Browser → API (poll) | blueprint status | done |
| Field | 1 | Crew app → Form viewer | open link with signed token (task, user, tenant) | template for the task's activity loaded from API |
|  | 2 | Form viewer → Render | submission API; task API for button actions | submission stored in Render; task status changed |

## 6. Refinement loops

Every change creates a new run; the project's `current_run_id` moves. Runs are never edited in place.

| User does | Run trigger | What runs | LLM | User confirms? | Replay key input changed |
| --- | --- | --- | --- | --- | --- |
| Edits feature tags or asset type on the Review map | `recompile` | new overlay version → engine `compile` → mapping for changed tasks | No | No, automatic | overlay version |
| Picks a job code on a needs-review feature | (same run) | feature\_mapping updated; learned\_mapping written | No | No | — |
| Asks the chat for a change | `edit` | chat writes Project Knowledge → engine `edit` (targeted AST edit) → `diff-explain` → proposed run | Yes | Yes, merges the diff | project knowledge version |
| NDM changes Render Domain Knowledge | `edit` | "changes pending" on affected projects → Apply → same path as chat | Yes | Yes | domain knowledge version |
| NDM changes tenant knowledge or promotes Project Knowledge upward (OPEN-12) | edit | new knowledge version → "changes pending" on affected projects → Apply → same path as chat | Yes | Yes | customer or domain knowledge version |
| Edits the task flow after a run exists | `edit` (QM-10) | engine `validate` conformance + `edit` → diff → merge | Yes | Yes | task flow hash |
| Answers a question again | `edit` | apply\_answers over current AST → validate → compile | Only if fix needed | Yes | answers |
| Uploads a new design version | `revision` | extract new files; `draft_layer` only for uncovered layers; join merges with base run's AST; full chain after | Yes | Review as normal | design hash |
| Rerun from Project Knowledge | `rerun` | every layer drafted from scratch | Yes | Review as normal | all |

Guards: `require_editable()` refuses every user-facing write once the project is `blueprinted`; recompiles refuse a `finalized` run. Same layers confirmed again = no new run; different layers = new run with a new design hash; changing layers mid-analysis is refused with `analysis_running`.

## 7. Deployment

All inside the Render AWS account, deployed by the render-services pipeline except the RDL engine (own repo, ADR-09). Nodes for the deployment diagram are the rows below; edges are the "Reaches" column.

| Unit | AWS service | Image / artifact | Limits | Scaling | Reaches | Status |
| --- | --- | --- | --- | --- | --- | --- |
| QuarterMaster MFE | S3 + CloudFront (existing scale-respawned bundle) | SystemJS bundle | — | CDN | API Gateway; S3 upload bucket (presigned); Mapbox | Current |
| API Gateway + Okta authorizer | API Gateway | — | 29 s per request | Managed | API Lambda | Current |
| API | Lambda, Python 3.11 (Mangum) | render-services zip | 29 s effective; no inline engine or LLM calls | Per request | RDS, S3, SNS | Current, inline recompile removed (ADR-07) |
| Stage worker | Lambda, Python 3.11 | render-services zip | 15 min; one LLM call + one deterministic step per invocation (QM-14) | Per SNS message; concurrency cap | RDS, S3, RDL engine (Invoke), Databricks, Render REST | Current |
| Sweeper | Lambda, scheduled (EventBridge) | render-services zip | — | — | RDS, SNS | Current |
| Long-running worker | ECS Fargate | GDAL image + ezdxf + KML parser + LibreDWG (OPEN-08) | No time limit | Task per job | RDS, S3 | Current |
| RDL engine | Lambda container image, Node 22 | rdl-engine image from ECR | 15 min; 10 GB memory/ephemeral; design cache per warm container; invoke response ≤ 6 MB, so larger results go to S3 by reference (OPEN-15) | Per invocation; reserved concurrency to protect short actions | S3 (presigned GET), Databricks | Proposed (QM-13, RA-02) |
| Form viewer | Static bundle (same CDN as MFE) | new | — | CDN | API; Render submission and task APIs | Proposed (QM-11) |
| Postgres | RDS PostgreSQL ≥ 15 | schema `respawned`, `quartermaster_*` | — | — | — | Current |
| S3 | one bucket, `<prefix>/<tenant_id>/<area>/` | eight areas (+ `forms`) | 500 MB per upload; `content-length-range` on presigned POST | — | — | Current |
| SNS topic | SNS | stage publish | — | — | Stage worker | Current |
| Databricks model serving | External | pinned model version, temperature 0 | rate limits per caller | — | — | Current |

**Network and IAM**

- Browser → API only through API Gateway with the Okta authorizer; `ProjectACL` / `TenantACL` on every QuarterMaster router.
- Stage worker → RDL engine by direct Lambda Invoke (IAM `lambda:InvokeFunction`); no HTTP, no API key, no public endpoint. Replaces the `rdl-engine.quartermaster.invalid:7070` placeholder.
- RDL engine → S3 by presigned GET minted by the worker; the engine has no IAM on the bucket (RA-03 to confirm, or grant read-only on the QuarterMaster prefix).
- Databricks credentials: two secrets, one for the engine function, one for render-services. The agents secret is retired.
- Crew app → Form viewer: link with a short-lived signed token carrying task, user and tenant; the viewer calls the API with it.
- `quartermaster-agents.quartermaster.invalid:8100` placeholder removed from `02-variables.tf`.

**Removed:** quartermaster-agents container (Python, LangGraph, port 8100). **Open:** malware scanning before extraction (RA-10); per-tenant feature flag (RA-09).

## 8. External contracts

| Contract | Owner | Called by | Used for | Needed |
| --- | --- | --- | --- | --- |
| Tenant, project, user, ACL | Render | API | Scope every request; `ProjectACL`, `TenantACL` | In place |
| Job code lists and codes (unit codes) | Render | Upload (list choice), unit\_code\_mapping, Configure | Candidate codes per task type; snapshot on the run (`unit_codes.job_codes`); drift check | In place |
| Task types | Render | Task flow templates, join, validate | Vocabulary the RDL must produce; pinned by hash in the vocabulary block | In place |
| Task tables (`respawned.tasks`, milestones, subsectors) | Render | Blueprint stage | Writes the plan; `integration_metadata.quartermaster.task_key` for idempotent retry | In place |
| Task status API | Render | Form viewer (button actions) | Change status, create follow-up task, flag issue | Shape to confirm (RA-06) |
| Form submission API | Render | Form viewer | Store filled form against a task; needs template version on each submission | Shape to confirm (RA-06) |
| Crew app link handoff | Render | Crew app → Form viewer | Signed short-lived token with task, user, tenant | To define (RA-06) |
| Databricks model serving | Render / Databricks | RDL engine, stage worker | All LLM calls; pinned model; temperature 0; every call logged in `llm_call`, replies cached in `llm_cache` | In place; agents credential retired |
| Mapbox | Vendor | Browser | Map rendering | In place |

## 9. Cross-cutting rules

- **Exactly-once stages.** One `stage` row per logical message, unique `idempotency_key` `{project}:{stage}:{step}:{entity}[:{part}]`; enqueue `INSERT … ON CONFLICT DO NOTHING` in the same transaction as the state change; claim by conditional `UPDATE … RETURNING`; complete row, results and next rows in one transaction; publish after commit. Sweeper re-drives stale rows by `updated_at` heartbeat.
- **Completion counting.** Upload: `files_total / files_registered / sources_registered / sources_finished` bumped by single `UPDATE … RETURNING`; the update that makes finished = registered enqueues `rdl_generation/start`. Run: `layers_total / layers_done` the same way; `join_enqueued` guards the single join. Join also waits for `layers_confirmed_at` on the task flow.
- **Determinism.** Every LLM call at temperature 0 against a pinned model, cached by hash of its full input. Parse, serialize, validate, compile, quantities and asset IDs are code. Same replay key (project ID, GeoJSON hashes, overlay version, exceptions version, knowledge version IDs, vocabulary hash, task flow hash, answers, pinned asset IDs, model version; engine version and prompt / slice template version to be added, OPEN-14) → same tasks in the same order.
- **Immutability.** GeoJSON and profiles are content-addressed and never deleted. Runs are history; a new run per change. Knowledge versions are written once under a token key. Finalized runs and blueprinted projects are read-only (`require_editable()`).
- **Tenant isolation.** `tenant_id` on every row; composite `(col, tenant_id)` FKs to parents with `UNIQUE (id, tenant_id)`; S3 keys under `<prefix>/<tenant_id>/`; `BlobStore` bound per tenant; DesignLoader cache keyed by the full tenant-prefixed key; stages run inside the tenant's LLM scope; retention and delete take a per-tenant advisory lock.
- **Failure handling.** A failed file or source is marked failed with its error and counts as finished; the rest of the upload carries on. No file passes → upload failed, no run. Validation failures go through fix, one attempt per invocation, chained up to a per-run cap; unresolved errors are shown with the tasks. Mapping below the confidence threshold → `needs_review`. Engine failure at start is retried; the last retry fails the run and the UI offers retry.
- **Time budget.** API 29 s: no inline engine or LLM calls. Stage worker and engine 15 min: one LLM call plus one deterministic step per invocation. Extraction unbounded on Fargate.
- **Human gates.** Layer choice; task flow confirmation; question answers; review of asset mapping and open questions; proposal merge after a diff; manual job code picks; blueprint create.
- **Observability.** `llm_call` per call with cost and tokens, rolled up on the run; `pipeline_event` per finding; per-stage count, duration and failure metrics; analytics in Databricks fed from Postgres.
