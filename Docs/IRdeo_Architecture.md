# IRdeo — Architecture Document
**Version:** 1.2
**Last updated:** May 11, 2026
**Status:** LIVING DOCUMENT — comprehensive technical architecture with code-level specifics
**Companion documents:** `IRdeo_Knowledge_Base.md` (v1.3+), `IRdeo_BRD.md` (v1.4+)

### Versioning Convention
- **v1.0** — first complete formal version
- **v1.1, v1.2, ...** — substantive revisions and additions
- **v2.0** — major restructure or fundamental approach change

### Changelog
- **v1.2 (May 11, 2026)** — Workflow simplification, aligned with KB v1.3 + BRD v1.4:
  - **Terminology change:** "segments" → "subchunks" throughout (matches user vocabulary)
  - **Take selection removed.** GenerationService produces one generation per subchunk. No take_1/take_2 directories. Section 10.3 simplified.
  - **Flat naming convention** for clips: `{chunk}_{subchunk_letter}_*.mp4`. Section 11.1 directory layout updated.
  - **Prompt Pack Export (Path A) added** — new Section 11.5 covers ZipPromptPackWriter and `/api/export/prompts` route. Power-user secondary path.
  - **Lyrics-audio sanity check** added to AnalysisService (Section 5.4 new).
  - **Filmora project + master docx require full song.** Partial generation OK; manual import via naming convention.
  - **JobKind updated:** removed LIPSYNC_CHUNK (now part of GENERATE_CHUNK pipeline), added EXPORT_PROMPT_PACK.
- **v1.1 (May 11, 2026)** — Three structural improvements from review pass:
  - **PreviewService added** (new Section 11.4) — ffmpeg-based stitching of winning takes into a single preview MP4 before Filmora export, giving the user an inline draft view in the web UI before opening their editor.
  - **Rate limiting hardened** (Sections 10.2, 10.3, 20.2) — explicit `asyncio.Semaphore` bounding concurrent fal.ai calls, tunable via settings, prevents 429 cascades when many takes generate in parallel.
  - **Vocal isolation moved to cloud** (Section 10.5) — replaced local Demucs/PyTorch dependency with Lalal.ai cloud API as primary, keeping the local Python process lightweight and removing GPU/VRAM requirements. Settings schema (13.2), pricing constants (14.1), Component Summary (3.2), and Open Questions (23) updated accordingly. ffmpeg added as system-level dependency (Section 21).
- **v1.0 (May 11, 2026)** — First complete formal version. Comprehensive technical architecture covering all system components, data flow, interfaces, deployment topology, state management, code-level specifics. Built on top of Knowledge Base v1.2 and BRD v1.3.

---

## 1. Document Overview

### 1.1 Purpose
This document defines the technical architecture of IRdeo. It translates the requirements specified in the BRD and the domain rules specified in the Knowledge Base into a concrete system design: components, interfaces, data flow, storage, deployment topology, and the code-level patterns that hold it all together.

This is the foundation document for build planning. The follow-on specs (UI Spec, Data Model Spec, API Integration Spec) will detail individual surface areas; this document defines how everything fits together.

### 1.2 Scope
- **In scope:** System components, their responsibilities, interfaces between them, data flow end-to-end, storage and state management, external integrations, deployment topology (local MVP + cloud SaaS), code-level patterns and conventions
- **Out of scope:** Specific JSON schemas (Data Model Spec), specific UI layouts (UI Spec), specific API integration details per provider (API Integration Spec), specific operational/deployment runbooks (Operations Spec, future)

### 1.3 Audience
- **Primary:** The builder (Rasti) executing implementation
- **Secondary:** AI coding agents (Claude Code CLI, etc.) acting as build executors — this document must be precise enough that an agent can implement components without ambiguity
- **Tertiary:** Future contributors / Phase 2 collaborators

### 1.4 Related Documents
- `IRdeo_Knowledge_Base.md` — Domain rules loaded into conversation LLM at runtime
- `IRdeo_BRD.md` — Business requirements, scope, user stories, functional requirements
- UI Spec (to be written) — User-facing interface design
- Data Model Spec (to be written) — JSON schemas for manifests, state objects
- API Integration Spec (to be written) — fal.ai, Claude, Whisper specifics
- Operations Spec (future) — Deployment, monitoring, backup, recovery

---

## 2. Architectural Principles

The following principles guide every design decision in this document. When in doubt, return to these.

### 2.1 Same Engine, Two Deployments
The Phase 1 MVP and Phase 2 SaaS share **identical core code**. The only differences are deployment topology and infrastructure-adjacent concerns (storage backend, auth layer, payment integration). No business logic differs between phases. This means every component is designed once with abstraction boundaries that allow swapping infrastructure (local filesystem ↔ cloud storage; local config ↔ vaulted user keys) without changing logic.

### 2.2 The Smart Layer Is the Product
The Knowledge Base + Conversation Engine + Prompt Generation pipeline is IRdeo's value. Video generation APIs (fal.ai → Kling/Veo) and lip sync APIs (fal.ai → Sync.so) are **commodity backends** behind a uniform interface. The architecture treats them as swappable, not as core.

### 2.3 User Has Final Authority
Per KB Section 0, KB rules are defaults, not absolutes. Every automated decision the system makes must be either (a) confirmable by the user, (b) overridable by the user, or (c) explicitly silent because the decision is mechanical and not creative. The architecture surfaces decisions to the user via the conversation layer; it does not bury them in code logic.

### 2.4 Stateless API Calls, Stateful System
The conversation LLM is stateless — every API call reconstructs the full context (KB + Song Context + Conversation History) from the system's persistent state. The system itself is stateful — it holds project data, conversation history, generation artifacts on local disk (MVP) or cloud storage (SaaS). The boundary is sharp.

### 2.5 Phases Are Idempotent Where Possible
Re-running a phase (e.g., regenerating a chunk, re-finalizing the Timestamps File) should produce consistent results without corrupting state. The system uses content-addressable artifacts and explicit state transitions, not implicit mutation.

### 2.6 Long-Running Jobs Are First-Class
Video generation takes 60–90 minutes per song. The architecture is built around async job orchestration from the start, not bolted on. Synchronous request/response patterns are reserved for fast operations (conversation turns, file uploads, queries).

### 2.7 Observable By Default
Every operation logs structured events. Generation progress streams to the UI in real time. State changes are traceable. Failures produce actionable error messages, not stack traces. This is essential because the user runs long jobs and needs to trust that the system is making progress.

### 2.8 Bring Your Own Keys
The user supplies their own fal.ai, Claude (Anthropic), and OpenAI API keys. IRdeo never proxies these on a shared account. This protects user costs (transparent, direct billing relationship with providers) and protects IRdeo (no exposure to reseller economics or upstream pricing changes).

---

## 3. High-Level System Architecture

### 3.1 Bird's-Eye View

```
┌─────────────────────────────────────────────────────────────────┐
│                      USER'S BROWSER                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Local Web UI (HTML + CSS + Vanilla JS / Alpine.js)       │  │
│  │  ─ Chat panel (text + voice)                              │  │
│  │  ─ Project sidebar                                        │  │
│  │  ─ File upload                                            │  │
│  │  ─ Timestamps File review/edit                            │  │
│  │  ─ Chunk definition                                       │  │
│  │  ─ Take selection grid (video playback)                   │  │
│  │  ─ Progress visualization (WebSocket)                     │  │
│  │  ─ Settings (API keys)                                    │  │
│  └─────────────────────┬─────────────────────────────────────┘  │
└────────────────────────┼────────────────────────────────────────┘
                         │ HTTP/REST + WebSocket
                         │ (localhost MVP; HTTPS SaaS)
┌────────────────────────┼────────────────────────────────────────┐
│                LOCAL FASTAPI BACKEND                            │
│  (single Python process, MVP; horizontally-scaled fleet, SaaS)  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  HTTP ROUTES (api/)                                     │    │
│  │  ─ /api/projects (CRUD)                                 │    │
│  │  ─ /api/chat (conversation)                             │    │
│  │  ─ /api/voice (Whisper transcription)                   │    │
│  │  ─ /api/upload (MP3, lyrics, photos)                    │    │
│  │  ─ /api/analyze (start analysis job)                    │    │
│  │  ─ /api/timestamps (review/finalize)                    │    │
│  │  ─ /api/chunks (define/update)                          │    │
│  │  ─ /api/generate (start generation job)                 │    │
│  │  ─ /api/regenerate-chunk (regenerate specific chunk)    │    │
│  │  ─ /api/lipsync (auto-runs after performer chunk gen)   │    │
│  │  ─ /api/preview/build (stitch chunks via ffmpeg)        │    │
│  │  ─ /api/preview/status (preview job status)             │    │
│  │  ─ /api/export/prompts (Path A: download prompt zip)    │    │
│  │  ─ /api/export (Path B: assemble Filmora + docx)        │    │
│  │  ─ /api/jobs/{id} (status polling)                      │    │
│  │  ─ /ws/jobs/{id} (WebSocket progress stream)            │    │
│  │  ─ /api/settings (API keys, defaults)                   │    │
│  │  ─ /api/health                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                    │
│  ┌─────────────────────────┼────────────────────────────────┐   │
│  │  SERVICE LAYER (services/)                               │   │
│  │                                                          │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                │   │
│  │  │ ConversationSvc │  │ AnalysisSvc     │                │   │
│  │  │ ─ Claude API    │  │ ─ Whisper       │                │   │
│  │  │ ─ KB + Song Ctx │  │ ─ librosa       │                │   │
│  │  │ ─ History mgmt  │  │ ─ Content cls   │                │   │
│  │  └─────────────────┘  └─────────────────┘                │   │
│  │                                                          │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                │   │
│  │  │ TimestampsSvc   │  │ ChunkSvc        │                │   │
│  │  │ ─ Draft TF      │  │ ─ Boundary prop │                │   │
│  │  │ ─ Whisper corr  │  │ ─ Manifest mgmt │                │   │
│  │  │ ─ Finalize      │  │ ─ User edits    │                │   │
│  │  └─────────────────┘  └─────────────────┘                │   │
│  │                                                          │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                │   │
│  │  │ PromptGenSvc    │  │ GenerationSvc   │                │   │
│  │  │ ─ Chunk prompt  │  │ ─ Job queue     │                │   │
│  │  │ ─ Segment decomp│  │ ─ fal.ai client │                │   │
│  │  │ ─ Continuity    │  │ ─ Retry logic   │                │   │
│  │  └─────────────────┘  └─────────────────┘                │   │
│  │                                                          │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                │   │
│  │  │ LipSyncSvc      │  │ OutputSvc       │                │   │
│  │  │ ─ Auto on       │  │ ─ Filmora gen   │                │   │
│  │  │   performer     │  │ ─ Master docx   │                │   │
│  │  │ ─ Vocal isolate │  │ ─ PromptPack    │                │   │
│  │  │ ─ Sync.so call  │  │ ─ Dir layout    │                │   │
│  │  └─────────────────┘  └─────────────────┘                │   │
│  │                                                          │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                │   │
│  │  │ ReferenceSvc    │  │ JobOrchestrator │                │   │
│  │  │ ─ Pre-baked     │  │ ─ Async tasks   │                │   │
│  │  │ ─ Live research │  │ ─ Status track  │                │   │
│  │  └─────────────────┘  └─────────────────┘                │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            │                                    │
│  ┌─────────────────────────┼────────────────────────────────┐   │
│  │  STORAGE LAYER (storage/)                                │   │
│  │                                                          │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                │   │
│  │  │ ProjectStore    │  │ MediaStore      │                │   │
│  │  │ ─ JSON manifests│  │ ─ MP3, photos   │                │   │
│  │  │ ─ Conv history  │  │ ─ Generated vid │                │   │
│  │  │ ─ Job records   │  │ ─ Lip-synced vid│                │   │
│  │  └─────────────────┘  └─────────────────┘                │   │
│  │                                                          │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                │   │
│  │  │ KnowledgeStore  │  │ SettingsStore   │                │   │
│  │  │ ─ KB markdown   │  │ ─ API keys      │                │   │
│  │  │ ─ Ref profiles  │  │ ─ User defaults │                │   │
│  │  └─────────────────┘  └─────────────────┘                │   │
│  │                                                          │   │
│  │  Local FS (MVP) │ Cloud storage (SaaS) — same interface  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                         │
                         │ External API calls (user's keys)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  EXTERNAL SERVICES                              │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ Anthropic    │  │ OpenAI       │  │ fal.ai       │           │
│  │ Claude API   │  │ Whisper API  │  │ Gateway      │           │
│  │ (Conv LLM)   │  │ (Transcribe) │  │              │           │
│  └──────────────┘  └──────────────┘  └──────┬───────┘           │
│                                             │                   │
│                                ┌────────────┴────────────┐      │
│                                │                         │      │
│                          ┌─────▼──────┐         ┌────────▼─────┐│
│                          │ Kling      │         │ Sync.so      ││
│                          │ (Video)    │         │ (Lip Sync)   ││
│                          │ Veo (Video)│         │              ││
│                          └────────────┘         └──────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Component Summary

| Layer | Component | Responsibility |
|-------|-----------|----------------|
| Frontend | Local Web UI | User-facing browser app |
| API | FastAPI Routes | HTTP/WebSocket endpoints |
| Services | ConversationService | Manages LLM conversations with KB + Song Context |
| Services | AnalysisService | Whisper + librosa + content classification |
| Services | TimestampsService | Draft and finalized Timestamps File lifecycle |
| Services | ChunkService | Chunk boundary proposal, manifest, user edits |
| Services | PromptGenService | Generates chunk-level and subchunk-level prompts |
| Services | GenerationService | Orchestrates video generation via fal.ai |
| Services | LipSyncService | Take selection + lip sync orchestration |
| Services | OutputService | Filmora project + master `.docx` assembly |
| Services | PromptPackService | Path A: zip prompts as text files for download |
| Services | PreviewService | ffmpeg-based stitching of chunks into preview MP4 |
| Services | VocalIsolationService | Vocal stem separation via Lalal.ai cloud API |
| Services | ReferenceService | Artist/song reference research (pre-baked + live) |
| Services | JobOrchestrator | Async job queue, status tracking, retries |
| Storage | ProjectStore | Per-project state (manifests, conversation history) |
| Storage | MediaStore | Binary media files (MP3, photos, video clips) |
| Storage | KnowledgeStore | KB markdown, reference profile library |
| Storage | SettingsStore | API keys (encrypted), user defaults |
| External | Anthropic Claude | Conversation LLM, content analysis |
| External | OpenAI Whisper | Speech-to-text |
| External | fal.ai → Kling | Video generation (performer + documentary) |
| External | fal.ai → Veo | Video generation (alternative, less common) |
| External | fal.ai → Sync.so | Lip sync |
| External | Lalal.ai | Vocal stem separation (cloud, replaces local Demucs) |
| System | ffmpeg | Local binary for preview stitching (user-installed prerequisite) |

---

## 4. Conversation Engine Architecture

The Conversation Engine is the heart of IRdeo — the AI Director that interviews users, applies KB rules, and produces engine-ready prompts. Everything else is plumbing.

### 4.1 Runtime Context Assembly

Per Knowledge Base Section 1, every conversation API call assembles three layers:

```python
# pseudocode for system_prompt construction
def build_system_prompt(project_id: str) -> str:
    kb_text = KnowledgeStore.load_kb_markdown()           # ~30KB stable
    song_ctx = ProjectStore.load_song_context(project_id)  # variable per song
    return f"{kb_text}\n\n---\n\nSONG CONTEXT:\n{song_ctx}"

def build_messages(project_id: str, new_user_msg: str) -> list[dict]:
    history = ProjectStore.load_conversation(project_id)  # all prior turns
    return history + [{"role": "user", "content": new_user_msg}]

async def chat_turn(project_id: str, user_msg: str) -> str:
    system_prompt = build_system_prompt(project_id)
    messages = build_messages(project_id, user_msg)
    response = await claude_client.messages.create(
        model="claude-opus-4-7",  # or user-configured model
        system=system_prompt,
        messages=messages,
        max_tokens=4096
    )
    ProjectStore.append_turn(project_id, user_msg, response.content[0].text)
    return response.content[0].text
```

### 4.2 Song Context Composition

The Song Context is everything per-song the LLM needs to know. It's assembled fresh each turn from persistent stores:

```python
def assemble_song_context(project_id: str) -> str:
    project = ProjectStore.load_project(project_id)
    parts = []
    parts.append(f"# Project: {project.song_title}")
    parts.append(f"Genre: {project.genre}")

    if project.timestamps_file_finalized:
        parts.append("## Timestamps File (FINALIZED — source of truth):")
        parts.append(MediaStore.read_text(project.timestamps_file_path))

    if project.chunk_manifest_exists:
        parts.append("## Chunk Manifest (current state):")
        parts.append(json.dumps(project.chunk_manifest, indent=2))

    if project.audio_analysis_exists:
        parts.append("## Audio Analysis:")
        parts.append(json.dumps(project.audio_analysis_summary, indent=2))

    if project.reference_research:
        parts.append("## Reference Research (for conversation guidance only — do not embed in prompts):")
        parts.append(project.reference_research_text)

    return "\n\n".join(parts)
```

### 4.3 Conversation State Persistence

Each project has its own conversation history. The history is a sequence of turns stored as JSON:

```python
# storage/conversation.py
@dataclass
class ConversationTurn:
    role: str  # "user" or "assistant"
    content: str
    timestamp: datetime
    turn_id: str  # uuid

class ConversationStore:
    def __init__(self, project_path: Path):
        self.path = project_path / "conversation.jsonl"

    def append(self, turn: ConversationTurn) -> None:
        with self.path.open("a") as f:
            f.write(json.dumps(asdict(turn), default=str) + "\n")

    def load_all(self) -> list[ConversationTurn]:
        if not self.path.exists():
            return []
        with self.path.open("r") as f:
            return [ConversationTurn(**json.loads(line)) for line in f]

    def as_messages_array(self) -> list[dict]:
        # Format expected by Claude/OpenAI APIs
        return [{"role": t.role, "content": t.content} for t in self.load_all()]
```

JSONL format chosen over single JSON file because:
- Append-only operations are atomic per line (no full-file rewrite)
- Conversation history grows linearly; never re-read the whole thing for writes
- Crash-resilient (partial write of last line is the worst case, easily detectable)
- Trivially debuggable (one turn per line in plain text)

### 4.4 Token Budget Management

LLM context window is 200K tokens. The system tracks usage and prevents overflow:

```python
class ContextBudget:
    MAX_TOKENS = 200_000
    RESERVED_FOR_RESPONSE = 4_096
    SAFETY_MARGIN = 8_000

    def estimate_tokens(self, text: str) -> int:
        # Approximate: 1 token ~= 4 chars for English
        return len(text) // 4

    def check_budget(self, system_prompt: str, messages: list[dict]) -> tuple[bool, int]:
        sys_tokens = self.estimate_tokens(system_prompt)
        msg_tokens = sum(self.estimate_tokens(m["content"]) for m in messages)
        total = sys_tokens + msg_tokens
        available = self.MAX_TOKENS - self.RESERVED_FOR_RESPONSE - self.SAFETY_MARGIN
        return (total <= available, available - total)

    def needs_summarization(self, history: list[dict]) -> bool:
        # Trigger if history alone exceeds 60% of available budget
        hist_tokens = sum(self.estimate_tokens(m["content"]) for m in history)
        return hist_tokens > (self.MAX_TOKENS * 0.6)
```

When `needs_summarization()` returns True, the system asynchronously generates a summary of the oldest N turns, replacing them with a single "[CONVERSATION SUMMARY: ...]" entry. This is a future enhancement; MVP can defer until token usage actually approaches limits in practice.

### 4.5 Conversation API Routes

```python
# api/routes/chat.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

router = APIRouter(prefix="/api/chat", tags=["chat"])

class ChatRequest(BaseModel):
    project_id: str
    message: str

class ChatResponse(BaseModel):
    reply: str
    actions: list[dict]  # optional structured actions (see Section 4.6)

@router.post("", response_model=ChatResponse)
async def chat(req: ChatRequest):
    project = ProjectStore.load(req.project_id)
    if not project:
        raise HTTPException(404, "Project not found")
    reply = await ConversationService.turn(req.project_id, req.message)
    actions = ActionParser.extract(reply)  # see Section 4.6
    return ChatResponse(reply=reply, actions=actions)
```

### 4.6 Structured Actions in Conversation Responses

Following the PASC chatbot pattern, the AI Director can include structured actions in its responses for the UI to render as buttons:

```python
# Action types specific to IRdeo
ACTION_TYPES = {
    "review_timestamps":  "Open timestamps file review panel",
    "review_chunks":      "Open chunk definition panel",
    "start_generation":   "Begin video generation",
    "regenerate_chunk":   "Regenerate chunk N",
    "show_prompt":        "Display generated prompt for chunk N",
    "upload_file":        "Prompt user for file upload",
    "export":             "Begin final output assembly",
    "settings":           "Open settings (API keys, defaults)",
}
```

Actions are embedded in LLM responses as a special tag the system parses:

```
[Conversational text here]

[ACTION:review_chunks|label=Review chunks]
[ACTION:upload_file|label=Upload reference photos|target=photos]
```

The parser strips action tags from the rendered text and returns them as structured data for the UI.

---

## 5. Audio Analysis Pipeline

### 5.1 Pipeline Stages

```
   MP3 input
      │
      ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Whisper    │───▶│   librosa    │───▶│   Content    │
│ (transcribe) │    │  (analyze)   │    │ Classifier   │
└──────────────┘    └──────────────┘    └──────────────┘
      │                    │                   │
      ▼                    ▼                   ▼
  raw transcript     audio features      sections + vocals
  + timestamps      (BPM, energy, beat)  + mood arc
                           │
                           ▼
                    ┌──────────────┐
                    │  Timestamps  │
                    │  File (draft)│
                    └──────────────┘
```

### 5.2 Service Implementation

```python
# services/analysis_service.py
from dataclasses import dataclass
from pathlib import Path

@dataclass
class AudioFeatures:
    bpm: float
    duration_seconds: float
    energy_curve: list[float]  # 2-sec resolution
    beat_positions: list[float]  # seconds
    brightness_curve: list[float]
    onset_strength: list[float]

@dataclass
class TranscriptionResult:
    text: str
    segments: list[dict]  # [{start, end, text}]
    language: str

class AnalysisService:
    def __init__(self, whisper_client, claude_client):
        self.whisper = whisper_client
        self.claude = claude_client

    async def analyze(self, mp3_path: Path, user_lyrics: str | None) -> dict:
        # Run Whisper + librosa in parallel
        transcript_task = asyncio.create_task(self._transcribe(mp3_path))
        features_task = asyncio.create_task(self._extract_features(mp3_path))
        transcript = await transcript_task
        features = await features_task

        # Content classification (uses Claude for section/vocal detection)
        classification = await self._classify_content(
            transcript, features, user_lyrics
        )
        return {
            "transcript": asdict(transcript),
            "features": asdict(features),
            "classification": classification,
        }

    async def _transcribe(self, mp3_path: Path) -> TranscriptionResult:
        with mp3_path.open("rb") as f:
            resp = await self.whisper.audio.transcriptions.create(
                model="whisper-1",
                file=f,
                response_format="verbose_json",
                timestamp_granularities=["segment"],
            )
        return TranscriptionResult(
            text=resp.text,
            segments=resp.segments,
            language=resp.language,
        )

    async def _extract_features(self, mp3_path: Path) -> AudioFeatures:
        # Runs in thread pool (librosa is CPU-bound, not async)
        return await asyncio.to_thread(self._librosa_analyze, mp3_path)

    def _librosa_analyze(self, mp3_path: Path) -> AudioFeatures:
        import librosa
        y, sr = librosa.load(str(mp3_path), sr=22050)
        duration = librosa.get_duration(y=y, sr=sr)
        tempo, beats = librosa.beat.beat_track(y=y, sr=sr)
        # 2-sec resolution energy curve
        frame_len = int(sr * 2)
        energy = [
            float(np.sqrt(np.mean(y[i:i+frame_len]**2)))
            for i in range(0, len(y), frame_len)
        ]
        # Spectral centroid as "brightness"
        centroid = librosa.feature.spectral_centroid(y=y, sr=sr)[0]
        brightness = [float(centroid[i:i+int(sr*2/512)].mean())
                      for i in range(0, len(centroid), int(sr*2/512))]
        onset = librosa.onset.onset_strength(y=y, sr=sr)
        return AudioFeatures(
            bpm=float(tempo),
            duration_seconds=float(duration),
            energy_curve=energy,
            beat_positions=[float(b) for b in librosa.frames_to_time(beats, sr=sr)],
            brightness_curve=brightness,
            onset_strength=[float(o) for o in onset.tolist()[::int(sr*2/512)]],
        )

    async def _classify_content(self, transcript, features, user_lyrics):
        # Use Claude to classify sections, detect vocal types, identify mood arc
        prompt = self._build_classification_prompt(transcript, features, user_lyrics)
        resp = await self.claude.messages.create(
            model="claude-opus-4-7",
            messages=[{"role": "user", "content": prompt}],
            max_tokens=4096,
        )
        return self._parse_classification(resp.content[0].text)
```

### 5.3 Skip-Whisper Path

When the user provides exact lyrics AND the audio is non-English or heavily instrumented, Whisper is skipped. The analysis service supports this via an explicit flag:

```python
async def analyze(self, mp3_path, user_lyrics, skip_whisper=False):
    if skip_whisper:
        if not user_lyrics:
            raise ValueError("Cannot skip Whisper without user-provided lyrics")
        # Synthesize a transcript from user lyrics with placeholder timestamps
        transcript = self._synthesize_transcript_from_lyrics(user_lyrics)
        features = await self._extract_features(mp3_path)
        classification = await self._classify_content(
            transcript, features, user_lyrics
        )
        return {...}  # same shape, timestamps will be refined in UI
```

In the skip-Whisper case, the UI presents manual timestamp assignment — the user listens to the song and clicks "mark timestamp here" while lyrics scroll, producing a Timestamps File without Whisper involvement.

### 5.4 Lyrics ↔ Audio Sanity Check

Per BRD FR-2.5 and KB Section 4, when both an MP3 and user lyrics are provided AND the audio language is English, the AnalysisService runs a token-overlap sanity check between Whisper output and user lyrics. The check catches a real production hazard: user uploads the wrong audio file or pastes lyrics from a different song, system spends compute on analysis, mismatch discovered only at the Timestamps File review step.

```python
# services/analysis_service.py (continued)
import re

class AnalysisService:
    ...

    def lyrics_audio_sanity_check(self, transcript: TranscriptionResult,
                                  user_lyrics: str | None) -> dict:
        """Compare Whisper transcript against user-provided lyrics for
        token overlap. Returns a result dict the UI can render as a warning
        if mismatch is detected. Advisory only — never blocks.

        Returns:
            {
                "should_warn": bool,
                "overlap_ratio": float,
                "threshold": float,
                "message": str | None,
                "skipped": bool,    # True if check skipped (non-English etc.)
                "skip_reason": str | None,
            }
        """
        # Skip if no user lyrics — nothing to compare against
        if not user_lyrics or not user_lyrics.strip():
            return {
                "should_warn": False, "overlap_ratio": 0.0, "threshold": 0.25,
                "message": None, "skipped": True,
                "skip_reason": "No user lyrics provided",
            }

        # Skip for non-English audio — Whisper unreliable enough to produce
        # false positives that would just annoy the user
        if transcript.language != "en":
            return {
                "should_warn": False, "overlap_ratio": 0.0, "threshold": 0.25,
                "message": None, "skipped": True,
                "skip_reason": f"Audio language is {transcript.language}, not English",
            }

        # Tokenize both — lowercase, alphanumeric words only
        def tokens(text: str) -> set[str]:
            return set(re.findall(r"[a-z0-9']+", text.lower()))

        whisper_tokens = tokens(transcript.text)
        lyric_tokens = tokens(user_lyrics)

        if not whisper_tokens or not lyric_tokens:
            return {
                "should_warn": False, "overlap_ratio": 0.0, "threshold": 0.25,
                "message": None, "skipped": True,
                "skip_reason": "Insufficient content to compare",
            }

        # Jaccard-style overlap: shared / smaller-set-size
        # (uses smaller set so partial transcript still matches full lyrics)
        shared = whisper_tokens & lyric_tokens
        smaller = min(len(whisper_tokens), len(lyric_tokens))
        overlap_ratio = len(shared) / smaller if smaller else 0.0

        threshold = 0.25  # generous; tunable

        if overlap_ratio >= threshold:
            return {
                "should_warn": False, "overlap_ratio": overlap_ratio,
                "threshold": threshold, "message": None,
                "skipped": False, "skip_reason": None,
            }

        return {
            "should_warn": True,
            "overlap_ratio": overlap_ratio,
            "threshold": threshold,
            "message": (
                f"I'm having trouble matching your lyrics to this audio. "
                f"Only {int(overlap_ratio * 100)}% of words overlap (expected "
                f"at least {int(threshold * 100)}%). Is this the right combination? "
                "You can proceed if you're sure, or upload a different file."
            ),
            "skipped": False, "skip_reason": None,
        }
```

The sanity check runs after Whisper completes but before the draft Timestamps File is generated. The result is returned to the UI as part of the analysis response; the UI renders it as an advisory banner if `should_warn` is True. The user can dismiss the warning and proceed, or upload different files.

**Tuning notes:**
- Threshold 0.25 (25% token overlap minimum) is intentionally generous — Whisper imperfection alone can lose 20–30% of tokens compared to clean source. Aim is to catch clear mismatches (10% or under), not borderline cases.
- Non-English audio always skips the check. We don't have a reliable transcript to compare against.
- The metric is set-based, not sequence-based — order doesn't matter. A song with the right tokens in a slightly different order still matches.
- AQ candidate (Section 23): if false-positive rate proves high in practice, tune threshold down to 0.15 or compute a smarter metric.

---

## 6. Timestamps File Lifecycle

### 6.1 State Machine

The Timestamps File transitions through three states:

```
   ┌───────────┐
   │  NONE     │  (no MP3 uploaded yet)
   └─────┬─────┘
         │ user uploads MP3
         ▼
   ┌───────────┐
   │  DRAFT    │  (auto-generated, not yet user-confirmed)
   └─────┬─────┘
         │ user confirms (after review/correction)
         ▼
   ┌───────────┐
   │ FINALIZED │  (locked, loaded into Song Context every turn)
   └─────┬─────┘
         │ user requests change later in session
         ▼
   ┌───────────┐
   │ AMENDED   │  (re-finalized after edit, treated as new FINALIZED)
   └───────────┘
```

### 6.2 Persistence Format

The Timestamps File is stored as `timestamps.md` in the project directory, in the canonical KB Section 2 format. State is tracked separately:

```python
# storage/project.py
@dataclass
class ProjectState:
    id: str
    song_title: str
    genre: str
    created_at: datetime
    timestamps_state: Literal["NONE", "DRAFT", "FINALIZED"]
    timestamps_finalized_at: datetime | None
    chunk_manifest_exists: bool
    generation_state: dict  # see Section 8
    output_state: dict  # see Section 11
```

The markdown file IS the artifact — no separate database row for the contents. This keeps the file system the source of truth and makes the data portable (you can email a `timestamps.md` to a collaborator and they can read it directly).

### 6.3 Service Implementation

```python
# services/timestamps_service.py
class TimestampsService:
    def __init__(self, project_store, analysis_service):
        self.projects = project_store
        self.analysis = analysis_service

    async def generate_draft(self, project_id: str) -> Path:
        project = self.projects.load(project_id)
        analysis = await self.analysis.analyze(
            project.mp3_path,
            project.user_lyrics,
            skip_whisper=project.skip_whisper,
        )
        md_text = self._format_canonical(analysis, project)
        ts_path = project.path / "timestamps.md"
        ts_path.write_text(md_text, encoding="utf-8")
        self.projects.set_state(project_id, "timestamps_state", "DRAFT")
        return ts_path

    def finalize(self, project_id: str, edited_md_text: str) -> Path:
        project = self.projects.load(project_id)
        ts_path = project.path / "timestamps.md"
        ts_path.write_text(edited_md_text, encoding="utf-8")
        self.projects.set_state(project_id, "timestamps_state", "FINALIZED")
        self.projects.set_state(project_id, "timestamps_finalized_at",
                                datetime.utcnow().isoformat())
        return ts_path

    def _format_canonical(self, analysis: dict, project) -> str:
        # Assembles the canonical KB Section 2 format from analysis output
        sections = self._detect_sections(analysis)
        energy_map = self._build_energy_map(analysis)
        corrections = self._compute_whisper_corrections(
            analysis["transcript"], project.user_lyrics
        )
        return CanonicalFormatter.assemble(
            song_title=project.song_title,
            artist=project.artist or "Unknown",
            duration=analysis["features"]["duration_seconds"],
            tempo=analysis["features"]["bpm"],
            sections=sections,
            energy_map=energy_map,
            whisper_corrections=corrections,
        )
```

---

## 7. Chunk Management

### 7.1 Two-Layer Model (Recap)

Per KB Section 3:
- **Chunks** — user-facing, ~1 min, aligned to musical sections
- **Segments** — engine-facing, max 8 sec, 7–8 per chunk

The chunk manifest stores chunk-level metadata; subchunks are derived at prompt-generation time.

### 7.2 Chunk Manifest Schema

Full schema lives in the Data Model Spec, but the shape here:

```python
@dataclass
class ChunkDefinition:
    chunk_id: str  # e.g., "chunk_01"
    start_seconds: float
    end_seconds: float
    section_label: str  # e.g., "Verse 2", from Timestamps File
    vocal_type: str  # e.g., "Male Heavy Rap"
    performer_present: bool
    performer_subchunks: list[tuple[float, float]] | None  # within-chunk
    reference_photo_set_id: str | None  # binding to library
    mood: str
    style_references: list[str]  # artist/song names
    model_preference: str  # "kling_motion_pro" | "kling_o3_pro" | "veo" | "auto"
    user_notes: str

@dataclass
class ChunkManifest:
    project_id: str
    version: int  # increments on edits
    chunks: list[ChunkDefinition]
    last_modified: datetime
```

Stored as `chunks.json` in the project directory.

### 7.3 Boundary Proposal Logic

```python
# services/chunk_service.py
class ChunkService:
    def propose_boundaries(self, timestamps_path: Path,
                          features: AudioFeatures) -> list[tuple[float, float]]:
        sections = self._parse_sections_from_timestamps(timestamps_path)
        proposed = []
        for section in sections:
            duration = section.end - section.start
            if duration <= 90:
                # Short enough, one chunk
                proposed.append((section.start, section.end))
            else:
                # Long section, split at lyrical pivots within
                pivots = self._find_internal_pivots(section, features.beat_positions)
                proposed.extend(self._split_at_pivots(section, pivots))
        return self._validate_no_mid_word_cuts(proposed, timestamps_path)
```

### 7.4 User Override

The UI presents the proposed boundaries visually (waveform with chunk dividers); user can drag dividers to adjust. Each move recomputes the manifest:

```python
@router.post("/api/chunks/redraw")
async def redraw_chunks(project_id: str, new_boundaries: list[ChunkBoundary]):
    manifest = ChunkService.load_manifest(project_id)
    manifest.chunks = ChunkService.apply_boundaries(manifest.chunks, new_boundaries)
    manifest.version += 1
    manifest.last_modified = datetime.utcnow()
    ChunkService.save_manifest(project_id, manifest)
    return {"manifest": manifest}
```

---

## 8. Prompt Generation

### 8.1 Two-Phase Approach

Prompt generation happens in two distinct passes:

**Pass 1 — Chunk-level prompt:** The AI Director writes a comprehensive narrative prompt for the entire chunk based on user input, KB rules, and Timestamps File. This is what the user might inspect via "show me the prompt for chunk 4."

**Pass 2 — Subchunk decomposition:** The same AI Director takes the chunk prompt and decomposes it into 7–8 subchunk prompts (max 8 sec each), each self-contained but continuity-aware.

### 8.2 Service Implementation

```python
# services/prompt_gen_service.py
class PromptGenService:
    def __init__(self, claude_client, knowledge_store):
        self.claude = claude_client
        self.kb = knowledge_store

    async def generate_chunk_prompt(self, project_id: str,
                                    chunk_id: str) -> str:
        chunk = ChunkService.load_chunk(project_id, chunk_id)
        timestamps = TimestampsService.load(project_id)
        genre_playbook = self.kb.get_genre_playbook(chunk.genre or "auto")

        system = self._build_prompt_gen_system_prompt(
            kb_section_8=self.kb.get_section("performer_integration"),
            kb_section_9=self.kb.get_section("continuity_rules"),
            kb_section_10=self.kb.get_section("audio_visual_translation"),
            genre_playbook=genre_playbook,
        )
        user_msg = self._build_chunk_input(chunk, timestamps)
        resp = await self.claude.messages.create(
            model="claude-opus-4-7",
            system=system,
            messages=[{"role": "user", "content": user_msg}],
            max_tokens=4096,
        )
        return resp.content[0].text

    async def decompose_into_subchunks(self, chunk_prompt: str,
                                       chunk: ChunkDefinition) -> list[SubchunkPrompt]:
        num_subchunks = self._calc_subchunks(chunk.end_seconds - chunk.start_seconds)
        system = self._build_subchunk_decomp_system_prompt()
        user_msg = self._build_decomp_input(chunk_prompt, chunk, num_subchunks)
        resp = await self.claude.messages.create(
            model="claude-opus-4-7",
            system=system,
            messages=[{"role": "user", "content": user_msg}],
            max_tokens=4096,
        )
        return self._parse_subchunks(resp.content[0].text, chunk)

    def _calc_subchunks(self, chunk_duration: float) -> int:
        # 8-sec max per subchunk; prefer fewer 8-sec over many short ones
        return max(1, int((chunk_duration + 7.99) // 8))
```

### 8.3 Subchunk Continuity Encoding

Each subchunk prompt explicitly carries continuity state from the previous subchunk, so the engine (which has no memory between calls) inherits visual state:

```python
@dataclass
class SubchunkPrompt:
    subchunk_id: str        # e.g., "01_a", "01_b"
    chunk_id: str           # e.g., "chunk_01"
    subchunk_letter: str    # "a", "b", "c", ...
    sequence_index: int
    start_seconds: float
    end_seconds: float
    duration: float         # 4-8 sec
    description_slug: str   # short slug for filename (e.g., "intro", "verse_start")
    prompt_text: str        # the actual text sent to the video API
    continuity_carry_from: str | None  # previous subchunk_id
    continuity_notes: str   # "same boardroom setting, lighting holds, performer eyeline to camera"
    model: str              # "kling_motion_pro" | "kling_o3_pro" | "veo"
    reference_photo_ids: list[str]
    expected_visual_summary: str  # for logs/debug, what we expect
```

---

## 9. Job Orchestration

### 9.1 Why Async Jobs

Video generation calls take 30–90 seconds each. A single song produces ~50–80 subchunk generations + 5–10 lip sync calls. That's 60–90 minutes of work. Synchronous HTTP requests time out long before that. So all generation work runs as background jobs.

### 9.2 Job Model

```python
# services/job_orchestrator.py
from enum import Enum

class JobStatus(str, Enum):
    PENDING = "pending"
    RUNNING = "running"
    SUCCEEDED = "succeeded"
    FAILED = "failed"
    CANCELLED = "cancelled"

class JobKind(str, Enum):
    ANALYSIS = "analysis"
    GENERATE_CHUNK = "generate_chunk"  # generates + auto-lipsyncs if performer
    GENERATE_SONG = "generate_song"    # all chunks in sequence
    REGENERATE_CHUNK = "regenerate_chunk"  # single chunk after user dissatisfaction
    BUILD_PREVIEW = "build_preview"
    EXPORT_PROMPT_PACK = "export_prompt_pack"  # Path A
    EXPORT = "export"  # Path B: Filmora + docx

@dataclass
class Job:
    id: str
    kind: JobKind
    project_id: str
    status: JobStatus
    progress: float  # 0.0 to 1.0
    current_step: str  # human-readable
    total_steps: int
    completed_steps: int
    created_at: datetime
    started_at: datetime | None
    completed_at: datetime | None
    error: str | None
    result: dict | None
    metadata: dict  # job-specific (e.g., {"chunk_id": "chunk_04"})

@dataclass
class JobEvent:
    job_id: str
    timestamp: datetime
    event_type: str  # "progress", "log", "error", "complete"
    message: str
    data: dict | None
```

### 9.3 Orchestrator Implementation

```python
class JobOrchestrator:
    def __init__(self, project_store, event_bus):
        self.projects = project_store
        self.events = event_bus
        self._running: dict[str, asyncio.Task] = {}

    async def submit(self, kind: JobKind, project_id: str,
                     metadata: dict = None) -> Job:
        job = Job(
            id=str(uuid.uuid4()),
            kind=kind,
            project_id=project_id,
            status=JobStatus.PENDING,
            progress=0.0,
            current_step="queued",
            total_steps=0,
            completed_steps=0,
            created_at=datetime.utcnow(),
            started_at=None,
            completed_at=None,
            error=None,
            result=None,
            metadata=metadata or {},
        )
        self.projects.save_job(job)
        task = asyncio.create_task(self._run(job))
        self._running[job.id] = task
        return job

    async def _run(self, job: Job):
        job.status = JobStatus.RUNNING
        job.started_at = datetime.utcnow()
        self.projects.update_job(job)
        self._emit(job, "progress", "Job started")

        try:
            handler = self._get_handler(job.kind)
            result = await handler(job, self._progress_callback(job))
            job.status = JobStatus.SUCCEEDED
            job.result = result
            self._emit(job, "complete", "Job succeeded", result)
        except asyncio.CancelledError:
            job.status = JobStatus.CANCELLED
            self._emit(job, "complete", "Job cancelled")
        except Exception as e:
            job.status = JobStatus.FAILED
            job.error = str(e)
            self._emit(job, "error", str(e))
        finally:
            job.completed_at = datetime.utcnow()
            self.projects.update_job(job)
            self._running.pop(job.id, None)

    def _progress_callback(self, job: Job):
        def cb(step_idx: int, total: int, msg: str):
            job.completed_steps = step_idx
            job.total_steps = total
            job.progress = step_idx / total if total > 0 else 0.0
            job.current_step = msg
            self.projects.update_job(job)
            self._emit(job, "progress", msg, {"step": step_idx, "total": total})
        return cb

    def _emit(self, job: Job, event_type: str, message: str, data: dict = None):
        evt = JobEvent(
            job_id=job.id,
            timestamp=datetime.utcnow(),
            event_type=event_type,
            message=message,
            data=data,
        )
        self.events.publish(evt)

    async def cancel(self, job_id: str):
        task = self._running.get(job_id)
        if task:
            task.cancel()
```

### 9.4 WebSocket Progress Streaming

The UI subscribes to job events via WebSocket:

```python
# api/routes/jobs.py
from fastapi import WebSocket

@router.websocket("/ws/jobs/{job_id}")
async def job_ws(ws: WebSocket, job_id: str):
    await ws.accept()
    subscription = event_bus.subscribe(job_id)
    try:
        async for event in subscription:
            await ws.send_json(event.to_dict())
    except WebSocketDisconnect:
        pass
    finally:
        event_bus.unsubscribe(subscription)
```

The UI displays a progress bar and live log; user can navigate away and return without losing state because progress is persisted in the job record.

### 9.5 MVP Process Model

MVP runs in a single Python process with `asyncio` cooperative multitasking. No external queue (Redis, RabbitMQ) needed. Jobs run as `asyncio.Task` instances within the FastAPI process. Job state persists to disk so a process crash doesn't lose tracking — on restart, the orchestrator scans for jobs in RUNNING status and marks them as FAILED (forcing user re-trigger). This is simpler than implementing crash recovery and acceptable for single-user MVP.

### 9.6 SaaS Process Model

For SaaS, the job orchestrator moves to a dedicated queue (Redis + Celery, or similar) so jobs survive process restarts and can be distributed across workers. The `JobOrchestrator` interface stays identical; only the backend changes. This is exactly the kind of abstraction that lets Phase 2 swap infrastructure without changing logic.

---

## 10. Generation & Lip Sync via fal.ai

### 10.1 fal.ai Integration Abstraction

fal.ai is the primary provider, but the abstraction allows swapping. The interface:

```python
# services/generation/providers/base.py
from abc import ABC, abstractmethod

class GenerationProvider(ABC):
    @abstractmethod
    async def generate_video(self, prompt: SegmentPrompt,
                            reference_images: list[bytes] = None) -> bytes:
        """Returns video bytes."""

    @abstractmethod
    async def lip_sync(self, video_bytes: bytes, audio_bytes: bytes,
                       model: str = "lipsync-2") -> bytes:
        """Returns lip-synced video bytes."""

    @abstractmethod
    def supported_models(self) -> list[str]:
        """List of model identifiers this provider supports."""

    @abstractmethod
    async def estimate_cost(self, prompt: SegmentPrompt) -> float:
        """USD cost estimate for the operation."""
```

### 10.2 fal.ai Provider Implementation

```python
# services/generation/providers/fal_ai.py
import fal_client

class FalAiProvider(GenerationProvider):
    MODEL_MAP = {
        "kling_motion_pro": "fal-ai/kling-video/v1.6/pro/image-to-video",
        "kling_o3_pro":     "fal-ai/kling-video/v1.6/pro/text-to-video",
        "veo_3":            "fal-ai/veo3",
        "veo_3_fast":       "fal-ai/veo3/fast",
    }
    LIPSYNC_MAP = {
        "lipsync-2":     "fal-ai/sync-lipsync/v2",
        "lipsync-2-pro": "fal-ai/sync-lipsync/v2-pro",
        "sync-3":        "fal-ai/sync-lipsync/v3",
    }

    def __init__(self, api_key: str, max_concurrent: int = 3):
        """
        Args:
            api_key: fal.ai API key
            max_concurrent: Maximum concurrent API calls. Tunable to match
                fal.ai's actual rate limits (verified during integration testing).
                The semaphore bounds ALL fal.ai calls system-wide (across video
                generation AND lip sync) to prevent 429 cascades when many
                subchunks generate in parallel. Configurable via settings
                (falai_max_concurrent).
        """
        fal_client.api_key = api_key
        self._semaphore = asyncio.Semaphore(max_concurrent)

    async def generate_video(self, prompt, reference_images=None):
        async with self._semaphore:
            model = self.MODEL_MAP[prompt.model]
            args = {
                "prompt": prompt.prompt_text,
                "duration": int(prompt.duration),
            }
            if reference_images:
                args["image_urls"] = [self._upload_image(img) for img in reference_images]

            result = await fal_client.subscribe_async(
                model,
                arguments=args,
                with_logs=True,
            )
            return await self._download(result["video"]["url"])

    async def lip_sync(self, video_bytes, audio_bytes, model="lipsync-2"):
        async with self._semaphore:
            model_id = self.LIPSYNC_MAP[model]
            video_url = await self._upload_temp(video_bytes, "video/mp4")
            audio_url = await self._upload_temp(audio_bytes, "audio/mp3")
            result = await fal_client.subscribe_async(
                model_id,
                arguments={"video_url": video_url, "audio_url": audio_url},
            )
            return await self._download(result["video"]["url"])

    def supported_models(self):
        return list(self.MODEL_MAP.keys()) + list(self.LIPSYNC_MAP.keys())

    async def estimate_cost(self, prompt):
        # See Cost Model section for pricing logic
        return CostEstimator.estimate(prompt, provider="fal_ai")
```

### 10.3 Generation Service Orchestration

```python
# services/generation_service.py
class GenerationService:
    def __init__(self, provider: GenerationProvider, output_dir: Path):
        self.provider = provider
        self.output = output_dir

    async def generate_song(self, project_id: str, progress_cb) -> dict:
        chunks = ChunkService.load_chunks(project_id)
        total_subchunks = sum(self._subchunks_for(c) for c in chunks)
        step = 0
        results = {}

        for chunk in chunks:
            chunk_results = await self._generate_chunk(
                chunk, lambda i, t, m: progress_cb(step + i, total_subchunks, m)
            )
            results[chunk.chunk_id] = chunk_results
            step += len(chunk_results)

        return results

    async def _generate_chunk(self, chunk: ChunkDefinition, progress_cb):
        # 1. Generate the chunk-level prompt
        chunk_prompt = await PromptGenService.generate_chunk_prompt(chunk)
        # 2. Decompose into subchunk prompts
        subchunks = await PromptGenService.decompose_into_subchunks(chunk_prompt, chunk)
        # 3. Generate one video per subchunk (no multi-take)
        chunk_output = []
        for idx, sub in enumerate(subchunks):
            video = await self._generate_with_retry(sub, chunk.reference_photos)
            # Flat naming: 01_a_*.mp4, 01_b_*.mp4, etc.
            clip_path = self._save_clip(chunk.chunk_id, sub.subchunk_letter,
                                        sub.description_slug, video)
            chunk_output.append({"subchunk": sub, "clip_path": clip_path})
            progress_cb(idx + 1, len(subchunks),
                       f"Generated {chunk.chunk_id}_{sub.subchunk_letter}")
        return chunk_output

    def _save_clip(self, chunk_id: str, subchunk_letter: str,
                   description_slug: str, video_bytes: bytes) -> Path:
        """Flat naming: 01_a_intro.mp4, 01_b_intro.mp4, 02_a_verse.mp4 etc."""
        chunk_num = chunk_id.removeprefix("chunk_")
        filename = f"{chunk_num}_{subchunk_letter}_{description_slug}.mp4"
        path = self.output / "chunks" / chunk_id / filename
        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_bytes(video_bytes)
        return path

    async def regenerate_chunk(self, project_id: str, chunk_id: str,
                              progress_cb) -> dict:
        """Regenerate a single chunk after user dissatisfaction.
        Overwrites previous output for this chunk."""
        chunk = ChunkService.load_chunk(project_id, chunk_id)
        # Optionally pick up updated prompt/model from chunk manifest changes
        return await self._generate_chunk(chunk, progress_cb)

    async def _generate_with_retry(self, prompt, refs, max_retries=3):
        for attempt in range(max_retries):
            try:
                return await self.provider.generate_video(prompt, refs)
            except (RateLimitError, TimeoutError) as e:
                if attempt == max_retries - 1:
                    raise
                await asyncio.sleep(2 ** attempt)  # exponential backoff
```

**Concurrency control — critical detail:** When generating an entire song, the GenerationService may fire many subchunk generations in parallel (~50–80 subchunks for a 5-minute song). Without bounded concurrency, fal.ai will return 429 rate-limit errors and the run will cascade-fail. The throttling lives inside the provider (FalAiProvider's `asyncio.Semaphore` from Section 10.2). The GenerationService can safely use `asyncio.gather()` for parallelism — the semaphore ensures the actual API hit rate stays bounded:

```python
async def _generate_subchunks_parallel(self, chunk: ChunkDefinition):
    """Fire all subchunk generations for a chunk in parallel; semaphore in
    provider enforces the actual concurrency cap."""
    return await asyncio.gather(*[
        self._generate_with_retry(sub, chunk.reference_photos)
        for sub in chunk.subchunks
    ])
```

The semaphore default is 3 concurrent calls; tunable via `settings.falai_max_concurrent`. If users see 429s in practice, they can lower the cap. If fal.ai allows higher concurrency, they can raise it for faster generation.

### 10.4 Lip Sync Service

Per BRD FR-8.11, lip sync runs automatically on performer chunks after take selection:

```python
# services/lipsync_service.py
class LipSyncService:
    def __init__(self, provider, vocal_isolator):
        self.provider = provider
        self.vocal_isolator = vocal_isolator

    async def lip_sync_performer_chunks(self, project_id: str, progress_cb):
        """Lip sync runs automatically on every performer chunk's generated
        clips (one per subchunk). No take selection — there's one clip to sync.
        Non-performer chunks skip this step entirely."""
        manifest = ChunkService.load_manifest(project_id)
        full_audio = self._load_song_audio(project_id)
        performer_chunks = [c for c in manifest.chunks if c.performer_present]

        for idx, chunk in enumerate(performer_chunks):
            audio_slice = self._slice_audio(full_audio,
                                            chunk.start_seconds,
                                            chunk.end_seconds)
            vocals_only = await self.vocal_isolator.isolate(audio_slice)
            # Lip-sync each subchunk's clip
            for sub in chunk.subchunks:
                clip_path = self._get_clip_path(project_id, chunk.chunk_id, sub)
                synced = await self.provider.lip_sync(
                    video_bytes=clip_path.read_bytes(),
                    audio_bytes=vocals_only,
                    model=chunk.lipsync_model or "lipsync-2",
                )
                # Overwrite the original clip with lipsynced version
                # (final per-subchunk artifact)
                clip_path.write_bytes(synced)
            progress_cb(idx + 1, len(performer_chunks),
                       f"Lip synced {chunk.chunk_id}")
```

### 10.5 Vocal Isolation (Cloud-First)

For best lip sync results, vocals should be isolated from the instrumental track. IRdeo handles this via a cloud API rather than a local model, for three reasons:
1. **No GPU/VRAM requirement** — local Demucs needs PyTorch + ~2GB VRAM for reasonable speed. Many user machines (especially laptops) can't run it cleanly.
2. **Lightweight local process** — keeps IRdeo's Python footprint under 500MB even with all services loaded. Architectural principle from Section 2.
3. **Better quality** — current cloud services (Lalal.ai's Phoenix/Orion engines, Mel-Roformer) consistently outperform open-source Demucs on vocal isolation specifically.

**Primary provider: Lalal.ai.** Credit-based pricing (~$10 per 90 min processed), no subscription floor, production-grade API, top-tier quality for vocal isolation. Aligns with our BYO-keys pay-per-use model.

```python
# services/vocal_isolation/providers/base.py
from abc import ABC, abstractmethod

class VocalIsolationProvider(ABC):
    @abstractmethod
    async def isolate(self, audio_bytes: bytes,
                     audio_format: str = "mp3") -> bytes:
        """Returns vocals-only audio bytes."""

    @abstractmethod
    async def estimate_cost(self, duration_seconds: float) -> float:
        """USD cost estimate."""


# services/vocal_isolation/providers/lalai.py
import httpx

class LalaiProvider(VocalIsolationProvider):
    BASE_URL = "https://www.lalal.ai/api"

    def __init__(self, api_key: str, engine: str = "orion"):
        """
        Args:
            engine: 'phoenix' | 'orion' (Lalal.ai's two top engines)
        """
        self.api_key = api_key
        self.engine = engine

    async def isolate(self, audio_bytes, audio_format="mp3"):
        async with httpx.AsyncClient(timeout=120.0) as client:
            # 1. Upload audio
            upload_resp = await client.post(
                f"{self.BASE_URL}/upload/",
                headers={"Authorization": f"license {self.api_key}"},
                content=audio_bytes,
                params={"format": audio_format},
            )
            file_id = upload_resp.json()["id"]

            # 2. Trigger separation (stem=vocals)
            split_resp = await client.post(
                f"{self.BASE_URL}/split/",
                headers={"Authorization": f"license {self.api_key}"},
                json=[{
                    "id": file_id,
                    "stem": "vocals",
                    "splitter": self.engine,
                }],
            )

            # 3. Poll for completion
            while True:
                check = await client.post(
                    f"{self.BASE_URL}/check/",
                    headers={"Authorization": f"license {self.api_key}"},
                    json={"id": file_id},
                )
                state = check.json()["result"][file_id]["task"]["state"]
                if state == "success":
                    vocals_url = check.json()["result"][file_id]["split"]["stem_track"]
                    break
                if state == "error":
                    raise UserError("Vocal isolation failed",
                                  suggestion="Check Lalal.ai account credits")
                await asyncio.sleep(2)

            # 4. Download vocals stem
            vocals_resp = await client.get(vocals_url)
            return vocals_resp.content

    async def estimate_cost(self, duration_seconds):
        # Lalal.ai pricing: ~$10 per 90 minutes = ~$0.0019/sec
        return duration_seconds * 0.0019


# services/vocal_isolation/providers/passthrough.py
class PassthroughProvider(VocalIsolationProvider):
    """Used when the user already supplies a vocals-only stem
    (typical with Suno output)."""
    async def isolate(self, audio_bytes, audio_format="mp3"):
        return audio_bytes

    async def estimate_cost(self, duration_seconds):
        return 0.0


# services/vocal_isolation_service.py
class VocalIsolationService:
    def __init__(self, provider: VocalIsolationProvider):
        self.provider = provider

    async def isolate_if_needed(self, project) -> bytes:
        if project.user_provided_vocals_only:
            # Skip — user supplied a clean stem
            return Path(project.vocals_stem_path).read_bytes()
        audio = Path(project.mp3_path).read_bytes()
        return await self.provider.isolate(audio)
```

**Provider abstraction allows fallback.** If Lalal.ai is unavailable or the user prefers an alternative, the same `VocalIsolationProvider` interface accommodates other backends — Replicate-hosted Demucs, ElevenLabs Audio Isolation, or eventually a local-fallback option for cost-sensitive offline use. Same pattern as Section 10.1 generation provider abstraction.

**When isolation is skipped:** If `project.user_provided_vocals_only` is True (user uploaded a Suno vocals stem separately), the PassthroughProvider returns the bytes unchanged — no API call, no cost.

---

## 11. Output Assembly

### 11.1 Per-Song Output Directory Structure

Per BRD FR-9.1 and FR-9.2:

```
~/IRdeo_Projects/
├── {project_id}_{song_slug}/
│   ├── source/
│   │   ├── original.mp3
│   │   ├── lyrics.txt (if provided)
│   │   ├── vocals_only.wav  (vocals stem after Lalal.ai isolation, if applicable)
│   │   └── upload_metadata.json
│   ├── reference_images/
│   │   ├── {photo_set_id}/
│   │   │   ├── photo_1.jpg
│   │   │   └── photo_2.jpg
│   ├── timestamps.md  (canonical Timestamps File)
│   ├── chunks.json  (chunk manifest)
│   ├── conversation.jsonl  (full conversation history)
│   ├── analysis.json  (audio analysis output)
│   ├── prompts/
│   │   ├── 01_chunk_prompt.txt
│   │   ├── 01_a.json  (subchunk prompt + metadata)
│   │   ├── 01_b.json
│   │   ├── 02_chunk_prompt.txt
│   │   ├── 02_a.json
│   │   ├── ...
│   ├── chunks/
│   │   ├── chunk_01/
│   │   │   ├── 01_a_intro.mp4    (final clip — lipsynced if performer)
│   │   │   ├── 01_b_intro.mp4
│   │   │   ├── 01_c_intro.mp4
│   │   │   └── chunk_metadata.json
│   │   ├── chunk_02/
│   │   │   ├── 02_a_verse.mp4
│   │   │   ├── 02_b_verse.mp4
│   │   │   ├── ...
│   │   ├── ...
│   ├── output/
│   │   ├── preview.mp4                (ffmpeg-stitched preview, available any time after generation)
│   │   ├── {song_slug}.wfp            (Filmora project — only when all chunks complete)
│   │   ├── {song_slug}.xml            (FCP XML fallback)
│   │   ├── {song_slug}_master.docx    (only when all chunks complete)
│   │   └── {song_slug}_prompts.zip    (Path A export, generated on demand)
│   ├── jobs/
│   │   ├── {job_id}.json  (job records for audit/debug)
│   ├── state.json  (top-level project state)
│   └── generation_log.jsonl  (event log)
```

**Key design choices:**

- **Flat clip naming** (`01_a_intro.mp4`, `01_b_intro.mp4`): chunk number + subchunk letter + brief descriptor. Sortable in any file browser. Easy to identify, select, and import manually into Filmora.
- **One clip per subchunk, period.** No `take_1/take_2/take_3` directories. If a chunk is bad, the user regenerates it — the new clip overwrites the old one.
- **Lipsync is in-place.** Performer chunks: the lip-synced version overwrites the silent generation in the same filename. No separate `lipsync.mp4` or `final.mp4` files.
- **Partial generation supported.** The chunks/ folder can hold a subset of the song's chunks. The user imports what exists into Filmora manually using the naming convention.
- **Whole-song deliverables (Filmora project, master docx) are gated.** They're only generated when ALL chunks for the song are complete. Output service detects partial state and surfaces a clear message.

### 11.2 Filmora Project Generation

The output service generates a Filmora-compatible project file. The `.wfp` format is investigated separately (BRD Open Question Q2); fallback formats are FCP XML or Premiere XML.

```python
# services/output/filmora_writer.py
class FilmoraWriter:
    def __init__(self, manifest, audio_path, chunk_paths):
        self.manifest = manifest
        self.audio = audio_path
        self.chunks = chunk_paths

    def write(self, output_path: Path):
        # If .wfp format is crackable:
        if FormatProbe.wfp_supported():
            self._write_wfp(output_path.with_suffix(".wfp"))
        else:
            self._write_fcp_xml(output_path.with_suffix(".xml"))

    def _write_wfp(self, path):
        # Implementation deferred until format is reverse-engineered
        raise NotImplementedError("WFP format investigation pending")

    def _write_fcp_xml(self, path):
        # Standard FCP XML — widely supported
        doc = self._build_xml_document()
        path.write_bytes(doc.tostring(pretty_print=True))
```

### 11.3 Master `.docx` Generation

```python
# services/output/docx_writer.py
from docx import Document

class MasterDocWriter:
    def write(self, project: Project, output_path: Path):
        doc = Document()
        doc.add_heading(f"IRdeo Master Document — {project.song_title}", 0)

        doc.add_heading("Suno Prompt", 1)
        doc.add_paragraph(project.suno_prompt or "(not recorded)")

        doc.add_heading("Production Notes", 1)
        doc.add_paragraph(project.production_notes or "(not recorded)")

        doc.add_heading("Album Context", 1)
        doc.add_paragraph(project.album_context or "(not specified)")

        doc.add_heading("Final Lyrics", 1)
        doc.add_paragraph(project.user_lyrics or project.whisper_transcript)

        doc.add_heading("Timestamps File", 1)
        doc.add_paragraph(MediaStore.read_text(project.timestamps_path))

        doc.add_heading("Video Prompts (per chunk)", 1)
        for chunk in project.chunks:
            doc.add_heading(f"Chunk {chunk.chunk_id}: {chunk.section_label}", 2)
            doc.add_paragraph(chunk.chunk_prompt)

        doc.add_heading("Whisper Corrections", 1)
        for old, new in project.whisper_corrections.items():
            doc.add_paragraph(f"• \"{old}\" → \"{new}\"", style="List Bullet")

        doc.save(str(output_path))
```

### 11.4 Preview Service (ffmpeg Stitch)

Before the user opens Filmora for real editing, they get an inline preview MP4 in the web UI — all winning takes (post-lipsync where applicable) stitched together with the song audio. This lets them validate the rough cut without leaving the browser and decide if any chunk needs regeneration before committing to the final export.

The preview is a **rough draft**, not a final product. No transitions, no effects, no color correction — just sequential concatenation of the chunks with the audio overlaid. Generated in seconds via ffmpeg. The user opens Filmora for the real creative work.

```python
# services/preview_service.py
import subprocess
from pathlib import Path

class PreviewService:
    def __init__(self, project_store, ffmpeg_binary: str = "ffmpeg"):
        self.projects = project_store
        self.ffmpeg = ffmpeg_binary

    async def build_preview(self, project_id: str, progress_cb) -> Path:
        """Stitch all per-subchunk clips into a single preview MP4.
        Clips are gathered from each chunk's directory in flat-naming order."""
        project = self.projects.load(project_id)
        chunks_ordered = sorted(project.chunks, key=lambda c: c.start_seconds)

        # Collect all clips across all chunks in playback order.
        # Each chunk's directory contains one .mp4 per subchunk
        # (e.g., 01_a_intro.mp4, 01_b_intro.mp4, 02_a_verse.mp4).
        # Sorting the chunk_dir alphabetically gives subchunk-letter order.
        all_clips = []
        for chunk in chunks_ordered:
            chunk_dir = project.path / "chunks" / chunk.chunk_id
            clip_paths = sorted(chunk_dir.glob("*.mp4"))
            if not clip_paths:
                raise UserError(
                    f"Chunk {chunk.chunk_id} has no clips yet.",
                    suggestion="Generate this chunk before building preview, "
                              "or build a partial preview from completed chunks only."
                )
            all_clips.extend(clip_paths)

        # Build ffmpeg concat list
        concat_file = project.path / "output" / "preview_concat.txt"
        concat_file.parent.mkdir(exist_ok=True)
        concat_file.write_text(
            "\n".join(f"file '{c.absolute()}'" for c in all_clips)
        )
        progress_cb(1, 4, "Built concat list")

        # Concat video clips (no re-encoding — fast)
        video_only = project.path / "output" / "preview_video.mp4"
        subprocess.run([
            self.ffmpeg, "-y",
            "-f", "concat", "-safe", "0",
            "-i", str(concat_file),
            "-c", "copy",
            str(video_only),
        ], check=True, capture_output=True)
        progress_cb(2, 4, "Concatenated clips")

        # Overlay the original audio (replace any audio from clips)
        preview_path = project.path / "output" / "preview.mp4"
        subprocess.run([
            self.ffmpeg, "-y",
            "-i", str(video_only),
            "-i", str(project.mp3_path),
            "-c:v", "copy",
            "-c:a", "aac",
            "-map", "0:v:0", "-map", "1:a:0",
            "-shortest",
            str(preview_path),
        ], check=True, capture_output=True)
        progress_cb(3, 4, "Attached audio track")

        # Cleanup intermediates
        video_only.unlink(missing_ok=True)
        concat_file.unlink(missing_ok=True)
        progress_cb(4, 4, "Preview ready")

        return preview_path
```

**API routes for preview:**

```python
# api/routes/preview.py
@router.post("/api/preview/build")
async def build_preview(project_id: str):
    job = await JobOrchestrator.submit(
        kind=JobKind.BUILD_PREVIEW,
        project_id=project_id,
    )
    return {"job_id": job.id}

@router.get("/api/preview/stream/{project_id}")
async def stream_preview(project_id: str):
    """Streams the preview MP4 for inline playback in the browser."""
    project = ProjectStore.load(project_id)
    preview_path = project.path / "output" / "preview.mp4"
    if not preview_path.exists():
        raise HTTPException(404, "Preview not yet built")
    return FileResponse(preview_path, media_type="video/mp4")
```

**System dependency: ffmpeg.** The user must have ffmpeg installed and on their PATH. Validated at startup by `irdeo doctor` (see Section 21.2). Cross-platform — works on Windows, macOS, Linux. The user only needs the binary; we don't bundle it.

**Workflow position:** Preview generation is fast (seconds for a 5-minute song since we copy streams without re-encoding). It runs:
1. After chunks have their flat-named clips in place (lipsync applied in-place for performer chunks per Section 10.4)
2. Before the user clicks "Export to Filmora"
3. As a quick job rather than a long one (no fal.ai involvement)

If the user regenerates a chunk, the preview is invalidated and can be rebuilt with a single click.

### 11.5 Prompt Pack Service (Path A Export)

Per BRD FR-9.7, after the conversation completes and prompts are generated for every chunk, the user can download a zip file containing the engineered prompts — one text file per chunk. This is the **Path A** output mode (power-user export option, secondary placement). Path A skips video generation entirely.

Use cases: user wants to paste prompts into AIVideo.com manually, compare IRdeo's prompt-engineering output against another tool, or save the prompts as a reference deliverable without generating video locally.

```python
# services/prompt_pack_service.py
import zipfile
from pathlib import Path
from datetime import datetime

class PromptPackService:
    def __init__(self, project_store):
        self.projects = project_store

    async def build_zip(self, project_id: str) -> Path:
        """Assemble engineered prompts into a downloadable zip.
        One text file per chunk; one summary README."""
        project = self.projects.load(project_id)
        if not project.prompts_generated:
            raise UserError(
                "Prompts have not been generated yet.",
                suggestion="Complete the conversation and chunk definition first."
            )

        output_dir = project.path / "output"
        output_dir.mkdir(exist_ok=True)
        zip_path = output_dir / f"{project.song_slug}_prompts.zip"

        with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as zf:
            # Add a README explaining what's in the zip
            zf.writestr("README.md", self._build_readme(project))

            # One file per chunk
            for chunk in project.chunks:
                content = self._format_chunk_prompt_file(project, chunk)
                # Filename matches the flat naming convention
                chunk_num = chunk.chunk_id.removeprefix("chunk_")
                filename = f"{chunk_num}_{chunk.section_slug}.txt"
                zf.writestr(filename, content)

            # Also include the full timestamps file for reference
            timestamps_md = (project.path / "timestamps.md").read_text(encoding="utf-8")
            zf.writestr("timestamps.md", timestamps_md)

        return zip_path

    def _build_readme(self, project) -> str:
        return f"""# {project.song_title} — Prompt Pack
Generated by IRdeo on {datetime.utcnow().isoformat()}Z

## Contents
This zip contains engineered video prompts for "{project.song_title}".
One text file per chunk, named per the IRdeo flat naming convention.

The prompts are designed to be pasted directly into:
- AIVideo.com (paste the chunk-level prompt)
- Direct API calls to Kling, Veo, or other video generation services
- Other prompt-aware video tools

## Files
- `README.md` — this file
- `timestamps.md` — the canonical Timestamps File for the song
- `{{NN}}_{{section_slug}}.txt` — one per chunk, with the engineered prompt

## How the Prompts Were Built
These prompts were generated by IRdeo's AI Director from your audio analysis,
your provided lyrics (if any), and your conversation about the visual vision
for each chunk. Genre playbooks, performer integration rules, and continuity
considerations have been applied per IRdeo's Knowledge Base.

## License
You own these prompts. Use them however you want.
"""

    def _format_chunk_prompt_file(self, project, chunk) -> str:
        """Format a single chunk's prompts into a human-readable text file."""
        lines = [
            f"# Chunk {chunk.chunk_id} — {chunk.section_label}",
            f"Time: {chunk.start_seconds:.2f}s – {chunk.end_seconds:.2f}s",
            f"Vocal type: {chunk.vocal_type}",
            f"Performer present: {chunk.performer_present}",
            f"Recommended model: {chunk.model_preference}",
            "",
            "## Chunk-Level Prompt (the full creative direction)",
            chunk.chunk_prompt,
            "",
            "## Subchunk Prompts (8-second slices for API generation)",
            "",
        ]
        for sub in chunk.subchunks:
            lines.append(f"### Subchunk {sub.subchunk_letter} "
                        f"({sub.start_seconds:.2f}s – {sub.end_seconds:.2f}s, "
                        f"{sub.duration:.1f}s)")
            lines.append(sub.prompt_text)
            if sub.continuity_notes:
                lines.append(f"\n_Continuity: {sub.continuity_notes}_")
            lines.append("")
        return "\n".join(lines)
```

**API route:**

```python
# api/routes/export.py
@router.post("/api/export/prompts")
async def export_prompts(project_id: str):
    """Build prompt pack zip and return download path."""
    job = await JobOrchestrator.submit(
        kind=JobKind.EXPORT_PROMPT_PACK,
        project_id=project_id,
    )
    return {"job_id": job.id}

@router.get("/api/export/prompts/download/{project_id}")
async def download_prompt_pack(project_id: str):
    """Stream the generated zip for download."""
    project = ProjectStore.load(project_id)
    zip_path = project.path / "output" / f"{project.song_slug}_prompts.zip"
    if not zip_path.exists():
        raise HTTPException(404, "Prompt pack not yet built")
    return FileResponse(
        zip_path,
        media_type="application/zip",
        filename=f"{project.song_slug}_prompts.zip",
    )
```

**Workflow position:** Path A becomes available after Phase 6 of the conversation flow (KB Section 13) — once chunks are defined and prompts are generated. The UI surfaces "Download Prompt Pack" as a secondary option alongside the primary "Generate Everything" button. Selecting Path A skips video generation, lip sync, preview, and Filmora export entirely. The user gets the zip and they're done.

**Cost:** Effectively free (LLM-only — the prompts were already generated as part of the conversation flow; zipping them up costs nothing additional).

---

## 12. Reference & Knowledge Stores

### 12.1 Knowledge Store

Loads the KB markdown into memory at startup; provides accessors:

```python
# storage/knowledge_store.py
class KnowledgeStore:
    KB_PATH = Path("knowledge/IRdeo_Knowledge_Base.md")

    def __init__(self):
        self._kb_text = self.KB_PATH.read_text(encoding="utf-8")
        self._sections = self._parse_sections(self._kb_text)

    def full_kb(self) -> str:
        return self._kb_text

    def get_section(self, name: str) -> str:
        return self._sections.get(name, "")

    def get_genre_playbook(self, genre: str) -> str:
        # Section 7.X based on genre
        return self._sections.get(f"genre_{genre.lower().replace(' ', '_')}", "")

    def reload(self):
        # For dev workflow — re-read KB when it changes on disk
        self._kb_text = self.KB_PATH.read_text(encoding="utf-8")
        self._sections = self._parse_sections(self._kb_text)
```

### 12.2 Reference Profile Library

Pre-baked artist profiles live as JSON files under `knowledge/references/`:

```
knowledge/references/
├── eminem_cleanin_out_my_closet.json
├── eminem_like_toy_soldiers.json
├── eminem_lose_yourself.json
├── portishead_glory_box.json
├── beth_gibbons_solo.json
└── ...
```

Each file:

```json
{
  "artist": "Eminem",
  "song_title": "Cleanin' Out My Closet",
  "genre": "Political Rap / Confessional Rap",
  "bpm_range": [97, 99],
  "vocal_style": "Male confessional rap, intense delivery",
  "mood": "Anger, vulnerability, accusation",
  "video_aesthetic": "Dark, narrative, performer-driven with B-roll cuts",
  "performer_dynamic": "Center frame, direct address to camera at peaks",
  "visual_conventions": [
    "Documentary cuts on factual lines",
    "Performer close-up on emotional admissions",
    "Cold color grade"
  ],
  "notes_for_ai_director": "User who says 'feels like Cleanin' Out My Closet' wants raw confessional energy. Match performer presence at emotional peaks; documentary fills the rest. Cold palette default."
}
```

### 12.3 Live Reference Research

For unknown references, the system performs web search and synthesis:

```python
# services/reference_service.py
class ReferenceService:
    async def lookup(self, query: str) -> dict:
        cached = self._cache.get(query)
        if cached:
            return cached

        local_profile = self._find_prebaked(query)
        if local_profile:
            return local_profile

        # Live web search
        search_results = await self.web_search.search(query)
        synthesized = await self._synthesize_profile(query, search_results)
        synthesized["source"] = "live_research"
        synthesized["confidence"] = "medium"
        synthesized["sources"] = [r["url"] for r in search_results[:3]]

        self._cache.set(query, synthesized)
        return synthesized

    async def _synthesize_profile(self, query, results):
        # Use Claude to read search results and produce a profile in the same schema
        # as pre-baked profiles, plus sources and confidence
        ...
```

The grounding rule from KB Section 6 applies: the synthesized profile must be presented to the user for confirmation before it's used in conversation context.

---

## 13. Settings & API Key Management

### 13.1 Local Storage (MVP)

API keys are stored in an encrypted file at `~/.irdeo/settings.json`:

```python
# storage/settings_store.py
from cryptography.fernet import Fernet

class SettingsStore:
    SETTINGS_PATH = Path.home() / ".irdeo" / "settings.json"
    KEY_PATH = Path.home() / ".irdeo" / ".key"

    def __init__(self):
        self.SETTINGS_PATH.parent.mkdir(exist_ok=True)
        self._fernet = self._load_or_create_key()

    def _load_or_create_key(self) -> Fernet:
        if self.KEY_PATH.exists():
            key = self.KEY_PATH.read_bytes()
        else:
            key = Fernet.generate_key()
            self.KEY_PATH.write_bytes(key)
            self.KEY_PATH.chmod(0o600)
        return Fernet(key)

    def save(self, settings: dict):
        plaintext = json.dumps(settings).encode()
        encrypted = self._fernet.encrypt(plaintext)
        self.SETTINGS_PATH.write_bytes(encrypted)
        self.SETTINGS_PATH.chmod(0o600)

    def load(self) -> dict:
        if not self.SETTINGS_PATH.exists():
            return {}
        encrypted = self.SETTINGS_PATH.read_bytes()
        plaintext = self._fernet.decrypt(encrypted)
        return json.loads(plaintext)
```

The encryption key lives next to the settings file with `0600` permissions. This is "encryption at rest" against casual filesystem inspection but not against an attacker with full machine access. For MVP this is acceptable; the threat model is "user's settings file is readable by other users on a shared machine," not "user's machine is compromised."

### 13.2 Settings Schema

```python
@dataclass
class Settings:
    # Core API keys
    anthropic_api_key: str | None
    openai_api_key: str | None  # for Whisper
    falai_api_key: str | None
    lalai_api_key: str | None  # for vocal isolation

    # Defaults
    default_video_model: str  # "kling_motion_pro"
    default_lipsync_model: str  # "lipsync-2"
    output_directory: Path  # ~/IRdeo_Projects
    claude_model: str  # "claude-opus-4-7"
    whisper_model: str  # "whisper-1"

    # Concurrency / rate-limit tuning
    falai_max_concurrent: int  # 3 — concurrent fal.ai calls cap; tune if 429s appear

    # Vocal isolation
    vocal_isolation_provider: str  # "lalai" | "passthrough" | "replicate"
    lalai_engine: str  # "phoenix" | "orion"
    user_supplies_vocals_only: bool  # if True, skip vocal isolation entirely
```

### 13.3 SaaS Migration Path

In Phase 2, `SettingsStore` is swapped for a vaulted store (AWS Secrets Manager, HashiCorp Vault, or equivalent). The interface stays identical; only the backend differs:

```python
# storage/settings_store_vault.py — SaaS implementation
class VaultedSettingsStore(SettingsStore):
    def __init__(self, user_id, vault_client):
        self.user_id = user_id
        self.vault = vault_client

    def save(self, settings: dict):
        self.vault.write(f"users/{self.user_id}/settings", settings)

    def load(self) -> dict:
        return self.vault.read(f"users/{self.user_id}/settings")
```

---

## 14. Cost Model & Estimation

### 14.1 Pricing Constants

Tracked in a module that's easy to update as provider pricing changes:

```python
# pricing.py
PRICING = {
    "anthropic_claude_opus_4_7": {
        "input_per_million_tokens": 15.0,
        "output_per_million_tokens": 75.0,
    },
    "openai_whisper": {
        "per_minute": 0.006,
    },
    "fal_kling_motion_pro": {
        "per_second_of_output": 0.40,  # placeholder; verify against fal.ai
    },
    "fal_kling_o3_pro": {
        "per_second_of_output": 0.10,
    },
    "fal_veo_3": {
        "per_second_of_output": 0.50,
    },
    "fal_sync_lipsync_2": {
        "per_second_of_output": 0.05,
    },
    "fal_sync_lipsync_2_pro": {
        "per_second_of_output": 0.083,
    },
    "fal_sync_3": {
        "per_second_of_output": 0.133,
    },
    "lalai_vocal_isolation": {
        "per_second_of_input": 0.0019,  # ~$10 per 90 min processed
    },
}
```

### 14.2 Estimation Service

```python
class CostEstimator:
    @staticmethod
    def estimate_song(project: Project) -> CostBreakdown:
        chunks = project.chunks
        analysis_cost = CostEstimator._whisper(project.duration_minutes)
        conversation_cost = CostEstimator._llm_for_song(chunks)
        generation_cost = sum(
            CostEstimator._video_for_chunk(c) for c in chunks
        )
        lipsync_cost = sum(
            CostEstimator._lipsync_for_chunk(c)
            for c in chunks if c.performer_present
        )
        vocal_iso_cost = (
            CostEstimator._vocal_isolation(project.duration_seconds)
            if not project.user_provided_vocals_only
            else 0.0
        )
        return CostBreakdown(
            analysis=analysis_cost,
            conversation=conversation_cost,
            generation=generation_cost,
            lipsync=lipsync_cost,
            vocal_isolation=vocal_iso_cost,
            total=analysis_cost + conversation_cost + generation_cost
                  + lipsync_cost + vocal_iso_cost,
        )
```

### 14.3 Pre-Generation Confirmation

Per BRD NFR-C2, the user sees an estimated cost before generation fires:

```python
@router.post("/api/generate/estimate")
async def estimate(project_id: str) -> CostBreakdown:
    project = ProjectStore.load(project_id)
    return CostEstimator.estimate_song(project)

@router.post("/api/generate/start")
async def start(project_id: str, confirm_cost_above: float | None = None):
    estimate = CostEstimator.estimate_song(ProjectStore.load(project_id))
    if confirm_cost_above and estimate.total > confirm_cost_above:
        raise HTTPException(400, f"Cost ${estimate.total:.2f} exceeds confirmation threshold")
    job = await JobOrchestrator.submit(JobKind.GENERATE_SONG, project_id)
    return {"job_id": job.id, "estimated_cost": estimate.total}
```

---

## 15. Error Handling & Resilience

### 15.1 Error Categories

| Category | Examples | Handling |
|----------|----------|----------|
| Transient | API rate limit, timeout, network blip | Retry with exponential backoff |
| Provider-side | fal.ai 503, model unavailable | Retry; fall back to alternative provider if configured |
| Invalid input | Bad MP3, missing API key, oversized file | Surface to user with actionable message |
| Logic / bug | Unexpected None, schema mismatch | Log with full stack, surface as "internal error, see logs" |
| Cost overrun | Generation cost exceeds threshold | Stop, surface estimate, require explicit confirmation |
| Quota / billing | Provider account out of credits | Surface to user immediately, pause job |

### 15.2 Retry Policy

```python
class RetryPolicy:
    @staticmethod
    async def with_retry(coro_factory, max_attempts=3, base_delay=1.0,
                         retryable=(RateLimitError, TimeoutError, NetworkError)):
        for attempt in range(max_attempts):
            try:
                return await coro_factory()
            except retryable as e:
                if attempt == max_attempts - 1:
                    raise
                delay = base_delay * (2 ** attempt) + random.uniform(0, 0.5)
                await asyncio.sleep(delay)
            except Exception:
                raise  # don't retry non-transient errors
```

### 15.3 User-Facing Error Messages

```python
class UserError(Exception):
    """An error that should be shown to the user, not just logged."""
    def __init__(self, message: str, suggestion: str = None):
        self.message = message
        self.suggestion = suggestion
        super().__init__(message)

# Example usage:
raise UserError(
    message="fal.ai API key not configured",
    suggestion="Go to Settings to add your fal.ai API key. You can get one at fal.ai/dashboard.",
)
```

The UI presents UserError messages as inline alerts in the chat with the suggestion as a follow-up bubble.

### 15.4 Job Crash Recovery

If the process crashes mid-job, jobs are left in RUNNING status on disk. On startup, the orchestrator scans for orphaned jobs:

```python
async def recover_on_startup(self):
    for job in self.projects.find_jobs_by_status(JobStatus.RUNNING):
        # Mark as failed; user can re-trigger if desired
        job.status = JobStatus.FAILED
        job.error = "Process crashed; job needs to be re-triggered"
        job.completed_at = datetime.utcnow()
        self.projects.update_job(job)
```

For SaaS with a durable queue (Redis/Celery), jobs survive process crashes natively — the orchestrator just resumes them.

---

## 16. Observability

### 16.1 Structured Logging

All events log as JSON via `structlog`:

```python
import structlog

logger = structlog.get_logger()

logger.info(
    "generation_subchunk_started",
    project_id=project_id,
    chunk_id=chunk.chunk_id,
    subchunk_id=subchunk.subchunk_id,
    model=subchunk.model,
    duration=subchunk.duration,
)
```

Logs land in `~/.irdeo/logs/irdeo.log` with daily rotation. Critical errors also emit to console.

### 16.2 Cost Tracking

Every external API call records actual cost:

```python
class CostTracker:
    def record(self, project_id: str, service: str, operation: str,
               input_units: float, cost_usd: float, metadata: dict = None):
        entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "project_id": project_id,
            "service": service,  # "claude" | "whisper" | "fal_video" | "fal_lipsync"
            "operation": operation,
            "input_units": input_units,
            "cost_usd": cost_usd,
            "metadata": metadata or {},
        }
        self._append_to_log(entry)

    def project_total(self, project_id: str) -> float:
        return sum(e["cost_usd"] for e in self._read_log()
                  if e["project_id"] == project_id)
```

Cost dashboards (project total, lifetime total, per-service breakdown) are surfaced in the UI from this log.

### 16.3 Health Check

```python
@router.get("/api/health")
async def health():
    return {
        "status": "ok",
        "version": __version__,
        "active_jobs": len(JobOrchestrator.running),
        "project_count": ProjectStore.count(),
        "api_keys_configured": {
            "anthropic": bool(SettingsStore.load().get("anthropic_api_key")),
            "openai": bool(SettingsStore.load().get("openai_api_key")),
            "falai": bool(SettingsStore.load().get("falai_api_key")),
        }
    }
```

---

## 17. Deployment Topology

### 17.1 MVP — Local Single Process

```
User's Machine (Windows / Mac / Linux)
├── Python 3.11+ venv
├── IRdeo package installed
├── FastAPI process (uvicorn) listening on localhost:8000
├── Local filesystem: ~/IRdeo_Projects/
├── Settings: ~/.irdeo/settings.json (encrypted)
├── Logs: ~/.irdeo/logs/
└── Browser tab open to http://localhost:8000
```

**Startup:** `irdeo start` command launches uvicorn, opens browser. `irdeo stop` shuts down cleanly. The user runs the tool when they want to use it; it's not a daemon.

**Cross-platform:** Windows is primary (Rasti's OS); macOS and Linux work identically because Python + browser is platform-neutral.

### 17.2 SaaS — Cloud Multi-Tenant

```
Cloud (AWS / GCP / Render)
├── Load balancer (HTTPS termination)
├── FastAPI fleet (horizontally scaled)
├── Job worker fleet (Celery or equivalent)
├── Redis (job queue + cache)
├── PostgreSQL (user accounts, project metadata)
├── Object storage (S3/GCS): per-user project files
├── Secrets vault (AWS Secrets Manager): user API keys
├── CDN: frontend assets
└── Observability: Datadog/Sentry/CloudWatch
```

The frontend code (HTML/CSS/JS) is identical between MVP and SaaS. The backend has a configuration switch that selects local vs cloud stores:

```python
# config.py
class Config:
    DEPLOYMENT_MODE = os.getenv("IRDEO_MODE", "local")  # "local" | "saas"

    @property
    def project_store(self):
        if self.DEPLOYMENT_MODE == "local":
            return LocalProjectStore(Path.home() / "IRdeo_Projects")
        return CloudProjectStore(s3_bucket="irdeo-projects")

    @property
    def settings_store(self):
        if self.DEPLOYMENT_MODE == "local":
            return LocalSettingsStore()
        return VaultedSettingsStore(vault_client=secrets_manager)
```

---

## 18. Security Model

### 18.1 MVP Threat Model

- **Attacker:** Other users on the same machine, or malware running with user privileges
- **Asset:** API keys, generated content, conversation history
- **Defense:** Filesystem permissions (`0600` on settings file), Fernet encryption at rest
- **Out of threat model:** Root-level compromise of user's machine, physical access to encrypted disk

### 18.2 SaaS Threat Model

- **Attacker:** External attackers, malicious users, insiders
- **Asset:** All MVP assets + payment info + other users' content
- **Defense:**
  - HTTPS everywhere (no plaintext over network)
  - User data isolated by tenant ID at the storage layer
  - API keys stored in secrets vault, never logged or surfaced to other tenants
  - Authentication via OAuth (no password handling)
  - Rate limiting per tenant
  - Audit log of all sensitive operations
  - Encryption at rest for all stored content

### 18.3 BYO Keys Discipline

User API keys are **only** used to call user-authorized services. The system never:
- Logs API keys
- Sends them to telemetry
- Includes them in error reports
- Embeds them in generated artifacts (the `.docx` output, the Filmora project file, etc.)
- Shares them across users (SaaS) — strict tenant isolation

---

## 19. Versioning & Compatibility

### 19.1 Internal Data Versions

Manifests carry a version number:

```python
@dataclass
class ChunkManifest:
    schema_version: int  # increments when shape changes
    project_id: str
    chunks: list[ChunkDefinition]
    ...
```

When the system reads a manifest with an older `schema_version`, it runs migration logic:

```python
def load_manifest(path: Path) -> ChunkManifest:
    raw = json.loads(path.read_text())
    return Migrations.upgrade_manifest(raw)
```

### 19.2 KB Version Pinning

A project records the KB version it was created with:

```python
@dataclass
class ProjectMetadata:
    irdeo_version: str
    kb_version: str  # e.g., "1.2"
    created_at: datetime
    ...
```

This is informational. The KB is loaded as system-prompt text every turn, so KB upgrades take effect immediately for all projects. Recording the version helps with debugging ("this project was created when KB v1.1 had different performer rules").

### 19.3 API Version

The FastAPI app exposes its version at `/api/version`. The frontend can detect mismatches and prompt the user to refresh.

---

## 20. Performance Considerations

### 20.1 Hot Paths

| Operation | Frequency | Latency target |
|-----------|-----------|----------------|
| Chat turn (text) | Per user message | < 10 sec (LLM-bound) |
| Voice transcription | Per voice message | < 5 sec |
| File upload (MP3 ~5MB) | Once per song | < 3 sec |
| Audio analysis | Once per song | < 5 min |
| Single subchunk generation | ~50–80 per song | 30–90 sec (API-bound) |
| Single lip sync | ~5–10 per song | 30–60 sec (API-bound) |
| Filmora project write | Once per song | < 5 sec |
| docx write | Once per song | < 5 sec |

### 20.2 Bottlenecks

Generation throughput is API-bound — fal.ai response time dominates. The system parallelizes subchunk generations within the bounds enforced by `FalAiProvider`'s `asyncio.Semaphore` (Section 10.2), defaulting to 3 concurrent calls across the entire process. This prevents the 429-cascade failure mode that would otherwise occur when many subchunks fire simultaneously.

Concrete bound: with `falai_max_concurrent = 3` and a typical 5-minute song (~50 subchunk generations + 5–10 lip sync calls), the total wall-clock time for generation runs roughly:
- (50 subchunks × 60 sec average) / 3 concurrent = ~17 minutes for video
- (10 lip sync × 45 sec average) / 3 concurrent = ~2.5 minutes for lip sync
- Plus serial overhead (vocal isolation ~10 sec, preview ~5 sec)
- **Total: ~20–25 minutes** of generation per song under normal conditions

If users observe consistent 429s, they lower `falai_max_concurrent` to 2 or 1. If fal.ai's actual rate limit is higher than 3 (to be verified during integration testing), users can raise it for faster runs.

### 20.3 Memory Footprint

- KB markdown: ~30KB stable
- Song Context (per project): 50KB–200KB
- Conversation history: grows linearly, target < 500KB before summarization triggers
- Audio analysis: < 5MB per song
- Video clips on disk: 100–500MB per song

Memory usage well under 1GB for active operation. Disk usage per finished song: 500MB–2GB.

---

## 21. Build & Project Structure

### 21.1 Repository Layout

```
irdeo/
├── README.md
├── pyproject.toml
├── requirements.txt
├── requirements-dev.txt
├── docs/
│   ├── IRdeo_Knowledge_Base.md
│   ├── IRdeo_BRD.md
│   ├── IRdeo_Architecture.md
│   ├── archive/
│   │   └── IRONRUST_VIDEO_STUDIO_HANDOFF.md
│   └── (more specs as written)
├── irdeo/
│   ├── __init__.py
│   ├── __main__.py
│   ├── version.py
│   ├── config.py
│   ├── pricing.py
│   ├── api/
│   │   ├── __init__.py
│   │   ├── app.py
│   │   └── routes/
│   │       ├── chat.py
│   │       ├── voice.py
│   │       ├── projects.py
│   │       ├── analysis.py
│   │       ├── timestamps.py
│   │       ├── chunks.py
│   │       ├── generation.py
│   │       ├── preview.py
│   │       ├── lipsync.py
│   │       ├── export.py
│   │       ├── jobs.py
│   │       ├── settings.py
│   │       └── health.py
│   ├── services/
│   │   ├── conversation_service.py
│   │   ├── analysis_service.py
│   │   ├── timestamps_service.py
│   │   ├── chunk_service.py
│   │   ├── prompt_gen_service.py
│   │   ├── generation_service.py
│   │   ├── lipsync_service.py
│   │   ├── output_service.py
│   │   ├── preview_service.py
│   │   ├── prompt_pack_service.py
│   │   ├── vocal_isolation_service.py
│   │   ├── reference_service.py
│   │   ├── job_orchestrator.py
│   │   ├── cost_estimator.py
│   │   ├── providers/
│   │   │   ├── base.py
│   │   │   ├── generation/
│   │   │   │   ├── base.py
│   │   │   │   └── fal_ai.py
│   │   │   └── vocal_isolation/
│   │   │       ├── base.py
│   │   │       ├── lalai.py
│   │   │       └── passthrough.py
│   ├── storage/
│   │   ├── project_store.py
│   │   ├── media_store.py
│   │   ├── knowledge_store.py
│   │   ├── settings_store.py
│   │   ├── conversation_store.py
│   │   └── cost_tracker.py
│   ├── models/
│   │   ├── project.py
│   │   ├── chunk.py
│   │   ├── subchunk.py
│   │   ├── job.py
│   │   ├── settings.py
│   │   └── cost.py
│   ├── core/
│   │   ├── retry.py
│   │   ├── errors.py
│   │   ├── token_budget.py
│   │   └── canonical_formatter.py
│   ├── cli/
│   │   ├── __init__.py
│   │   └── commands.py
│   └── frontend/
│       ├── index.html
│       ├── style.css
│       ├── app.js
│       └── components/
├── knowledge/
│   ├── IRdeo_Knowledge_Base.md  (symlink or copy of docs/)
│   └── references/
│       └── (artist profile JSON files)
└── tests/
    ├── conftest.py
    ├── unit/
    └── integration/
```

### 21.2 Entry Points

```python
# irdeo/__main__.py
def main():
    parser = argparse.ArgumentParser()
    sub = parser.add_subparsers(dest="command")
    sub.add_parser("start", help="Start IRdeo local web server")
    sub.add_parser("stop", help="Stop running IRdeo server")
    sub.add_parser("doctor", help="Diagnose configuration issues")
    args = parser.parse_args()
    if args.command == "start":
        from .api.app import run_server
        run_server()
    elif args.command == "doctor":
        from .cli.commands import doctor
        doctor()
```

`irdeo start` does:
1. Validates settings (API keys present? output dir writable?)
2. **Checks ffmpeg is on PATH** — required for PreviewService (Section 11.4). Fails with actionable error if missing.
3. Starts uvicorn on `localhost:8000`
4. Opens `http://localhost:8000` in default browser
5. Logs to console + file

`irdeo doctor` does:
- All `start` validations without actually starting the server
- API key validation (try a cheap call to each provider, confirm credentials work)
- ffmpeg presence and version check
- Output directory permissions
- Disk space check
- Reports each item with ✓ or ✗ + suggestion

### System Prerequisites

Beyond Python dependencies, IRdeo requires:
- **ffmpeg** on PATH — for preview stitching (Section 11.4). Cross-platform binary, user installs separately. Install instructions:
  - Windows: `winget install ffmpeg` or download from ffmpeg.org
  - macOS: `brew install ffmpeg`
  - Linux: `apt install ffmpeg` / equivalent
- **Modern web browser** — Chrome, Firefox, Safari, or Edge (last 2 major versions)
- **Network connectivity** to: api.anthropic.com, api.openai.com, fal.ai, www.lalal.ai

### 21.3 Dependency Injection

Services accept their dependencies via constructor injection. This makes testing trivial (swap real clients for mocks) and supports swapping providers (real fal.ai vs test stub):

```python
# irdeo/api/app.py
def build_app(config: Config) -> FastAPI:
    app = FastAPI()
    # Wire dependencies
    knowledge_store = KnowledgeStore()
    settings = SettingsStore()
    claude_client = anthropic.AsyncAnthropic(api_key=settings.load()["anthropic_api_key"])
    whisper_client = openai.AsyncOpenAI(api_key=settings.load()["openai_api_key"])
    fal_provider = FalAiProvider(api_key=settings.load()["falai_api_key"])

    project_store = LocalProjectStore(config.output_directory)
    event_bus = InMemoryEventBus()
    job_orchestrator = JobOrchestrator(project_store, event_bus)

    conversation = ConversationService(claude_client, knowledge_store, project_store)
    analysis = AnalysisService(whisper_client, claude_client)
    # ... wire others

    # Mount routes with dependencies
    app.include_router(chat_router(conversation))
    # ... others

    return app
```

---

## 22. Testing Strategy

### 22.1 Unit Tests

Each service has unit tests with all external dependencies mocked. Coverage targets:
- All service classes: 80%+ coverage
- Critical logic (token budget, retry policy, prompt assembly): 95%+

### 22.2 Integration Tests

Integration tests cover full pipelines with stubbed external APIs:
- Full song workflow: upload MP3 → analyze → finalize → chunk → generate (stubbed) → lip sync (stubbed) → export
- Conversation flows: user provides various inputs, assert correct KB rule application

### 22.3 Real-API Smoke Tests

A separate test suite runs against real fal.ai/Claude/Whisper APIs. Gated behind environment variable (`IRDEO_REAL_API_TESTS=1`) to control cost. Run weekly + before releases.

### 22.4 Knowledge Base Tests

The KB itself gets test coverage — synthetic conversation transcripts that assert the AI Director follows KB rules:
- Given a Slovak ballad genre, performer choices follow Section 7.2
- Given a performer-present chunk, lip sync is mandated per Section 8
- Given a user override of a KB rule, the assistant acknowledges and confirms per Section 0

---

## 23. Open Architecture Questions

These need resolution during build but don't block this document being approved:

- **AQ-1** Should the system pool LLM API connections, or create per-request? Likely per-request for MVP simplicity.
- **AQ-2** Should chunked uploads be supported for large MP3 files? Probably not for MVP (5MB files are fine in one shot).
- **AQ-3** Should the system pre-warm the LLM with the KB on startup to reduce first-turn latency? Worth testing.
- **AQ-4** Should generated videos be transcoded to a uniform format (H.264, 1080p) before Filmora import, or passed through as fal.ai returns them? Investigation needed.
- **AQ-5** When the Filmora `.wfp` format is investigated (BRD Q2), the output writer needs concrete spec. Defer until that investigation completes.
- **AQ-6** SaaS migration path for project file storage: lazy-migrate user's local projects to cloud on first SaaS login, or fresh start? UX decision.
- **AQ-7** ChromaDB for KB embedding or plain markdown loading? Plain markdown is simpler at current KB size; ChromaDB becomes useful when reference profile library grows large.
- **AQ-8** ~~Job concurrency limit per project — how many subchunk generations run in parallel?~~ **RESOLVED v1.1: explicit `asyncio.Semaphore(3)` in FalAiProvider (Section 10.2), tunable via `settings.falai_max_concurrent`. Verify the 3-call default against real fal.ai limits during integration testing; adjust if higher concurrency is allowed.**
- **AQ-9** Lalal.ai cost validation at song volume — verify estimated $0.0019/sec ($10/90min) holds in practice across mixed audio types. If Lalal.ai cost dominates per-song spend disproportionately, evaluate cheaper alternatives (Replicate-hosted Demucs).
- **AQ-10** Lalal.ai engine selection — Phoenix vs Orion. Test both on representative IRdeo songs (rap with heavy beat, atmospheric ballad with vocals over piano) to determine default. May vary by genre.
- **AQ-11** Preview re-encoding tradeoff — preview currently uses `-c copy` (no re-encoding, fastest) but assumes all chunk clips share compatible codecs from fal.ai. If clips come back with mismatched encoding parameters, ffmpeg concat may fail or produce corrupted preview. May need to add a normalize-encoding step before concat. Test during integration.
- **AQ-12** ffmpeg version requirements — minimum supported version. Likely 4.x+ but verify the specific filters/options used in PreviewService work on older versions still common on user machines.

---

## 24. Provenance

This Architecture Document was derived from:
- Knowledge Base v1.2 (`IRdeo_Knowledge_Base.md`) — domain rules
- BRD v1.3 (`IRdeo_BRD.md`) — requirements and scope
- PASC AI Assistant architecture (`AI_Assistant_Technical_Architecture.md`) — proven reference pattern for FastAPI + ChromaDB + Whisper + LLM + browser frontend
- Sessions 04, 05, 06, 07 — manual production experience informing component design
- Direct discussion with Rasti throughout the spec process

Every architectural decision is grounded in either an explicit BRD requirement, a KB rule, a lesson from past production, or a clearly-named architectural principle in Section 2. Speculative choices are flagged in Section 23 (Open Architecture Questions).
