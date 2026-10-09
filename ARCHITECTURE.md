# ContentPilot AI — System Architecture

```mermaid
graph TD
    Client[React 18 Dashboard] -->|HTTP / REST| API[Express API Gateway]
    API --> SecurityGuard[Security Guard: SSRF / Path Traversal]
    SecurityGuard --> Operators[Autonomous Operator]
    Operators --> StateMachine[15-State Pipeline Machine]
    Operators --> AIProvider[Google Gemini AI Provider]
    Operators --> FFmpeg[FFmpeg Media Engine]
    Operators --> ReviewStudio[Review Studio Gate]
    ReviewStudio -->|Explicit Approval| SafetyChecker[Pre-Upload Safety Check]
    SafetyChecker --> YouTubeEngine[YouTube Upload Engine]
    YouTubeEngine --> YouTube[YouTube Data API v3]
    YouTube --> Analytics[Analytics Ingestion]
    Analytics --> Learning[Learning Agent]
    Learning --> Experiments[Hypothesis Experiments]

    Scheduler[Persistent Cron Scheduler] --> SQLite[(SQLite WAL Database)]
    JobWorker[Persistent Job Worker] --> SQLite
    API --> SQLite
```

---

## 1. Core Architectural Pillars

1. **Deterministic 20-Stage Autonomous Flow**:
   - Every video project transitions through 20 deterministic operational checkpoints:
     `Inspect Channel` -> `Inspect Analytics` -> `Research` -> `Strategy` -> `Select Topic` -> `Script` -> `SEO` -> `Thumbnail` -> `Narration` -> `Visual Assets` -> `Video Assembly` -> `MP4 Integrity` -> `Quality Control (QC)` -> `Rights & Provenance` -> `Send to Review Studio` -> **`MANDATORY HUMAN APPROVAL GATE`** -> `YouTube Upload` -> `Capture Telemetry` -> `Learning` -> `Future Strategy`.
2. **Strict Human Governance Gate**:
   - Step 16 strictly halts autonomous execution until explicit approval is granted in Review Studio.
   - Pre-upload safety checker enforces 5 non-bypassable constraints.
3. **Resilient SQLite Persistence**:
   - All state transitions, jobs, and scheduled tasks are backed by SQLite running in Write-Ahead Logging (`WAL`) mode with foreign key integrity.
4. **Zero-Faking Policy**:
   - ContentPilot AI guarantees that metrics, video files, audio tracks, and citations reflect genuine system operations, never simulated placeholders.
5. **Auto-Fix Self-Healing**:
   - The engine automatically categorizes transient errors into 9 classifications and attempts safe recovery before escalating.
