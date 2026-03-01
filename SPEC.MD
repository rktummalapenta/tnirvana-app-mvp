# TNirvana Agent OS - Implementation Spec (MVP)

## 1. Scope
Build a track-first platform where users work in top-level tracks (`intelligence_scout`, `deep_research`, `instagram`, future `news`) and create projects under each track.

MVP must support:
- Intelligence Scout run from natural-language question.
- Deep Research run with citations.
- Instagram carousel workflow with rendered image preview.
- HITL review with actions: `approve`, `reject`, `edit`, `add_context`.
- Pre-final rendering for reports/carousels before publish.
- Tauri macOS desktop app as a first-class MVP client.

## 2. Product Behavior
### 2.1 Track-first UX
- User selects a track tab.
- User creates/selects project under that track.
- User starts workflow run.

### 2.2 Scout workflow behavior
Input:
- `seed_question` (natural language).

Execution:
1. Generate multi-angle sub-questions (baseline, contrarian, gaps, current trends, risks).
2. Retrieve with `perplexity_search` using `sonar-pro`.
3. Run gap analysis and second retrieval pass.
4. Enrich citations and extract insights.
5. Compare with prior runs and produce `change_summary` + timeline update.
6. Synthesize Scout report.
7. Render preview (`html`, `pdf`).
8. HITL review.
9. Finalize and publish.

## 3. System Components
- `ui-shell` (Tauri/Web): tabs, projects, runs, preview viewer, HITL queue.
- `app-server` (FastAPI): auth, projects, profiles, runs, artifacts, review tasks.
- `orchestrator` (FastAPI worker/service): checkpointed YAML execution.
- `llm-gateway`: OpenAI + Claude routing, fallback, streaming.
- `tools-runtime`: Perplexity/search/fetch/extract/render/publish tools.
- `storage`: DynamoDB metadata + S3 artifacts.

### 3.1 Tauri App Requirements (MVP)
- Platform: macOS (Apple Silicon + Intel).
- Stack: Tauri + React UI.
- Required screens:
  - Track tabs and project list.
  - Run detail with step status.
  - Preview viewer (HTML/PDF/image carousel).
  - HITL queue with actions `approve/reject/edit/add_context`.
- Security:
  - Store session credentials/tokens in macOS Keychain.
  - No direct AWS/S3 credentials in client.
  - Artifact access only through API-provided pre-signed URLs.
- Data behavior:
  - Read/write only via App Server APIs.
  - Live status via `/v1/runs/{run_id}/events` with `/status` polling fallback.
  - Local cache for recent run list, preview metadata, and draft review comments.
- Packaging:
  - Signed macOS app bundle and versioned update channel.

## 4. Data Contracts
## 4.1 DynamoDB tables
- `TrackCatalog(track_type)`
- `Projects(track_type, project_id)`
- `ScoutProfiles(project_id, profile_id=primary)`
- `Runs(project_id, started_at#run_id)`
- `RunEvents(run_id, event_at#seq)` (for live status/event stream)
- `Artifacts(run_id, artifact_id)`
- `ReviewTasks(status_bucket, created_at#task_id)`
- `TimelineIndex(project_id, summary_at)`

## 4.2 Key entities
- `ScoutProfile`: `project_id`, `seed_question`, `question_lenses[]`, `cadence`, `lookback_runs`.
- `SourceItem`: normalized retrieved source + snippet/citation data.
- `InsightItem`: insight + linked source IDs.
- `ChangeSummary`: new/changed/faded insights vs prior run.
- `ArtifactManifest`: canonical list of run artifacts and checksums.

Run status fields (in `Runs`):
- `status`, `state`, `current_step`, `completed_steps`, `total_steps`, `progress_pct`, `last_heartbeat_at`, `eta_seconds`.

## 5. Artifact and Preview Model
Artifact layers:
- S3 file (content)
- DynamoDB artifact row (index)
- manifest JSON (inventory)

Credential boundary:
- Client never gets S3 credentials.
- Backend services access S3 via IAM role.
- Client reads artifacts using API-issued pre-signed URLs.

Required Scout artifacts:
- `input/seed_question.json`
- `planning/generated_questions.json`
- `retrieval/perplexity_sonar_results.json`
- `analysis/insights.json`
- `analysis/change_summary.json`
- `output/scout_report.md`
- `preview/scout_report_preview.html`
- `preview/scout_report_preview.pdf`

Instagram required preview artifacts:
- `preview/carousel_slide_01.png` ... `preview/carousel_slide_n.png`

## 6. API Surface (MVP)
- `GET /v1/tracks`
- `POST /v1/tracks/{track_type}/projects`
- `GET /v1/tracks/{track_type}/projects`
- `GET /v1/projects/{project_id}`
- `POST /v1/projects/{project_id}/scout-profile`
- `GET /v1/projects/{project_id}/scout-profile`
- `POST /v1/projects/{project_id}/runs`
- `GET /v1/projects/{project_id}/runs`
- `GET /v1/runs/{run_id}`
- `GET /v1/runs/{run_id}/status`
- `GET /v1/runs/{run_id}/events` (SSE)
- `GET /v1/runs/{run_id}/preview`
- `POST /v1/runs/{run_id}/resume`
- `POST /v1/runs/{run_id}/cancel`
- `GET /v1/runs/{run_id}/artifacts`
- `GET /v1/review-tasks?status=pending`
- `POST /v1/review-tasks/{task_id}/decision`

Artifact API behavior:
- Artifact/preview endpoints return metadata plus pre-signed URLs for read access.
- Client never calls S3 with long-lived credentials.

Review decision payload (MVP):
```json
{
  "action": "approve|reject|edit|add_context",
  "comments": "string",
  "additional_context": "string",
  "resume_from_step": "optional_step_id"
}
```

## 7. Workflow Engine Requirements
- Step types: `llm`, `tool`, `reducer`, `loop`, `hitl`.
- Checkpoint after every step.
- Idempotent retries with bounded attempts and backoff.
- Resume from checkpoint and/or `resume_from_step` after HITL.
- `add_context` persists context and injects into subsequent prompts.
- Emit run events on step transitions and periodic heartbeat updates.

## 7.1 Job Status Requirements
- Status API must return:
  - `status`, `state`, `current_step`, `progress_pct`, `last_heartbeat_at`, `eta_seconds`.
- Event stream must emit:
  - `run.started`, `step.started`, `step.completed`, `step.failed`, `run.waiting_for_review`, `run.completed`.
- UI behavior:
  - Run detail subscribes to SSE when available.
  - Poll `/status` every 5-10 seconds as fallback.

## 8. Tools Runtime Requirements
Required tools:
- `perplexity_search` (use `sonar-pro` for Scout)
- `fetch_url`
- `extract_citation_snippets`
- `html_to_text`
- `pdf_read`
- `render_report_preview`
- `render_carousel_images`
- `export_markdown_to_html`
- `publish_artifacts`

Tool contract:
- JSON-schema validated input/output
- timeout + retryable error shape
- provenance metadata on outputs

## 9. HITL and Review Requirements
- Run pauses at HITL with `ReviewTask`.
- Reviewer sees preview artifacts.
- Supported actions:
  - `approve`: continue finalize/publish.
  - `reject`: transition `REVISION_REQUIRED`.
  - `edit`: save edited artifact and continue.
  - `add_context`: persist reviewer context and resume from configured step.

## 10. Non-Functional Requirements
- Observability: structured logs, traces, token/cost metrics.
- Security: Secrets Manager for API keys; SSE-KMS on S3; DynamoDB encryption.
- Reliability: retry policies, dead-letter failure state.
- Desktop quality: Tauri app launch under 3s on warm start, stable preview rendering for HTML/PDF/PNG.

## 11. Acceptance Criteria
- Scout run from one natural-language question produces multi-angle queries, insights, change summary, timeline update, and preview artifacts.
- Deep research outputs previewable report before publish.
- Instagram workflow renders carousel preview images before publish.
- HITL `add_context` action can alter subsequent run output without losing previous artifacts.
- All produced artifacts discoverable via `/v1/runs/{run_id}/artifacts` and `/v1/runs/{run_id}/preview`.
- Tauri app can complete end-to-end flow: create/select project, start run, view preview, submit HITL action, and resume run.
- User can observe live run status/progress in UI via `/status` and `/events`.
