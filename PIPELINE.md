# ContentPilot AI — Pipeline & State Machine Specification

ContentPilot AI executes content generation through an immutable, deterministic, 15-stage pipeline. Every stage transitions via explicit validation gates, records persistent checkpoint artifacts in SQLite, and guarantees zero faking.

---

## 1. End-to-End Pipeline State Flow

```
[1. RESEARCH]
      │ ✓ Opportunities scored & validated via Zod
      ▼
[2. STRATEGY]
      │ ✓ Hook hypothesis, format & length target established
      ▼
[3. SCRIPT]
      │ ✓ Scene blueprints, visual prompts, voiceover text & claims recorded
      ▼
[4. SEO] ─── AND ─── [5. THUMBNAIL]
      │                   │
      ▼                   ▼
      │ ✓ Titles, tags & 3x 1280x720 rendered concepts ready
      └─────────┬─────────┘
                ▼
[6. NARRATION (TTS)]
      │ ✓ Verified non-zero-byte audio tracks & word alignments
      ▼
[7. VISUAL ASSETS]
      │ ✓ Validated 1080p images/clips per scene
      ▼
[8. VIDEO PRODUCTION (FFmpeg)]
      │ ✓ Master video render with dynamic motion & audio ducking
      ▼
[9. QUALITY CONTROL (QC)]
      │ ✓ FFprobe integrity, LUFS loudness (-14 ± 1) & black-frame check
      ▼
[10. RIGHTS & PROVENANCE]
      │ ✓ Synthetic media flag & source-to-claim audit generated
      ▼
[11. HUMAN APPROVAL GATE] ◄── [AUTO_PUBLISH=false default strictly held]
      │
      ├── [REVISION REQUESTED] ──► Rollback only affected scene / metadata
      │
      └── [APPROVED BY HUMAN]
                ▼
[12. YOUTUBE PUBLISHING]
      │ ✓ Idempotent HTTP Resumable upload & metadata sync
      ▼
[13. ANALYTICS SYNC]
      │ ✓ Ingest 24h, 7d, 28d retention curves & CTR
      ▼
[14. CLOSED-LOOP LEARNING]
      │ ✓ Extract correlations & actionable strategy adjustments
      ▼
[15. NEXT STRATEGY / CADENCE]
```

---

## 2. Granular Stage Specifications & Validation Gates

### Stage 1: Research
- **Input**: Channel profile, target keywords, audience persona.
- **Processing**: Research Agent uses Gemini to query search intent, competitor saturation, and content whitespace.
- **Validation Gate**: Output must parse as a non-empty list of structured opportunities matching `ResearchOpportunitySchema` with `opportunityScore >= 50`.
- **Checkpoint Artifact**: `artifacts/research.json` (SHA-256 hashed).

---

### Stage 2: Strategy
- **Input**: Selected research opportunity and channel cadence.
- **Processing**: Strategy Agent defines target runtime, format (explainer, essay, tutorial), hook hypothesis, and pacing.
- **Validation Gate**: Target length must be bounded (e.g. 60-90s for Shorts, 300-900s for Long Form); format must be an approved enum.
- **Checkpoint Artifact**: `artifacts/strategy.json`.

---

### Stage 3: Script
- **Input**: Approved strategy document.
- **Processing**: Script Agent produces structured scene blueprints with exact narration words, visual scene prompts, and factual claims.
- **Validation Gate**:
  - Script must include a designated Hook (<= 15s duration).
  - Every scene must have non-empty `scriptText` and non-empty `visualPrompt`.
  - Estimated total word count must align with strategy target duration within a ±15% window.
- **Checkpoint Artifact**: `artifacts/script.json`.

---

### Stage 4: SEO Optimization
- **Input**: Complete script and target keywords.
- **Processing**: SEO Agent generates 3 distinct high-CTR title variations, full YouTube description, tags list, and timestamps.
- **Validation Gate**:
  - Titles must be <= 100 characters each.
  - Tags must not exceed 500 characters total.
  - Description must include structured chapter timestamps matching scene durations.
- **Checkpoint Artifact**: `artifacts/seo.json`.

---

### Stage 5: Thumbnail Conception & Render
- **Input**: Script hook, selected titles, brand visual guidelines.
- **Processing**: Thumbnail Agent produces 3 visual prompt concepts, invokes Imagen 3 / image generator, and overlays typography via Sharp.
- **Validation Gate**:
  - Image files must be on disk, readable, exactly 1280x720 pixels, and >= 50KB in size.
- **Checkpoint Artifact**: `thumbnails/thumb_01.jpg`, `thumbnails/thumb_02.jpg`.

---

### Stage 6: Narration (Voiceover Synthesis)
- **Input**: Scene blueprints script text, selected voice ID.
- **Processing**: Narration Agent synthesizes per-scene audio using Gemini TTS / ElevenLabs.
- **Validation Gate (Zero-Faking Guarantee)**:
  - Every generated audio file must be verified via FFprobe: duration must be > 0.5s, format must be valid WAV/MP3, and audio must not be pure silence.
  - Silent placeholder generation is strictly prohibited and treated as an unrecoverable stage error.
- **Checkpoint Artifact**: `audio/scene_01.wav`, `audio/master_narration.wav`.

---

### Stage 7: Visual Asset Generation
- **Input**: Per-scene visual prompts and project aspect ratio (16:9 or 9:16).
- **Processing**: Visual Asset Agent generates high-resolution imagery or video clips for each scene.
- **Validation Gate**:
  - Exactly one valid visual file must exist for every scene blueprint.
  - Image dimensions must match target aspect ratio (1920x1080 or 1080x1920).
- **Checkpoint Artifact**: `visuals/scene_01.png`, `visuals/scene_02.mp4`.

---

### Stage 8: Video Production (FFmpeg Assembly)
- **Input**: Visual assets, master narration audio, background music, captions (.srt).
- **Processing**: Video Production Agent composes an FFmpeg complex filter graph:
  - Applies Ken Burns subtle zoom/pan to static images.
  - Ducks background music by -18dB during active speech.
  - Stitches scenes with smooth transitions.
  - Encodes master output as H.264 video with AAC audio in an MP4 container.
- **Validation Gate**:
  - Render process must exit with return code 0. Output file must exist and be non-empty.
- **Checkpoint Artifact**: `video/master_render.mp4`.

---

### Stage 9: Quality Control (QC)
- **Input**: `video/master_render.mp4` and source script.
- **Processing**: QC Agent runs FFprobe and FFmpeg diagnostic filters.
- **Validation Gate (Enforced Checklist)**:
  1. *Loudness*: Integrated loudness must measure between -13.0 and -15.0 LUFS.
  2. *Black Frames*: No black-frame segments exceeding 0.5s permitted.
  3. *Sync Drift*: Audio stream and video stream duration delta must not exceed 200ms.
  4. *Resolution/Framerate*: Must match exact project specification at 30 or 60 FPS.
- **Checkpoint Artifact**: `logs/qc_report.json`.

---

### Stage 10: Rights & Provenance Verification
- **Input**: Cited research URLs, factual claims from Script Agent, asset model logs.
- **Processing**: Compiles source-to-claim attribution audit and flags mandatory YouTube synthetic media disclosures.
- **Validation Gate**:
  - `containsSyntheticMedia` must be set to `true` (for YouTube AI declaration compliance).
  - Provenance summary must be generated with explicit legal disclaimer.
- **Checkpoint Artifact**: `artifacts/provenance.json`.

---

### Stage 11: Human Approval Gate
- **Input**: Fully rendered video, QC report, metadata, and thumbnail previews.
- **Behavior**:
  - System pauses pipeline. State transitions to `AWAITING_APPROVAL`.
  - Emits in-app alert and operational notification.
  - User can inspect video frame-by-frame, edit metadata, or trigger granular scene re-renders.
  - Pipeline cannot proceed to publication without an explicit signed approval payload logged in `AuditLog`.
  - Default `AUTO_PUBLISH` is strictly `false`.

---

### Stage 12: YouTube Publishing
- **Input**: Approved production bundle, Google OAuth credentials.
- **Processing**:
  1. Computes unique `idempotencyKey` from production ID and video hash.
  2. Checks `PublishSchedule` to confirm upload has not already succeeded.
  3. Initiates YouTube Resumable Upload session.
  4. Streams video in 8MB chunks, recording byte offset in SQLite.
  5. If connection breaks, queries resumable URI for `Range` header and resumes from byte offset.
  6. Sets custom thumbnail, scheduled release date, tags, and category.
- **Validation Gate**: YouTube API must return a valid 11-character `videoId`.

---

### Stage 13-15: Analytics, Learning & Next Strategy
- **Scheduled Ingestion**: Polls YouTube Analytics API at 24h, 7d, and 28d intervals.
- **Learning Analysis**: Computes retention dip correlations against scene types and pacing.
- **Strategy Refinement**: Updates `ChannelLearning` knowledge rules and automatically informs the next topic research cycle.

---

## 3. Granular Scene Invalidation & Regeneration Logic

ContentPilot AI prevents wasteful full re-renders when minor revisions occur:

| User Action | Invalidated Stages | Preserved Stages |
| :--- | :--- | :--- |
| **Edit Title or Description** | None (Metadata update only) | All Media & Video Render |
| **Swap Thumbnail Concept** | Stage 5 (Thumbnail only) | All Video, Audio & Script |
| **Regenerate Single Scene Visual** | Stage 7 (Scene N visual), Stage 8 (Video), Stage 9 (QC) | Script, Audio/Narration, SEO, Other Scenes |
| **Re-record Single Scene Audio** | Stage 6 (Scene N audio), Stage 8 (Video), Stage 9 (QC) | Script, Visual Assets, SEO, Other Scenes |
| **Rewrite Entire Script** | Stages 3, 4, 5, 6, 7, 8, 9, 10 | Stage 1 (Research), Stage 2 (Strategy) |
