# IRdeo — Operations Specification
**Version:** 1.0
**Last updated:** May 11, 2026
**Status:** LIVING DOCUMENT — operational concerns for MVP and SaaS phases
**Companion documents:** `IRdeo_Knowledge_Base.md` (v1.3+), `IRdeo_BRD.md` (v1.4+), `IRdeo_Architecture.md` (v1.2+), `IRdeo_UI_Spec.md` (v1.0+), `IRdeo_Data_Model.md` (v1.0+), `IRdeo_API_Integration.md` (v1.0+), `IRdeo_Output_Spec.md` (v1.0+)

### Versioning Convention
- **v1.0** — first complete formal version
- **v1.1, v1.2, ...** — substantive revisions and additions
- **v2.0** — major restructure or fundamental approach change

### Changelog
- **v1.0 (May 11, 2026)** — First complete formal version. Covers MVP operational concerns (local deployment, logging, doctor diagnostics, backup discipline, key rotation) AND SaaS Phase 2 operational concerns (cloud deployment, monitoring, scaling, security operations, support tooling, cost management). MVP sections are concrete and implementable. SaaS sections are forward-looking — concrete decisions deferred to Phase 2 when SaaS infrastructure is chosen. Built on top of all six prior specs.

---

## 1. Document Overview

### 1.1 Purpose
This document covers operational concerns that don't fit neatly into Architecture, BRD, or API Integration: deployment, monitoring, logging, backup, scaling, security operations, support tooling, and cost management at scale. It's the "how to run IRdeo in production" document.

The doc has two distinct scopes:

- **MVP operations (concrete):** How Rasti runs IRdeo locally. Validated patterns, specific commands, real procedures.
- **SaaS operations (forward-looking):** How a hypothetical Phase 2 SaaS would operate. Architectural patterns, deferred decisions where reasonable, infrastructure-agnostic where possible.

This split keeps the doc useful TODAY (MVP) while preserving forward-looking thinking for WHEN Phase 2 happens (SaaS).

### 1.2 Scope

**In scope:**
- MVP: deployment, logging, diagnostics, backup, key rotation, troubleshooting
- SaaS Phase 2: deployment topology, monitoring/alerting, scaling, security operations, support tooling, cost controls

**Out of scope:**
- Provider-side operational concerns (Anthropic, OpenAI, fal.ai, Lalal.ai — they handle their own ops)
- End-user training (covered in help drawer per UI Spec)
- Customer success / sales / marketing operations

### 1.3 Design Principles for Operations

**P1 — Observable by default.** Every operation logs structured events. Failures produce actionable messages. State changes are traceable. The user (MVP) or operator (SaaS) should never have to wonder "what's it doing right now?"

**P2 — Self-diagnostic.** IRdeo can introspect its own health. `irdeo doctor` (MVP) and equivalent admin tools (SaaS) report the state of every dependency.

**P3 — Backup discipline.** User data should never be lost to a single failure. MVP relies on user filesystem hygiene; SaaS adds redundant cloud storage and point-in-time recovery.

**P4 — Graceful degradation.** When something breaks, IRdeo surfaces the problem clearly and continues working on what's still possible. Generation fails for chunk 4 → user can still work on chunks 1-3 and 5-9.

**P5 — Cost transparency.** Both user (MVP) and operator (SaaS) always know what's been spent and what's projected. Surprises are not allowed.

### 1.4 Related Documents
- `IRdeo_Architecture.md` — System components, deployment topology (Section 17)
- `IRdeo_API_Integration.md` — Provider relationships, key management
- `IRdeo_BRD.md` — Phase 2 SaaS scope (Section 6.2)
- `IRdeo_Data_Model.md` — Backup-relevant file layouts

---

# PART 1 — MVP OPERATIONS

## 2. Local Deployment

### 2.1 System Prerequisites

Per Architecture Section 21.2:

- **Python 3.11+** — runtime
- **ffmpeg on PATH** — preview stitching (Output Spec Section 6)
- **Modern browser** — Chrome, Firefox, Safari, Edge (last 2 major versions)
- **Disk space** — 5GB minimum (active project) + ongoing accumulation for completed projects (~500MB-2GB per song)
- **Network connectivity** — to Anthropic, OpenAI, fal.ai, Lalal.ai endpoints

### 2.2 Installation

```bash
# Clone the repo
git clone https://github.com/{rasti}/irdeo.git
cd irdeo

# Create virtual environment
python -m venv .venv
source .venv/bin/activate   # macOS/Linux
.venv\Scripts\activate      # Windows

# Install dependencies
pip install -e .            # editable install for development
# or
pip install irdeo           # when published to PyPI (post-MVP)

# Initial setup
irdeo init                  # creates ~/.irdeo/ directory, default settings.json
irdeo doctor                # diagnoses missing dependencies
```

### 2.3 First Run

```bash
irdeo start
```

This command:
1. Validates ffmpeg is on PATH
2. Validates settings file exists at `~/.irdeo/settings.json`
3. Validates `IRdeo_Projects` output directory is writable
4. Starts FastAPI/uvicorn on `localhost:8000` (configurable via `--port`)
5. Opens the user's default browser to `http://localhost:8000`
6. Logs server output to console + `~/.irdeo/logs/irdeo.log`

If any validation fails, IRdeo prints actionable error messages and exits before starting the server. Example:

```
ERROR: ffmpeg not found on PATH.

IRdeo requires ffmpeg for preview generation.

Install instructions:
  macOS:   brew install ffmpeg
  Ubuntu:  sudo apt install ffmpeg
  Windows: winget install ffmpeg (or download from ffmpeg.org)

After installing, verify with: ffmpeg -version
Then re-run: irdeo start
```

### 2.4 Shutdown

```bash
# Graceful shutdown (in another terminal or via UI menu)
irdeo stop

# Or just Ctrl+C in the terminal running irdeo start
```

Graceful shutdown:
1. Finishes the currently-streaming chat response (if any)
2. Marks active background jobs as `cancelled` (so they don't appear as orphaned `running` on next start)
3. Closes WebSocket connections cleanly
4. Persists in-memory state to disk
5. Stops uvicorn

Force quit (Ctrl+C twice) skips the cleanup. The startup logic detects orphaned `running` jobs and marks them as `failed` (Architecture Section 15.4).

### 2.5 Running Multiple Instances

IRdeo MVP is single-user, single-instance per machine. Running `irdeo start` while another instance is running:

- Detects the existing instance on port 8000
- Prints "IRdeo is already running. Visit http://localhost:8000"
- Does not start a second instance

For dev work needing multiple instances, use `--port` to choose a different port:

```bash
irdeo start --port 8001   # second instance on alternate port
```

## 3. The `irdeo doctor` Command

The diagnostic Swiss army knife. Run when something feels wrong.

### 3.1 What It Checks

```bash
$ irdeo doctor

IRdeo Doctor — Diagnostic Report

System Prerequisites
[✓] Python 3.11.4
[✓] ffmpeg 6.0 (at /usr/local/bin/ffmpeg)
[✓] Output directory writable: /Users/rasti/IRdeo_Projects
[✓] Settings file exists: /Users/rasti/.irdeo/settings.json
[✓] Settings file permissions: 0600 (correct)
[✓] Encryption key file permissions: 0600 (correct)
[✓] Free disk space: 87 GB available

API Keys
[✓] Anthropic API key configured (sk-ant-...4f8a)
[✓] Anthropic API: reachable, key valid
[✓] OpenAI API key configured (sk-...92ce)
[✓] OpenAI API: reachable, key valid
[✓] fal.ai API key configured (fal_...d3b1)
[?] fal.ai API: format valid, full check deferred
[✓] Lalal.ai API key configured (license-...8c4f)
[✓] Lalal.ai API: reachable, 87.5 minutes credits remaining

Background Jobs
[✓] No orphaned running jobs
[!] 2 failed jobs in last 7 days — review with: irdeo jobs --failed

Projects
[✓] 13 projects in index
[✓] Index consistent with filesystem (0 missing, 0 extra)
[!] 1 project has incomplete state.json (project_id p_3f9a82b1)
     Run: irdeo repair p_3f9a82b1

Backups
[!] No automated backups configured
     Recommend: set up time-based backups for ~/IRdeo_Projects

Logs
[✓] Log file size: 4.2 MB (rotates at 50 MB)
[✓] No unredacted credentials in logs (last 1000 lines scanned)

Performance
[✓] Conversation budget threshold: 120000 tokens (default)
[✓] fal.ai max concurrent: 3 (default)

OVERALL STATUS: ✓ Healthy (3 advisory items)
```

### 3.2 Levels of Output

- **[✓]** — All good, nothing to do
- **[?]** — Indeterminate (full check skipped, not a failure)
- **[!]** — Advisory; non-critical but worth attention
- **[✗]** — Critical failure; something is broken

If any `[✗]` items exist, `irdeo doctor` exits with non-zero status (useful for scripted health checks).

### 3.3 Subcommands

```bash
irdeo doctor --quick          # skip API key validation (no network calls)
irdeo doctor --fix            # attempt auto-repair for [!] items
irdeo doctor --json           # machine-readable output for tooling
```

### 3.4 Implementation Notes

`doctor` is a CLI-only command (not exposed via the web UI) because:
- It runs before the server can start (validates whether starting is possible)
- It's a power-user / troubleshooting tool, not part of the conversation workflow
- It can call out to providers and report results without requiring an active session

## 4. Logging

### 4.1 Log Locations

```
~/.irdeo/logs/
├── irdeo.log            ← current log file
├── irdeo.2026-05-10.log ← yesterday's rotated log
├── irdeo.2026-05-09.log
└── ...
```

### 4.2 Rotation Policy

- **Trigger:** Daily rotation OR file size > 50 MB (whichever comes first)
- **Retention:** Last 30 days; older logs deleted
- **Compression:** Logs older than 1 day compressed with gzip

```
~/.irdeo/logs/
├── irdeo.log            ← uncompressed, current
├── irdeo.2026-05-10.log ← uncompressed, yesterday
├── irdeo.2026-05-09.log.gz   ← compressed, 2 days old
├── irdeo.2026-05-08.log.gz
└── ...
```

### 4.3 Log Format

Structured JSON (per Architecture Section 16.1, via `structlog`):

```jsonl
{"timestamp": "2026-05-11T09:02:11.402Z", "level": "info", "event": "job_started", "job_id": "j_2c9e1d4f", "kind": "generate_song", "project_id": "p_8f3a2b1c"}
{"timestamp": "2026-05-11T09:02:24.108Z", "level": "info", "event": "subchunk_started", "subchunk_id": "01_a", "model": "kling_motion_pro"}
{"timestamp": "2026-05-11T09:03:24.108Z", "level": "info", "event": "subchunk_completed", "subchunk_id": "01_a", "duration_ms": 82888, "cost_usd": 0.32}
{"timestamp": "2026-05-11T09:05:11.402Z", "level": "warning", "event": "fal_rate_limit", "subchunk_id": "01_b", "retry_after_seconds": 8}
{"timestamp": "2026-05-11T09:05:19.401Z", "level": "info", "event": "fal_retry", "subchunk_id": "01_b", "attempt": 2}
```

Console output during `irdeo start` is human-readable; log files are JSON for tooling.

### 4.4 Security: Redaction

Per API Integration Spec Section 3.4, the `SafeLogger` wrapper redacts any text matching credential patterns before writing. Periodic doctor check verifies no unredacted credentials are present in logs (regex scan over the last N lines).

### 4.5 Inspecting Logs

```bash
irdeo logs                    # tail current log
irdeo logs --json             # raw JSON output
irdeo logs --since="1h ago"   # filter by time
irdeo logs --level=error      # filter by level
irdeo logs --grep "subchunk"  # text search
```

## 5. Backup & Recovery

### 5.1 What Needs Backing Up

Per Data Model Spec Section 2:

| Data | Importance | Recovery Cost If Lost |
|------|------------|----------------------|
| `~/IRdeo_Projects/{project_id}/source/original.mp3` | HIGH if not elsewhere | Re-upload from user's archive |
| `~/IRdeo_Projects/{project_id}/timestamps.md` | MEDIUM | Re-generate (Whisper + manual correction) |
| `~/IRdeo_Projects/{project_id}/chunks.json` | HIGH | Manually rebuild from conversation |
| `~/IRdeo_Projects/{project_id}/conversation.jsonl` | MEDIUM | Lost forever; reconstruct context manually |
| `~/IRdeo_Projects/{project_id}/chunks/**/*.mp4` | HIGH | Re-generate ($15-25 per song re-run) |
| `~/IRdeo_Projects/{project_id}/output/*.docx` | LOW | Regenerate from other state |
| `~/.irdeo/settings.json` (encrypted) | HIGH | Rotate all API keys and re-enter |
| `~/.irdeo/.key` (Fernet encryption key) | CRITICAL | Without it, settings.json is unreadable |

### 5.2 MVP Backup Strategy

IRdeo does NOT implement automated backups in MVP. The user is responsible for backing up `~/IRdeo_Projects/` and `~/.irdeo/` via their existing tooling:

- **Time Machine** (macOS) — automatic
- **File History** (Windows) — automatic
- **Backblaze / Carbonite** — automatic
- **rsync to external drive** — manual but reliable
- **Cloud sync** (Dropbox, iCloud Drive, Google Drive) — automatic but watch for sync issues with locked files

The `doctor` command surfaces an advisory `[!] No automated backups configured` if it can't detect a backup tool active on the user's `~/IRdeo_Projects` directory.

### 5.3 Backup Discipline

Recommend to user:
- Back up BEFORE major destructive operations (delete project, migrate schema, regenerate large batches)
- Test recovery occasionally (delete a backed-up project, restore from backup, verify it works)
- Keep at least one off-machine backup (external drive, cloud) for disaster scenarios

### 5.4 In-Process Backups

For migrations and destructive operations within IRdeo itself, `atomic_write` and `backup_before_modify` patterns from Data Model Spec Section 19 create local backups in `.backups/` subdirectories. These are NOT a substitute for user-level backups — they protect against IRdeo bugs, not user data loss scenarios.

### 5.5 Recovery Procedures

**Project corrupted / state.json invalid:**
```bash
irdeo repair {project_id}
```
This reconstructs `state.json` by scanning the project directory for what exists. Conversation history, chunk manifest, and clips are preserved; only the top-level state is rebuilt.

**Settings file lost or corrupted:**
```bash
irdeo init --reset-settings
```
This prompts for fresh API keys and rebuilds `settings.json`. Existing projects are unaffected.

**Fernet encryption key lost:**
This is unrecoverable — settings.json cannot be decrypted without the key. Recovery:
1. Delete the corrupted `settings.json` and `.key`
2. Run `irdeo init --reset-settings`
3. Re-enter all API keys (you may need to look them up in provider dashboards)

**Project clips deleted but state intact:**
The user can regenerate any chunk via the UI. Job orchestrator handles this gracefully.

## 6. Key Rotation (MVP)

### 6.1 When to Rotate

- **Scheduled:** every 6-12 months (defensive hygiene)
- **Suspected compromise:** immediately
- **Provider-initiated:** when provider notifies you of a security incident
- **Personnel changes:** if you've shared the key with someone who shouldn't have access (collaborators, contractors who left)

### 6.2 Rotation Procedure

1. **At the provider:** create a new API key (the old one is still active)
2. **At IRdeo:** open Settings drawer → update the relevant API key → click Test → click Save
3. **At the provider:** revoke the old API key (now that the new one is confirmed working)

The order matters — create new BEFORE revoking old, so there's no downtime if the new key has a problem.

### 6.3 Logging Rotation Events

Every key save event logs:
```json
{"event": "settings_key_rotated", "service": "anthropic", "timestamp": "...", "previous_key_redacted": "sk-ant-...4f8a", "new_key_redacted": "sk-ant-...7c3e"}
```

Useful for audit trail. The user can review past rotations with `irdeo logs --grep settings_key_rotated`.

## 7. Troubleshooting Common Issues (MVP)

### 7.1 "Generation failed for chunk X"

Most common cause: transient fal.ai issue (rate limit, server error). The retry policy handles most cases automatically. If the failure persists:

1. Check `irdeo logs --grep chunk_id={X}` for the specific error
2. Common errors:
   - **Content policy rejection** — prompt was flagged; edit the chunk's prompt
   - **Quota exhausted** — top up fal.ai account
   - **Reference photo issues** — try different photos or remove the photo set
3. Retry via the UI's "Regenerate chunk" button

### 7.2 "Cost is higher than expected"

1. Open cost breakdown popover (UI Spec Section 3.5)
2. Compare actual vs. estimated costs per service
3. Common causes of overrun:
   - Many chunks set to Veo 3 (premium model)
   - Long performer chunks driving high lip sync cost
   - Many regenerations on bad chunks
4. Mitigations:
   - Switch chunks from Veo to Kling where quality is acceptable
   - Use Kling O3 Pro for non-performer chunks (cheaper than Kling Motion Pro)
   - Use lipsync-2 (standard) instead of lipsync-2-pro for less critical chunks

### 7.3 "Preview build failed"

Most common: ffmpeg codec mismatch across clips.

1. Check log for the specific error
2. PreviewService should auto-fall back to re-encoding (Output Spec Section 6.4); if that also failed:
   - Check ffmpeg version (`ffmpeg -version`) — need 4.0 or newer
   - Try manual rebuild: `irdeo preview rebuild {project_id} --force-reencode`
3. If still failing: open each clip in a media player to verify they're not corrupted

### 7.4 "Filmora project file won't open"

1. Verify Filmora version is current (older versions may not support newer FCPXML)
2. Try the FCP XML fallback if you got `.wfp`:
   ```bash
   irdeo export {project_id} --format=fcpxml
   ```
3. If FCPXML also fails: use the manual import workflow (Output Spec Section 10)

### 7.5 "I can't log into Settings — API keys disappeared"

Settings file may be corrupted. Run:
```bash
irdeo doctor
```
If settings integrity check fails, follow the recovery procedure (Section 5.5).

### 7.6 General Diagnostic Flow

When any unfamiliar issue arises:
1. `irdeo doctor` — diagnose overall state
2. `irdeo logs --since="10m ago"` — recent log context
3. Identify the specific error message
4. Search the Help drawer (in UI) or this Troubleshooting section
5. If still stuck: file an issue with the relevant log lines redacted of any keys

---

# PART 2 — SAAS PHASE 2 OPERATIONS

This part addresses operational concerns when IRdeo becomes a hosted multi-tenant service. Decisions here are forward-looking — concrete infrastructure choices are deferred to when SaaS is actually being built. The patterns are infrastructure-agnostic where possible.

## 8. SaaS Deployment Architecture

### 8.1 High-Level Topology

```
┌──────────────────────────────────────────────────────────────────────┐
│                          INTERNET                                    │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                          ┌──────▼──────┐
                          │  HTTPS LB   │  (Cloudflare / AWS ALB / GCP LB)
                          │  TLS term.  │
                          └──────┬──────┘
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
       ┌────▼─────┐        ┌─────▼────┐         ┌────▼──────┐
       │ FastAPI  │        │ FastAPI  │         │ FastAPI   │
       │ Worker 1 │        │ Worker 2 │         │ Worker N  │
       └────┬─────┘        └─────┬────┘         └────┬──────┘
            │                    │                    │
            └────────────────────┼────────────────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
       ┌────▼──────┐       ┌─────▼─────┐        ┌────▼──────┐
       │  Redis    │       │ PostgreSQL│        │  Object   │
       │  (queue)  │       │ (metadata)│        │  Storage  │
       └────┬──────┘       └───────────┘        │  (S3/GCS) │
            │                                    └───────────┘
            │
       ┌────▼──────────────────────────────────┐
       │   Celery / RQ Workers (job runners)   │
       │   ─ generate_song                     │
       │   ─ build_preview                     │
       │   ─ export_prompt_pack                │
       │   ─ export                            │
       └───────────────────────────────────────┘

                       ┌──────────────────┐
                       │  Secrets Vault   │
                       │  (per-user keys) │
                       └──────────────────┘

                       ┌──────────────────┐
                       │  Observability   │
                       │  (Datadog/etc.)  │
                       └──────────────────┘
```

### 8.2 Component Responsibilities

| Component | Responsibility | TBD During SaaS Build |
|-----------|----------------|-----------------------|
| Load Balancer | TLS termination, request routing | Provider choice (AWS ALB, Cloudflare, GCP LB) |
| FastAPI Workers | HTTP/WebSocket handling, stateless | Container orchestrator (Kubernetes, ECS, Cloud Run) |
| Redis | Job queue + WebSocket pub/sub | Managed (ElastiCache, Memorystore) vs self-hosted |
| PostgreSQL | User accounts, billing, project metadata | Managed (RDS, Cloud SQL, Aurora) |
| Object Storage | Project files (clips, MP3s, etc.) | S3 vs GCS vs Azure Blob (cost/region driven) |
| Job Workers | Long-running generation tasks | Celery vs RQ vs Dramatiq (Python ecosystem) |
| Secrets Vault | Per-user API keys | AWS Secrets Manager vs HashiCorp Vault vs GCP Secret Manager |
| Observability | Logs, metrics, traces, alerts | Datadog, New Relic, Grafana stack, or cloud-native |

### 8.3 What Stays From MVP

Critically — the application code stays the same. The `Architecture.md` Section 17.2 config switch selects local vs cloud backends:

```python
class Config:
    DEPLOYMENT_MODE = os.getenv("IRDEO_MODE", "local")  # "local" | "saas"

    @property
    def project_store(self):
        if self.DEPLOYMENT_MODE == "local":
            return LocalProjectStore(Path.home() / "IRdeo_Projects")
        return CloudProjectStore(s3_bucket=os.getenv("S3_BUCKET"))

    @property
    def settings_store(self):
        if self.DEPLOYMENT_MODE == "local":
            return LocalSettingsStore()
        return VaultedSettingsStore(vault_client=secrets_manager_client)

    @property
    def job_orchestrator(self):
        if self.DEPLOYMENT_MODE == "local":
            return InProcessJobOrchestrator()
        return CeleryJobOrchestrator(redis_url=os.getenv("REDIS_URL"))
```

Every code path that worked for MVP works for SaaS with different store/orchestrator backends. This is the architectural payoff of the abstraction discipline.

## 9. Monitoring & Alerting (SaaS)

### 9.1 What to Monitor

| Category | Specific Metrics |
|----------|------------------|
| **Application health** | Worker uptime, request latency p50/p95/p99, error rate, queue depth |
| **External APIs** | Per-provider response time, error rate, quota usage (where queryable) |
| **Database** | Query latency, connection pool saturation, replication lag |
| **Object storage** | Latency, throughput, request errors, bytes stored |
| **Jobs** | Active job count by kind, job duration p50/p95/p99, failure rate by kind |
| **Cost** | Per-user spend rate, total platform cost burn, ratio of provider costs to revenue |
| **Security** | Failed auth attempts, API key rotation events, vault access patterns |
| **Business** | Active users, conversion rate (signup → first generation), session length |

### 9.2 Alert Tiers

**Critical (page someone immediately):**
- All FastAPI workers down
- Database unreachable
- Object storage unreachable
- Job queue depth > threshold (10× expected)
- Vault unreachable
- Cost burn rate > 2× projected for sustained period

**Warning (email/Slack, no page):**
- Single worker degraded
- Provider error rate > 5%
- Replication lag > 30 seconds
- Queue depth > 2× expected
- Storage costs trending above plan

**Info (dashboard, no notification):**
- Normal operations
- Periodic stats summaries

### 9.3 Alert Hygiene Rules

- Every alert must be actionable. If no one knows what to do when it fires, it shouldn't exist.
- Every alert must have a runbook reference (Section 14).
- Alerts that fire and resolve repeatedly within a day get a flapping alert about themselves.
- New alerts go through a "shake-out period" — 1 week in warning-only mode before promoting to critical.

### 9.4 Dashboards

Recommended dashboards (decision deferred to observability tool choice):

1. **System health overview** — green/yellow/red per component
2. **Request flow** — latency and error rate per endpoint
3. **Job queue** — pending, running, completed/failed per kind
4. **Provider health** — per-provider response time, error rate, quota
5. **Cost & burn rate** — current spend by category, projected month-end
6. **Per-tenant view** — drill into a specific user's recent activity for support

## 10. Logging at SaaS Scale

### 10.1 Centralization

Logs from all FastAPI workers and job runners flow to a central log aggregator (specific tool TBD: ELK, Datadog Logs, CloudWatch Logs, Grafana Loki, etc.).

Structured JSON format from MVP (Architecture Section 16.1) carries through unchanged — same log shape, different destination.

### 10.2 Retention & Cost Controls

| Log Tier | Retention | Notes |
|----------|-----------|-------|
| Hot (queryable, indexed) | 30 days | Most expensive; only critical recent logs |
| Warm (queryable, slower) | 90 days | Investigations, support tickets |
| Cold (archived, queryable on request) | 1 year | Compliance, audit |
| Deleted | beyond 1 year | Per data retention policy |

Adjust per actual compliance requirements when SaaS launches.

### 10.3 Per-Tenant Log Isolation

Every log entry includes `user_id` (or `tenant_id`). When a support ticket comes in, an operator can pull just that user's logs without seeing other users' data.

```python
# Example query pattern
logs.query(filters={"user_id": "u_abc123", "timestamp": "2026-05-11T*"})
```

Critical: ensure operator access to per-tenant logs is audited (Section 12).

### 10.4 Sensitive Data Discipline (Still Critical)

The `SafeLogger` patterns from MVP carry through. Even at SaaS scale, API keys, payment info, and personal data NEVER appear in logs. Periodic automated scans verify this.

## 11. Scaling Patterns

### 11.1 Vertical vs Horizontal

- **FastAPI workers:** horizontal (add more workers behind LB). Stateless, easy.
- **Job workers (Celery):** horizontal (add more worker processes). Queue depth drives scaling decisions.
- **PostgreSQL:** vertical initially; sharding/read replicas at high scale (TBD threshold)
- **Redis:** vertical initially; clustering at high scale (TBD threshold)
- **Object storage:** infinite by design (S3/GCS scale transparently)

### 11.2 Auto-Scaling Triggers

| Component | Scale Up When | Scale Down When |
|-----------|---------------|-----------------|
| FastAPI workers | CPU > 70% for 5 min OR queue depth growing | CPU < 30% for 15 min |
| Job workers | Queue depth > 5× worker count | Queue depth = 0 for 10 min |
| Database connections | Pool > 80% saturated | Pool < 20% saturated |

Specific values are tunable based on real traffic patterns post-launch.

### 11.3 Capacity Planning

**Per-user expected resource usage:**
- 1 active conversation: ~50KB memory in cache, 5 req/min during active use
- 1 generation job: ~100MB scratch space, 60-90 min wall-clock, 50-80 fal.ai API calls
- Idle user: 0 resources

**Sizing examples (rough, refine post-launch):**
- 100 concurrent active users: 2-4 FastAPI workers, 4-8 job workers
- 1000 concurrent active users: 8-16 FastAPI workers, 16-32 job workers
- 10000 concurrent active users: requires deeper consideration of bottlenecks (database connection limits, fal.ai concurrency, storage I/O)

### 11.4 Provider Concurrency Coordination

Per API Integration Spec Section 10, fal.ai has client-side concurrency limits. In MVP, this is `asyncio.Semaphore(3)` within a single process. In SaaS, multiple workers may each make fal.ai calls simultaneously.

**Coordination patterns:**
- **Per-user limit:** each user has their own semaphore-equivalent (Redis-backed counter), so one user's heavy load doesn't starve others
- **Global limit:** if fal.ai imposes a total account concurrency limit, a global counter across all workers (Redis) enforces it
- **Per-job assignment:** generation jobs are assigned slots from a pool; if no slot, queued

Specific implementation deferred to SaaS build; the BYO-keys architecture means each user has their own fal.ai account, so per-user limits matter more than global.

## 12. Security Operations (SaaS)

### 12.1 Audit Logging

Every sensitive operation produces an audit log entry, separate from application logs:

| Event | Fields |
|-------|--------|
| User login | `user_id`, `ip`, `user_agent`, `auth_method`, `success` |
| API key rotation | `user_id`, `service`, `timestamp` (no keys logged) |
| Settings change | `user_id`, `setting_key`, `previous_value_hash`, `new_value_hash` |
| Project access | `user_id`, `project_id`, `operation` (read/write/delete) |
| Admin action | `admin_user_id`, `target_user_id`, `action`, `reason` |
| Vault access | `user_id`, `service`, `purpose`, `request_id` |

Audit logs:
- Stored separately from application logs (different retention, different access controls)
- Retained for 1 year minimum (or per compliance requirements)
- Queryable by user (their own audit log, transparency) and by operators (subject to audit-of-audit)
- Tamper-evident (cryptographically signed, append-only)

### 12.2 Admin Access Discipline

Operators who need access to user data (for support) must:
1. Request access for a specific user, with reason
2. Access auto-logged with the reason
3. Access auto-expires after the support task completes
4. Reviewed periodically by a separate team member

Never:
- "All admins can see all users" — no broad access by default
- Persistent admin sessions
- Sharing admin credentials

### 12.3 Incident Response Plan

Pre-defined procedures for security incidents:

**Suspected key leak:**
1. Force rotation of affected user's keys (revoke at provider, prompt user for new keys)
2. Audit logs for the relevant time period
3. Notify the user with specifics
4. Post-mortem on how the leak occurred

**Suspected unauthorized access:**
1. Lock the user's account (no further activity until verified)
2. Force password/OAuth re-auth
3. Notify user via known-good contact method
4. Audit account activity for the relevant period
5. Restore access only after user confirms identity

**Data corruption / loss:**
1. Identify scope (one user, many users, all users)
2. Restore from backups (Section 13)
3. Communicate honestly with affected users
4. Post-mortem and process update

**Provider compromise (e.g., Anthropic announces breach):**
1. Notify all users with affected provider keys
2. Encourage immediate key rotation
3. Provide one-click rotation tool
4. Audit for any suspicious activity

### 12.4 Compliance Considerations

Decision deferred to SaaS launch jurisdiction:
- **GDPR (EU users):** data deletion rights, data portability, breach notification
- **CCPA (California users):** similar to GDPR
- **HIPAA / SOC 2 / ISO 27001:** if pursuing enterprise customers
- **Payment card industry (PCI):** Stripe handles this; IRdeo never sees raw card data

## 13. Backup & Recovery (SaaS)

### 13.1 Backup Strategy

**Object storage (project files):** native versioning + cross-region replication
- Object versioning: every write creates a new version; old versions retained 30 days
- Cross-region replication: copies to a second region for disaster recovery

**PostgreSQL:** point-in-time recovery (PITR)
- Continuous WAL archiving
- 7-day PITR window
- Daily full backups, retained 30 days
- Weekly full backups, retained 1 year

**Redis:** ephemeral by design
- Job queue: if Redis fails, in-flight jobs are retried (Section 11.4)
- WebSocket state: ephemeral; reconnect rebuilds state
- No critical data in Redis worth backing up

**Vault:** native backup per vault product (AWS Secrets Manager has automated backups; HashiCorp Vault has snapshot capability)

### 13.2 Recovery Time Objectives (RTO) & Recovery Point Objectives (RPO)

Target SLAs (negotiable):

| Scenario | RTO (time to restore) | RPO (data loss tolerance) |
|----------|----------------------|---------------------------|
| Single worker failure | < 1 minute | 0 (no data lost) |
| Database failure (failover) | < 5 minutes | < 1 minute |
| Region failure (failover to backup region) | < 30 minutes | < 5 minutes |
| Total data loss (catastrophic) | < 24 hours | < 1 hour |

### 13.3 Recovery Testing

- **Restore drills** quarterly: pick a random backup, restore to a staging environment, verify it works
- **Region failover drills** semi-annually: simulated region outage, verify failover completes within RTO
- **Tabletop exercises** for major scenarios: walk through the response without actually breaking anything

## 14. Runbooks (SaaS)

Specific procedures for common operational scenarios. Each runbook follows a template:

**Trigger:** What alert or symptom indicates this scenario
**Severity:** How urgent
**Who's on call:** Roles responsible
**Diagnosis steps:** How to confirm the issue and gather info
**Resolution steps:** Specific actions to fix
**Verification:** How to confirm the fix worked
**Post-mortem:** Required if user-facing impact occurred

Initial runbook list (specific content deferred to SaaS build):

| Runbook | Topic |
|---------|-------|
| RB-1 | All workers down |
| RB-2 | Database failover required |
| RB-3 | Job queue stuck / not processing |
| RB-4 | Provider (fal.ai/Anthropic/etc.) outage |
| RB-5 | Cost burn rate alert |
| RB-6 | Security incident (suspected breach) |
| RB-7 | Region failover |
| RB-8 | Storage near quota |
| RB-9 | Suspected DDoS / abuse |
| RB-10 | Single tenant in distress (high errors for one user) |

Runbooks are living documents; updated after every incident based on lessons learned.

## 15. Support Tooling (SaaS)

### 15.1 Admin Console

A web interface for IRdeo support staff (separate from the user-facing UI). Features:

- **User search:** find user by email, ID, organization
- **User detail view:** account info, projects, recent activity, audit log
- **Project detail view:** state, generation history, cost log, conversation
- **Quick actions (audit-logged):** reset password (force re-auth), pause account, refund, escalate
- **System health:** the dashboards from Section 9.4

Decision deferred: build custom admin UI vs. use existing tools (Retool, Forest Admin, etc.). Likely cheaper to use existing tools at MVP-of-SaaS scale.

### 15.2 Customer Support Workflow

1. Support ticket arrives (email, chat, etc.)
2. Support agent looks up user in admin console
3. Agent reviews user's recent activity and audit log
4. Agent identifies the issue
5. Agent either:
   - Provides self-service guidance
   - Performs a fix (audit-logged with reason)
   - Escalates to engineering

### 15.3 Self-Service Tools

Reduce support load by giving users tools to fix common issues themselves:

- **In-app help drawer** (UI Spec Section 3.7) with troubleshooting articles
- **Doctor-equivalent in UI:** "Run diagnostics" button that mirrors `irdeo doctor` but in browser
- **Restart job:** UI-driven job restart without support intervention
- **Cost breakdown:** clear visibility into where costs came from

## 16. Cost Management at SaaS Scale

### 16.1 Cost Categories

IRdeo's cost structure has two layers:

**Pass-through costs (user pays directly via BYO keys):**
- Anthropic API calls
- OpenAI API calls
- fal.ai API calls
- Lalal.ai API calls

**IRdeo platform costs:**
- Compute (FastAPI workers, job workers)
- Database (PostgreSQL)
- Object storage (S3/GCS bytes + ops)
- Bandwidth (CDN, data egress)
- Observability tools
- Vault service
- Domain, SSL, etc.

IRdeo's subscription fee covers the platform costs and yields the margin. The user's API spend doesn't touch IRdeo's books.

### 16.2 Per-User Platform Cost Budget

Estimate the per-user platform cost based on usage tier:

| Tier | Generations/month | Estimated platform cost | Subscription price | Margin |
|------|-------------------|--------------------------|--------------------|---------| 
| Hobby | < 5 | $2-$5 | $19/mo | ~70-80% |
| Pro | 5-30 | $10-$30 | $49/mo | ~50-60% |
| Studio | 30+ | $40-$100 | $149/mo | ~30-40% |

Numbers are illustrative; real pricing determined by competitive analysis and unit economics at launch.

### 16.3 Cost Alerts

- **Per-user:** alert if a single user's platform cost projects to exceed their tier's expected range (may indicate abuse or a tier mismatch)
- **Aggregate:** alert if total platform burn rate exceeds plan
- **Provider-side anomaly:** if fal.ai cost for a single user spikes 10× their normal, investigate (their key may be compromised)

### 16.4 Cost Optimization Levers

When platform costs need trimming:
- Storage lifecycle policies (move old project files to cheaper tiers automatically)
- Worker right-sizing (fewer/smaller workers during low-traffic hours)
- Caching (prompt-pack-export and preview rebuild caching)
- Database query optimization
- CDN tuning (cache headers, edge regions)

## 17. SaaS Launch Readiness Checklist

Before public launch, verify:

**Infrastructure:**
- [ ] All MVP-to-SaaS abstractions (config switches) tested at production scale
- [ ] Database backup + restore tested
- [ ] Cross-region failover tested
- [ ] Auto-scaling triggers calibrated against load tests

**Security:**
- [ ] Penetration test completed and findings remediated
- [ ] Audit logging verified
- [ ] Incident response runbooks reviewed
- [ ] On-call rotation established
- [ ] Compliance requirements identified and met

**Observability:**
- [ ] Monitoring dashboards live
- [ ] Alerts wired to on-call
- [ ] Log aggregation working
- [ ] Per-tenant log isolation verified

**Support:**
- [ ] Admin console operational
- [ ] Support team trained
- [ ] Documentation available (user-facing + internal)
- [ ] Self-service tools live

**Cost:**
- [ ] Per-tier cost budgets validated
- [ ] Cost alerts wired
- [ ] Cost optimization playbook ready

**Legal/Business:**
- [ ] Terms of service published
- [ ] Privacy policy published
- [ ] Billing integration (Stripe) tested end-to-end
- [ ] Refund/dispute process documented

This is a starting checklist; expand based on specific launch context.

---

## 18. Open Operations Questions

Things to resolve as MVP matures into SaaS planning:

- **OpQ-1** Specific cloud provider (AWS / GCP / Azure / Render / Fly.io) — decision driven by team familiarity, pricing, region requirements
- **OpQ-2** Container orchestrator — Kubernetes (heavy, powerful), ECS/Cloud Run (managed, simpler), bare VMs (cheapest, most ops work)
- **OpQ-3** Observability stack — Datadog (full-featured but expensive), Grafana stack (cheap, more ops work), cloud-native (CloudWatch/StackDriver, locked-in)
- **OpQ-4** Job queue choice — Celery (battle-tested but heavy), RQ (simpler), Dramatiq (modern alternative)
- **OpQ-5** Database choice — Postgres on RDS (default safe choice) vs Aurora (more expensive, higher performance) vs CockroachDB (geo-distributed)
- **OpQ-6** Compliance scope — what regulatory regimes (GDPR, SOC 2, HIPAA) apply at launch
- **OpQ-7** Pricing model — usage-based vs tiered subscription vs hybrid
- **OpQ-8** Free tier — offer one or not? Affects fraud/abuse considerations
- **OpQ-9** Team structure — sole operator, small team, distributed? Affects on-call requirements
- **OpQ-10** Service Level Agreement (SLA) commitments to users — what uptime promised
- **OpQ-11** Multi-region from launch, or single region with expansion later

These are decisions for the SaaS planning phase, not the MVP build phase. Listed here to ensure they're not forgotten when SaaS becomes the next focus.

---

## 19. Provenance

This Operations Specification was derived from:
- All six prior IRdeo specs
- Architecture v1.2 Section 17 (deployment topology)
- API Integration v1.0 Section 13 (SaaS evolution path)
- Industry-standard operational patterns (12-factor app, SRE practices, defense-in-depth security)
- Common patterns from comparable SaaS architectures (multi-tenant Python applications)

MVP operations (Sections 2-7) are concrete and implementable from MVP launch.
SaaS operations (Sections 8-17) are forward-looking architectural patterns with specific tool/infrastructure decisions deferred to when SaaS is actually being built. Defer markers throughout indicate "TBD in SaaS build" — these are not gaps, they're correctly-staged decisions.
