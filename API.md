# ContentPilot AI — REST API Reference

Base URL: `http://localhost:3001`

---

## 1. System & Operations

### `GET /health`
Returns quick health heartbeat and governance flags.
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "governance": { "autoPublish": false, "requireHumanApproval": true }
}
```

### `GET /health/detailed` or `GET /api/health`
Performs live probes of all 8 core services (Backend, Database, Gemini, YouTube, FFmpeg, Storage, Scheduler, Worker).
```json
{
  "success": true,
  "status": "healthy",
  "components": [
    { "name": "Backend Core", "status": "healthy", "message": "Running on Node v20.18.0" },
    { "name": "Database (SQLite/WAL)", "status": "healthy", "latencyMs": 2 }
  ]
}
```

### `GET /api/overview`
Query Parameters: `?channelId=<id>`
Returns aggregated active jobs, pending approvals, scheduled videos, and lifetime views.

---

## 2. Autonomous Operator

### `GET /api/operator/status`
Query Parameters: `?channelId=<id>`
Returns active pipeline state, stage index (1-20), approval gate state, and recent transitions.

### `POST /api/operator/run`
Body:
```json
{
  "channelId": "channel_cuid",
  "targetAudience": "Engineers",
  "existingProjectId": "optional_project_id_for_resumption"
}
```

### `POST /api/operator/pause`
Body: `{ "projectId": "project_cuid" }`

### `POST /api/operator/resume`
Body: `{ "projectId": "project_cuid", "channelId": "channel_cuid" }`

### `POST /api/operator/cancel`
Body: `{ "projectId": "project_cuid" }`

---

## 3. Review Studio & Approvals

### `GET /api/review/pending`
Query Parameters: `?channelId=<id>`
Lists all projects currently halted at Stage 16 awaiting explicit human review.

### `GET /api/review/review/:projectId`
Returns complete 11-element inspection bundle (video, thumbnail, script, SEO, sources, QC, rights, logs).

### `POST /api/review/:projectId/approve`
Body: `{ "approvedBy": "editor@domain.com", "notes": "Approved" }`
Advances state machine from `AWAITING_APPROVAL` to `APPROVED`.

### `POST /api/review/:projectId/reject`
Body: `{ "reviewer": "editor@domain.com", "reason": "Violates brand voice" }`
Transitions state machine to `REJECTED`.

### `POST /api/review/:projectId/edit`
Body: `{ "edits": { "title": "New Title", "description": "New Desc" }, "editor": "editor@domain.com" }`

### `POST /api/review/:projectId/regenerate`
Body: `{ "stage": "visual", "requestedBy": "editor@domain.com" }`

### `POST /api/review/:projectId/send-back`
Body: `{ "targetStage": "SCRIPT_READY", "notes": "Fix tone", "reviewer": "editor@domain.com" }`

### `POST /api/review/:projectId/schedule`
Body: `{ "scheduledPublishTime": "2026-10-10T14:00:00Z", "privacyStatus": "private", "scheduledBy": "editor@domain.com" }`

---

## 4. YouTube OAuth & Upload Engine

### `GET /api/youtube/auth-url`
Returns Google OAuth 2.0 consent screen URL.

### `POST /api/youtube/callback`
Body: `{ "code": "4/0A...", "channelId": "channel_cuid" }`
Exchanges auth code for tokens, encrypts via AES-256-GCM, and connects channel.

### `POST /api/youtube/upload/:projectId`
Body: `{ "channelId": "channel_cuid", "privacyStatus": "private" }`
Enforces `preUploadSafetyChecker` and uploads video with idempotency guarantee.

---

## 5. Scheduler & Queue

### `GET /api/scheduler/tasks`
Lists all persistent scheduled tasks.

### `POST /api/scheduler/tasks`
Creates a new scheduled task (one-time, daily, weekly, custom interval).

### `GET /api/publishing-queue`
Lists videos queued for YouTube release.

### `POST /api/publishing-queue/:id/cancel`
Cancels queued upload.

---

## 6. Analytics, Learning & Experiments

### `POST /api/analytics/sync/:channelId`
Polls YouTube Analytics and captures lifetime snapshots.

### `GET /api/learning/insights/:channelId`
Returns data-backed insights with evidence, confidence rating, and recommendations.

### `POST /api/experiments`
Creates A/B hypothesis test for titles, thumbnails, or topics.

### `POST /api/experiments/:id/approve`
Human editorial sign-off on experiment winner before applying to channel strategy.
