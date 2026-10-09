# ContentPilot AI — Multi-Agent System Specification

ContentPilot AI employs an orchestrated ensemble of 16 specialized, autonomous agents. Each agent possesses bounded responsibilities, strongly typed contracts enforced via **Zod**, immutable state checkpointing, and isolated failure domains.

---

## Agent System Overview

```
                        ┌───────────────────────────────┐
                        │      Autonomous Operator      │
                        └──────────────┬────────────────┘
                                       │ orchestrates
   ┌───────────────────────────────────┼───────────────────────────────────┐
   ▼                                   ▼                                   ▼
[Research Agent]            [Rights/Provenance Agent]           [Auto-Fix Agent]
   │                                   │                               (Supervisor)
   ▼                                   ▼
[Strategy Agent]               [Approval Agent] ◄── [Quality Control (QC) Agent]
   │                                   │                      ▲
   ▼                                   ▼                      │
[Script Agent]                 [Publishing Agent]     [Video Production Agent]
   │                                   │                      ▲
   ├───────────────┬────────────────┐  ▼                      │
   ▼               ▼                ▼ [Analytics Agent]       ├─────────────────┐
[SEO Agent] [Thumbnail Agent] [Narration Agent]               ▼                 ▼
                                       │             [Visual Asset Agent]  (Audio Master)
                                       ▼                      │
                                [Learning Agent] ─────────────┘
```

---

## Detailed Agent Catalog

### 1. Research Agent
- **Purpose**: Discovers high-probability content topics, analyzes competitive channels, and tracks emerging audience trends.
- **Inputs**:
  - `channelProfile`: Niche, target demographic, forbidden topics, brand voice.
  - `focusKeywords`: Array of seed terms or industry domains.
  - `historicalPerformance`: Past topics and audience response metrics.
- **Outputs** (Zod Enforced):
  - `opportunities`: Array of scored topics (`topic`, `angle`, `searchVolumeScore`, `competitionScore`, `opportunityScore`, `rationale`, `sources`).
- **Prompt Strategy**: Few-shot contextual analysis directing Gemini to identify information gaps and untapped angles rather than generic rehashes.
- **Fallback**: Fallback to evergreen pillar backlog if trending signals are cold.

---

### 2. Strategy Agent
- **Purpose**: Translates a chosen content opportunity into an executable video strategy aligned with channel growth KPIs.
- **Inputs**:
  - `selectedTopic`: Chosen research opportunity.
  - `channelStrategy`: Target cadence, video format (Explainer, Documentary, Tutorial, Short), target duration.
- **Outputs** (Zod Enforced):
  - `strategy`: `targetLength` (seconds), `format`, `hookHypothesis`, `coreValueProposition`, `targetAudiencePersona`, `retentionPacingStrategy`.
- **Interactions**: Passes strategic boundaries to the Script and SEO Agents.

---

### 3. Script Agent
- **Purpose**: Writes the complete, cinematic, retention-engineered screenplay with scene-by-scene visual and audio directions.
- **Inputs**:
  - `strategy`: Formatted video strategy.
  - `brandVoice`: Tone parameters (e.g. authoritative, witty, empathetic).
- **Outputs** (Zod Enforced):
  - `title`: Working title.
  - `hook`: `text`, `visualCue`, `targetDuration` (must be <= 15s).
  - `scenes`: Array of `SceneBlueprint`:
    - `index`: Numeric position (1..N).
    - `label`: Semantic section name (e.g., "Setup", "The Hidden Flaw").
    - `scriptText`: Exact voiceover narration words.
    - `visualPrompt`: Detailed diffusion/video generation prompt.
    - `durationEstimate`: Seconds calculated from word count (~140 wpm).
    - `claims`: Array of statements requiring source verification.
  - `callToAction`: `scriptText`, `visualCue`.
- **Validation**: Rejects scripts with missing hooks or scenes without visual prompts.

---

### 4. SEO Agent
- **Purpose**: Optimizes video discoverability, search indexing, and organic click-through rate.
- **Inputs**:
  - `script`: Full screenplay content.
  - `strategy`: Target keyword focus.
- **Outputs** (Zod Enforced):
  - `titles`: Array of 3 distinct high-CTR title variations (Curiosity Gap, Direct Benefit, Question).
  - `description`: Structured YouTube description (Hook paragraph, Timestamps/Chapters, Value summary, Links, Legal disclaimer).
  - `tags`: Array of prioritized comma-separated tags (< 500 characters total).
  - `searchIntentScore`: Score from 0 to 100 evaluating algorithmic discoverability.

---

### 5. Thumbnail Agent
- **Purpose**: Conceptualizes and designs high-CTR, visually striking thumbnails.
- **Inputs**:
  - `script`: Screenplay theme and core tension.
  - `seoTitles`: Recommended titles.
  - `visualStyle`: Channel brand color palette and styling guidelines.
- **Outputs** (Zod Enforced):
  - `concepts`: 3 distinct thumbnail layouts:
    - `prompt`: Image generation prompt optimized for Imagen 3 / FLUX.
    - `overlayText`: Punchy 2-4 word high-contrast text overlay.
    - `focalElement`: Primary visual anchor (face, object, diagram).
    - `colorScheme`: Contrast pair (e.g., Electric Yellow on Dark Slate).
  - `renderedThumbnails`: File paths of composited 1280x720 JPEG/PNG assets.

---

### 6. Narration Agent
- **Purpose**: Produces studio-grade voiceover narration with exact word-level timing and natural prosody.
- **Inputs**:
  - `scenes`: Script scene array.
  - `voiceProfile`: Voice ID, provider (Gemini TTS / ElevenLabs), pitch, speed.
- **Outputs**:
  - `audioFiles`: Per-scene audio files (.wav) and merged mastered narration track.
  - `durations`: Exact measured audio duration per scene (using FFprobe).
  - `wordTimestamps`: Word-level alignment timestamps for subtitle sync.
- **Zero-Faking Guarantee**: Halts with an error if TTS audio is silent, truncated, or unrendered.

---

### 7. Visual Asset Agent
- **Purpose**: Generates high-fidelity visual assets (images, motion graphics, video clips) for every scene.
- **Inputs**:
  - `scenes`: Scene prompts and required aspect ratio (16:9 or 9:16).
  - `visualTheme`: Artistic style tokens (e.g., cinematic photorealism, matte painting).
- **Outputs**:
  - `sceneAssets`: Map of `sceneIndex` to validated image or video asset path on disk.
  - `motionParameters`: Zoom direction, pan coordinates, and Ken Burns speed for static images.
- **Validation**: Verifies files exist, are readable, match aspect ratio dimensions, and contain no corruption.

---

### 8. Video Production Agent
- **Purpose**: Compiles all visual, audio, transition, and caption elements into the final master video using FFmpeg.
- **Inputs**:
  - `sceneAssets`: Images and video clips.
  - `audioTrack`: Master narration track + background music track.
  - `subtitles`: Synchronized subtitle file (.srt).
- **Outputs**:
  - `finalVideoPath`: Master rendered MP4 file (H.264 / AAC).
  - `renderMetrics`: Encoding duration, bitrate, file size, frame count.
- **Security & Safety**: Generates deterministic FFmpeg filter graphs executed exclusively via argument arrays—no shell string evaluation.

---

### 9. Quality Control (QC) Agent
- **Purpose**: Automated gatekeeper executing rigorous audio, visual, and file standards before human presentation.
- **Inputs**:
  - `finalVideoPath`: Output from Video Production Agent.
  - `script`: Original screenplay.
- **Verification Checks**:
  1. **Container & Stream Validation**: FFprobe verification of H.264 video stream, AAC audio stream.
  2. **EBU R128 Audio Normalization**: Verifies Integrated Loudness is within -14.0 ± 1.0 LUFS and True Peak <= -1.0 dBFS.
  3. **Visual Quality**: Black-frame detection (`blackdetect` filter) ensures no unintentional blank pauses > 0.5s.
  4. **Audio Sync Drift**: Confirms audio and video durations match within 200 milliseconds.
  5. **Resolution & Aspect**: Validates exact target dimensions (1920x1080 or 1080x1920) at stable 30 or 60 FPS.
- **Outputs**:
  - `passed`: Boolean.
  - `score`: Overall quality score (0-100).
  - `findings`: Array of warnings or critical blocking errors.

---

### 10. Rights & Provenance Agent
- **Purpose**: Ensures research integrity, tracks content provenance, and flags synthetic media disclosures.
- **Inputs**:
  - `researchSources`: Citations collected during research.
  - `scriptClaims`: Factual assertions made in the script.
  - `generatedAssets`: Logs of AI models used for visuals and voice.
- **Outputs**:
  - `provenanceSummary`: Source-to-claim mapping.
  - `syntheticMediaDisclosure`: Flagged as required (`containsSyntheticMedia = true`) for YouTube compliance.
  - `legalAuditNotice`: Explicit disclaimer stating that AI analysis does not constitute legal copyright clearance.

---

### 11. Approval Agent (Human-in-the-Loop Gatekeeper)
- **Purpose**: Manages the editorial review process, enforcing human authorization before any external publication.
- **Inputs**:
  - Complete production bundle (QC report, preview video, SEO metadata, thumbnails, provenance).
- **Behaviors**:
  - By default, locks pipeline at `AWAITING_APPROVAL`.
  - Presents interactive preview, scene editor, and metadata controls in the web UI.
  - Supports `Approve`, `Request Revisions` (with targeted feedback for specific agents), or `Reject`.
  - Ensures `AUTO_PUBLISH` is strictly `false` unless explicitly overridden by authorized admin credentials.

---

### 12. Publishing Agent
- **Purpose**: Executes bulletproof, idempotent uploads to YouTube using official Google APIs.
- **Inputs**:
  - Approved video file, thumbnail, title, description, tags, schedule time, and privacy setting.
- **Mechanisms**:
  - **HTTP Resumable Upload**: Uses YouTube Resumable Upload protocol with chunked byte transfer and automatic offset query on interruption.
  - **Idempotency Guard**: Stores `uploadSessionUri` and YouTube video ID in SQLite to guarantee retries never duplicate uploads.
  - **Metadata & Thumbnail Sync**: Sets custom thumbnail, captions, and publication schedule via YouTube Data API v3.

---

### 13. Analytics Agent
- **Purpose**: Gathers post-publication performance data directly from YouTube Analytics and Data APIs.
- **Inputs**:
  - YouTube Video ID and Channel ID.
- **Outputs**:
  - `snapshots`: 24h, 7d, 28d performance records (Impressions, Views, CTR, Average View Duration, Retention Curve array, Subscribers Gained, Estimated Revenue).

---

### 14. Learning Agent
- **Purpose**: Identifies algorithmic patterns across video performance to continually optimize future strategies.
- **Inputs**:
  - Historical video analytics snapshots + production attributes (topic, format, hook duration, voice, thumbnail colors).
- **Outputs**:
  - `channelLearnings`: High-confidence statistical correlations (e.g., "Videos starting with a curiosity question have +22% higher 30s retention").
  - `strategyRecommendations`: Actionable modifications fed back into Strategy and Research agents.

---

### 15. Autonomous Operator
- **Purpose**: Master orchestrator governing channel publishing schedules and pipeline cadences.
- **Responsibilities**:
  - Evaluates active channel strategy (e.g. 2 videos/week on Tuesdays and Thursdays).
  - Triggers batch generation runs when pipeline backlog drops below target buffer.
  - Manages operator run lifecycle: `QUEUED` → `RESEARCHING` → `PRODUCING` → `AWAITING_APPROVAL`.
  - Halts execution and emits notifications if human review queues exceed capacity.

---

### 16. Auto-Fix Agent
- **Purpose**: Cross-cutting supervisory agent that classifies errors across all pipeline stages and executes safe, bounded self-healing.
- **Error Taxonomy & Remediations**:
  - `LLM_JSON_PARSE_ERROR`: Re-prompts the model with explicit schema violation details.
  - `AUDIO_CLIPPING`: Re-runs audio normalization with lower target gain.
  - `FFMPEG_DIMENSION_ERROR`: Pads image/video inputs to even dimensions (divisible by 2) automatically.
  - `RATE_LIMIT_EXCEEDED`: Applies exponential backoff with jitter.
  - `TRANSIENT_NETWORK_DROP`: Safely re-attempts HTTP requests with circuit breaker limits (max 3 tries).
