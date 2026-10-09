# ContentPilot AI — Troubleshooting & Incident Response

This guide covers common issues, root cause diagnoses, and recovery procedures.

---

## 1. Auto-Fix Engine Diagnostic Classifications

ContentPilot AI automatically categorizes all runtime failures into a 9-class error taxonomy:

| Classification | Meaning | Automatic Recovery Strategy |
| :--- | :--- | :--- |
| `TRANSIENT` | Network timeout, temporary 503 | Exponential backoff retry |
| `VALIDATION` | Malformed JSON from AI | LIFO-stack structural repair |
| `MEDIA` | FFmpeg dimension/aspect ratio mismatch | Dimension padding to even mod-2 dimensions |
| `QUOTA` | YouTube API 10,000 units/day limit reached | Backoff cool-down without consuming rapid retries |
| `AUTHENTICATION`| Expired or revoked OAuth refresh token | Halt retries, flag channel for user re-authentication |
| `CONFIGURATION` | Missing environment variable | Halt execution with exact variable diagnosis |
| `DATABASE` | SQLite busy timeout | Auto-retry with jitter |
| `NETWORK` | DNS failure, connection reset | Scheduled retry |
| `UNKNOWN` | Unhandled runtime exception | Escalated to system error log |

---

## 2. Common Issues & Solutions

### Issue 1: YouTube API Quota Exceeded (`quotaExceeded`)
- **Symptom**: YouTube upload returns HTTP 403 `quotaExceeded: YouTube API quota limit reached for today`.
- **Diagnosis**: Standard Google Cloud projects have a daily limit of 10,000 quota units. A video upload costs 1,600 units.
- **Resolution**:
  1. The Auto-Fix engine escalates the error and schedules the upload for the next UTC midnight window.
  2. To increase limits, request a quota increase in Google Cloud Console under **APIs & Services -> YouTube Data API v3 -> Quotas**.

### Issue 2: OAuth Token Expired or Invalid Grant (`invalid_grant`)
- **Symptom**: `Token has been expired or revoked`.
- **Diagnosis**: User changed Google account password or OAuth consent screen was set to "Testing" (tokens expire in 7 days).
- **Resolution**:
  1. Set OAuth consent screen to "In production" in Google Cloud Console to avoid 7-day token expiration.
  2. In the dashboard, navigate to **YouTube Channel** and click **Connect YouTube OAuth** to re-authenticate.

### Issue 3: FFmpeg Execution Failure
- **Symptom**: Render error during video assembly.
- **Diagnosis**: Missing codec libraries or non-even frame dimensions (e.g., 1081x1920).
- **Resolution**:
  1. Verify FFmpeg installation via `ffmpeg -version`.
  2. ContentPilot AI automatically applies filter `-vf "pad=ceil(iw/2)*2:ceil(ih/2)*2"` to ensure width and height are divisible by 2.

### Issue 4: SQLite Database Lock (`SQLITE_BUSY`)
- **Symptom**: `database is locked` error during high concurrent write traffic.
- **Diagnosis**: SQLite running without WAL mode or prolonged transactions.
- **Resolution**:
  1. Verify WAL mode is enabled:
     ```bash
     sqlite3 backend/prisma/dev.db "PRAGMA journal_mode;"
     # Output must be: wal
     ```
  2. ContentPilot AI initializes SQLite with `PRAGMA busy_timeout = 5000;` to buffer concurrent writes.

---

## 3. Crash Recovery Procedures

If the server crashes during an active pipeline run:
1. **Restart the server**:
   ```bash
   npm run start
   ```
2. **Automatic Crash Sweep**:
   - `persistentJobQueue.recoverOrphanedJobs()` runs on startup and identifies any job in `RUNNING` state without a fresh heartbeat (>30s).
   - Orphaned jobs are automatically rescheduled with incremented retry counts.
3. **Pipeline Resumption**:
   - The Autonomous Operator resumes from the last valid checkpoint without re-synthesizing earlier stages.
