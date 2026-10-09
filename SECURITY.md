# ContentPilot AI — Security, Compliance & Governance Architecture

Security, privacy, and regulatory governance are foundational requirements in ContentPilot AI. This document outlines the defensive engineering controls, threat mitigations, and compliance protocols enforced across the platform.

---

## 1. Core Security Tenets

1. **Zero Secret Leakage**: No API keys, credentials, or secrets may ever be committed to source code or transmitted to the client frontend bundle.
2. **Defensive Shell Sandboxing**: AI outputs are strictly treated as untrusted data. AI-generated text is **never** passed into an execution shell (`exec`, `eval`, or raw bash/powershell strings).
3. **Mandatory Human Governance**: Automated publishing is disabled by default (`AUTO_PUBLISH=false`). Every video requires explicit human cryptographic sign-off.
4. **Resilient Data Protection**: Refresh tokens and sensitive credentials stored at rest in SQLite are encrypted using authenticated encryption (AES-256-GCM).
5. **No AI Legal Clearance Claims**: AI fact-checking and provenance tools assist human verification but must never legally certify copyright or trademark indemnity.

---

## 2. Threat Modeling & Mitigation Matrix

| Threat | Attack Vector | ContentPilot AI Defensive Control |
| :--- | :--- | :--- |
| **Command Injection** | Malicious prompt injection attempting to run shell commands via FFmpeg or subagent | Strict `execFile` usage with immutable argument arrays; regex validation on file paths; zero raw shell string evaluation. |
| **Frontend Credential Leak** | Developer inadvertently bundling `process.env.GEMINI_API_KEY` into client bundle | Architecture physically separates backend API (`server/`) and frontend (`client/`). Frontend consumes only backend REST/SSE endpoints. Zero direct LLM calls from client. |
| **Duplicate Video Flooding** | Network retry triggering duplicate uploads to YouTube channel | Database-enforced `idempotencyKey` on `PublishSchedule`. Upload session verification queries existing video state before initiating transfer. |
| **Token Theft at Rest** | Local access to SQLite database file exposing YouTube OAuth refresh tokens | Secrets encrypted at rest via AES-256-GCM using an application master key (`APP_SECRET_KEY`) stored outside the database. |
| **Hallucinated Publication** | System claiming a video was published without actual YouTube insertion | Strict verification: Publication is only marked successful when YouTube Data API returns an active, queryable video ID. |
| **Fake Media Generation** | System using placeholder gradients or silence when API keys fail | Zero-Faking Policy: Audio/visual validators fail fast with actionable error logs rather than generating bogus placeholders. |
| **Prompt Injection / Jailbreak** | Adversarial input in research topic inducing harmful script output | Strict system prompts, content safety filters in Gemini, and mandatory human review before publishing. |

---

## 3. Shell Execution & FFmpeg Security Architecture

ContentPilot AI enforces strict guidelines on process execution:

### Prohibited Code Pattern (Unsafe):
```javascript
// FORBIDDEN IN CONTENTPILOT AI
const { exec } = require('child_process');
exec(`ffmpeg -i ${userInput} output.mp4`); // Vulnerable to command injection!
```

### Mandated Code Pattern (Secure):
```typescript
import { execFile } from 'node:child_process';
import { promisify } from 'node:util';

const execFileAsync = promisify(execFile);

// Validated parameter array with no shell interpolation
export async function executeFFmpeg(args: string[]): Promise<{ stdout: string; stderr: string }> {
  // Validate every argument against dangerous shell meta-characters
  for (const arg of args) {
    if (/[\r\n\0]/.test(arg)) {
      throw new SecurityError(`Invalid character detected in FFmpeg parameter: ${arg}`);
    }
  }

  return execFileAsync(getFFmpegPath(), args, {
    maxBuffer: 64 * 1024 * 1024,
    windowsHide: true,
  });
}
```

---

## 4. YouTube OAuth 2.0 & Token Vault

The YouTube integration adheres to Google OAuth 2.0 best practices:

1. **PKCE Flow (Proof Key for Code Exchange)**: Protects the authorization flow against authorization code interception attacks.
2. **Minimal Required Scopes**:
   - `https://www.googleapis.com/auth/youtube.upload` (Video uploads)
   - `https://www.googleapis.com/auth/youtube` (Thumbnail and playlist updates)
   - `https://www.googleapis.com/auth/yt-analytics.readonly` (Performance sync)
3. **At-Rest Token Encryption (AES-256-GCM)**:
   ```typescript
   import { createCipheriv, createDecipheriv, randomBytes } from 'node:crypto';

   export function encryptSecret(plainText: string, masterKey: Buffer): string {
     const iv = randomBytes(12);
     const cipher = createCipheriv('aes-256-gcm', masterKey, iv);
     const encrypted = Buffer.concat([cipher.update(plainText, 'utf8'), cipher.final()]);
     const tag = cipher.getAuthTag();
     return `${iv.toString('hex')}:${tag.toString('hex')}:${encrypted.toString('hex')}`;
   }
   ```

---

## 5. Provenance, Copyright & Synthetic Media Disclosures

### 5.1 YouTube AI Disclosure Compliance
Under YouTube policy, creators must disclose when realistic content is created with altered or synthetic media (including generative AI).
- ContentPilot AI automatically sets `containsSyntheticMedia = true` in the production provenance bundle.
- During publication, the YouTube metadata snippet is flagged with `selfDeclaredMadeForKids = false` and disclosure flags indicating generative media usage.

### 5.2 Strict Legal Disclaimer
> [!CAUTION]
> ContentPilot AI provenance verification verifies that factual statements in scripts are paired with research sources. **It does NOT and CANNOT constitute legal copyright, fair use, or trademark clearance.** Human operators must review content and ensure they possess appropriate rights, licenses, and clearances for all source materials.

---

## 6. Audit Logging & Non-Repudiation

Every state-altering event is written to an immutable `AuditLog` table in SQLite:

- Timestamp (UTC ISO-8601)
- Actor (`system`, `operator_run`, or authenticated user)
- Action (`PIPELINE_INIT`, `SCENE_REVISION`, `APPROVAL_GRANTED`, `APPROVAL_REJECTED`, `YOUTUBE_PUBLISHED`)
- Entity ID & Type
- Payload diff / snapshot

Audit logs cannot be modified or deleted via standard application APIs.
