# QuarterMaster — Architecture Decision Log

Nov 6, 2026 · Stalin Pereira

## Scope

This log records the architecture decisions for QuarterMaster taken in the design review of 6 November 2026, against the conversion branch as documented in the End-to-End Flow, Architecture Standard (2026-11-05) and Database Schema (head d3a58b7c1e94). Decisions are grouped by whether they are agreed, need Render's approval, or are still open. Existing ADR-01 to ADR-09 are re-stated at the end with their new status.

Terms used: **activity** = one task type placed at one position in a task flow, with its milestone, crew days and predecessors. **AST** = the RDL program parsed into JSON by the RDL engine. **Join** = the single LLM call that writes the RDL program from the per-layer outputs.

## Decisions made

Agreed in the design review of 6 November 2026. These are the baseline for the clean architecture; current code is "what exists now", not the target.

| ID | Decision | Context | Consequences |
| --- | --- | --- | --- |
| QM-01 | Layer selection happens before any AI call. Only kept layers are profiled and drafted. | Code already does this (`upload.excluded_layers`, profile at `rdl_generation/start`). The per-file profile at extraction has no readers. | Confirms ADR-02 (drop per-file profile) and ADR-03 (profile once at Start). Saves LLM cost on unused layers. |
| QM-02 | Drafting is per-layer JSON then one join call that writes RDL text. No whole-program draft. | Today the initial run drafts the whole program in the agents service; only revisions use `draft-layer` + `draft-join`. The output-token budget, not input, is the binding limit. | Initial run and revision use the same path. Per-layer output is a small JSON: layer name, attribute keys, field mappings, render asset type guess, confidence. Join takes all layer JSONs + knowledge + task flow and writes the RDL program and dependencies. |
| QM-03 | Asset-type mapping is folded into the per-layer call. No separate `map_assets` step. | `map_assets` is a second LLM pass over the same profile. | One call per layer instead of two. Confidence per layer comes from the same call (<75% = needs review). |
| QM-04 | The RDL engine parses RDL text into a JSON AST and serializes the AST back to RDL text, in code. The LLM only ever reads and writes RDL text. | JSON is 2–4x the tokens of the DSL; the DSL exists to keep model output compact. Everything else in the system wants a structured, editable form. | Question answers, overlay edits and task-flow conformance operate on the AST. Project Knowledge and the diff view are generated from the AST. Round trip must be lossless; rule IDs must be stable. See OPEN-01. |
| QM-05 | Question planner runs after the join, over the AST. Answers are applied as edits to the AST before compile. | Questions depend on the identified render asset types ("how should asset X be calculated"), so they cannot be asked before the asset guess exists. | New order: extract → choose layers → per-layer draft → confirm task flow → join → questions → apply answers → validate/fix → compile. |
| QM-06 | Only user-submitted answers are stored on the project. Defaults are never copied. Each run pins the version of the questions knowledge file it resolved against. | Copying defaults per project would freeze them; a knowledge update would not reach reruns. | `run.question_plan` + `knowledge_version_ids` reconstruct "answered X, defaulted Y from version N" at display time. Past runs stay reproducible; new runs pick up new defaults. |
| QM-07 | The task flow is a contract, not a hint. The RDL must conform: every task type in the RDL exists in the flow and every RDL dependency respects the flow's order. | Today the flow is read at draft time and `staged_task.placement_id` is recorded but unused. | New engine validation rule (no AI). Task types outside the flow are errors through fix. Feature types with no activity become a "we don't know what to do with this" notice. `placement_id` becomes a real compile output. |
| QM-08 | The unit of the task flow is an **activity**: ID, task type, milestone, crew days, predecessor activities. The same task type may appear on several activities. | A task type can sit in two milestones (inspect pole in pre-construction and close-out). Task type alone is not unique. | Task flow template = ordered list of activities. Each compiled task belongs to exactly one activity. When a task type is on two activities the join needs a discriminating condition; if it cannot find one, it is surfaced as an open question. |
| QM-09 | Milestone and crew days live on the activity, not in the RDL. Compile reads them from the activity a task lands on. | Removes the `unknown_milestone` fix loop. Crew days are a property of a kind of work, not of a feature. | RDL drops milestone literals. Flat crew days per activity to start; a rate (days per 1,000 ft) multiplied by quantity is the extension. Configure › Dependencies mostly collapses into the task flow page. |
| QM-10 | Editing the task flow after the RDL exists goes through the targeted-edit → diff → merge path, like a knowledge change. The task flow is part of the replay key. | Rules themselves must change, so a recompile is not enough. | Task flow hash is already in `compile_summary.task_flow`; promote it to a first-class replay key input. |
| QM-11 | Forms: Quartermaster owns the template, the viewer and the conditional/button logic. Render's crew app opens a link; the viewer posts to Render's submission API and, for buttons, to Render's task API. Whether templates attach to an activity or a task type is decided by RA-05. | Render has a form builder and viewer but they are limited and slow to build in, and support for extending their schema is uncertain. | Closed vocabulary: field types; condition operators (equals, not empty, greater than, ...); button actions (change task status, create follow-up task, attach photo, flag issue). Templates versioned per tenant; project-level forks are new versions. Needs tables of their own instead of the `forms` key in `project_setting`. |
| QM-12 | Retire the quartermaster-agents service. RDL-touching AI calls move to the RDL engine; non-RDL AI calls become plain LLM calls in the Python backend. | Three services hold Databricks credentials and prompt logic; the agents service's validator can drift from the engine's. With QM-02 its initial draft path is no longer needed. | Engine: per-layer draft, join, fix, edit, diff-explain, project knowledge, chat apply-change. Backend: question planner, unit code selection, job code suggestions, form matching, conversational chat. Backend stays the orchestrator (stage rows, calls the engine). One fewer container, credential set and validator. |
| QM-13 | The RDL engine runs as a Lambda container image, invoked directly by the backend. | Backend is already Lambda; the engine is stateless and each action fits one invocation. Closes the hosting placeholder. | No HTTP host, no API key, no network path to manage. Engine function gets the 15-minute timeout. Backend must call it from a stage job, never inline from the API, so ADR-07 becomes mandatory. The design cache (ADR-04) is per warm container only. |
| QM-14 | No single invocation waits on more than one LLM call plus one deterministic step. Each fix attempt is its own invocation; the backend chains attempts up to a per-run cap. | Five fix attempts at 2–3 minutes each would exceed 15 minutes. | Attempt count and cap recorded on the run. Extraction stays on Fargate because it is the only step that can genuinely exceed 15 minutes. |
| QM-15 | Extraction stays on Fargate (long-running worker); every other stage on Lambda. | Large DWG or multi-shapefile ZIPs have no bounded runtime. | Reaffirms ADR-01 and the current split. |
| QM-16 | Knowledge is sliced by consumer, not sent whole. Each LLM call gets only the knowledge kinds it uses. | test-2 spend (Oct 26–30): each RDL call carried \~130k input tokens, including all of knowledge/global. 85% of it (forms, questions) is never used by RDL drafting; rdl\_run was 79% of spend. | Per-layer: design\_extraction + element\_task\_matching (\~6k tokens). Join, fix, edit: those + RDL instructions + task types (\~8–10k). Question planner: questions. Form matching: forms. Per-layer input drops from \~130k to under 10k tokens. How to slice is OPEN-13. |
| QM-17 | The knowledge slice is a byte-identical prompt prefix, pinned per run, so prompt caching applies. The first per-layer call is sent alone to warm the cache; the rest fan out after it. | Cache reads cost 1.429 DBU per 1M tokens against 28.571 for input (20× cheaper). 90–99% of input was fixed text and 53% of prompts were exact repeats. Parallel calls started together all miss the cache. | Layer profile and other variable input go after the prefix. Spend report estimate: $3.80 → $1.50 per run from caching alone. Retrieval of candidate asset types only if the vocabulary is still too large after QM-16 and QM-17. |
| QM-18 | Runs are immutable. Every change creates a new run and moves project.current\_run\_id. Review edits are an append-only change set merged with the baseline GeoJSON at compile. | Keeps every run reproducible from its replay key; the baseline GeoJSON is never edited. | overlay\_version, tag\_overlay and feature\_exception are append-only. Finalised runs and blueprinted projects are read-only (require\_editable()). Project Knowledge is written in sync with each run's RDL. |

## Decisions needing Render approval

Each of these changes something Render owns or operates. Owner on Render's side is to be confirmed (Andy for hosting).

| ID | Decision proposed | Why Render must approve | What we need from them | Status |
| --- | --- | --- | --- | --- |
| RA-01 | Retire quartermaster-agents; engine and backend are the only two LLM callers (QM-12). | Removes a service from their deployment surface; changes which services hold Databricks credentials. | Agreement, and removal of the agents placeholder from `02-variables.tf`. | Pending |
| RA-02 | RDL engine runs as a Lambda container image in the Render account, invoked directly (QM-13). | Render owns the pipelines and on-call. Replaces the `rdl-engine.quartermaster.invalid:7070` placeholder. | Decision on Lambda vs ECS; who owns the engine's pipeline; whether the engine lives in its own repo (ADR-09). | Pending |
| RA-03 | Engine downloads designs from S3 through presigned URLs (ADR-04) rather than an IAM role. | Security posture on their account. | Confirm presigned URLs are acceptable, or grant read-only IAM on the QuarterMaster prefix. | Pending |
| RA-04 | Task flow becomes a contract the RDL must conform to; the unit is an activity (QM-07, QM-08). | Changes the task-flow data they seed as Render defaults, and the vocabulary that appears in their UI. | Confirm the word "activity" (or their own term); confirm milestone and crew days belong on the activity (QM-09). | Pending |
| RA-05 | Forms attach to an activity rather than a task type. | The pre-construction and close-out inspection of the same task type may need different forms. | Confirm with the NDM team whether forms differ by position in the flow. | Pending |
| RA-06 | Form rendering, display and logic stay in Quartermaster; the crew app opens a Quartermaster link (QM-11). | Their crew app must show a link and pass task, user and tenant context (signed short-lived token). | Submission API shape (flat vs structured, photos/signatures, template version on each submission); task API for button actions; context handoff mechanism. | Pending |
| RA-07 | Review recompiles move into a worker job (ADR-07), now mandatory. | Review page gains a pending state per edit. | UX sign-off on the polling pending state. | Pending |
| RA-08 | Knowledge source of truth: S3 versioned objects + Postgres index (as built), not a Databricks volume. | The design conversation assumed Databricks volumes; the code uses S3. Prompts must read one place. | Confirm S3 is the store; Databricks reads from it if needed for serving. | Pending |
| RA-09 | Per-tenant feature flag for QuarterMaster. | Gating mechanism is Render's. | Agree the flag mechanism. | Pending |
| RA-10 | Malware scanning of uploads before extraction. | Security requirement is theirs to set. | Decide whether required, which scanner, and where it sits. | Pending |

## Open items pending investigation

On our side, before the decisions above can be built.

| ID | Item | Blocks | Owner |
| --- | --- | --- | --- |
| OPEN-01 | Does the RDL engine export the program as JSON and import JSON back to RDL? Is the round trip lossless (same program, not just a valid one)? Does the parser expose stable rule IDs? | QM-04, QM-05, QM-07, QM-10 | Stalin |
| OPEN-02 | Exact schema of the per-layer JSON (layer, attribute keys, asset type guess, field mappings, confidence) and how the join prompt consumes it. | QM-02, QM-03 | Stalin |
| OPEN-03 | Form spec v1: field types, condition grammar, button action list, versioning model, submission payload. | QM-11, RA-06 | Stalin |
| OPEN-04 | Which of the 112 questions edit which AST fields; whether any question cannot be expressed as an AST edit. | QM-05 | Stalin |
| OPEN-05 | Crew days: flat per activity for v1, or rate per quantity from the start. | QM-09 | Stalin / Render NDM |
| OPEN-06 | What to keep from the agents service before retiring it (validator, retry loop, chat graph). | QM-12 | Stalin |
| OPEN-07 | Fragment library (`rdl/fragment-library`) and `extraction_key` reuse: keep both; confirm the fragment key still matches after per-layer output changes to JSON. | QM-02 | Stalin |
| OPEN-08 | `dwg2dxf` / LibreDWG missing from the Fargate image. | Extraction of DWG | Stalin |
| OPEN-09 | PostGIS vs GeoJSON in S3 for spatial queries. Earlier decision (D-3) was no PostGIS; still listed open in the standard. | Review map on big designs | Stalin |
| OPEN-10 | Review map sends the whole design; vector tiles via the existing tile-server later. | Review UX | Stalin |
| OPEN-11 | UI does not send `expected_versions` on Configure saves (last write wins per key). | Concurrency on Configure | Frontend |
| OPEN-12 | Knowledge promotion flow: NDMs periodically review Project Knowledge and promote items to tenant (customer) knowledge or global Render domain knowledge. Needs a review queue, a diff against existing knowledge, promote/reject actions, a new knowledge version, and "changes pending" on affected projects. | QM-16, knowledge hierarchy | Stalin |
| OPEN-13 | Reorganise the knowledge folder by consumer now, or slice at prompt-assembly time? The first is simpler for versioning and the replay key; the second puts the slice definition in the replay key. | QM-16, QM-17 | Stalin (ask Jude) |
| OPEN-14 | Replay key omits the engine version and the prompt / knowledge-slice template version. Either can change output with the same model and inputs. | Determinism, QM-10 | Stalin |
| OPEN-15 | A direct Lambda invoke returns at most 6 MB. Compile, AST and diff results on large designs should go to S3 and come back by reference, like design inputs (ADR-04). | QM-13 | Stalin |
| OPEN-16 | Form viewer hosting: served by render-services or from the MFE CDN. | QM-11, RA-06 | Stalin |

## Status of existing ADRs

From the Architecture Standard of 2026-11-05, with the effect of this review.

| ADR | Title | Previous status | Status after review |
| --- | --- | --- | --- |
| ADR-01 | Extraction stays in render-services | Accepted | Unchanged (QM-15) |
| ADR-02 | Drop the per-file profile at extraction | Proposed | Accepted (QM-01) |
| ADR-03 | Profile once at Start, on kept layers, reused by design hash | Accepted · reuse Proposed | Accepted including reuse (QM-01) |
| ADR-04 | Engine receives S3 references, not inline GeoJSON | Proposed | Accepted; cache is per warm Lambda only (QM-13). Presigned-URL access needs RA-03 |
| ADR-05 | Profiling stays in the TypeScript engine | Accepted | Unchanged |
| ADR-06 | Save a layer summary at extraction for the Layers page | Proposed | Accepted |
| ADR-07 | Move review recompiles into a worker job | Proposed | Mandatory (QM-13); UX sign-off RA-07 |
| ADR-08 | Engine LLM providers trimmed to Databricks | Proposed | Accepted |
| ADR-09 | RDL engine lives in its own repository | Open | Still open; bundled with RA-02 |
| D-1 | One queue, one worker | Adopted | Superseded by stage rows + sweeper (as built) |
| D-3 | No PostGIS | Decided | Reopened as OPEN-09 |
| D-5 | No email | Decided | Unchanged; status in DB, UI polls |
| D-6 | TypeScript engine, Python everything else | Decided | Unchanged; agents service removed (QM-12) |
| D-11 | Lambda, not Fargate | Decided | Confirmed for engine (QM-13); extraction stays Fargate (QM-15) |
| D-14 | Draft in parts, not one call | Decided | Confirmed and now applied to the initial run (QM-02) |
