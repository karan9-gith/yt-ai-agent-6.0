# ContentPilot AI — Database Architecture & Schema Specification

ContentPilot AI uses **SQLite** as its embedded transactional database, managed via **Prisma ORM**. SQLite provides zero-latency local operations, ACID compliance, zero infrastructure overhead, and full support for atomic transactions.

---

## 1. Engine Configuration & Pragmas

To ensure enterprise-grade concurrency and data safety across simultaneous background worker threads and API requests, SQLite is configured with the following PRAGMAs on every connection:

```sql
PRAGMA journal_mode = WAL;          -- Write-Ahead Logging for high read/write concurrency
PRAGMA synchronous = NORMAL;         -- Durability with low disk I/O latency
PRAGMA foreign_keys = ON;            -- Strict foreign key constraint enforcement
PRAGMA busy_timeout = 5000;          -- 5-second wait before SQLITE_BUSY error
PRAGMA cache_size = -64000;          -- 64MB memory cache
```

---

## 2. Entity-Relationship Model

```
┌──────────────────┐       ┌──────────────────────┐
│  ChannelProfile  │──────►│   ChannelStrategy    │
└──────────────────┘       └──────────┬───────────┘
                                      │
                                      ▼
                           ┌──────────────────────┐       ┌──────────────────┐
                           │     OperatorRun      │──────►│   ContentIdea    │
                           └──────────┬───────────┘       └──────────────────┘
                                      │ triggers
                                      ▼
                           ┌──────────────────────┐
                           │      Production      │
                           └──────────┬───────────┘
                                      │
         ┌───────────────┬────────────┼─────────────┬────────────────┐
         ▼               ▼            ▼             ▼                ▼
   ┌───────────┐   ┌───────────┐┌───────────┐ ┌───────────┐    ┌────────────┐
   │Checkpoints│   │  Scenes   ││SeoMetadata│ │Thumbnails │    │ QC Report  │
   └───────────┘   └─────┬─────┘└───────────┘ └───────────┘    └────────────┘
                         │
                         ▼
                   ┌───────────┐
                   │ Revisions │
                   └───────────┘
                         │
         ┌───────────────┴────────────┬──────────────────────────────┐
         ▼                            ▼                              ▼
   ┌───────────┐                ┌───────────┐                  ┌───────────┐
   │Provenance │                │  Review   │                  │ Publishing│
   └───────────┘                └───────────┘                  └─────┬─────┘
                                                                     │
                                                                     ▼
                                                               ┌───────────┐
                                                               │ Analytics │
                                                               └───────────┘

Cross-Cutting Infrastructure Entities:
- JobQueue (Persistent worker tasks with crash recovery)
- AuditLog (Immutable security and action ledger)
- SystemSetting (Encrypted configuration and credentials)
- Notification (In-app notifications and alerts)
```

---

## 3. Prisma Schema Definition (`schema.prisma`)

```prisma
datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

// ==========================================
// CHANNEL PROFILE & STRATEGY
// ==========================================

model ChannelProfile {
  id             String            @id @default(cuid())
  channelName    String
  niche          String
  targetAudience String
  brandVoice     String
  forbiddenTopics String           @default("[]") // JSON Array
  visualStyle    String
  timezone       String            @default("UTC")
  createdAt      DateTime          @default(now())
  updatedAt      DateTime          @updatedAt
  strategies     ChannelStrategy[]
}

model ChannelStrategy {
  id              String         @id @default(cuid())
  profileId       String
  profile         ChannelProfile @relation(fields: [profileId], references: [id], onDelete: Cascade)
  objective       String
  targetCadence   Int            @default(2) // Videos per week
  defaultFormat   String         @default("explainer") // explainer, documentary, short, tutorial
  targetLengthSec Int            @default(480) // 8 minutes default
  primaryKpi      String         @default("views")
  contentPillars  String         @default("[]") // JSON Array
  status          String         @default("draft") // draft, active, archived
  createdAt       DateTime       @default(now())
  updatedAt       DateTime       @updatedAt
  operatorRuns    OperatorRun[]
  productions     Production[]
}

model ContentIdea {
  id               String       @id @default(cuid())
  topic            String
  angle            String?
  opportunityScore Float        @default(0.0)
  searchVolume     String?
  competition      String?
  status           String       @default("backlog") // backlog, planned, discarded, produced
  sourceData       String       @default("{}") // JSON metadata
  createdAt        DateTime     @default(now())
  updatedAt        DateTime     @updatedAt
  productions      Production[]
}

// ==========================================
// AUTONOMOUS OPERATOR & PRODUCTION RUNS
// ==========================================

model OperatorRun {
  id          String          @id @default(cuid())
  strategyId  String
  strategy    ChannelStrategy @relation(fields: [strategyId], references: [id])
  status      String          @default("queued") // queued, running, completed, paused, failed
  stage       String          @default("planning")
  progress    Int             @default(0) // 0 - 100
  planData    String          @default("[]") // JSON plan items
  error       String?
  startedAt   DateTime        @default(now())
  completedAt DateTime?
  productions Production[]
}

model Production {
  id             String                     @id @default(cuid())
  strategyId     String?
  strategy       ChannelStrategy?           @relation(fields: [strategyId], references: [id])
  operatorRunId  String?
  operatorRun    OperatorRun?               @relation(fields: [operatorRunId], references: [id])
  ideaId         String?
  idea           ContentIdea?               @relation(fields: [ideaId], references: [id])
  title          String
  status         String                     @default("draft") 
  // Status flow: draft, researching, scripting, producing, qc_verifying, awaiting_approval, approved, publishing, published, failed
  format         String                     @default("long_form") // long_form, short
  targetDuration Float                      @default(480.0)
  actualDuration Float?
  outputPath     String?
  createdAt      DateTime                   @default(now())
  updatedAt      DateTime                   @updatedAt

  checkpoints    ProductionStageCheckpoint[]
  scenes         ProductionScene[]
  seo            SeoMetadata?
  thumbnails     ThumbnailVariant[]
  qcReport       QualityControlReport?
  provenance     ContentProvenance?
  review         ContentReview?
  publishRecord  PublishSchedule?
  snapshots      PerformanceSnapshot[]
}

model ProductionStageCheckpoint {
  id           String     @id @default(cuid())
  productionId String
  production   Production @relation(fields: [productionId], references: [id], onDelete: Cascade)
  stage        String     // research, strategy, script, seo, thumbnail, narration, visual, production, qc
  status       String     @default("completed") // pending, running, completed, invalid
  artifactHash String     // SHA-256 hash of artifact payload for integrity
  artifactJson String     // Full serializable output of the stage
  error        String?
  createdAt    DateTime   @default(now())
  completedAt  DateTime?

  @@unique([productionId, stage])
}

// ==========================================
// SCENE MANIFEST & REVISIONS
// ==========================================

model ProductionScene {
  id              String                    @id @default(cuid())
  productionId    String
  production      Production                @relation(fields: [productionId], references: [id], onDelete: Cascade)
  position        Int
  label           String
  scriptText      String
  visualPrompt    String
  durationSec     Float                     @default(5.0)
  assetType       String                    @default("missing") // missing, image, video
  assetPath       String?
  audioPath       String?
  narrationStatus String                    @default("pending") // pending, generated, error
  status          String                    @default("draft") // draft, ready, regenerating, error
  createdAt       DateTime                  @default(now())
  updatedAt       DateTime                  @updatedAt
  revisions       ProductionSceneRevision[]

  @@unique([productionId, position])
}

model ProductionSceneRevision {
  id           String          @id @default(cuid())
  sceneId      String
  scene        ProductionScene @relation(fields: [sceneId], references: [id], onDelete: Cascade)
  action       String          // regenerate_visual, rewrite_script, re_record_audio
  beforeState  String          // JSON snapshot
  afterState   String          // JSON snapshot
  createdAt    DateTime        @default(now())
}

// ==========================================
// METADATA, ASSETS & QC
// ==========================================

model SeoMetadata {
  id                String     @id @default(cuid())
  productionId      String     @unique
  production        Production @relation(fields: [productionId], references: [id], onDelete: Cascade)
  titleVariants     String     // JSON Array of title options
  selectedTitle     String
  description       String
  tags              String     // JSON Array of tags
  searchIntentScore Float      @default(0)
  updatedAt         DateTime   @updatedAt
}

model ThumbnailVariant {
  id           String     @id @default(cuid())
  productionId String
  production   Production @relation(fields: [productionId], references: [id], onDelete: Cascade)
  conceptPrompt String
  overlayText  String
  assetPath    String
  isSelected   Boolean    @default(false)
  createdAt    DateTime   @default(now())
}

model QualityControlReport {
  id            String     @id @default(cuid())
  productionId  String     @unique
  production    Production @relation(fields: [productionId], references: [id], onDelete: Cascade)
  passed        Boolean    @default(false)
  overallScore  Float      @default(0)
  loudnessLufs  Float
  durationDrift Float      // Audio duration vs Video duration delta in seconds
  findingsJson  String     @default("[]") // Detailed QC findings
  verifiedAt    DateTime   @default(now())
}

model ContentProvenance {
  id                         String     @id @default(cuid())
  productionId               String     @unique
  production                 Production @relation(fields: [productionId], references: [id], onDelete: Cascade)
  sourcesJson                String     @default("[]") // Cited sources & URLs
  claimsJson                 String     @default("[]") // Factual statements checked
  containsSyntheticMedia     Boolean    @default(true) // Required for YouTube AI disclosure
  legalNoticeAcknowledged    Boolean    @default(false)
  updatedAt                  DateTime   @updatedAt
}

// ==========================================
// HUMAN APPROVAL & PUBLISHING
// ==========================================

model ContentReview {
  id           String     @id @default(cuid())
  productionId String     @unique
  production   Production @relation(fields: [productionId], references: [id], onDelete: Cascade)
  status       String     @default("needs_review") // needs_review, approved, changes_requested, rejected
  reviewNotes  String?
  reviewedBy   String?
  reviewedAt   DateTime?
  updatedAt    DateTime   @updatedAt
}

model PublishSchedule {
  id               String     @id @default(cuid())
  productionId     String     @unique
  production       Production @relation(fields: [productionId], references: [id], onDelete: Cascade)
  youtubeVideoId   String?
  youtubeUrl       String?
  scheduledPublish DateTime?
  privacyStatus    String     @default("private") // private, unlisted, public
  uploadSessionUri String?    // Resumable upload session URI
  bytesUploaded    Int        @default(0)
  status           String     @default("queued") // queued, uploading, published, failed, reconciliation_required
  idempotencyKey   String     @unique
  error            String?
  publishedAt      DateTime?
  createdAt        DateTime   @default(now())
  updatedAt        DateTime   @updatedAt
}

// ==========================================
// ANALYTICS & LEARNING
// ==========================================

model PerformanceSnapshot {
  id                String     @id @default(cuid())
  productionId      String
  production        Production @relation(fields: [productionId], references: [id])
  youtubeVideoId    String
  measurementWindow String     // 24h, 7d, 28d, lifetime
  views             Int        @default(0)
  impressions       Int        @default(0)
  ctr               Float      @default(0.0)
  averageViewDuration Float    @default(0.0)
  retentionCurveJson String    @default("[]") // Seconds vs percentage
  subscribersGained Int        @default(0)
  capturedAt        DateTime   @default(now())

  @@index([youtubeVideoId, measurementWindow])
}

model ChannelLearning {
  id             String   @id @default(cuid())
  category       String   // hook, pacing, thumbnail, topic
  finding        String
  confidence     Float    @default(0.0)
  impactScore    Float    @default(0.0)
  actionRuleJson String   // Actionable instruction for agents
  createdAt      DateTime @default(now())
}

// ==========================================
// PERSISTENT JOB QUEUE & AUDIT SYSTEM
// ==========================================

model JobQueue {
  id             String    @id @default(cuid())
  type           String    // agent_task, video_render, youtube_upload, analytics_sync
  payloadJson    String
  status         String    @default("pending") // pending, running, completed, failed, cancelled
  priority       Int       @default(0) // Higher = faster execution
  attempts       Int       @default(0)
  maxAttempts    Int       @default(3)
  lastError      String?
  lockedBy       String?   // Worker ID
  lastHeartbeat  DateTime?
  scheduledFor   DateTime  @default(now())
  completedAt    DateTime?
  createdAt      DateTime  @default(now())
  updatedAt      DateTime  @updatedAt

  @@index([status, scheduledFor, priority])
}

model AuditLog {
  id         String   @id @default(cuid())
  entityType String
  entityId   String
  action     String   // CREATE, UPDATE, APPROVE, PUBLISH, ROLLBACK
  actor      String   @default("system")
  payload    String   @default("{}")
  createdAt  DateTime @default(now())

  @@index([entityType, entityId])
}

model Notification {
  id        String   @id @default(cuid())
  level     String   @default("info") // info, warning, error, action_required
  title     String
  message   String
  dataJson  String?
  isRead    Boolean  @default(false)
  createdAt DateTime @default(now())
}

model SystemSetting {
  key         String   @id
  value       String   // Encrypted if sensitive
  isSecret    Boolean  @default(false)
  description String?
  updatedAt   DateTime @updatedAt
}
```

---

## 4. Crash Recovery & Job Queue Operational Semantics

The persistent `JobQueue` uses optimistic locking with heartbeat detection:

1. **Worker Acquisition**:
   ```sql
   UPDATE JobQueue 
   SET status = 'running', lockedBy = :workerId, lastHeartbeat = CURRENT_TIMESTAMP
   WHERE id = (
     SELECT id FROM JobQueue 
     WHERE status = 'pending' AND scheduledFor <= CURRENT_TIMESTAMP
     ORDER BY priority DESC, createdAt ASC
     LIMIT 1
   );
   ```
2. **Crash Recovery Sweep** (executed every 30s):
   ```sql
   UPDATE JobQueue
   SET status = 'pending', lockedBy = NULL, attempts = attempts + 1
   WHERE status = 'running' 
     AND lastHeartbeat < datetime('now', '-2 minutes')
     AND attempts < maxAttempts;
   ```
3. **Dead-Letter Classification**:
   Jobs exceeding `maxAttempts` are marked `failed`, logging an audit record and triggering an actionable notification in the dashboard.
