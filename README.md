# ContentPilot AI 🚀

Enterprise-grade autonomous YouTube operations platform powered by multi-agent orchestration, strict human governance, deterministic media rendering, and closed-loop algorithmic learning.

---

## 1. What It Is

**ContentPilot AI** is an end-to-end autonomous content operations engine and review dashboard for YouTube channels. It orchestrates a 20-stage pipeline: market research, competitive analysis, strategic ideation, script synthesis, thumbnail concept generation, voice narration, rendering, audio loudness normalization (EBU R128), state-machine safety checks, mandatory human editorial review, and scheduled YouTube Data API v3 publishing.

---

## 2. Features

- **Autonomous Multi-Agent Pipeline**: Research, Strategy, Scripting, SEO, Thumbnails, Video Assembly, and Quality Control.
- **Mandatory Human Editorial Gate**: AI cannot publish content autonomously. Every project halts at the Human Approval Gate until explicitly signed off in Review Studio.
- **Google Gemini Integration**: Real multimodal AI generation (dynamic market research, script synthesis, SEO metadata, and thumbnail concepts) powered by the `@google/genai` SDK using `GEMINI_API_KEY`.
- **YouTube OAuth 2.0 Integration**: Real Google Cloud OAuth 2.0 connectivity with resumable video uploading, playlist selection, and scheduled release.
- **Real-Time Pipeline Telemetry**: Live status updates across jobs, render progress, and background worker queues.
- **Closed-Loop Learning Agent**: Ingests lifetime performance snapshots to iteratively refine title hooks and content retention strategies.
- **A/B Experimentation Engine**: Systematic split-testing for titles, thumbnails, and publishing schedules.
- **Auto-Fix Self-Healing**: Automated recovery from transient quota limits, LLM output anomalies, and audio loudness drifts.
- **19 Dedicated Operations Consoles**: Complete management across research, scripts, production, review, distribution, analytics, scheduler, and system health.

---

## 3. Project Structure

```
contentpilot-ai/
├── client/                     # Frontend Single Page Application (React 18, Vite, TypeScript)
│   ├── src/
│   │   ├── components/         # Header, Sidebar, VideoPlayer, EmptyState
│   │   ├── pages/              # 19 Operations Consoles (Overview, Review, Scripts, etc.)
│   │   ├── services/           # Direct backend API client
│   │   └── types/              # TypeScript models and domain definitions
│   ├── package.json
│   ├── tsconfig.json
│   └── vite.config.ts          # Vite build & local dev proxy configuration
├── backend/                    # Backend API & Pipeline Runtime (Node.js, Express, TypeScript)
│   ├── src/
│   │   ├── api/                # 19 Modular REST API routes
│   │   ├── config/             # Zod environment schema & validation
│   │   ├── core/               # Pipeline state machine & state types
│   │   ├── database/           # Prisma SQLite client & WAL pragma configuration
│   │   ├── jobs/               # Background persistent queue & worker manager
│   │   ├── services/           # Gemini AI provider, YouTube OAuth, scheduler, auto-fix
│   │   └── worker.ts           # Standalone worker runtime
│   ├── prisma/
│   │   └── schema.prisma       # Prisma SQLite database schema
│   ├── tests/                  # Vitest unit, integration, and end-to-end test suites
│   ├── package.json
│   ├── tsconfig.json
│   └── vitest.config.ts        # Test runner configuration
├── package.json                # Root npm workspaces configuration
├── package-lock.json           # Locked dependency tree
├── .gitignore                  # Source-control exclusion rules
├── .env.example                # Local environment variable template
├── API.md                      # REST API endpoints documentation
├── ARCHITECTURE.md             # Architecture overview & state machine
├── ENVIRONMENT.md              # Configuration reference
├── SECURITY.md                 # Security & governance controls
├── SETUP.md                    # Setup and prerequisite instructions
├── TROUBLESHOOTING.md          # Diagnostic & recovery guide
└── README.md                   # Project documentation
```

---

## 4. Local Development

### Prerequisites
- **Node.js**: Version `v20.x` or later (LTS recommended)
- **npm**: Version `10.x` or later
- **FFmpeg & FFprobe**: Version `6.0` or later installed and accessible in system `PATH`
- **SQLite**: Embedded automatically via Prisma (no daemon required)

### Installation & First Run

```bash
# 1. Install all dependencies across workspaces
npm install

# 2. Configure environment variables
cp .env.example .env

# 3. Initialize SQLite database schema
npm run prisma:generate
npx prisma db push --schema=backend/prisma/schema.prisma

# 4. Run backend and frontend concurrently
npm run dev:all
```

- **Frontend UI**: `http://localhost:5173`
- **Backend API**: `http://localhost:3001`
- **Health Check**: `http://localhost:3001/health`

### Available NPM Commands (Run from Project Root)

| Command | Action |
| :--- | :--- |
| `npm run dev:all` | Start both backend server and Vite frontend concurrently |
| `npm run dev` | Start backend development server with hot-reload (`tsx watch`) |
| `npm run dev:client` | Start frontend Vite development server (`vite`) |
| `npm run build` | Build all workspaces (client and backend) |
| `npm run build:client` | Build Vite frontend production bundle |
| `npm run build:backend` | Generate Prisma client and compile backend TypeScript (`tsc`) |
| `npm test` | Run complete backend test suite (9 test suites, 74 tests) |
| `npm run typecheck` | Perform TypeScript type checking without emitting files |
| `npm run lint` | Run TypeScript lint check across workspaces |
| `npm run prisma:generate` | Generate Prisma Client from schema |
| `npm run prisma:push` | Sync SQLite database schema with Prisma definition |

> **Workspaces Tip**: All npm workspace commands (`--workspace=backend`, etc.) must be run from the repository root. If run from inside a subfolder, npm will return `npm error No workspaces found`. See [SETUP.md](file:///c:/Users/Antplay/Downloads/ai%20youtube%20agent/SETUP.md) for Android Termux PRoot-Distro guide.

---

## 5. Google Gemini Configuration

ContentPilot AI integrates with Google Gemini for multimodal content operations:

1. Obtain an API key at [Google AI Studio](https://aistudio.google.com/).
2. Set your key in `.env`:
   ```env
   GEMINI_API_KEY=your_actual_gemini_api_key
   GEMINI_MODEL=gemini-2.5-flash
   ```
3. If the key is missing or not configured, the application reports a configuration error requiring `GEMINI_API_KEY` rather than producing synthetic results.

---

## 6. YouTube Data API v3 & OAuth 2.0 Configuration

To enable YouTube publishing and analytics synchronization:

1. Open [Google Cloud Console](https://console.cloud.google.com/).
2. Enable the **YouTube Data API v3**.
3. Create **OAuth 2.0 Client IDs** (Web application):
   - **Authorized redirect URI**: `http://localhost:3001/api/auth/youtube/callback`
4. Configure in `.env`:
   ```env
   YOUTUBE_CLIENT_ID=your_oauth_client_id.apps.googleusercontent.com
   YOUTUBE_CLIENT_SECRET=your_oauth_client_secret
   YOUTUBE_REDIRECT_URI=http://localhost:3001/api/auth/youtube/callback
   ```
5. In the dashboard, navigate to **YouTube Channel** and click **Connect YouTube OAuth**.

---

## 7. Environment Variables

Reference template in `.env.example`:

| Variable | Required | Description | Default |
| :--- | :--- | :--- | :--- |
| `NODE_ENV` | Optional | Runtime environment (`development`, `test`, `production`) | `development` |
| `PORT` | Optional | Backend API port | `3001` |
| `CORS_ORIGIN` | Optional | Allowed client origin | `http://localhost:5173` |
| `APP_SECRET_KEY` | **Required** | Secret key for AES-256 token encryption | None |
| `DATABASE_URL` | **Required** | SQLite connection string | `file:./dev.db` |
| `AUTO_PUBLISH` | Optional | Auto-publish toggle (Must default to false) | `false` |
| `REQUIRE_HUMAN_APPROVAL` | Optional | Enforces human approval in Review Studio | `true` |
| `GEMINI_API_KEY` | **Required for AI** | Google Gemini API key | None |
| `GEMINI_MODEL` | Optional | Model identifier | `gemini-2.5-flash` |
| `YOUTUBE_CLIENT_ID` | Optional | YouTube OAuth Client ID | None |
| `YOUTUBE_CLIENT_SECRET` | Optional | YouTube OAuth Client Secret | None |
| `YOUTUBE_REDIRECT_URI` | Optional | OAuth callback redirect URL | `http://localhost:3001/api/auth/youtube/callback` |
| `STORAGE_DIR` | Optional | Local asset and media storage directory | `./storage` |

---

## 8. Security & Operational Governance

- **Zero Secret Exposure**: Server credentials never leave the local backend environment.
- **Frontend Cleanliness**: No private keys or OAuth secrets are compiled into the client bundle.
- **Encrypted Token Storage**: YouTube OAuth refresh tokens are encrypted at rest using AES-256-GCM.
- **Non-Bypassable Governance**: Pre-upload safety guards enforce `status === 'APPROVED'` and QC verification before any network request to YouTube is dispatched.
- **Strict CORS Policy**: Whitelists only explicitly configured frontend origins.

---

## 📄 License

MIT License. Designed and built for autonomous, governed YouTube production.
