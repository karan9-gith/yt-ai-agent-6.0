# ContentPilot AI — Environment Variables Specification

This document details all configuration parameters recognized by ContentPilot AI.

---

## 1. Core Server & Security

| Variable | Required | Default | Description | Sensitivity |
| :--- | :--- | :--- | :--- | :--- |
| `NODE_ENV` | Optional | `development` | Runtime environment (`development`, `production`, `test`) | Low |
| `PORT` | Optional | `3001` | HTTP listening port for Express backend | Low |
| `CORS_ORIGIN` | Optional | `http://localhost:5173` | Allowed origin for Cross-Origin Resource Sharing | Low |
| `APP_SECRET_KEY` | **Required** | None | 32+ character key used to derive AES-256-GCM cipher for OAuth tokens | **CRITICAL** |
| `DATABASE_URL` | **Required** | `file:./dev.db` | SQLite database URI | Medium |

---

## 2. Operational Safety & Governance

| Variable | Required | Default | Description | Sensitivity |
| :--- | :--- | :--- | :--- | :--- |
| `AUTO_PUBLISH` | Optional | `false` | When `false`, publishing without human signoff is impossible | High |
| `REQUIRE_HUMAN_APPROVAL` | Optional | `true` | Enforces explicit human approval in Review Studio | High |

---

## 3. AI Providers

| Variable | Required | Default | Description | Sensitivity |
| :--- | :--- | :--- | :--- | :--- |
| `GEMINI_API_KEY` | Recommended | None | Google AI Studio API Key for research, scripts, and SEO | **CRITICAL** |
| `GEMINI_MODEL` | Optional | `gemini-2.5-flash` | Gemini model identifier | Low |
| `OPENAI_API_KEY` | Optional | None | OpenAI API key for secondary LLM operations | **CRITICAL** |
| `OPENAI_MODEL` | Optional | `gpt-4o-mini` | OpenAI model identifier | Low |
| `ELEVENLABS_API_KEY` | Optional | None | ElevenLabs voice synthesis API key | **CRITICAL** |
| `ELEVENLABS_VOICE_ID` | Optional | `21m00Tcm4TlvDq8ikWAM` | Default voice ID for ElevenLabs synthesis | Low |
| `REPLICATE_API_TOKEN` | Optional | None | Replicate token for alternative image diffusion | **CRITICAL** |

---

## 4. YouTube OAuth Integration

| Variable | Required | Default | Description | Sensitivity |
| :--- | :--- | :--- | :--- | :--- |
| `YOUTUBE_CLIENT_ID` | Optional | None | Google Cloud Console OAuth 2.0 Web Client ID | Medium |
| `YOUTUBE_CLIENT_SECRET` | Optional | None | Google Cloud Console OAuth 2.0 Client Secret | **CRITICAL** |
| `YOUTUBE_REDIRECT_URI` | Optional | `http://localhost:3001/api/auth/youtube/callback` | OAuth redirect URI configured in Cloud Console | Low |

---

## 5. Media & Storage

| Variable | Required | Default | Description | Sensitivity |
| :--- | :--- | :--- | :--- | :--- |
| `FFMPEG_PATH` | Optional | System `PATH` | Custom path to `ffmpeg` binary if not in PATH | Low |
| `FFPROBE_PATH` | Optional | System `PATH` | Custom path to `ffprobe` binary if not in PATH | Low |
| `STORAGE_DIR` | Optional | `./storage` | Directory where video renders and assets are stored | Medium |
| `MAX_CONCURRENT_RENDERS` | Optional | `1` | Maximum parallel video rendering operations | Low |

---

## 6. Job Worker & Scheduler

| Variable | Required | Default | Description | Sensitivity |
| :--- | :--- | :--- | :--- | :--- |
| `JOB_WORKER_POLL_INTERVAL_MS` | Optional | `2000` | Polling frequency for persistent SQLite job worker | Low |
| `JOB_HEARTBEAT_INTERVAL_SEC` | Optional | `30` | Heartbeat interval to detect orphaned jobs | Low |
| `JOB_MAX_ATTEMPTS` | Optional | `3` | Maximum retry attempts for transient worker failures | Low |
