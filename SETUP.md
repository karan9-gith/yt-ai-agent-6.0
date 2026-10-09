# ContentPilot AI — Comprehensive Setup Guide

This guide walks you through setting up ContentPilot AI for local development and operations.

---

## 1. Prerequisites & System Dependencies

1. **Node.js**: Version `v20.x` or later (LTS recommended).
2. **FFmpeg & FFprobe**: Version `6.0` or later installed and accessible via system `PATH`.
   - **Windows**: Install via `winget install Gyan.FFmpeg` or extract from [gyan.dev](https://www.gyan.dev/ffmpeg/builds/) and add `bin` to PATH. Verify via `ffmpeg -version`.
   - **macOS**: `brew install ffmpeg`
   - **Linux (Ubuntu/Debian)**: `sudo apt-get update && sudo apt-get install -y ffmpeg`
3. **SQLite**: Embedded automatically via Prisma (no standalone daemon required).

---

## 2. Repository Installation

Clone the repository and install dependencies in both backend and frontend workspaces:

```bash
# Clone the repository
git clone https://github.com/your-org/contentpilot-ai.git
cd contentpilot-ai

# Install all workspace dependencies at root (automatically sets up backend & client)
npm install
```

---

## 3. Environment Configuration

Copy the environment template and configure your secrets:

```bash
cp .env.example .env
```

### Essential Environment Variables:

| Variable | Description | Default / Example |
| :--- | :--- | :--- |
| `NODE_ENV` | Application environment (`development`, `production`, `test`) | `development` |
| `PORT` | Backend HTTP API port | `3001` |
| `CORS_ORIGIN` | Allowed client origin for CORS | `http://localhost:5173` |
| `APP_SECRET_KEY` | 32+ byte string used for AES-256 token encryption | *Set a secure random key* |
| `DATABASE_URL` | SQLite file connection string | `file:./dev.db` |
| `AUTO_PUBLISH` | Auto-publishing policy (**Must default to false**) | `false` |
| `REQUIRE_HUMAN_APPROVAL`| Mandatory human editorial review | `true` |
| `GEMINI_API_KEY` | Google AI Studio API key | Obtain from Google AI Studio |
| `GEMINI_MODEL` | Gemini LLM model | `gemini-2.5-flash` |
| `YOUTUBE_CLIENT_ID` | Google Cloud OAuth 2.0 Client ID | `your_oauth_client_id` |
| `YOUTUBE_CLIENT_SECRET` | Google Cloud OAuth 2.0 Client Secret | `your_oauth_client_secret` |
| `YOUTUBE_REDIRECT_URI` | OAuth 2.0 authorized callback redirect URI | `http://localhost:3001/api/auth/youtube/callback` |
| `STORAGE_DIR` | Directory for rendered videos and audio assets | `./storage` |

---

## 4. Google Gemini API Setup

1. Visit [Google AI Studio](https://aistudio.google.com/).
2. Create or select a Google Cloud project.
3. Generate a new API key.
4. Set in `.env`:
   ```env
   GEMINI_API_KEY=your_gemini_api_key
   GEMINI_MODEL=gemini-2.5-flash
   ```

---

## 5. YouTube Data API v3 & OAuth 2.0 Setup

1. Open [Google Cloud Console](https://console.cloud.google.com/).
2. Enable the **YouTube Data API v3**.
3. Navigate to **APIs & Services -> OAuth consent screen**:
   - User Type: External
   - Add scopes:
     - `https://www.googleapis.com/auth/youtube.upload`
     - `https://www.googleapis.com/auth/youtube.readonly`
     - `https://www.googleapis.com/auth/youtube.force-ssl`
     - `https://www.googleapis.com/auth/yt-analytics.readonly`
4. Navigate to **APIs & Services -> Credentials**:
   - Create Credentials -> **OAuth 2.0 Client IDs**.
   - Application type: **Web application**.
   - Name: `ContentPilot AI Studio`.
   - Authorized redirect URIs:
     `http://localhost:3001/api/auth/youtube/callback`
5. Copy Client ID and Client Secret into `.env`:
   ```env
   YOUTUBE_CLIENT_ID=your_client_id.apps.googleusercontent.com
   YOUTUBE_CLIENT_SECRET=your_client_secret
   YOUTUBE_REDIRECT_URI=http://localhost:3001/api/auth/youtube/callback
   ```

---

## 6. Database Initialization

Run Prisma code generation and apply database schema migrations:

```bash
# From repository root:
npm run prisma:generate
npm run prisma:push
```

---

## 7. Development & Running Commands

All commands can be executed directly from the project root using npm workspace scripts:

### Run Both Backend and Frontend Concurrently:
```bash
npm run dev:all
# Backend: http://localhost:3001 | Frontend: http://localhost:5173
```

### Run Backend Only:
```bash
npm run dev
# Starts Express backend on http://localhost:3001 with hot reload
```

### Run Frontend Only:
```bash
npm run dev:client
# Starts Vite React dashboard on http://localhost:5173
```

> **Note on Workspaces Error**:
> If you ever encounter:
> `npm error No workspaces found: --workspace=backend`
> This occurs when executing a workspace flag command from inside a subfolder (`cd backend` or `cd client`) instead of the repository root. Always run workspace commands (`--workspace=backend`) from the root directory where the master `package.json` resides.

---

## 8. Build & Test Commands

Run the comprehensive test suite (all 9 suites, unit, integration, and E2E):

```bash
npm test
```

Run TypeScript compilation and build across workspaces:

```bash
npm run typecheck
npm run build
```

---

## 9. Android Termux Compatibility Guide

ContentPilot AI can be developed or managed on Android devices via Termux. However, certain runtime binaries (notably Prisma's query engine and FFmpeg) have specific platform constraints:

### Key Constraints in Native Termux:
1. **Prisma Engine Incompatibility with Bionic Libc**:
   - Native Termux uses Android's Bionic libc rather than Linux glibc or musl.
   - Standard `@prisma/engines` binaries compiled for `linux-arm64-openssl-*` expect glibc and fail to load under native Termux.
2. **Native Node.js Addons**:
   - Packages like `sharp` require prebuilt binaries that target glibc.

### Recommended Solution: PRoot-Distro (Ubuntu/Debian)
The recommended approach for running ContentPilot AI on Android is using `proot-distro` to run a full glibc-based Ubuntu environment:

```bash
# 1. Inside Termux, install PRoot-Distro
pkg update && pkg install -y proot-distro

# 2. Install Ubuntu
proot-distro install ubuntu

# 3. Log in to Ubuntu
proot-distro login ubuntu

# 4. Install Node.js 20, npm, and FFmpeg inside Ubuntu
apt update && apt install -y curl ffmpeg
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# 5. Clone and run ContentPilot AI inside PRoot Ubuntu
git clone <repo-url> contentpilot-ai
cd contentpilot-ai
npm install
npm run prisma:generate
npm run prisma:push
npm run dev:all
```

Inside PRoot Ubuntu, standard Linux ARM64 binaries for Prisma and Node.js execute reliably without compatibility issues.

---

## 10. Database Backup Instructions

Because ContentPilot AI uses SQLite with Write-Ahead Logging (`WAL`), create consistent hot backups using the safe SQLite backup API or copy when idle:

```bash
# Hot backup using sqlite3 CLI
sqlite3 backend/prisma/dev.db ".backup 'backend/prisma/backup-$(date +%Y%m%d%H%M%S).db'"
```

---

## 11. Known Limitations

1. **Daily YouTube Quota**: Standard Google Cloud projects receive 10,000 YouTube Data API units/day. A video upload costs 1,600 units. ContentPilot AI auto-fix schedules backoff retries when quota is exhausted.
2. **Local FFmpeg Performance**: Video render speed depends on CPU hardware encoding capabilities.

