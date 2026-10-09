# ContentPilot AI — System Architecture Specification

## 1. Executive Summary & Vision

**ContentPilot AI** is an enterprise-grade, autonomous YouTube automation and channel operations platform. Designed from first principles to overcome the brittle scripting, monolithic coupling, and simulated faking of prior experimental agents, ContentPilot AI implements a resilient, human-governed, multi-agent pipeline.

The platform coordinates 16 specialized AI agents across content research, strategic planning, scriptwriting, SEO optimization, thumbnail generation, voice synthesis, visual asset assembly, FFmpeg-based video rendering, quality assurance, provenance verification, human approval, idempotent publishing, and closed-loop algorithmic learning.

```
       ┌─────────────────────────────────────────────────────────────┐
       │                   ContentPilot AI Core                      │
       └─────────────────────────────────────────────────────────────┘
                                      │
    ┌─────────────────────────┬───────┴─────────┬────────────────────────┐
    ▼                         ▼                 ▼                        ▼
┌──────────────┐      ┌───────────────┐ ┌──────────────┐      ┌────────────────────┐
│ Multi-Agent  │      │  State & Job  │ │ Video Engine │      │ YouTube Publishing │
│ Orchestrator │◄────►│ System (Queue)│ │ (FFmpeg/QC)  │◄────►│ & Analytics Engine │
└──────────────┘      └───────────────┘ └──────────────┘      └────────────────────┘
    ▲                         ▲                 ▲                        ▲
    │                         │                 │                        │
    └─────────────────────────┼─────────────────┼────────────────────────┘
                              ▼                 ▼
                     ┌──────────────────┐ ┌───────────────┐
                     │  SQLite Database │ │ Human Approval│
                     │   (Prisma ORM)   │ │  Dashboard    │
                     └──────────────────┘ └───────────────┘
```

---

## 2. Reference Repository Architectural Evaluation

A comprehensive audit of the reference implementation (`darkzOGx/youtube-automation-agent`) revealed critical architectural patterns as well as fundamental structural deficiencies that ContentPilot AI resolves:

### 2.1 Architectural Comparison Matrix

| Dimension | Reference Architecture (`youtube-automation-agent`) | ContentPilot AI Architecture |
| :--- | :--- | :--- |
| **Code Structure** | Monolithic (`index.js` 88KB, `db.js` 110KB, `app.js` 106KB) | Clean Layered Modular Architecture (Monorepo: `server/` & `client/`) |
| **Language & Typing** | Untyped JavaScript (CommonJS) | Strict TypeScript with end-to-end type safety |
| **Database Layer** | Raw SQL queries with string concatenation, callback hell | Prisma ORM with SQLite, typed models, and atomic transactions |
| **Data Validation** | Manual ad-hoc checks; fragile regex JSON parsing | Zod schemas for all API payloads, agent I/O, and LLM structured outputs |
| **Job Execution** | In-memory `Promise` map with fragile polling; lost on crash | Persistent SQLite-backed Job Queue with worker heartbeats & restart recovery |
| **Stage Checkpoints** | Basic checkpoint table; prone to stale state | Immutable Artifact Hashing, deterministic stage cache, granular invalidated rollback |
| **Generation Integrity** | **Fakes generation** via gradient placeholders when keys/tools miss | **Zero-Faking Guarantee**: strict validation; halts with actionable diagnostics |
| **YouTube Upload** | Raw streaming `videos.insert`; high timeout failure rate | Official Resumable Upload protocol with chunking, retry offset, & idempotency keys |
| **Video Production** | Ad-hoc exec calls; minimal validation | Strongly typed FFmpeg/FFprobe pipeline with LUFS audio check & black frame QC |
| **Human-in-the-Loop** | Inconsistent approval gates; optional manual flags | **Strict Human-Gated Approval by default**; `AUTO_PUBLISH=false` enforced |
| **Legal & Provenance** | Superficial source listing | Rigorous provenance audit trails, claim flags, and synthetic media disclosures |
| **Frontend** | Vanilla JS monolith manipulating raw DOM strings | React 18+ with Vite, TanStack Query, typed components, responsive dark dashboard |

---

## 3. Core Architectural Principles

1. **Human Governance First**: Automation accelerates creativity but must never bypass human editorial judgment. The default posture of the system is `APPROVAL_REQUIRED = true` and `AUTO_PUBLISH = false`.
2. **Absolute Generation Integrity**: The platform will **never fake** rendering, synthesis, audio generation, or uploads. If a dependency, credential, or external service is unavailable, the pipeline records a structured error and pauses for intervention.
3. **Fault Tolerance & Crash Survival**: All agent executions and pipeline transitions are executed as durable jobs in SQLite. If the Node.js process terminates abruptly, active jobs are reclaimed and safely resumed from their last verified checkpoint upon restart.
4. **Idempotent Operations**: Every external side-effect (YouTube upload, image generation, external API request) carries an idempotency token to ensure that retries cannot produce duplicate videos or double charges.
5. **Least-Privilege Security & Zero Shell Injection**: AI agents output structured JSON strictly validated by Zod schemas. AI outputs are never directly interpolated into shell commands. All FFmpeg/FFprobe invocations execute via sanitized `execFile` parameter arrays.
6. **Provider-Agnostic Abstraction**: Google Gemini serves as the primary intelligence backbone, encapsulated behind unified abstract provider interfaces (`IAIService`, `ITextProvider`, `IImageProvider`, `ITTSProvider`, `IVideoProvider`) to allow zero-downtime provider switching.

---

## 4. System Layer Topology

```
┌────────────────────────────────────────────────────────────────────────┐
│                        PRESENTATION LAYER (Vite + React)               │
│  - Pipeline Visualizer & Stage Stepper                                 │
│  - Scene Manifest Editor & Media Previewer                             │
│  - Human Approval Studio & Provenance Inspector                       │
│  - Analytics Dashboard & Recommendation Tuning Engine                  │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ HTTP REST + SSE / WebSockets
┌───────────────────────────────────▼────────────────────────────────────┐
│                     APPLICATION API LAYER (Node / Express / TS)         │
│  - Route Controllers with Zod Request/Response Validation              │
│  - OAuth 2.0 PKCE YouTube Handlers & Session Authentication            │
│  - Real-time Event Streaming (SSE) for Pipeline Telemetry              │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼────────────────────────────────────┐
│                     ORCHESTRATION & DOMAIN LOGIC LAYER                 │
│  - Autonomous Operator Engine (Cadence & Run Manager)                  │
│  - Pipeline Coordinator (State Transitions & Stage Enforcers)          │
│  - Multi-Agent Runtime (16 Isolated Specialized Agents)                │
│  - Auto-Fix & Diagnostics Classifier                                   │
└───────────────────┬────────────────────────────────┬───────────────────┘
                    │                                │
┌───────────────────▼──────────────┐   ┌─────────────▼───────────────────┐
│     INFRASTRUCTURE SERVICES      │   │     PERSISTENCE & JOBS LAYER    │
│  - AI Provider Factory (Gemini)  │   │  - Persistent SQLite Job Queue  │
│  - FFmpeg / FFprobe Media Engine │   │  - Prisma ORM Client            │
│  - YouTube Resumable Client      │   │  - Artifact File Store          │
│  - Secure Token Vault (AES-GCM)  │   │  - Structured Audit Log Stream  │
└──────────────────────────────────┘   └─────────────────────────────────┘
```

---

## 5. Technology Stack Decisions & Justifications

### 5.1 Backend: Node.js & TypeScript
- **Node.js (v20+ LTS)**: High-performance asynchronous I/O, ideal for streaming media uploads, child process orchestration, and event-driven architectures.
- **TypeScript (v5+)**: Full static type safety eliminates an entire class of runtime errors common in untyped automation bots.
- **Express / Fastify**: Lightweight, battle-tested HTTP engine with modular routing and robust middleware pipelines.

### 5.2 Persistence & ORM: SQLite + Prisma
- **SQLite 3 (via better-sqlite3 or Prisma engine)**: Zero-latency embedded database, zero external infrastructure overhead, WAL (Write-Ahead Logging) mode enabled for high-concurrency read/write transactions, and effortless backup/portability.
- **Prisma ORM**: Declarative schema migrations, fully typed client generation, relations, and compile-time query safety.

### 5.3 Validation & Structured Output: Zod
- **Zod (v3.23+)**: Enforces strict schema validation for client requests, environment configurations, and critically, LLM JSON responses. Every agent prompt specifies a JSON schema that is validated via Zod before data enters the database or pipeline.

### 5.4 Video & Media Rendering: FFmpeg & FFprobe
- **Native FFmpeg/FFprobe**: Industry-standard media processing. Provides frame-accurate trimming, multi-track audio normalization (EBU R128), video stitching, Ken Burns pan-and-zoom motion filters, and automated quality validation (black frame and silence detection).

### 5.5 Primary Intelligence: Google Gemini
- **`@google/genai` (Gemini 2.5/3.0 series)**: Exceptional context windows, multi-modal reasoning capabilities, native JSON output schemas, and high throughput. Integrated behind an extensible provider abstraction.

### 5.6 Frontend: React 18 + Vite + Tailwind/Modern CSS
- **Vite + React 18**: Ultra-fast hot module replacement, predictable component lifecycle, and optimized bundle size.
- **TanStack Query**: Server state caching, background refetching, and real-time synchronization.
- **Lucide Icons & Tailwind CSS / Modern Tokens**: Sleek, modern dark-mode aesthetic with responsive, high-density operational views.

---

## 6. High-Level Agent Workflow & Pipeline Topology

The complete lifecycle follows a strictly sequential, gate-checked pipeline:

```
[1. Research Agent]
         │ (Topic scoring, competitor signals, keyword clusters)
         ▼
[2. Strategy Agent]
         │ (Format, target duration, hook concept, pillar alignment)
         ▼
[3. Script Agent]
         │ (Scene blueprints, full narration, visual prompts, claim annotations)
         ▼
[4. SEO Agent]
         │ (3x Title variants, description, tags, chapters, search intent)
         ▼
[5. Thumbnail Agent]
         │ (Concept prompts, image synthesis, visual typography layouts)
         ▼
[6. Narration Agent]
         │ (TTS synthesis, word-level audio alignment, silence verification)
         ▼
[7. Visual Asset Agent]
         │ (AI image/video scene generation, asset validation, Ken Burns fx)
         ▼
[8. Video Production Agent]
         │ (FFmpeg timeline assembly, audio mixing, subtitles burn-in)
         ▼
[9. Quality Control (QC) Agent]
         │ (FFprobe checks, LUFS loudness, resolution/fps, sync validation)
         ▼
[10. Rights & Provenance Agent]
         │ (Source attribution, synthetic media disclosure, legal claim check)
         ▼
[11. Approval Agent (Human Gate)] ──[REJECT: Back to Scene/Script]
         │ (Preview review, script diff, manual sign-off required)
         ▼ [APPROVED]
[12. Publishing Agent]
         │ (Resumable YouTube upload, thumbnail set, metadata sync, scheduler)
         ▼
[13. Analytics Agent]
         │ (Performance tracking: CTR, retention curve, views, watch time)
         ▼
[14. Learning Agent]
         │ (Correlation engine, retention dip diagnosis, recommendation generation)
         ▼
[15. Autonomous Operator & Next Strategy]
         │ (Channel cadence governor, next topic prioritization)
         ▼
[16. Auto-Fix Agent (Cross-Cutting Supervisor)]
         │ (Monitors failures across any stage, executes safe bounded repairs)
```

---

## 7. Storage & File System Organization

Media assets and artifacts are organized deterministically using unique generation IDs:

```
storage/
├── productions/
│   └── {productionId}/
│       ├── artifacts/
│       │   ├── strategy.json
│       │   ├── script.json
│       │   ├── seo.json
│       │   └── provenance.json
│       ├── audio/
│       │   ├── narration.wav
│       │   ├── narration_normalized.wav
│       │   └── bgm.mp3
│       ├── visuals/
│       │   ├── scene_01.png
│       │   ├── scene_02.mp4
│       │   └── raw/
│       ├── thumbnails/
│       │   ├── thumb_concept_1.png
│       │   ├── thumb_concept_2.png
│       │   └── thumb_final.png
│       ├── video/
│       │   ├── intermediate_timeline.mp4
│       │   └── final_render.mp4
│       └── logs/
│           ├── ffmpeg.log
│           └── qc_report.json
└── backups/
    └── sqlite_wal_snapshots/
```

---

## 8. Reliability, Idempotency & Error Handling Design

1. **Job Heartbeats & Crash Recovery**: Every running job writes periodic heartbeats (`last_heartbeat_at`). A background supervisor detects jobs stuck in `running` without a recent heartbeat (e.g. after a process crash) and re-queues them safely from the last valid checkpoint.
2. **Resumable Network I/O**: Network uploads track byte offsets in the database. If an upload connection drops, the publishing agent queries the YouTube Resumable URI for the last received byte and resumes without restarting the upload from 0%.
3. **Stage Checkpoint Invalidation Cascades**: When a user modifies an earlier stage (e.g., editing the script in Scene 3), only downstream artifacts (audio for Scene 3, video render, QC report) are invalidated. Upstream artifacts (strategy, SEO, unaffected scene visuals) remain cached.
4. **Auto-Fix Sandbox**: When an agent or FFmpeg operation fails, the Auto-Fix Agent inspects the error taxonomy (e.g., `AUDIO_CLIPPING`, `FFMPEG_DIMENSION_MISMATCH`, `RATE_LIMIT_EXCEEDED`). It adjusts parameters (e.g. padding dimensions to divisible-by-2, applying exponential backoff) and re-attempts execution up to 3 bounded tries.
