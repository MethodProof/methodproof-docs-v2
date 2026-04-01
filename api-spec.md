# MethodProof API Specification

Base URL: `https://api.methodproof.com` (production) / `http://localhost:8000` (local dev)

All endpoints return JSON. Protected endpoints require `Authorization: Bearer <jwt>` header. All responses include `X-Correlation-ID` header for tracing.

---

## Auth

### POST /auth/register

Create a new company and admin user.

**Request:**
```json
{
  "company_name": "Acme Corp",
  "email": "admin@acme.com",
  "password": "s3cureP@ss"
}
```

**Response (201):**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "uuid",
    "email": "admin@acme.com",
    "role": "admin",
    "company_id": "uuid"
  }
}
```

```bash
curl -X POST http://localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"company_name":"Acme Corp","email":"admin@acme.com","password":"s3cureP@ss"}'
```

### POST /auth/login

Authenticate and receive a JWT (24h expiry).

**Request:**
```json
{
  "email": "admin@acme.com",
  "password": "s3cureP@ss"
}
```

**Response (200):**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "uuid",
    "email": "admin@acme.com",
    "role": "admin",
    "company_id": "uuid"
  }
}
```

```bash
curl -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@acme.com","password":"s3cureP@ss"}'
```

### POST /auth/sso/saml

Enterprise SSO callback (Wave 6).

### GET /auth/me

Return the current authenticated user.

**Response (200):**
```json
{
  "id": "uuid",
  "email": "admin@acme.com",
  "role": "admin",
  "company_id": "uuid",
  "company_name": "Acme Corp"
}
```

```bash
curl http://localhost:8000/auth/me \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

**Error (401):**
```json
{"detail": "Invalid or expired token"}
```

---

## Assessments

All assessment endpoints are scoped to the authenticated user's company.

### POST /assessments

**Request:**
```json
{
  "title": "Senior Backend Engineer — System Design",
  "description": "Build a rate limiter service...",
  "time_limit_minutes": 90,
  "config": {
    "allowed_models": ["claude-sonnet-4-6", "claude-haiku-4-5"],
    "token_budget": 50000,
    "allowed_tools": ["web_search", "documentation"],
    "blocked_domains": ["chegg.com", "coursehero.com"],
    "project_template": "rate-limiter-starter",
    "webcam_required": false,
    "languages": ["python", "go"]
  }
}
```

**Response (201):**
```json
{
  "id": "uuid",
  "title": "Senior Backend Engineer — System Design",
  "description": "Build a rate limiter service...",
  "time_limit_minutes": 90,
  "config": { "..." },
  "created_at": "2026-03-30T10:00:00Z"
}
```

### GET /assessments

Paginated list for company.

**Query params:** `page` (default 1), `per_page` (default 20)

**Response (200):**
```json
{
  "items": [
    {
      "id": "uuid",
      "title": "Senior Backend Engineer — System Design",
      "candidate_count": 12,
      "created_at": "2026-03-30T10:00:00Z"
    }
  ],
  "total": 1,
  "page": 1,
  "per_page": 20
}
```

### GET /assessments/:id

Full assessment with config.

### PUT /assessments/:id

Update assessment config. Same body as POST.

### DELETE /assessments/:id

Soft delete (sets status to `archived`). Returns `204`.

### POST /assessments/:id/invite

**Request:**
```json
{
  "candidate_email": "candidate@example.com",
  "accommodations": {
    "extra_time_minutes": 30
  }
}
```

**Response (201):**
```json
{
  "session_id": "uuid",
  "candidate_email": "candidate@example.com",
  "status": "pending",
  "invite_sent_at": "2026-03-30T10:05:00Z"
}
```

### GET /assessments/:id/candidates

**Response (200):**
```json
{
  "candidates": [
    {
      "session_id": "uuid",
      "email": "candidate@example.com",
      "status": "completed",
      "overall_score": 78,
      "started_at": "2026-03-30T11:00:00Z",
      "ended_at": "2026-03-30T12:28:00Z"
    }
  ]
}
```

### POST /assessments/:id/rubric

Set rubric signal weights.

**Request:**
```json
{
  "name": "Security-Focused Backend",
  "base": "default",
  "weight_overrides": {
    "testing_behavior": 0.25,
    "code_judgment": 0.25,
    "token_efficiency": 0.05
  },
  "custom_signals": [
    {
      "name": "security_awareness",
      "weight": 0.15,
      "query": "security_edge_cases.cypher",
      "description": "Did the candidate consider auth, input validation, injection?"
    }
  ]
}
```

### GET /assessments/:id/rubric

Returns the rubric config for this assessment.

### GET /assessments/:id/analytics

Aggregate analytics across all candidates for this assessment (Wave 6).

---

## Templates

### POST /templates

**Request:**
```json
{
  "repo_url": "https://github.com/acme/rate-limiter-starter",
  "assessment_id": "uuid"
}
```

### GET /templates

List templates for company.

### GET /templates/:id

Template metadata parsed from `methodproof.yaml`.

### POST /templates/:id/validate

Clones repo and checks for required files: `README.md`, `methodproof.yaml`, `tests/public/`.

**Response (200):**
```json
{
  "valid": true,
  "errors": []
}
```

**Response (200, invalid):**
```json
{
  "valid": false,
  "errors": ["Missing file: methodproof.yaml", "Missing directory: tests/public/"]
}
```

---

## Sessions — Container-facing

### POST /sessions

Initialize session from invite token.

**Request:**
```json
{
  "session_token": "invite-token-from-email-link"
}
```

**Response (200):**
```json
{
  "session_id": "uuid",
  "config": {
    "allowed_models": ["claude-sonnet-4-6", "claude-haiku-4-5"],
    "token_budget": 50000,
    "blocked_domains": ["chegg.com", "coursehero.com"],
    "time_limit_minutes": 90
  }
}
```

**Error (401):** Invalid or expired session token.

### GET /sessions/:id/config

Full MCP + proxy config for the container.

**Response (200):**
```json
{
  "allowed_models": ["claude-sonnet-4-6", "claude-haiku-4-5"],
  "token_budget": 50000,
  "blocked_domains": ["chegg.com", "coursehero.com"],
  "time_limit_minutes": 90,
  "allowed_tools": ["web_search", "documentation"],
  "project_template": "rate-limiter-starter"
}
```

### POST /sessions/:id/heartbeat

**Request:**
```json
{
  "local_time": "2026-03-30T14:23:01.000Z"
}
```

**Response (200):**
```json
{
  "server_time": "2026-03-30T14:23:01.123Z"
}
```

If no heartbeat for >5 minutes, session status is set to `abandoned`.

### POST /sessions/:id/events

Batch telemetry events from the container.

**Request:** See [Telemetry Event Schema](#telemetry-event-schema) below.

**Response (200):**
```json
{
  "accepted": 47,
  "rejected": 3,
  "rejections": [
    {"id": "uuid", "reason": "duplicate event ID"},
    {"id": "uuid", "reason": "invalid action type"}
  ]
}
```

Rate limit: 100 events per batch, 10 batches per minute per session.

### POST /sessions/:id/screenshots

Multipart upload of screenshot batch with metadata JSON.

**Response (200):**
```json
{
  "stored": [
    {"key": "company-id/session-id/screenshots/2026-03-30T14:23:01.jpg", "size_bytes": 48200}
  ]
}
```

### POST /sessions/:id/attention

**Request:**
```json
{
  "points": [
    {
      "timestamp": "2026-03-30T14:23:01.000Z",
      "gaze_x": 0.45,
      "gaze_y": 0.62,
      "head_pose": {"pitch": 5.2, "yaw": -3.1, "roll": 0.8},
      "face_present": true
    }
  ]
}
```

### POST /sessions/:id/webcam

Multipart webcam clip upload. Validates consent record includes webcam category.

**Response (200):**
```json
{
  "key": "company-id/session-id/webcam/clip-001.webm",
  "signed_url": "https://s3.../..."
}
```

### PUT /sessions/:id/submit

Candidate submits their work. Sets status to `completed`.

**Response (200):**
```json
{
  "status": "completed",
  "ended_at": "2026-03-30T12:28:00Z"
}
```

**Error (409):** Session already completed.

### PUT /sessions/:id/timeout

System timeout. Sets status to `timed_out`.

---

## Sessions — Extension-facing

### POST /sessions/:id/browser-events

Same batch format as container `/events`, but only accepts browser-prefixed types: `browser_search`, `browser_visit`, `browser_tab_switch`, `browser_copy`, `browser_ai_chat`.

Returns 400 for non-browser event types.

### GET /sessions/:id/extension-config

**Response (200):**
```json
{
  "categories": {
    "search": ["google.com", "bing.com", "duckduckgo.com", "kagi.com", "perplexity.ai"],
    "docs": ["docs.python.org", "developer.mozilla.org", "devdocs.io", "readthedocs.io"],
    "code_host": ["github.com", "gitlab.com", "bitbucket.org"],
    "qa": ["stackoverflow.com", "stackexchange.com", "reddit.com/r/programming"],
    "ai_chat": ["chat.openai.com", "claude.ai", "gemini.google.com", "perplexity.ai"],
    "cheating": ["chegg.com", "coursehero.com", "brainly.com"],
    "package": ["pypi.org", "npmjs.com", "crates.io", "pkg.go.dev"],
    "reference": ["wikipedia.org", "w3schools.com", "geeksforgeeks.org"]
  },
  "redaction_params": ["auth", "token", "key", "password", "secret", "session", "jwt", "api_key"],
  "server_time": "2026-03-30T14:23:01.123Z"
}
```

### POST /sessions/:id/extension-heartbeat

**Request:**
```json
{
  "local_time": "2026-03-30T14:23:01.000Z"
}
```

**Response (200):**
```json
{
  "server_time": "2026-03-30T14:23:01.123Z"
}
```

If no extension heartbeat for >5 minutes during an active session, a Moment node with `type=extension_disconnect` is created.

---

## Review — Dashboard-facing

### GET /sessions/:id/graph

Full session process graph for D3 visualization.

**Response (200):**
```json
{
  "nodes": [
    {
      "id": "uuid",
      "type": "Action",
      "label": "llm_prompt",
      "properties": {
        "timestamp": "2026-03-30T14:23:01.234Z",
        "duration_ms": null,
        "metadata": {"model": "claude-sonnet-4-6", "token_count": 342}
      }
    },
    {
      "id": "uuid",
      "type": "Artifact",
      "label": "file",
      "properties": {"path": "src/limiter.py", "content_hash": "abc123"}
    }
  ],
  "edges": [
    {"source": "uuid", "target": "uuid", "type": "NEXT", "properties": {"gap_ms": 1200}},
    {"source": "uuid", "target": "uuid", "type": "INFORMED", "properties": {}}
  ]
}
```

### GET /sessions/:id/graph/subgraph

Filtered subgraph.

**Query params:** `types` (comma-separated action types), `start` (ISO timestamp), `end` (ISO timestamp)

### GET /sessions/:id/timeline

Narrated timeline with chapters.

**Response (200):**
```json
{
  "chapters": [
    {
      "title": "Problem Analysis",
      "start_time": "2026-03-30T11:00:00Z",
      "end_time": "2026-03-30T11:04:30Z",
      "description": "Candidate read the README and explored the starter code...",
      "screenshot_refs": ["company/session/screenshots/t1.jpg", "company/session/screenshots/t2.jpg"]
    }
  ]
}
```

### GET /sessions/:id/moments

**Response (200):**
```json
{
  "moments": [
    {
      "id": "uuid",
      "type": "rapid_iteration",
      "timestamp": "2026-03-30T11:15:00Z",
      "duration_ms": 90000,
      "description": "6 LLM prompts in 90 seconds — candidate was iterating on the rate limiting algorithm",
      "action_ids": ["uuid", "uuid", "uuid"]
    }
  ]
}
```

### GET /sessions/:id/scores

**Response (200):**
```json
{
  "session_id": "uuid",
  "overall_score": 78,
  "process_score": 82,
  "artifact_score": 74,
  "calibration": "72nd percentile",
  "signals": {
    "problem_comprehension": {"score": 85, "weight": 0.10, "evidence": "42s before first prompt"},
    "prompt_engineering": {"score": 90, "weight": 0.20, "evidence": "7 prompts, avg specificity 0.84"},
    "iteration_quality": {"score": 75, "weight": 0.15, "evidence": "2 dead ends, avg recovery 45s"},
    "code_judgment": {"score": 80, "weight": 0.20, "evidence": "modified 6/8 AI outputs"},
    "token_efficiency": {"score": 70, "weight": 0.10, "evidence": "23,400 tokens for medium complexity"},
    "testing_behavior": {"score": 65, "weight": 0.15, "evidence": "tests written after implementation"},
    "tool_selection": {"score": 88, "weight": 0.10, "evidence": "Haiku for boilerplate, Sonnet for design"}
  }
}
```

### GET /sessions/:id/artifacts

**Response (200):**
```json
{
  "artifacts": [
    {
      "id": "uuid",
      "path": "src/limiter.py",
      "type": "file",
      "content_hash": "abc123",
      "is_ai_generated": false,
      "linked_action_ids": ["uuid", "uuid"],
      "snapshot_url": "https://s3.../signed-url"
    }
  ]
}
```

### GET /sessions/:id/screenshots/:ref

Returns signed S3 URL for the screenshot.

### GET /sessions/:id/webcam/:ref

Returns signed S3 URL for the webcam clip.

### GET /compare

Side-by-side comparison of two sessions.

**Query params:** `sessions` (comma-separated IDs), `signals` (comma-separated signal names)

---

## Webhooks

### POST /webhooks

**Request:**
```json
{
  "url": "https://acme.com/hooks/methodproof",
  "events": ["session.started", "session.completed", "scores.ready", "session.abandoned"],
  "secret": "whsec_abc123"
}
```

### GET /webhooks

List registered webhooks for company.

### DELETE /webhooks/:id

Remove webhook. Returns `204`.

**Webhook delivery:** On event, payload is signed with HMAC-SHA256 using the webhook secret. Signature sent in `X-MethodProof-Signature` header. Retries: 1s, 5s, 30s (max 3 attempts).

---

## Health

### GET /health

Shallow health check (no auth required).

**Response (200):**
```json
{"status": "ok"}
```

### GET /health/deep

Deep health check with dependency status (no auth required).

**Response (200):**
```json
{
  "status": "ok",
  "checks": {
    "neo4j": {"status": "ok", "latency_ms": 12},
    "postgres": {"status": "ok", "latency_ms": 5},
    "redis": {"status": "ok", "latency_ms": 2},
    "s3": {"status": "ok", "latency_ms": 45}
  },
  "timestamp": "2026-03-30T14:23:01.123Z"
}
```

Returns `503` if Neo4j or Postgres unreachable.

---

## Telemetry Event Schema

All telemetry (container and extension) uses this batch format:

```json
{
  "events": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "type": "<action_type>",
      "timestamp": "2026-03-30T14:23:01.234Z",
      "duration_ms": null,
      "metadata": {}
    }
  ]
}
```

### Action Types (15 total)

| Type | Source | Metadata |
|------|--------|----------|
| `llm_prompt` | MCP Server | `{model, prompt_text, token_count, temperature, tools_available}` |
| `llm_completion` | MCP Server | `{model, response_text, token_count, finish_reason, latency_ms}` |
| `web_search` | Network Proxy | `{engine, query, result_count, clicked_results}` |
| `web_visit` | Network Proxy | `{url, domain, duration_ms, category}` |
| `file_edit` | FS Watcher | `{path, diff, lines_added, lines_removed, is_ai_generated}` |
| `file_create` | FS Watcher | `{path, size, language}` |
| `file_delete` | FS Watcher | `{path}` |
| `terminal_cmd` | Terminal Monitor | `{command, exit_code, output_snippet}` |
| `test_run` | Terminal Monitor | `{framework, passed, failed, duration_ms}` |
| `git_commit` | FS Watcher | `{hash, message, files_changed}` |
| `browser_search` | Extension | `{engine, query, result_count}` |
| `browser_visit` | Extension | `{url, domain, category, duration_ms, title}` |
| `browser_tab_switch` | Extension | `{from_domain, to_domain, to_category}` |
| `browser_copy` | Extension | `{source_url, source_domain, text_length, text_snippet}` |
| `browser_ai_chat` | Extension | `{platform, url, detected_input}` |

Container events (types 1-10) go to `POST /sessions/:id/events`.
Extension events (types 11-15) go to `POST /sessions/:id/browser-events`.

### Metadata Per Type

**llm_prompt:**
```json
{
  "model": "claude-sonnet-4-6",
  "prompt_text": "Write a Python function that...",
  "token_count": 342,
  "temperature": 0.7,
  "tools_available": ["web_search", "documentation"]
}
```

**llm_completion:**
```json
{
  "model": "claude-sonnet-4-6",
  "response_text": "Here is the implementation...",
  "token_count": 1205,
  "finish_reason": "end_turn",
  "latency_ms": 2340
}
```

**web_search:**
```json
{
  "engine": "google.com",
  "query": "python rate limiter sliding window",
  "result_count": 10,
  "clicked_results": ["stackoverflow.com/q/123", "docs.python.org/..."]
}
```

**web_visit:**
```json
{
  "url": "https://docs.python.org/3/library/collections.html",
  "domain": "docs.python.org",
  "duration_ms": 45000,
  "category": "docs"
}
```

**file_edit:**
```json
{
  "path": "src/limiter.py",
  "diff": "@@ -10,3 +10,8 @@\n+def sliding_window(...):\n+    ...",
  "lines_added": 5,
  "lines_removed": 0,
  "is_ai_generated": true
}
```

**file_create:**
```json
{
  "path": "tests/test_limiter.py",
  "size": 1240,
  "language": "python"
}
```

**file_delete:**
```json
{
  "path": "src/old_limiter.py"
}
```

**terminal_cmd:**
```json
{
  "command": "pytest tests/ -v",
  "exit_code": 1,
  "output_snippet": "FAILED tests/test_limiter.py::test_window - AssertionError..."
}
```

**test_run:**
```json
{
  "framework": "pytest",
  "passed": 8,
  "failed": 2,
  "duration_ms": 3400
}
```

**git_commit:**
```json
{
  "hash": "a1b2c3d",
  "message": "Add sliding window rate limiter",
  "files_changed": ["src/limiter.py", "tests/test_limiter.py"]
}
```

**browser_search:**
```json
{
  "engine": "google.com",
  "query": "redis sliding window rate limiting",
  "result_count": 15
}
```

**browser_visit:**
```json
{
  "url": "https://stackoverflow.com/questions/123/rate-limiting",
  "domain": "stackoverflow.com",
  "category": "qa",
  "duration_ms": 32000,
  "title": "How to implement rate limiting in Python"
}
```

**browser_tab_switch:**
```json
{
  "from_domain": "stackoverflow.com",
  "to_domain": "docs.python.org",
  "to_category": "docs"
}
```

**browser_copy:**
```json
{
  "source_url": "https://stackoverflow.com/questions/123",
  "source_domain": "stackoverflow.com",
  "text_length": 85,
  "text_snippet": "from collections import deque\n\ndef sliding_window(max_requests, window_seconds):\n..."
}
```

**browser_ai_chat:**
```json
{
  "platform": "chatgpt",
  "url": "https://chat.openai.com/c/abc123",
  "detected_input": true
}
```
