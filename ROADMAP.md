# ContentPilot AI — Implementation Roadmap

This document outlines the structured, phase-by-phase implementation plan for building the complete, production-grade **ContentPilot AI** platform.

---

## Roadmap Overview

```
Phase 1: Foundation & Infrastructure (DB, Queue, Core Engine, Security)
    │
    ▼
Phase 2: Intelligence & Planning Agents (Research, Strategy, Script, SEO)
    │
    ▼
Phase 3: Media Generation & Synthesis (Narration, Visual Assets, Thumbnails)
    │
    ▼
Phase 4: Video Assembly, FFmpeg Pipeline & QC Engine
    │
    ▼
Phase 5: Human Approval Studio & Idempotent YouTube Publishing
    │
    ▼
Phase 6: Closed-Loop Analytics, Autonomous Operator & Learning Engine
    │
    ▼
Phase 7: Full Integration, End-to-End Testing & Hardening
```

---

## Phase 1: Foundation & Core Infrastructure
**Objective**: Establish the rock-solid, typed backbone of the application with zero debt.

### Deliverables:
1. **Monorepo & Build Configuration**:
   - Backend TypeScript setup with `tsconfig.json`, strict linting, and paths.
   - Frontend Vite + React + TypeScript scaffold with Tailwind CSS / design system tokens.
2. **Database Engine & Prisma Schema**:
   - SQLite initialization with WAL (Write-Ahead Logging) mode and foreign keys enabled.
   - Comprehensive Prisma schema (all 20+ core tables) with typed relations and indexes.
   - Seed scripts for initial settings and channel profile.
3. **Persistent SQLite Job Queue System**:
   - `JobQueue` manager supporting FIFO, priority ordering, and exponential backoff retry.
   - Crash recovery supervisor: detects uncompleted jobs upon server boot and safely restarts them.
   - Worker concurrency governor: prevents resource exhaustion (e.g. max 1 video render at a time).
4. **Structured Logging & Audit Trail**:
   - Winston or Pino logger with JSON structured output and file rotation.
   - Audit trail logger recording all state mutations and human actions.
5. **AI Provider Abstraction Layer**:
   - Interface contracts: `IAIService`, `ITextProvider`, `IImageProvider`, `ITTSProvider`.
   - Primary Google Gemini integration with strict Zod structured output validation.
   - Mock/Test AI provider for deterministic unit testing without API costs.

### Verification Gate:
- Pass all unit tests for DB operations, job queue persistence through simulated SIGKILL, and Gemini structured output parsing.

---

## Phase 2: Intelligence & Content Planning Agents
**Objective**: Implement the upstream creative intelligence agents with strict schema enforcement.

### Deliverables:
1. **Research Agent**:
   - Trend extraction, search volume analysis, and competitor content gap evaluation.
   - Output: Scored content opportunities with citations and search intent analysis.
2. **Strategy Agent**:
   - Video format selection (Deep-dive, tutorial, commentary, short-form).
   - Pillar alignment, target audience persona, and retention hypothesis design.
3. **Script Agent**:
   - Structured screenplay generation: Hook (0-15s), Setup, Multi-part Narrative, Climax, CTA.
   - Scene-level breakdown with explicit visual instructions, voiceover copy, and duration targets.
   - Fact/claim annotations tagging statements requiring external provenance verification.
4. **SEO Agent**:
   - High-CTR title variants (3 distinct angles: Curiosity, Direct Value, Provocative).
   - Search-optimized description with timestamps, video tags, and hashtag optimization.

### Verification Gate:
- Run automated generation for a sample niche topic. Verify that all 4 agents generate validated Zod records and store immutable checkpoints in the database.

---

## Phase 3: Media Generation & Synthesis
**Objective**: Build reliable multi-modal media generation with zero faking and graceful degradation.

### Deliverables:
1. **Narration Agent (Voice Synthesis)**:
   - Gemini / ElevenLabs / Azure Speech provider integrations.
   - Sentence-level audio synthesis, automatic silence trimming, and loudness normalization.
   - Exact word-count-to-duration matching.
   - Zero-faking guarantee: Halts with clear error if TTS credentials fail.
2. **Thumbnail Agent**:
   - Visual concept generator creating 3 distinct compositional prompts.
   - High-resolution image generation (Gemini Imagen 3 / FLUX).
   - Sharp-based image compositing: rule-of-thirds focal placement, high-contrast overlay text, branding badges.
3. **Visual Asset Agent**:
   - Scene-by-scene visual generation matching script visual prompts.
   - Automatic aspect ratio (16:9 long form or 9:16 Shorts) enforcement.
   - B-roll asset fallback to curated public domain or verified licensed sources.

### Verification Gate:
- Complete end-to-end media generation for a 3-scene test script. Validate all audio files and image files are valid, non-zero-byte media on disk.

---

## Phase 4: Video Production Pipeline, FFmpeg & Quality Control
**Objective**: Construct an enterprise-grade, deterministic video rendering and QC pipeline.

### Deliverables:
1. **FFmpeg Video Assembler**:
   - Dynamic filter graph builder: Ken Burns slow-pan & zoom motion for static visuals.
   - Cross-dissolve scene transitions and audio cross-fades.
   - Subtitle/captions generation (.srt / .vtt) with burned-in animated styling.
   - Audio mastering: Voiceover + background music ducking (side-chain compression) targeting YouTube standard -14 LUFS.
2. **Quality Control (QC) Agent**:
   - Automated FFprobe file integrity verification (container, codec, framerate, duration).
   - EBU R128 loudness verification (ensuring within -14 ± 1 LUFS).
   - Black-frame detection and silent-audio segment detection.
   - Audio/video duration drift check (tolerance < 0.2s).
3. **Rights & Provenance Agent**:
   - Verification of all research sources and citations.
   - Synthetic media disclosure generator for YouTube compliance.
   - Explicit disclaimer: Flags human review needed for legal and trademark compliance.

### Verification Gate:
- Render a complete 60-second test video. QC Agent must automatically pass the video or generate a structured failure report with precise timecodes.

---

## Phase 5: Human Approval Studio & Idempotent YouTube Publishing
**Objective**: Create the human-in-the-loop control center and rock-solid YouTube publication pipeline.

### Deliverables:
1. **Approval Studio (Frontend + API)**:
   - High-performance video player with scene timeline navigation.
   - Side-by-side script, SEO, and thumbnail comparison view.
   - Scene editor: allows regenerative tweaking of single scenes without re-rendering the whole video.
   - One-click Approve / Reject with structured editorial revision notes.
   - Enforcement of `AUTO_PUBLISH=false` default.
2. **OAuth 2.0 & Token Management**:
   - Google OAuth 2.0 flow with PKCE and automatic refresh token rotation.
   - Encrypted token storage (AES-256-GCM) with master key rotation.
3. **Publishing Agent (Resumable Uploads)**:
   - Chunked HTTP Resumable Upload protocol implementation (`/upload/youtube/v3/videos?uploadType=resumable`).
   - Idempotency key registry: prevents duplicate uploads during network retries.
   - Automated thumbnail setting, schedule release setting, playlist assignment, and privacy controls.

### Verification Gate:
- Perform an end-to-end simulated/sandbox YouTube upload with network interruption injection. Verify upload resumes from byte offset without creating duplicate entries.

---

## Phase 6: Analytics, Closed-Loop Learning & Autonomous Operator
**Objective**: Close the loop with algorithmic performance monitoring and autonomous operations.

### Deliverables:
1. **Analytics Agent**:
   - Scheduled synchronization with YouTube Analytics API.
   - Metrics ingestion: Impressions, CTR, Audience Retention curve, Watch Time, Subscriber delta.
2. **Learning Agent**:
   - Retention curve drop-off analyzer: correlates drop-off timecodes with scene types.
   - Baseline statistical calculations and performance scoring.
   - Actionable strategy recommendation generator (e.g. "Hooks shorter than 8s perform +34% better").
3. **Autonomous Operator Engine**:
   - Cadence governor: schedules automated generation runs based on channel strategy.
   - Operator run lifecycle: Queued → Researching → Planning → Generation Jobs → Awaiting Approval.
4. **Auto-Fix Agent**:
   - Cross-cutting error classifier and remediation engine.
   - Handles automatic retry of transient network timeouts, FFmpeg dimension padding, and LLM formatting repairs.

### Verification Gate:
- Ingest synthetic and real historical performance snapshots. Verify learning engine updates channel strategy parameters and feeds back into the next research run.

---

## Phase 7: Polish, Dashboard UI & Production Hardening
**Objective**: Deliver a world-class user experience, documentation, and operational tooling.

### Deliverables:
1. **Executive Operations Dashboard**:
   - Real-time pipeline status board (Active jobs, Queued, In Review, Published).
   - Channel health telemetry and growth charts.
   - Settings & credentials management center with real-time status diagnostics.
2. **Operational Hardening & Resilience**:
   - Production process monitoring and recovery.
   - Health check endpoints, graceful shutdown handlers, and DB backup scripts.
