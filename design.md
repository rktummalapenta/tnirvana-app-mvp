# TNirvana Agent OS — Intelligence Scout Platform

## 1. Executive Summary
TNirvana Agent OS is a service-based platform for recurring intelligence collection, deep research, and content production with human review gates. It standardizes how topics are tracked over time, how sources are cited, and how final artifacts are approved and published.

The core value is one shared execution engine (workflow orchestrator + LLM gateway + tools runtime) with multiple workflow recipes. This keeps behavior consistent (retrieval quality, citations, auditing, HITL) while allowing different outputs (briefs, reports, articles, social content).

### MVP goals
- Deliver repeatable `intelligence_scout` runs with change-summary analysis against prior runs.
- Deliver `deep_research` runs with iterative retrieval/synthesis and citations.
- Deliver publishing workflows for Forbes/Medium long-form and Instagram carousel outputs.
- Enforce HITL approval gates before publish/export.
- Persist all run metadata in DynamoDB and artifacts in S3.
- Support timeline memory: “what changed over time” per Scout project.

### Non-goals (MVP)
- Full autonomous publishing to external platforms without human approval.
- Real-time stream ingestion infrastructure (RSS hosting explicitly out of scope).
- Fine-grained multi-tenant billing and chargeback.
- On-device model inference.

## 2. Product Workflows
The product is track-first:
1. User enters a top-level track tab (`Intelligence Scout`, `Deep Research`, `Instagram`, future `News`, etc.).
2. User creates/selects a `Project` inside that track.
3. User runs workflows inside that track project.

Workflows are separate recipes because output contracts, QA checks, and HITL gates differ. They share one engine so orchestration, tooling, model routing, retries, observability, and storage stay consistent.
All workflows include a `render_preview` stage before final approval/publish so users can review report/carousel outputs visually.

### Track model and workflow recipes
- Track tabs are extensible and registered in a track catalog (`track_type`, `label`, `enabled`, `order`, `capabilities`).
- Current track tabs:
  - `intelligence_scout`
  - `deep_research`
  - `instagram`
  - `news` (future extension, same engine)

### Workflows
- `intelligence_scout`
  - Lives under the `Intelligence Scout` tab.
  - Input is a natural-language user question. System expands it into multiple angle/gap/trend sub-questions, retrieves with Perplexity Sonar Pro, then synthesizes agentic insights.
  - Compares with previous Scout runs (if any) to produce timeline change analysis.
  - Primary output: executive brief + structured change summary + timeline summary + rendered preview.
- `deep_research`
  - Lives under the `Deep Research` tab.
  - Multi-iteration research loop (retrieve -> analyze gaps -> retrieve more -> synthesize) with citation-backed report.
  - Primary output: long-form research report + source matrix + rendered preview.
- `longform_article`
  - Triggered from the `Deep Research` tab when user chooses article mode.
  - Workflow: outline -> draft -> style/voice pass -> compliance checks -> HITL.
  - Supports `target_publication: forbes | medium`.
- `instagram_carousel`
  - Lives under the `Instagram` tab.
  - Social content workflow focused on “Men inner peace and struggles”: angle selection -> carousel slide script -> caption -> hashtag set -> render carousel images -> HITL.

### Why separate recipes, shared engine
- Separate recipes encode domain-specific prompts, artifact schemas, and review policies.
- Shared engine guarantees common controls: retries, checkpoints, model/tool governance, provenance, and security.

## 3. System Architecture Overview
```text
+--------------------+      +---------------------+
| Tauri/Web UI Shell | <--> | App Server (FastAPI)|
+--------------------+      +----------+----------+
                                      |
                                      v
                           +----------+-----------+
                           | Workflow Orchestrator|
                           +----+-------------+---+
                                |             |
                                v             v
                     +----------+---+   +-----+----------------+
                     | LLM Gateway |   | Skills/Tools Runtime |
                     +------+------+
                            |
                            v
                    +-------+-------------------------------+
                    | OpenAI / Anthropic + Perplexity APIs |
                    +---------------------------------------+

          +-------------------+         +-----------------------+
          | DynamoDB (state)  | <-----> | S3 (artifacts/files) |
          +-------------------+         +-----------------------+
```

### Data flow summary
1. UI selects a track tab, creates/selects a project under that track, then starts run via App Server.
2. Orchestrator executes YAML graph and checkpoints run state.
3. Steps call LLM Gateway and Tools Runtime.
4. Intermediate/final artifacts persist to S3; run/task metadata persists to DynamoDB.
5. HITL tasks pause runs; approval resumes from checkpoint.

### Local-first vs cloud-first coexistence
- Local-first (Tauri)
  - UI and lightweight cache run locally.
  - Credentials stored in OS Keychain; API access to cloud App Server.
  - Optional local queue for offline draft edits and deferred sync.
- Cloud-first (Web)
  - Same APIs and backend services.
  - No local install requirement.
- Coexistence
  - Both clients are thin shells over shared backend contracts.
  - Run IDs, artifacts, and review tasks are source-of-truth in AWS.

## 4. Services (Service-Based Approach)

### 4.1 UI Shell
Options:
- Tauri macOS desktop (React)
- Web app (React/Next.js)

Navigation model:
- Track tabs (create/select project inside each):
  - Intelligence Scout
  - Deep Research
  - Instagram
  - News (future)
- Project list scoped to selected track
- Runs
- Preview (pre-final render)
- Review Queue (HITL)
- Artifacts Viewer
- Settings

Responsibilities:
- Manage user sessions and track-first navigation.
- Start runs, inspect step progress, perform review actions (approve/reject/edit).
- Show rendered previews (report HTML/PDF, carousel image set) before final approval.
- Render artifacts (markdown/html/json).

### 4.2 App Server (API)
Responsibilities:
- AuthN/AuthZ boundary.
- Track catalog APIs (dynamic top-level tabs).
- CRUD for projects under each track.
- Scout-specific question profile APIs (natural-language question + generated question set).
- Run creation/query/cancel/resume.
- Preview retrieval APIs for staged render output.
- Artifact metadata lookup and signed URL issuance.
- ReviewTask queue and decisions, including additional-context injection from reviewer.

Minimal REST endpoints:
- `GET /v1/tracks`
- `POST /v1/tracks/{track_type}/projects`
- `GET /v1/tracks/{track_type}/projects`
- `GET /v1/projects/{project_id}`
- `POST /v1/projects/{project_id}/scout-profile`
- `GET /v1/projects/{project_id}/scout-profile`
- `POST /v1/projects/{project_id}/runs`
- `GET /v1/projects/{project_id}/runs`
- `GET /v1/runs/{run_id}`
- `GET /v1/runs/{run_id}/preview`
- `POST /v1/runs/{run_id}/resume`
- `POST /v1/runs/{run_id}/cancel`
- `GET /v1/runs/{run_id}/artifacts`
- `GET /v1/review-tasks?status=pending`
- `POST /v1/review-tasks/{task_id}/decision`

### 4.3 Workflow Orchestrator
Responsibilities:
- Load workflow YAML definition.
- Build execution graph, apply budgets/policies.
- Persist per-step checkpoints and retries.
- Pause/resume around HITL steps.

Execution guarantees:
- At-least-once step execution with idempotency tokens.
- Deterministic replay from checkpoints.

### 4.4 LLM Gateway
Responsibilities:
- Provider-agnostic chat/completion interface.
- Model routing across OpenAI/Claude by policy.
- Tool-calling normalization and stream multiplexing.
- Retry/fallback with circuit-breaker rules.

### 4.5 Skills Runtime (Tools)
Responsibilities:
- Execute approved tools with validation, permissions, and rate limits.
- Standardize tool outputs (JSON + provenance metadata).
- Maintain tool-level audit logs.

Initial tools:
- `perplexity_search`
- `fetch_url`
- `extract_citation_snippets`
- `html_to_text`
- `pdf_read`
- `publish_artifacts`
- `export_markdown_to_html`
- `render_report_preview`
- `render_carousel_images`

### 4.6 Storage Layer
- DynamoDB for metadata, run state, indexing, review queue.
- S3 for artifact blobs and manifests.

### Error handling and idempotency
- Client-request idempotency: `Idempotency-Key` header on mutating APIs.
- Orchestrator step idempotency key: `{run_id}:{step_id}:{attempt_group}`.
- Retries: exponential backoff + jitter; bounded max attempts per step.
- Dead-letter state: run transitions to `FAILED_RETRY_EXHAUSTED` with diagnostic artifact.
- Partial failures: failed step can be edited/retried via HITL intervention.

## 5. Data Model (DynamoDB + S3)

### 5.0 Plain-language hierarchy
- `Track/Tab`: top-level workspace (`intelligence_scout`, `deep_research`, `instagram`, future `news`).
- `Project`: created inside a selected track tab.
- `Scout Profile`: Scout-only project config containing the user’s natural-language question and generated sub-questions.
- `Run`: one execution of a workflow inside a project tab.
- `Artifact`: files produced by a run.
- `ReviewTask`: HITL approval item for a run step.
- `Timeline entry`: chronological “what changed” snapshot for a Scout project.

### 5.1 DynamoDB tables

#### TrackCatalog
- PK: `track_type`
- Attributes: `label`, `enabled`, `display_order`, `capabilities`, `created_at`, `updated_at`

#### Projects
- PK: `track_type`
- SK: `project_id`
- Attributes: `owner_id`, `name`, `description`, `status`, `created_at`, `updated_at`
- GSI1: `project_id` (partition), `track_type` (sort) for direct project fetch
- GSI2: `owner_id` (partition), `created_at` (sort) for listing all user projects

#### Runs
- PK: `project_id`
- SK: `started_at#run_id`
- Attributes: `track_type`, `workflow_id`, `status`, `state`, `started_at`, `ended_at`, `current_step`, `budget_usage`, `input_hash`
- GSI1: `run_id` (partition), `started_at` (sort) for direct run lookup
- GSI2: `track_type` (partition), `started_at` (sort) for track-level operations
- GSI3: `status` (partition), `started_at` (sort) for operations dashboard

#### ScoutProfiles
- PK: `project_id`
- SK: `profile_id` (use `primary` for MVP)
- Attributes: `seed_question`, `question_lenses[]`, `cadence`, `lookback_runs`, `created_at`, `updated_at`

#### Artifacts
- PK: `run_id`
- SK: `artifact_id`
- Attributes: `project_id`, `track_type`, `artifact_type`, `stage`, `render_variant`, `s3_uri`, `sha256`, `content_type`, `created_at`, `manifest_ref`
- GSI1: `project_id` (partition), `created_at` (sort)

#### ReviewTasks
- PK: `status_bucket`
- SK: `created_at#task_id`
- Attributes: `task_id`, `project_id`, `run_id`, `step_id`, `assignee`, `decision`, `comments`, `additional_context`, `preview_artifact_refs`, `resolved_at`, `resume_token`
- GSI1: `assignee` (partition), `created_at` (sort)
- GSI2: `run_id` (partition), `created_at` (sort)

#### TimelineIndex
- PK: `project_id`
- SK: `summary_at` (ISO timestamp)
- Attributes: `run_id`, `summary`, `key_changes`, `change_score`, `top_sources`, `question_lenses_used`
- GSI1: `run_id` (partition), `summary_at` (sort)

### 5.2 Query patterns
- List projects under selected tab: `Projects(track_type, project_id*)`.
- List project runs ordered by time: `Runs(project_id, started_at#run_id*)`.
- Fetch artifacts for run: `Artifacts(run_id, artifact_id*)`.
- Fetch preview artifacts for run: `Artifacts(run_id, artifact_id*)` filtered by `stage='preview'`.
- Fetch pending review tasks: `ReviewTasks(PENDING#yyyy-mm-dd, created_at#task_id*)`.
- Build trendline: `TimelineIndex(project_id, summary_at between t1..t2)`.
- Fetch Scout profile for project: `ScoutProfiles(project_id, primary)`.

### 5.3 Hot partition avoidance
- Project writes are distributed by `track_type + project_id`; use UUID/ULID for `project_id`.
- Review queue can hot-spot; use `status_bucket` (example: `PENDING#2026-03-01`) and rotate by day.
- For very active Scout projects, shard TimelineIndex PK as `project_id#shard_n` if needed.
- Batch artifact metadata writes and use adaptive retry with jitter.

### 5.4 S3 structure
`s3://<bucket>/tracks/<track_type>/projects/<project_id>/runs/<run_id>/<stage>/<artifact_name>`

Recommended layout:
```text
s3://tnirvana-artifacts/
  tracks/
    intelligence_scout/
      projects/proj_123/
        runs/run_2026_03_01_001/
          input/seed_question.json
          planning/generated_questions.json
          retrieval/perplexity_sonar_results.json
          enrichment/citations.json
          analysis/insights.json
          analysis/change_summary.json
          output/scout_report.md
          output/scout_report.html
          preview/scout_report_preview.html
          preview/scout_report_preview.pdf
          manifest/artifact_manifest.json
```

### 5.5 Artifact model (plain and practical)
An artifact has three layers:
- `Artifact file` (S3 object): actual content generated during run (JSON/MD/HTML/CSV).
- `Artifact metadata row` (DynamoDB Artifacts): lightweight index to locate/filter artifacts.
- `Artifact manifest` (single JSON per run): canonical inventory of every artifact and checksum.

Why this split:
- UI listing/search is fast from DynamoDB metadata.
- Large content stays cheap in S3.
- Manifest enables replay/debug/audit and deterministic export.

MVP Scout artifact set:
- `input/seed_question.json`: user’s original natural-language question.
- `planning/generated_questions.json`: expanded angle/gap/trend sub-questions.
- `retrieval/perplexity_sonar_results.json`: raw Perplexity Sonar Pro outputs.
- `analysis/insights.json`: extracted insights with source references.
- `analysis/change_summary.json`: new/changed/faded insights vs prior runs.
- `output/scout_report.md`: final agentic Scout report (HITL-reviewed).
- `preview/scout_report_preview.html`: rendered preview before final approval.

### 5.6 Core entities (Pydantic-friendly)
Entity intent:
- `ScoutProfile`: what the Scout project should monitor and how to query it.
- `SourceItem`: one cited source record fetched from search/web/PDF.
- `InsightItem`: one distilled insight backed by one or more sources.
- `ChangeSummary`: what is new/changed/faded versus prior run.
- `TimelineSummary`: chronological trendline snapshot used in “what changed over time”.

```python
from pydantic import BaseModel
from typing import List, Dict, Optional

class ScoutProfile(BaseModel):
    project_id: str
    seed_question: str
    question_lenses: List[str] = []
    cadence: str  # daily|weekly|monthly
    lookback_runs: int = 5

class SourceItem(BaseModel):
    source_id: str
    title: str
    url: str
    source: str
    published_at: Optional[str] = None
    snippet: Optional[str] = None
    citation_snippet: Optional[str] = None
    relevance_score: float = 0.0
    credibility_score: float = 0.0

class InsightItem(BaseModel):
    insight_id: str
    statement: str
    category: str
    source_ids: List[str]
    confidence: float

class ChangeSummary(BaseModel):
    project_id: str
    run_id: str
    previous_run_id: Optional[str] = None
    new_insights: List[str]
    changed_insights: List[str]
    faded_insights: List[str]
    summary: str

class TimelineSummary(BaseModel):
    project_id: str
    run_id: str
    summary_at: str
    trendline: str
    key_changes: List[str]
    change_score: float

class ArtifactManifest(BaseModel):
    project_id: str
    track_type: str
    run_id: str
    workflow_id: str
    artifacts: List[Dict[str, str]]
    created_at: str
```

## 6. Workflow YAML Design

### 6.1 Workflow YAML schema concept
```yaml
metadata:
  workflow_id: string
  version: string
  description: string
inputs:
  track_type: string
  project_id: string
  user_question: string|null
  params: object
budgets:
  max_tokens_total: int
  max_tool_calls: int
  max_runtime_seconds: int
policy:
  model_routing:
    <step_id>: { provider: openai|anthropic, model: string, fallback: [..] }
  tools_allowed: [tool_name]
  retry:
    max_attempts: int
    backoff_ms: int
memory:
  load_prior_runs: true|false
  lookback_runs: int
  build_timeline: true|false
artifacts:
  required: [artifact_type]
  preview_required: true|false
  preview_formats: [html|pdf|png]
  publish_targets: [s3]
steps:
  - id: string
    type: llm|tool|reducer|loop|hitl
    input_from: [step_id]
    config: object
hitl:
  required_steps: [step_id]
  allow_additional_context: true|false
  timeout_hours: int
  on_reject: revise|abort
```

### 6.2 Full YAML example: `intelligence_scout.yaml`
```yaml
metadata:
  workflow_id: intelligence_scout
  version: 1.0.0
  description: Recurring topic intelligence from natural-language question with timeline comparison.

inputs:
  track_type: intelligence_scout
  project_id: "${project_id}"
  user_question: "${user_question}"
  params:
    top_k_sources_per_question: 8
    max_generated_questions: 8

budgets:
  max_tokens_total: 180000
  max_tool_calls: 80
  max_runtime_seconds: 1800

policy:
  model_routing:
    generate_questions:
      provider: openai
      model: gpt-4.1
      fallback:
        - provider: anthropic
          model: claude-3-7-sonnet
    gap_analysis:
      provider: anthropic
      model: claude-3-7-sonnet
      fallback:
        - provider: openai
          model: gpt-4.1
    extract_insights:
      provider: openai
      model: gpt-4.1
      fallback:
        - provider: anthropic
          model: claude-3-7-sonnet
    synthesize_scout_report:
      provider: anthropic
      model: claude-3-7-sonnet
      fallback:
        - provider: openai
          model: gpt-4.1
    qa_check:
      provider: openai
      model: gpt-4.1-mini
      fallback: []
  tools_allowed:
    - perplexity_search
    - fetch_url
    - extract_citation_snippets
    - html_to_text
    - pdf_read
    - render_report_preview
    - export_markdown_to_html
    - publish_artifacts
  retry:
    max_attempts: 3
    backoff_ms: 800

memory:
  load_prior_runs: true
  lookback_runs: 5
  build_timeline: true

artifacts:
  required:
    - generated_questions
    - sources_raw
    - sources_enriched
    - insights
    - change_summary
    - scout_report_md
    - scout_report_html
    - timeline_summary
    - scout_report_preview_html
    - scout_report_preview_pdf
  preview_required: true
  preview_formats:
    - html
    - pdf
  publish_targets:
    - s3

steps:
  - id: load_context
    type: reducer
    input_from: []
    config:
      operation: load_scout_profile_and_prior_runs

  - id: generate_questions
    type: llm
    input_from: [load_context]
    config:
      prompt_template: prompts/scout_generate_questions.md
      output_schema: string[]
      rules:
        include_angles: [baseline, contrarian, gaps, current_trends, risks]
        max_questions: 8

  - id: retrieve_initial
    type: loop
    input_from: [generate_questions]
    config:
      iterator: generated_questions
      body:
        - type: tool
          tool: perplexity_search
          args:
            model: sonar-pro
            mode: search_with_citations
            query: "${item}"
            top_k: 8

  - id: gap_analysis
    type: llm
    input_from: [retrieve_initial]
    config:
      prompt_template: prompts/scout_gap_analysis.md
      output_schema: string[]
      rules:
        max_gap_questions: 4

  - id: retrieve_gaps
    type: loop
    input_from: [gap_analysis]
    config:
      iterator: gap_questions
      body:
        - type: tool
          tool: perplexity_search
          args:
            model: sonar-pro
            mode: search_with_citations
            query: "${item}"
            top_k: 8

  - id: enrich_citations
    type: loop
    input_from: [retrieve_initial, retrieve_gaps]
    config:
      iterator: source_items
      body:
        - type: tool
          tool: fetch_url
        - type: tool
          tool: html_to_text
        - type: tool
          tool: extract_citation_snippets

  - id: extract_insights
    type: llm
    input_from: [enrich_citations, load_context]
    config:
      prompt_template: prompts/scout_extract_insights.md
      output_schema: InsightItem[]

  - id: dedupe_rank
    type: reducer
    input_from: [extract_insights]
    config:
      operation: dedupe_and_rank_insights

  - id: summarize_changes
    type: reducer
    input_from: [dedupe_rank, load_context]
    config:
      operation: compute_change_summary
      output_schema: ChangeSummary
      compare_with_prior_runs: true

  - id: synthesize_scout_report
    type: llm
    input_from: [summarize_changes, dedupe_rank]
    config:
      prompt_template: prompts/scout_summarize.md
      output_schema: markdown

  - id: qa_check
    type: llm
    input_from: [synthesize_scout_report, enrich_citations]
    config:
      prompt_template: prompts/scout_qa.md
      output_schema: qa_report

  - id: render_preview
    type: tool
    input_from: [qa_check, synthesize_scout_report]
    config:
      tool: render_report_preview
      args:
        formats: [html, pdf]

  - id: hitl_review
    type: hitl
    input_from: [render_preview]
    config:
      queue: review_tasks
      actions: [approve, reject, edit, add_context]
      additional_context_target_steps: [generate_questions, synthesize_scout_report]

  - id: export_html
    type: tool
    input_from: [hitl_review]
    config:
      tool: export_markdown_to_html

  - id: publish
    type: tool
    input_from: [export_html]
    config:
      tool: publish_artifacts
      args:
        destination: s3

hitl:
  required_steps:
    - hitl_review
  allow_additional_context: true
  timeout_hours: 48
  on_reject: revise
```

### 6.3 YAML skeletons (other workflows)
```yaml
# deep_research.yaml
metadata: { workflow_id: deep_research, version: 1.0.0 }
memory: { load_prior_runs: true, lookback_runs: 3, build_timeline: true }
steps:
  - { id: retrieve_round_1, type: tool }
  - { id: gap_analysis, type: llm }
  - { id: retrieve_round_2, type: tool }
  - { id: synthesize_report, type: llm }
  - { id: source_matrix, type: reducer }
  - { id: render_preview, type: tool }
  - { id: hitl_review, type: hitl }
  - { id: publish, type: tool }
```

```yaml
# longform_article.yaml (triggered from Deep Research tab)
metadata: { workflow_id: longform_article, version: 1.0.0 }
inputs:
  params: { target_publication: forbes }
policy:
  model_routing:
    outline: { provider: anthropic, model: claude-3-7-sonnet }
    draft: { provider: openai, model: gpt-4.1 }
steps:
  - { id: ingest_research, type: reducer }
  - { id: outline, type: llm }
  - { id: draft, type: llm }
  - { id: editorial_pass, type: llm }
  - { id: render_preview, type: tool }
  - { id: hitl_review, type: hitl }
  - { id: publish, type: tool }
```

```yaml
# instagram_carousel.yaml
metadata: { workflow_id: instagram_carousel, version: 1.0.0 }
policy:
  tools_allowed: [render_carousel_images, publish_artifacts]
steps:
  - { id: choose_angle, type: llm }
  - { id: script_slides, type: llm }
  - { id: caption_hashtags, type: llm }
  - { id: safety_tone_check, type: llm }
  - { id: render_carousel_preview, type: tool }
  - { id: hitl_review, type: hitl }
  - { id: publish, type: tool }
```

## 7. Agent Loop (Engine) + HITL

### 7.1 Run state machine
```text
LOAD_CONTEXT
  -> GENERATE_QUESTIONS
  -> RETRIEVE_INITIAL (SONAR_PRO)
  -> GAP_ANALYSIS
  -> RETRIEVE_GAPS (SONAR_PRO)
  -> ENRICH_CITATIONS
  -> EXTRACT_INSIGHTS
  -> DEDUPE/RANK
  -> CHANGE_SUMMARY
  -> SYNTHESIZE_SCOUT_REPORT
  -> QA
  -> RENDER_PREVIEW
  -> HITL
  -> FINALIZE
  -> PUBLISH
  -> COMPLETE
```

Failure paths:
- Any step can transition to `RETRYING`, then back to current step.
- Retry exhaustion transitions to `FAILED_RETRY_EXHAUSTED`.
- HITL reject can transition to `REVISION_REQUIRED` and resume at configured step.
- HITL add-context transitions to `CONTEXT_ADDED` and resumes from configured target step.

### 7.2 Checkpoints and resumability
- Persist checkpoint after each step with:
  - `run_id`, `step_id`, `step_input_ref`, `step_output_ref`, `status`, `attempt_count`.
- Resume algorithm:
  - Load latest successful checkpoint.
  - Skip completed idempotent steps.
  - Continue from first incomplete or revision-target step.

### 7.3 HITL mechanics
- `hitl` step creates a `ReviewTask` item in DynamoDB.
- Run status becomes `WAITING_FOR_REVIEW`.
- Reviewer action options:
  - `approve`: continue to next step.
  - `reject`: route to configured revise/abort behavior.
  - `edit`: save reviewer edits as artifact revision, then continue.
  - `add_context`: attach reviewer context/instructions and resume from configured step (for example `generate_questions` or `synthesize_scout_report`).
- UI Review Queue consumes pending tasks and calls `/review-tasks/{task_id}/decision`.
- App Server posts resume signal to orchestrator with `resume_token` and optional `additional_context`.

## 8. LLM Gateway Details

### 8.1 Unified request format
```json
{
  "request_id": "req_...",
  "run_id": "run_...",
  "step_id": "extract_insights",
  "provider_hint": "openai",
  "model": "gpt-4.1",
  "messages": [{"role": "system", "content": "..."}],
  "tools": [{"name": "perplexity_search", "input_schema": {}}],
  "response_format": {"type": "json_schema", "name": "InsightItemArray"},
  "stream": true,
  "timeout_ms": 60000
}
```

### 8.2 Unified response format
```json
{
  "request_id": "req_...",
  "provider": "openai",
  "model": "gpt-4.1",
  "output": {"type": "json", "data": []},
  "tool_calls": [{"name": "fetch_url", "arguments": {"url": "..."}}],
  "usage": {"input_tokens": 1200, "output_tokens": 640},
  "latency_ms": 4120,
  "finish_reason": "stop"
}
```

### 8.3 Tool-calling normalization
- Normalize provider-specific tool call structures into a single internal schema.
- Enforce schema validation before tool execution.
- Store raw provider payload for audit/debug while exposing normalized payload to orchestrator.

### 8.4 Streaming
- Gateway emits token events and structured partials over SSE/WebSocket.
- Orchestrator can stream to UI while still checkpointing final consolidated output.

### 8.5 Routing and fallback
- Per-step policy chooses primary provider/model.
- Fallback triggered by timeout, rate-limit, or policy-defined error classes.
- Circuit breaker opens after repeated failures and temporarily reroutes traffic.

### 8.6 Observability
- Trace IDs propagated across API -> orchestrator -> gateway -> tools.
- Metrics:
  - request count, latency p50/p95, error rate
  - tokens in/out per model
  - estimated cost per run/workflow/project
- Logs:
  - structured JSON logs with redaction of sensitive fields
- Cost estimation:
  - compute from provider pricing map x token usage (stored per step)

### 8.7 Security
- API keys stored in AWS Secrets Manager; never persisted in artifacts.
- Prompt/response redaction policies for PII-like patterns.
- Audit events for model/tool invocations and review decisions.

## 9. Skills/Tools Runtime

### 9.1 Tool contract
```json
{
  "name": "tool_name",
  "version": "1.0.0",
  "input_schema": {},
  "output_schema": {},
  "permissions": ["network:http", "s3:write"],
  "timeout_ms": 30000,
  "idempotent": true
}
```

Execution envelope:
- Input validated against JSON Schema.
- Output includes `provenance` (`source_url`, `fetched_at`, `hash`).
- Deterministic error object on failure (`code`, `message`, `retryable`).

### 9.2 Initial tools
- `perplexity_search`
  - Input: query/options; Output: ranked results with citations.
- `fetch_url`
  - Input: URL; Output: raw content + metadata.
- `extract_citation_snippets`
  - Input: content + quote hints; Output: snippet spans and confidence.
- `html_to_text`
  - Input: html; Output: normalized plain text.
- `pdf_read`
  - Input: URL or bytes ref + pages; Output: extracted text.
- `render_report_preview`
  - Input: markdown + template + formats; Output: rendered preview artifacts (html/pdf).
- `render_carousel_images`
  - Input: carousel script + style template; Output: rendered slide images for review.
- `publish_artifacts`
  - Input: artifact refs + destination; Output: publish receipts.
- `export_markdown_to_html`
  - Input: markdown; Output: html.

### 9.3 Safety policies and rate limiting
- Per-tool allowlist enforced by workflow policy.
- Domain restriction support for `fetch_url` where needed.
- Max payload size and timeout caps per tool.
- Token-bucket rate limiting by tool and project.
- Automatic quarantine of malformed/unsafe outputs.

## 10. Tauri macOS App Architecture
- React UI runs inside Tauri WebView; Tauri Rust layer handles secure native integrations.
- Credentials (session tokens, optional encrypted API tokens) stored in macOS Keychain.
- App communicates only with App Server APIs; no direct DynamoDB/S3 credentials in client.
- Local cache:
  - recent runs, review drafts, and artifact metadata in local sqlite/file cache.
- Offline behavior (optional MVP+):
  - queue non-destructive actions (notes/edits) and sync when online.
  - run execution still requires cloud services.
- Packaging/updates:
  - signed macOS bundles, staged rollout, and in-app update checks.

## 11. AWS Infrastructure

### 11.1 Reference architecture
```text
[Client: Tauri/Web]
      |
      v
[API Gateway]
      |
      v
[App Server - ECS Fargate]
      |
      +----------------------------+
      |                            |
      v                            v
[Workflow Orchestrator - ECS]   [LLM Gateway - ECS]
      |                            |
      v                            v
[DynamoDB]                    [External LLM/Perplexity APIs]
      |
      v
[S3 Artifacts]

[CloudWatch Logs/Metrics/Alarms]
[Secrets Manager]
```

- API ingress: API Gateway (HTTP API). ALB is acceptable if consolidating container routing.
- Compute: ECS Fargate for long-running orchestrator/gateway; Lambda can be used for lightweight tool adapters.
- Storage: DynamoDB + S3.
- Secret storage: Secrets Manager.
- Observability: CloudWatch + X-Ray/OpenTelemetry exporters.
- Optional note: Step Functions can replace parts of orchestrator, but custom orchestrator is preferred for YAML graph flexibility and HITL resume semantics.

### 11.2 IAM boundaries (least privilege)
- App Server task role:
  - DynamoDB CRUD only on track-catalog/project/scout-profile/run/review tables.
  - S3 read/write only under artifact bucket prefix for allowed projects.
- Orchestrator task role:
  - Run-state tables + artifact bucket writes.
- LLM Gateway task role:
  - Read provider keys from Secrets Manager.
- Deny-by-default SCP/permission boundaries for wildcard resource access.

## 12. Security, Privacy, Compliance
- Data minimization:
  - Store only required source snippets and metadata; avoid full-page retention unless needed.
- PII handling:
  - Detect/redact sensitive content in prompts/logs/artifacts where policy requires.
- Artifact retention:
  - S3 lifecycle policies (e.g., hot 90 days, archive 365 days, purge thereafter per policy).
- Encryption at rest:
  - S3 SSE-KMS and DynamoDB encryption enabled.
- Encryption in transit:
  - TLS for all API and external provider traffic.
- Audit logs:
  - Immutable event records for run transitions, tool calls, model invocations, and HITL decisions.

## 13. Phased Implementation Plan (Codex-friendly)

### Phase 0: Foundations
Deliverables:
- Monorepo service skeleton (`api`, `orchestrator`, `llm_gateway`, `tools_runtime`, `ui`).
- Shared schemas (Pydantic models + JSON Schemas).
- Baseline CI, lint, type checks.

### Phase 1: Persistence backbone
Deliverables:
- DynamoDB table creation + repositories.
- S3 artifact writer + manifest generation.
- Run create/read/update APIs.
- ReviewTask decision payload supports `additional_context`.

### Phase 2: Retrieval and citation enrichment
Deliverables:
- `perplexity_search`, `fetch_url`, `html_to_text`, `extract_citation_snippets`, `pdf_read` tools.
- Configure `perplexity_search` default profile to Sonar Pro for Scout workflows.
- Provenance fields and retry-safe tool execution envelope.

### Phase 3: Intelligence Scout + timeline
Deliverables:
- `intelligence_scout.yaml` implementation.
- Natural-language seed question -> generated question lenses -> gap-filling query loop.
- Change-summary computation and TimelineIndex updates.
- Preview rendering (`html/pdf`) before HITL approval.
- HITL queue integration for Scout report approval.

### Phase 4: Deep Research
Deliverables:
- Iterative research loop with gap analysis.
- Citation-backed report generation + QA checks.

### Phase 5: Publishing workflows + Tauri polish
Deliverables:
- `longform_article` (with `target_publication=forbes|medium`) and `instagram_carousel` workflows.
- Editorial HITL gates with `add_context` action and resume behavior.
- Carousel image rendering before final publish.
- Tauri UX polish, review queue ergonomics, and release packaging.

### MVP acceptance criteria
- A Scout run from one natural-language question completes end-to-end with generated multi-angle questions, citations, change summary, and timeline update.
- Reviewer can approve/reject/edit/add_context and resume run without data loss.
- Reports and carousel images are rendered in preview before final publish.
- Deep research report includes verifiable source mapping.
- Publishing workflows produce expected artifacts (outline/draft or carousel/caption).
- Run observability includes token/cost metrics and traceable step logs.

## 14. Appendix

### 14.1 Minimal API routes
```text
GET    /v1/tracks
POST   /v1/tracks/{track_type}/projects
GET    /v1/tracks/{track_type}/projects
GET    /v1/projects/{project_id}
POST   /v1/projects/{project_id}/scout-profile
GET    /v1/projects/{project_id}/scout-profile
POST   /v1/projects/{project_id}/runs
GET    /v1/projects/{project_id}/runs
GET    /v1/runs/{run_id}
GET    /v1/runs/{run_id}/preview
POST   /v1/runs/{run_id}/resume
POST   /v1/runs/{run_id}/cancel
GET    /v1/runs/{run_id}/artifacts
GET    /v1/review-tasks?status=pending
POST   /v1/review-tasks/{task_id}/decision
```

### 14.2 Example artifact manifest JSON
```json
{
  "run_id": "run_2026_03_01_001",
  "workflow_id": "intelligence_scout",
  "project_id": "proj_123",
  "track_type": "intelligence_scout",
  "artifacts": [
    {
      "artifact_id": "art_001",
      "type": "generated_questions",
      "format": "json",
      "s3_uri": "s3://tnirvana-artifacts/tracks/intelligence_scout/projects/proj_123/runs/run_2026_03_01_001/planning/generated_questions.json",
      "sha256": "..."
    },
    {
      "artifact_id": "art_002",
      "type": "perplexity_sonar_results",
      "format": "json",
      "s3_uri": "s3://tnirvana-artifacts/tracks/intelligence_scout/projects/proj_123/runs/run_2026_03_01_001/retrieval/perplexity_sonar_results.json",
      "sha256": "..."
    },
    {
      "artifact_id": "art_003",
      "type": "change_summary",
      "format": "json",
      "s3_uri": "s3://tnirvana-artifacts/tracks/intelligence_scout/projects/proj_123/runs/run_2026_03_01_001/analysis/change_summary.json",
      "sha256": "..."
    },
    {
      "artifact_id": "art_004",
      "type": "scout_report_md",
      "format": "markdown",
      "s3_uri": "s3://tnirvana-artifacts/tracks/intelligence_scout/projects/proj_123/runs/run_2026_03_01_001/output/scout_report.md",
      "sha256": "..."
    },
    {
      "artifact_id": "art_005",
      "type": "scout_report_preview_pdf",
      "format": "pdf",
      "s3_uri": "s3://tnirvana-artifacts/tracks/intelligence_scout/projects/proj_123/runs/run_2026_03_01_001/preview/scout_report_preview.pdf",
      "sha256": "..."
    }
  ],
  "created_at": "2026-03-01T12:00:00Z"
}
```

### 14.3 Example ReviewTask payload
```json
{
  "task_id": "rt_789",
  "project_id": "proj_123",
  "track_type": "intelligence_scout",
  "run_id": "run_2026_03_01_001",
  "step_id": "hitl_review",
  "status": "PENDING",
  "assignee": "editor_42",
  "actions": ["approve", "reject", "edit", "add_context"],
  "artifact_refs": ["art_003", "art_004", "art_005"],
  "preview_artifact_refs": ["art_005"],
  "additional_context": "Expand coverage on 2026 policy changes and compare with last 3 runs.",
  "resume_token": "resume_abc123",
  "created_at": "2026-03-01T12:05:00Z",
  "sla_due_at": "2026-03-03T12:05:00Z"
}
```
