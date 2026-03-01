# TNirvana Agent OS - Implementation Phases

## Phase 0 - Repo and Contracts
Goal:
- Stand up codebase structure and shared contracts.

Deliverables:
- Service folders: `api/`, `orchestrator/`, `llm_gateway/`, `tools_runtime/`, `ui/`.
- Shared `schemas/` package for Pydantic + JSON Schemas.
- Base CI: lint, typecheck, unit tests.

Tasks:
- Define core models: `ScoutProfile`, `Run`, `Artifact`, `ReviewTask`, `ChangeSummary`.
- Define workflow YAML parser and validation.
- Add config management and env loading strategy.

Exit criteria:
- CI passes on empty skeleton with schema validation tests.

## Phase 1 - Persistence and Core APIs
Goal:
- Implement DynamoDB/S3 persistence and baseline API contracts.

Deliverables:
- DynamoDB repositories for all MVP tables.
- S3 artifact writer + manifest writer.
- FastAPI endpoints for tracks/projects/runs/artifacts/review tasks.

Tasks:
- Implement idempotency key support on mutating endpoints.
- Implement run metadata storage and query APIs.
- Implement review task creation/update including `additional_context`.

Exit criteria:
- Can create project, create run record, and store/list artifacts.

## Phase 2 - Tools Runtime and Retrieval
Goal:
- Implement tool execution layer and Sonar Pro retrieval path.

Deliverables:
- Tools: `perplexity_search`, `fetch_url`, `html_to_text`, `extract_citation_snippets`, `pdf_read`.
- `perplexity_search` configured for `sonar-pro` in Scout profile.
- Tool-level logging, retries, and rate limits.

Tasks:
- Add unified tool invocation contract.
- Add provenance metadata on tool outputs.
- Add integration tests for tool timeout + retry behavior.

Exit criteria:
- Scout retrieval pipeline runs and writes retrieval artifacts to S3.

## Phase 3 - Scout Orchestration + Timeline
Goal:
- Deliver complete Scout workflow with trend comparison.

Deliverables:
- `intelligence_scout.yaml` execution end-to-end.
- Multi-angle question generation + gap query loop.
- Prior-run comparison and `TimelineIndex` updates.

Tasks:
- Implement reducers: dedupe/rank insights, compute change summary.
- Implement timeline summarizer.
- Add run checkpointing and resume logic.

Exit criteria:
- One Scout run generates report + change summary + timeline entries.

## Phase 4 - Render Preview + HITL Extensions
Goal:
- Add pre-final rendering and HITL add-context loop.

Deliverables:
- Tools: `render_report_preview`, `render_carousel_images`.
- API: `GET /v1/runs/{run_id}/preview`.
- HITL actions: `approve`, `reject`, `edit`, `add_context`.

Tasks:
- Render HTML/PDF preview for Scout/Research reports.
- Render PNG slide previews for Instagram carousel.
- Inject `additional_context` into resumed run prompt context.
- Add audit records for all review decisions.

Exit criteria:
- Reviewer can inspect preview, add context, resume, and see updated preview.

## Phase 5 - Deep Research + Publishing Workflows
Goal:
- Deliver remaining workflow recipes and final UX polish.

Deliverables:
- `deep_research.yaml`, `longform_article.yaml`, `instagram_carousel.yaml`.
- Tauri review queue + preview viewer UX polish.
- Export/publish integration via `publish_artifacts`.

Tasks:
- Implement deep research source matrix.
- Implement longform article flow for Forbes/Medium output modes.
- Implement Instagram carousel generation and preview approval.

Exit criteria:
- All three tracks can run to HITL preview and publish successfully.

## Cross-Phase Quality Gates
- Unit tests for reducers, schema validation, run state transitions.
- Integration tests for API + orchestrator + tools runtime.
- Cost/latency dashboards available in CloudWatch.
- Security checks: secret handling, redaction, IAM least privilege.

## MVP Milestone Definition
MVP is complete when:
- Scout run from natural-language question executes end-to-end with Sonar Pro retrieval, timeline comparison, and preview before publish.
- HITL supports `add_context` and successful resume.
- Deep Research and Instagram workflows generate previewable outputs and publish artifacts.
