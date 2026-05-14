# IRdeo — API Integration Specification
**Version:** 1.0
**Last updated:** May 11, 2026
**Status:** LIVING DOCUMENT — comprehensive API integration reference
**Companion documents:** `IRdeo_Knowledge_Base.md` (v1.3+), `IRdeo_BRD.md` (v1.4+), `IRdeo_Architecture.md` (v1.2+), `IRdeo_UI_Spec.md` (v1.0+), `IRdeo_Data_Model.md` (v1.0+)

### Versioning Convention
- **v1.0** — first complete formal version
- **v1.1, v1.2, ...** — substantive revisions and additions
- **v2.0** — major restructure or fundamental approach change

### Changelog
- **v1.0 (May 11, 2026)** — First complete formal version. Defines integration with Anthropic Claude, OpenAI Whisper, fal.ai (Kling, Veo, Sync.so), and Lalal.ai. Centralized key management. Comprehensive security model. Per-provider API surfaces, authentication, rate limits, error handling, retry semantics. Built on top of KB v1.3, BRD v1.4, Architecture v1.2, UI Spec v1.0, Data Model v1.0.

---

## 1. Document Overview

### 1.1 Purpose
This document specifies how IRdeo integrates with external API providers. It covers authentication patterns, request/response shapes, model identifiers, pricing, rate limits, error semantics, retry policies, and — critically — the security and key management discipline that prevents API credentials from being exposed or leaked.

This document is the source of truth for all external API integration code. Implementers (CLI agents or humans) should not need to consult provider docs for the basics; provider docs are referenced for edge cases and future evolution.

### 1.2 Scope
- **In scope:** All four external service integrations (Anthropic Claude, OpenAI Whisper, fal.ai, Lalal.ai), centralized key management, security model, request/response patterns, error handling, retry policies, development workflow, sign-up instructions, SaaS evolution path
- **Out of scope:** Detailed internal architecture (Architecture Doc), data persistence (Data Model Spec), UI design (UI Spec), provider-side billing UX (provider's own concern)

### 1.3 Audience
- **Primary:** Build implementers wiring external APIs
- **Secondary:** Operators managing keys in production
- **Tertiary:** Future contributors adding new provider integrations

### 1.4 Provider Catalog at a Glance

| Provider | Used For | Auth Method | Pricing Model | SDK Available |
|----------|----------|-------------|---------------|---------------|
| **Anthropic Claude** | Conversation LLM, content classification, prompt generation | API key (header) | Per-token (input/output) | `anthropic` (Python) |
| **OpenAI Whisper** | Audio transcription | API key (bearer) | Per-minute audio | `openai` (Python) |
| **fal.ai** | Video generation (Kling, Veo) + Lip sync (Sync.so) | API key (header) | Per-second of output | `fal-client` (Python) |
| **Lalal.ai** | Vocal stem isolation | License key (header) | Per-second processed, credit-based | None (REST) |

All four are accessed via user-provided API keys — IRdeo never proxies through a shared account (the BYO-keys principle from Architecture Section 2.8).

### 1.5 Related Documents
- `IRdeo_Knowledge_Base.md` — Domain rules
- `IRdeo_BRD.md` — Functional requirements driving each integration
- `IRdeo_Architecture.md` — Service-layer context (Section 10 for fal.ai, Section 5 for Whisper, Section 4 for Claude, Section 10.5 for Lalal.ai)
- `IRdeo_Data_Model.md` — Cost tracking schema (Section 13), settings schema (Section 14)
- Provider documentation URLs (referenced throughout)

---

## 2. Centralized Key Management

### 2.1 Single Source of Truth

ALL API keys for IRdeo live in exactly one place at runtime: the encrypted settings file at `~/.irdeo/settings.json` (Architecture Section 13.1, Data Model Section 14). Application code retrieves keys only through the SettingsStore interface. Keys are NEVER:

- Hardcoded in source code
- Stored in environment variables that persist across reboots
- Written to log files
- Sent to telemetry endpoints
- Embedded in generated artifacts (`.docx`, `.zip`, Filmora project files)
- Included in error reports or stack traces
- Shared across user accounts (SaaS)

```python
# RIGHT — retrieve via SettingsStore
class ClaudeClient:
    def __init__(self, settings_store: SettingsStore):
        settings = settings_store.load()
        self._client = anthropic.AsyncAnthropic(api_key=settings.anthropic_api_key)

# WRONG — never do this
import os
client = anthropic.AsyncAnthropic(api_key=os.environ["ANTHROPIC_API_KEY"])  # NO

# WRONG — never do this
client = anthropic.AsyncAnthropic(api_key="sk-ant-xxxxx")  # NO
```

### 2.2 Development Workflow with `.env`

For developer convenience during local development ONLY (not for end users), a `.env` file can be used to populate the SettingsStore on first run:

```bash
# .env — for developer setup ONLY, never committed
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
FALAI_API_KEY=fal_...
LALAI_API_KEY=lalai_...
```

A bootstrap script reads `.env` once and writes the encrypted `settings.json`. After that, `.env` should be deleted or rotated. The SettingsStore is the only place that reads keys at runtime.

```python
# irdeo/cli/commands.py
def bootstrap_from_env():
    """Developer-only convenience: populate settings from .env on first run."""
    from dotenv import load_dotenv
    load_dotenv()
    settings_store = SettingsStore()
    current = settings_store.load() or {}

    for env_key, settings_key in [
        ("ANTHROPIC_API_KEY", "anthropic_api_key"),
        ("OPENAI_API_KEY", "openai_api_key"),
        ("FALAI_API_KEY", "falai_api_key"),
        ("LALAI_API_KEY", "lalai_api_key"),
    ]:
        if value := os.environ.get(env_key):
            current[settings_key] = value

    settings_store.save(current)
    print("Settings populated from .env. You can now delete the .env file.")
```

**End users never touch `.env`.** They enter API keys via the Settings drawer in the local web UI (UI Spec Section 3.6).

### 2.3 `.gitignore` Discipline

The repository root MUST contain a `.gitignore` that blocks all credential-bearing files:

```gitignore
# Credentials — NEVER commit
.env
.env.*
*.key
*.pem
secrets/
.irdeo/

# Encrypted settings (still don't commit even though encrypted)
settings.json
~/.irdeo/

# Build artifacts that might capture state
*.log
__pycache__/
.pytest_cache/
dist/
build/

# IDE / OS
.vscode/
.idea/
.DS_Store
Thumbs.db
```

This file is committed (it's the protection itself); the files it blocks are not.

### 2.4 Pre-Commit Hooks

To prevent accidental commits of API keys, IRdeo's repository includes a pre-commit hook:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
```

These scan staged changes for high-entropy strings, known credential patterns, and provider-specific key formats. The commit fails if anything looks like a credential.

### 2.5 Key Validation on Save

When the user enters a key in the Settings UI, IRdeo validates it before saving:

```python
class KeyValidator:
    """Verify a key works before persisting it."""

    @staticmethod
    async def validate_anthropic(key: str) -> tuple[bool, str]:
        try:
            client = anthropic.AsyncAnthropic(api_key=key)
            # Cheapest possible call — just a token check
            await client.messages.create(
                model="claude-haiku-4-5-20251001",
                max_tokens=1,
                messages=[{"role": "user", "content": "."}],
            )
            return True, "Key valid"
        except anthropic.AuthenticationError:
            return False, "Invalid API key"
        except anthropic.RateLimitError:
            return True, "Key valid (rate limited; key works)"
        except Exception as e:
            return False, f"Validation failed: {e}"

    @staticmethod
    async def validate_openai(key: str) -> tuple[bool, str]:
        try:
            client = openai.AsyncOpenAI(api_key=key)
            # List models is a cheap auth check
            await client.models.list()
            return True, "Key valid"
        except openai.AuthenticationError:
            return False, "Invalid API key"
        except Exception as e:
            return False, f"Validation failed: {e}"

    @staticmethod
    async def validate_falai(key: str) -> tuple[bool, str]:
        # fal.ai doesn't have a free auth-only endpoint; check format
        if not key.startswith("fal_") and ":" not in key:
            return False, "Key format looks wrong (expected fal_... or key:secret)"
        # Optionally: hit a low-cost endpoint to verify
        return True, "Key format valid (full validation deferred until first use)"

    @staticmethod
    async def validate_lalai(key: str) -> tuple[bool, str]:
        try:
            async with httpx.AsyncClient() as client:
                resp = await client.get(
                    "https://www.lalal.ai/api/limit/",
                    headers={"Authorization": f"license {key}"},
                )
                if resp.status_code == 200:
                    return True, f"Key valid; credits remaining: {resp.json().get('balance', 'unknown')}"
                return False, f"Auth failed: {resp.status_code}"
        except Exception as e:
            return False, f"Validation failed: {e}"
```

Failed validation surfaces in the UI with the error message; the key is not saved.

---

## 3. Security Model

### 3.1 Threat Model — MVP (Local)

**Attackers:**
- Other users on the same machine
- Malware running with user privileges
- Casual filesystem inspection (someone glancing at the user's drive)

**Assets:**
- API keys (financial exposure if abused)
- Generated content (creative property)
- Conversation history (potentially personal)

**Defenses:**
- Fernet encryption at rest for settings file (Architecture Section 13.1)
- Filesystem permissions: `0600` on settings file AND key file
- Settings stored at `~/.irdeo/settings.json`; encryption key at `~/.irdeo/.key`
- Both files owned by the user, readable only by the user

**Out of MVP threat model:**
- Root-level machine compromise (game over regardless)
- Physical access to encrypted disk (system-level concern)
- Side-channel attacks on memory

### 3.2 Threat Model — SaaS (Cloud, Phase 2)

**Attackers:**
- External attackers (network)
- Malicious users (other tenants)
- Insiders (IRdeo operators)

**Assets:**
- All MVP assets, PLUS:
- Payment information (Stripe-handled, never touches IRdeo storage)
- Cross-user content (must remain isolated)

**Defenses:**
- TLS everywhere (HTTPS-only, no plaintext over network)
- User data isolated by `user_id` at storage layer
- API keys stored in dedicated secrets vault (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager)
- Vault retrieves keys per-request; never cached in plaintext beyond request scope
- All authentication via OAuth (Google, GitHub) — IRdeo never handles passwords
- Rate limiting per tenant (prevents one user from exhausting fal.ai concurrency for others)
- Audit log of all sensitive operations (key access, settings changes)
- Encrypted at rest for all stored content

### 3.3 Key Rotation Discipline

**MVP:**
- User-initiated rotation via Settings UI: replace key, validate, save (overwrites old)
- Old key is overwritten in place — no key history retained
- If user suspects compromise, they rotate at the provider (revoke + new key) and update IRdeo settings

**SaaS:**
- Same user-initiated flow
- Plus: scheduled mandatory rotation reminders (every N months, configurable)
- Plus: audit log captures all rotation events

### 3.4 What IRdeo Never Logs

The structured logger (Architecture Section 16.1) has an enforced blocklist:

```python
# core/logging.py
SENSITIVE_PATTERNS = [
    r'sk-ant-[a-zA-Z0-9-_]+',      # Anthropic keys
    r'sk-[a-zA-Z0-9]{32,}',         # OpenAI keys
    r'fal_[a-f0-9]{32,}',           # fal.ai keys
    r'fal-[a-zA-Z0-9_-]+:[a-zA-Z0-9_-]+',  # fal.ai key:secret format
]

class SafeLogger:
    @staticmethod
    def sanitize(text: str) -> str:
        for pattern in SENSITIVE_PATTERNS:
            text = re.sub(pattern, '[REDACTED_KEY]', text)
        return text

    @staticmethod
    def log(level: str, event: str, **kwargs):
        sanitized_kwargs = {k: SafeLogger.sanitize(str(v)) for k, v in kwargs.items()}
        structlog.get_logger().log(level, event, **sanitized_kwargs)
```

Even if a developer accidentally tries to log a key, the sanitizer redacts it before it hits the log file.

### 3.5 Error Reporting Discipline

When errors are reported (logs, UI, telemetry), any stack trace or HTTP response body that might contain keys is sanitized through the same `SafeLogger.sanitize()` filter. The UserError class (Architecture Section 15.3) never includes raw provider response bodies in messages shown to the user.

### 3.6 Network Egress Allowlist (Future)

For maximum paranoia (likely SaaS phase, optional for advanced MVP users), the application can be deployed behind a network egress allowlist permitting connections only to:

- `api.anthropic.com`
- `api.openai.com`
- `*.fal.ai`, `fal.run`
- `www.lalal.ai`, `api.lalal.ai`

This prevents any code path from exfiltrating credentials to a malicious endpoint, even if other security layers fail.

---

## 4. Anthropic Claude Integration

### 4.1 Use Cases in IRdeo

| Use Case | Service | Model |
|----------|---------|-------|
| Conversation LLM (AI Director) | ConversationService | `claude-opus-4-7` |
| Content classification (sections, vocal types, mood) | AnalysisService | `claude-opus-4-7` |
| Lyrics-audio sanity check supplementary reasoning | AnalysisService | `claude-haiku-4-5-20251001` (cheap) |
| Chunk-level prompt generation | PromptGenService | `claude-opus-4-7` |
| Subchunk decomposition | PromptGenService | `claude-opus-4-7` |
| Conversation history summarization | ConversationService | `claude-opus-4-7` |

The user may override the default model via Settings (`claude_model`).

### 4.2 Authentication

```python
import anthropic

client = anthropic.AsyncAnthropic(api_key=settings.anthropic_api_key)
```

The SDK handles the `x-api-key` header and the `anthropic-version` header automatically. Use the async client (`AsyncAnthropic`) for compatibility with FastAPI's async handlers.

### 4.3 Endpoint and Request Shape

**Endpoint:** `POST https://api.anthropic.com/v1/messages` (handled by SDK)

**Headers:**
- `x-api-key: {api_key}`
- `anthropic-version: 2023-06-01`
- `content-type: application/json`

**Request body:**
```json
{
  "model": "claude-opus-4-7",
  "max_tokens": 4096,
  "system": "...IRdeo Knowledge Base + Song Context...",
  "messages": [
    {"role": "user", "content": "Let's start with the song"},
    {"role": "assistant", "content": "Got it! Upload the MP3..."},
    {"role": "user", "content": "Uploaded. What's next?"}
  ]
}
```

### 4.4 Response Shape

```json
{
  "id": "msg_01abc...",
  "type": "message",
  "role": "assistant",
  "content": [
    {"type": "text", "text": "Excellent. Now — what genre is this song?"}
  ],
  "model": "claude-opus-4-7",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 4827,
    "output_tokens": 142
  }
}
```

The `usage` object drives cost tracking (Section 11).

### 4.5 Rate Limits

Anthropic uses tier-based rate limits that scale with usage. Refer to current Anthropic documentation (https://docs.claude.com/en/api/rate-limits) for exact numbers; they're tier-dependent and evolve over time.

For IRdeo's expected MVP load (one user, sequential conversations):
- Default tier limits are well within IRdeo's needs
- One conversation turn ≈ 1 request, ~5K input tokens, ~500 output tokens
- A 60-minute production session ≈ 30-60 turns
- Well under any tier limit

If 429 is encountered: retry per Section 9.2 (exponential backoff).

### 4.6 Pricing (Verify Current Rates at docs.claude.com)

Current pricing model: per-million-tokens for input and output, separate rates. Examples (verify current values):

- Claude Opus 4.7: ~$15 input / ~$75 output per million tokens
- Claude Haiku 4.5: significantly cheaper, ~$1 input / ~$5 output per million tokens

Cost per conversation turn for IRdeo: typically $0.05–$0.30 depending on KB+context size.

The pricing constants live in `irdeo/pricing.py` (Architecture Section 14.1) and must be updated when Anthropic changes rates.

### 4.7 Service Implementation

```python
# services/conversation_service.py
import anthropic
from anthropic import (
    AuthenticationError, RateLimitError, APITimeoutError,
    APIConnectionError, APIStatusError
)

class ConversationService:
    def __init__(self, settings_store, knowledge_store, project_store):
        settings = settings_store.load()
        if not settings.get("anthropic_api_key"):
            raise UserError(
                "Anthropic API key not configured.",
                suggestion="Go to Settings to add your Anthropic API key."
            )
        self.client = anthropic.AsyncAnthropic(
            api_key=settings["anthropic_api_key"]
        )
        self.model = settings.get("claude_model", "claude-opus-4-7")
        self.knowledge = knowledge_store
        self.projects = project_store

    async def turn(self, project_id: str, user_message: str) -> dict:
        system_prompt = self._build_system_prompt(project_id)
        messages = self._build_messages(project_id, user_message)

        try:
            response = await self.client.messages.create(
                model=self.model,
                max_tokens=4096,
                system=system_prompt,
                messages=messages,
            )
        except AuthenticationError:
            raise UserError(
                "Anthropic API key is invalid or expired.",
                suggestion="Check Settings; you may need to rotate the key."
            )
        except RateLimitError as e:
            # Caller handles retry; raise structured error
            raise TransientAPIError("anthropic", "rate_limit", retry_after=e.response.headers.get("retry-after"))
        except APITimeoutError:
            raise TransientAPIError("anthropic", "timeout")
        except APIConnectionError:
            raise TransientAPIError("anthropic", "connection")
        except APIStatusError as e:
            if e.status_code >= 500:
                raise TransientAPIError("anthropic", f"server_{e.status_code}")
            raise UserError(f"Anthropic API error: {e.status_code}")

        # Extract text and log cost
        text = response.content[0].text
        await self._record_cost(project_id, response.usage)

        # Append to conversation history
        self.projects.append_turn(project_id, "user", user_message)
        self.projects.append_turn(project_id, "assistant", text,
                                  metadata={"model_used": self.model})

        return {
            "reply": text,
            "usage": {
                "input_tokens": response.usage.input_tokens,
                "output_tokens": response.usage.output_tokens,
            }
        }
```

### 4.8 Conversation Context Management

The system prompt (KB + Song Context) can grow large. Per Architecture Section 4.4, IRdeo tracks token usage and triggers summarization when approaching context limits.

Claude Opus 4.x has a context window of 200K tokens. IRdeo reserves:
- ~30K for KB
- ~30K for Song Context (varies)
- ~4K for response
- ~10K safety margin

Leaving ~126K for conversation history. Summarization triggers when conversation history alone exceeds 100K tokens.

### 4.9 Streaming (Future Enhancement)

The Anthropic SDK supports streaming responses for lower perceived latency. Not implemented in MVP, but the architecture allows easy upgrade — the response handler would consume server-sent events instead of awaiting the full response. Tracked as AQ-Anthropic-1 in Section 12.

---

## 5. OpenAI Whisper Integration

### 5.1 Use Cases in IRdeo

| Use Case | Endpoint |
|----------|----------|
| MP3 transcription (full song) | `/v1/audio/transcriptions` |
| Voice input transcription (microphone) | `/v1/audio/transcriptions` |

### 5.2 Authentication

```python
import openai

client = openai.AsyncOpenAI(api_key=settings.openai_api_key)
```

The SDK uses `Authorization: Bearer {key}` automatically.

### 5.3 Endpoint and Request Shape

**Endpoint:** `POST https://api.openai.com/v1/audio/transcriptions` (handled by SDK)

**Form-data fields:**
- `file` — binary audio file
- `model` — `whisper-1` (only model currently)
- `response_format` — `verbose_json` (we want timestamps and language detection)
- `timestamp_granularities[]` — `segment` (or `word` for higher precision; segment is default and sufficient)
- `language` — optional ISO 639-1 code to hint language (e.g., `en`, `sk`); if omitted, auto-detected

### 5.4 Response Shape (verbose_json)

```json
{
  "task": "transcribe",
  "language": "english",
  "duration": 285.34,
  "text": "Yeah... I used to believe you. Defended you. Repeated what you told me...",
  "segments": [
    {
      "id": 0,
      "seek": 0,
      "start": 0.0,
      "end": 1.84,
      "text": "Yeah...",
      "tokens": [...],
      "temperature": 0.0,
      "avg_logprob": -0.31,
      "compression_ratio": 1.23,
      "no_speech_prob": 0.02
    },
    {
      "id": 1,
      "seek": 0,
      "start": 2.10,
      "end": 5.32,
      "text": "I used to believe you.",
      ...
    }
  ]
}
```

`language` may come back as full name ("english") rather than ISO code; IRdeo normalizes to ISO 639-1 (`en`, `sk`, etc.) for internal use.

### 5.5 File Size Limit

OpenAI's Whisper endpoint enforces a 25 MB file size limit per request. IRdeo's MP3 uploads are typically 4–10 MB for a 5-minute song; well under the limit. For longer audio (rare in music video work), IRdeo would split the file into ≤24 MB chunks before sending; not implemented in MVP since songs don't approach this size.

### 5.6 Pricing

$0.006 per minute of audio input. A 5-minute song costs $0.03 to transcribe — negligible.

### 5.7 Service Implementation

```python
# services/analysis_service.py — Whisper portion
class AnalysisService:
    async def _transcribe(self, mp3_path: Path,
                          language_hint: str | None = None) -> TranscriptionResult:
        """Transcribe via OpenAI Whisper. Handles file open and API call."""
        if not self._openai_client:
            raise UserError(
                "OpenAI API key not configured.",
                suggestion="Go to Settings to add your OpenAI API key (needed for Whisper)."
            )

        try:
            with mp3_path.open("rb") as f:
                kwargs = {
                    "model": "whisper-1",
                    "file": f,
                    "response_format": "verbose_json",
                    "timestamp_granularities": ["segment"],
                }
                if language_hint:
                    kwargs["language"] = language_hint

                resp = await self._openai_client.audio.transcriptions.create(**kwargs)
        except openai.AuthenticationError:
            raise UserError("OpenAI API key is invalid or expired.")
        except openai.RateLimitError:
            raise TransientAPIError("openai", "rate_limit")
        except openai.APITimeoutError:
            raise TransientAPIError("openai", "timeout")

        # Normalize language to ISO 639-1
        lang_normalized = self._normalize_language(resp.language)

        # Record cost
        await self._record_cost(
            service="openai",
            operation="whisper_transcribe",
            input_units=resp.duration,
            input_unit_type="audio_seconds",
            cost_usd=resp.duration * 0.0001,  # $0.006/min ÷ 60 = $0.0001/sec
        )

        return TranscriptionResult(
            text=resp.text,
            segments=[
                {"start": s.start, "end": s.end, "text": s.text}
                for s in resp.segments
            ],
            language=lang_normalized,
            duration=resp.duration,
        )

    def _normalize_language(self, lang_str: str) -> str:
        """Convert 'english' → 'en', 'slovak' → 'sk', etc."""
        mapping = {
            "english": "en", "slovak": "sk", "czech": "cs",
            "spanish": "es", "french": "fr", "german": "de",
            "italian": "it", "portuguese": "pt", "russian": "ru",
            # extend as needed
        }
        return mapping.get(lang_str.lower(), lang_str.lower()[:2])
```

### 5.8 Voice Input Variant

For microphone voice input (UI Spec Section 11.1), the same endpoint is used with a short audio clip (typically 2–30 seconds of speech):

```python
async def transcribe_voice_input(self, audio_bytes: bytes,
                                 mime_type: str = "audio/webm") -> str:
    # Write to temp file (Whisper SDK wants file-like input)
    with tempfile.NamedTemporaryFile(suffix=".webm", delete=False) as tmp:
        tmp.write(audio_bytes)
        tmp_path = Path(tmp.name)

    try:
        with tmp_path.open("rb") as f:
            resp = await self._openai_client.audio.transcriptions.create(
                model="whisper-1",
                file=f,
                response_format="json",  # simpler for voice input
            )
        return resp.text
    finally:
        tmp_path.unlink(missing_ok=True)
```

---

## 6. fal.ai Integration

### 6.1 Use Cases in IRdeo

| Use Case | Model Family | Model IDs (verify at fal.ai) |
|----------|--------------|-------------------------------|
| Performer chunks with reference photo | Kling Motion Pro | `fal-ai/kling-video/v1.6/pro/image-to-video` |
| Documentary chunks (no reference photo) | Kling O3 Pro | `fal-ai/kling-video/v1.6/pro/text-to-video` |
| Premium cinematic chunks (optional) | Veo 3 | `fal-ai/veo3` |
| Fast/budget alternative | Veo 3 Fast | `fal-ai/veo3/fast` |
| Lip sync (default) | Sync.so lipsync-2 | `fal-ai/sync-lipsync/v2` |
| Lip sync (premium quality) | Sync.so lipsync-2-pro | `fal-ai/sync-lipsync/v2-pro` |
| Lip sync (4K with obstruction detection) | Sync.so sync-3 | `fal-ai/sync-lipsync/v3` |

Model IDs are placeholders based on fal.ai's URL pattern; verify current model availability and exact paths at https://fal.ai/models when integrating.

### 6.2 Authentication

```python
import fal_client

# Set API key for the entire process
fal_client.api_key = settings.falai_api_key
```

fal.ai accepts either a single string key (`fal_XXXXX`) or a key:secret pair (`fal-XXX:XXX`). The SDK handles either format.

### 6.3 The fal.ai Async Pattern

fal.ai jobs are long-running (30–90 seconds for video generation). The SDK provides three usage patterns:

1. **`subscribe`** — synchronous wait, easy but blocks
2. **`subscribe_async`** — async wait, returns when complete (RECOMMENDED for IRdeo)
3. **`submit` + poll** — fire-and-forget, manual polling (for very long jobs or batch processing)

IRdeo uses `subscribe_async` because:
- Generations complete in <2 minutes (no need for fire-and-forget)
- Async integrates cleanly with FastAPI's async handlers
- Built-in retry on transient failures
- Streaming logs (if `with_logs=True`)

### 6.4 Video Generation Request

```python
result = await fal_client.subscribe_async(
    "fal-ai/kling-video/v1.6/pro/image-to-video",
    arguments={
        "prompt": "Dark intimate room, performer watching TV...",
        "image_url": "https://...",  # or data URL with base64
        "duration": 8,  # seconds
        "aspect_ratio": "16:9",
        "negative_prompt": "blurry, distorted",
    },
    with_logs=True,
    on_queue_update=on_queue_update,  # optional progress callback
)
```

### 6.5 Video Generation Response

```python
{
    "video": {
        "url": "https://fal.media/files/.../output.mp4",
        "content_type": "video/mp4",
        "file_name": "output.mp4",
        "file_size": 4823910
    },
    "seed": 12345,  # for reproducibility (rerun with same seed)
    "timings": {
        "inference": 67.3
    }
}
```

The video URL is temporary (signed URL with expiration). IRdeo MUST download the video immediately and store it locally; URLs expire in ~24 hours.

### 6.6 Reference Photo Upload

For Kling Motion Pro (image-to-video), reference photos must be accessible by fal.ai. Two approaches:

1. **Upload to fal.ai's temporary storage:** use `fal_client.upload_file()` which returns a signed URL valid for the duration of the job
2. **Public URL:** if IRdeo deploys to a host with public URLs (SaaS), the photo can be referenced directly

MVP uses approach 1 (temporary upload):

```python
async def _upload_reference_photo(self, photo_path: Path) -> str:
    """Upload a reference photo to fal.ai; return temporary URL."""
    photo_bytes = photo_path.read_bytes()
    # fal-client's upload helper
    url = await fal_client.upload_async(
        photo_bytes,
        content_type="image/jpeg" if photo_path.suffix == ".jpg" else "image/png",
    )
    return url
```

### 6.7 Lip Sync Request

```python
result = await fal_client.subscribe_async(
    "fal-ai/sync-lipsync/v2",
    arguments={
        "video_url": video_url,    # the generated silent video
        "audio_url": audio_url,    # the vocals-only audio for this subchunk
        # Optional model-specific parameters
        "sync_mode": "loop",
        "model": "sync-1.7.1",
    },
)
```

Both `video_url` and `audio_url` need to be accessible by fal.ai. IRdeo uploads them temporarily as above before calling lip sync.

### 6.8 fal.ai Error Categories

| Error | HTTP | Retry? | Notes |
|-------|------|--------|-------|
| `401 Unauthorized` | 401 | No | Bad API key — UserError, fix in Settings |
| `403 Forbidden` | 403 | No | Account doesn't have access to this model — UserError |
| `429 Too Many Requests` | 429 | Yes | Rate limited — backoff and retry (Section 9) |
| `500 Internal Server Error` | 500 | Yes | Transient; retry up to 3 times |
| `502 Bad Gateway` | 502 | Yes | Transient |
| `503 Service Unavailable` | 503 | Yes | Transient; respect Retry-After if provided |
| `504 Gateway Timeout` | 504 | Yes | Job took too long; retry once, then fail |
| `Content policy violation` | 400 | No | Prompt rejected — UserError, user must edit prompt |
| `Model unavailable` | 503 | Yes | Model may be temporarily down — retry, then fail |
| `Insufficient credits` | 402 | No | Out of account credits — UserError, user must top up |

### 6.9 Concurrency Control (Recap)

Per Architecture Section 10.2, all fal.ai calls go through a single `asyncio.Semaphore` bounding concurrent calls to `settings.falai_max_concurrent` (default 3). This prevents 429 cascades when many subchunks generate in parallel.

### 6.10 Pricing (Verify Current Rates at fal.ai)

Rough estimates (these MUST be verified at integration time):

| Model | Approximate Cost |
|-------|-----------------|
| Kling Motion Pro (image-to-video) | ~$0.40 per 8-second clip |
| Kling O3 Pro (text-to-video) | ~$0.10 per 8-second clip |
| Veo 3 | ~$0.50 per 8-second clip |
| Veo 3 Fast | ~$0.25 per 8-second clip |
| Sync.so lipsync-2 | ~$0.05 per second of output |
| Sync.so lipsync-2-pro | ~$0.083 per second of output |
| Sync.so sync-3 | ~$0.133 per second of output |

Always verify by running a small test job before locking pricing into `pricing.py`.

### 6.11 Service Implementation (Full)

```python
# services/providers/generation/fal_ai.py
import asyncio
import fal_client
import httpx
from pathlib import Path

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
        fal_client.api_key = api_key
        self._semaphore = asyncio.Semaphore(max_concurrent)

    async def generate_video(self, prompt: SubchunkPrompt,
                            reference_image_paths: list[Path] = None) -> bytes:
        async with self._semaphore:
            model = self.MODEL_MAP[prompt.model]
            args = {
                "prompt": prompt.prompt_text,
                "duration": int(prompt.duration_seconds),
                "aspect_ratio": "16:9",
            }

            if reference_image_paths:
                # Upload images temporarily to fal.ai
                image_urls = []
                for path in reference_image_paths:
                    url = await self._upload_temp(path.read_bytes(),
                                                  self._guess_image_mime(path))
                    image_urls.append(url)
                args["image_url"] = image_urls[0]  # Kling takes primary image
                # Some models support multiple image refs; check model docs

            try:
                result = await fal_client.subscribe_async(
                    model,
                    arguments=args,
                    with_logs=True,
                )
            except fal_client.UserError as e:
                # fal.ai's own error class for known issues
                if "content policy" in str(e).lower():
                    raise UserError(
                        f"Prompt rejected by content filter for {prompt.subchunk_id}",
                        suggestion="Try rephrasing the chunk's prompt or visuals."
                    )
                raise
            except Exception as e:
                # Unknown error — let retry policy handle it
                raise TransientAPIError("fal_ai", str(e))

            return await self._download(result["video"]["url"])

    async def lip_sync(self, video_bytes: bytes, audio_bytes: bytes,
                       model: str = "lipsync-2") -> bytes:
        async with self._semaphore:
            model_id = self.LIPSYNC_MAP[model]
            video_url = await self._upload_temp(video_bytes, "video/mp4")
            audio_url = await self._upload_temp(audio_bytes, "audio/mp3")

            try:
                result = await fal_client.subscribe_async(
                    model_id,
                    arguments={
                        "video_url": video_url,
                        "audio_url": audio_url,
                    },
                )
            except Exception as e:
                raise TransientAPIError("fal_ai_lipsync", str(e))

            return await self._download(result["video"]["url"])

    async def _upload_temp(self, content: bytes, content_type: str) -> str:
        """Upload bytes to fal.ai temporary storage; return signed URL."""
        return await fal_client.upload_async(content, content_type=content_type)

    async def _download(self, url: str) -> bytes:
        """Download generated content immediately (URLs expire)."""
        async with httpx.AsyncClient(timeout=120.0) as client:
            resp = await client.get(url)
            resp.raise_for_status()
            return resp.content

    def _guess_image_mime(self, path: Path) -> str:
        ext = path.suffix.lower()
        return {
            ".jpg": "image/jpeg",
            ".jpeg": "image/jpeg",
            ".png": "image/png",
            ".webp": "image/webp",
        }.get(ext, "application/octet-stream")
```

---

## 7. Lalal.ai Integration

### 7.1 Use Case in IRdeo

Vocal stem isolation for performer chunks before lip sync. Per Architecture Section 10.5, Lalal.ai replaces local Demucs to keep the IRdeo process lightweight.

### 7.2 Authentication

License-key-based, header authentication:

```
Authorization: license {license_key}
```

The license key is obtained by signing up at https://www.lalal.ai/api/ and purchasing credits. Unlike Anthropic/OpenAI/fal.ai, Lalal.ai operates on a credit model (you buy a quota of audio-minutes in advance) rather than per-call billing.

### 7.3 API Flow

Lalal.ai requires a multi-step async flow:

```
Upload audio → Start split → Poll status → Download stem
```

1. **POST /api/upload/** — upload audio, get file ID
2. **POST /api/split/** — start vocal/instrumental separation
3. **POST /api/check/** — poll for completion (every 2 seconds)
4. **GET /api/download/** — when complete, download the stem URL returned in check response

### 7.4 Upload Endpoint

**Endpoint:** `POST https://www.lalal.ai/api/upload/`

**Headers:**
- `Authorization: license {key}`

**Body:** raw audio bytes

**Query params:**
- `format` — `mp3`, `wav`, `flac`, `ogg`, `aiff`

**Response:**
```json
{
  "status": "success",
  "id": "abc123def456...",  // file ID for subsequent calls
  "size": 6842150,
  "duration": 285.34,
  "expires": "2026-05-12T14:25:30.000Z"
}
```

Uploaded files expire after ~24 hours.

### 7.5 Split Endpoint

**Endpoint:** `POST https://www.lalal.ai/api/split/`

**Headers:**
- `Authorization: license {key}`

**Body (form-encoded or JSON):**
```json
{
  "id": "abc123def456...",
  "stem": "vocals",      // or "drums", "bass", "piano", "instrumental"
  "splitter": "orion"    // or "phoenix" (both top-tier; orion is newer)
}
```

**Response:**
```json
{
  "status": "success",
  "result": {
    "abc123def456...": {
      "task": {
        "state": "progress",
        "progress": 0
      }
    }
  }
}
```

### 7.6 Check (Poll) Endpoint

**Endpoint:** `POST https://www.lalal.ai/api/check/`

**Headers:**
- `Authorization: license {key}`

**Body (form-encoded):** `id=abc123def456`

**Response (in progress):**
```json
{
  "status": "success",
  "result": {
    "abc123def456...": {
      "task": {
        "state": "progress",
        "progress": 45
      }
    }
  }
}
```

**Response (complete):**
```json
{
  "status": "success",
  "result": {
    "abc123def456...": {
      "task": {
        "state": "success"
      },
      "split": {
        "duration": 285.34,
        "stem_track": "https://api.lalal.ai/download/...stem.wav",
        "back_track": "https://api.lalal.ai/download/...instrumental.wav"
      }
    }
  }
}
```

**Polling discipline:** poll every 2 seconds; typical job completes in 10–30 seconds.

### 7.7 Credit Check Endpoint

**Endpoint:** `GET https://www.lalal.ai/api/limit/`

**Headers:**
- `Authorization: license {key}`

**Response:**
```json
{
  "status": "success",
  "email": "user@example.com",
  "balance": 87.5,   // remaining audio-minutes
  "package": "Starter"
}
```

Used for the key validator (Section 2.5) and the cost dashboard.

### 7.8 Error Categories

| Error | Notes |
|-------|-------|
| `401` | Invalid license key — UserError |
| `402` | No credits remaining — UserError with link to top up |
| `403` | Daily quota exceeded — TransientAPIError (retry tomorrow) or UserError |
| `413` | File too large (>200 MB typically) — UserError |
| `task.state == "error"` | Processing failed — usually a corrupted audio file or unsupported format |

### 7.9 Pricing

Credit-based; current rates approximately $10 per 90 minutes of audio processed (verify at lalal.ai/pricing). IRdeo pays per song:

- 5-minute song × 1 stem (vocals only) = 5 minutes processed = ~$0.56 per song

Negligible relative to video generation cost.

### 7.10 Service Implementation

```python
# services/providers/vocal_isolation/lalai.py
import asyncio
import httpx

class LalaiProvider(VocalIsolationProvider):
    BASE_URL = "https://www.lalal.ai/api"

    def __init__(self, api_key: str, engine: str = "orion"):
        self.api_key = api_key
        self.engine = engine
        self._headers = {"Authorization": f"license {api_key}"}

    async def isolate(self, audio_bytes: bytes,
                     audio_format: str = "mp3") -> bytes:
        async with httpx.AsyncClient(timeout=180.0) as client:
            # Step 1: Upload
            upload_resp = await client.post(
                f"{self.BASE_URL}/upload/",
                headers=self._headers,
                content=audio_bytes,
                params={"format": audio_format},
            )
            upload_resp.raise_for_status()
            upload_data = upload_resp.json()
            if upload_data.get("status") != "success":
                raise TransientAPIError("lalai_upload", str(upload_data))
            file_id = upload_data["id"]

            # Step 2: Start split
            split_resp = await client.post(
                f"{self.BASE_URL}/split/",
                headers={**self._headers, "Content-Type": "application/json"},
                json={
                    "id": file_id,
                    "stem": "vocals",
                    "splitter": self.engine,
                },
            )
            split_resp.raise_for_status()

            # Step 3: Poll for completion
            stem_url = None
            for attempt in range(150):  # 5 minutes max
                check_resp = await client.post(
                    f"{self.BASE_URL}/check/",
                    headers=self._headers,
                    data={"id": file_id},
                )
                check_resp.raise_for_status()
                state_data = check_resp.json()["result"][file_id]
                state = state_data["task"]["state"]

                if state == "success":
                    stem_url = state_data["split"]["stem_track"]
                    break
                if state == "error":
                    raise UserError(
                        "Vocal isolation failed.",
                        suggestion="The audio file may be corrupted or unsupported. Try re-exporting."
                    )

                await asyncio.sleep(2)

            if not stem_url:
                raise TransientAPIError("lalai", "polling_timeout")

            # Step 4: Download stem
            stem_resp = await client.get(stem_url)
            stem_resp.raise_for_status()

            # Record cost
            duration_sec = upload_data["duration"]
            await self._record_cost(
                service="lalai",
                operation="vocal_isolation",
                input_units=duration_sec,
                input_unit_type="input_seconds",
                cost_usd=duration_sec * 0.0019,  # ~$10/90min
                metadata={"engine": self.engine, "file_id": file_id},
            )

            return stem_resp.content

    async def estimate_cost(self, duration_seconds: float) -> float:
        return duration_seconds * 0.0019
```

---

## 8. Cross-Provider Error Categorization

Errors from any external API fall into four categories with different handling:

### 8.1 Authentication Errors (HTTP 401, 403)

**Examples:**
- Bad API key
- Expired/revoked key
- Wrong key for the endpoint
- Insufficient permissions on the account

**Handling:** Raise `UserError` immediately. Do not retry. Surface to UI with "Open Settings" action. Halt the affected job.

### 8.2 Quota / Billing Errors (HTTP 402, 429 with quota signal)

**Examples:**
- Anthropic monthly quota exhausted
- OpenAI account out of credits
- fal.ai account out of funds
- Lalal.ai credit balance zero

**Handling:** Raise `UserError` with provider-specific top-up link. Do not retry — the user must fix it.

### 8.3 Transient Errors (HTTP 429 rate limit, 500-504, network failures)

**Examples:**
- Rate limit hit (recoverable with backoff)
- Provider server temporarily unavailable
- Network blip
- Request timeout

**Handling:** Retry with exponential backoff (Section 9.2). After max retries, surface as job failure with retry option.

### 8.4 Content / Policy Errors (HTTP 400 with content policy signal)

**Examples:**
- Prompt rejected by content filter (rare in IRdeo's domain but possible)
- Audio rejected by Whisper (corrupted file)
- Reference image rejected by Kling (e.g., flagged as celebrity)

**Handling:** Raise `UserError` with specific guidance. Do not retry — the input itself is the problem.

### 8.5 Unified Exception Hierarchy

```python
# core/errors.py

class IRdeoError(Exception):
    """Base for all IRdeo-raised errors."""
    pass

class UserError(IRdeoError):
    """An error the user can act on. Shown in UI."""
    def __init__(self, message: str, suggestion: str = None):
        self.message = message
        self.suggestion = suggestion
        super().__init__(message)

class TransientAPIError(IRdeoError):
    """An external API failed transiently. Eligible for retry."""
    def __init__(self, service: str, error_code: str, retry_after: int = None):
        self.service = service
        self.error_code = error_code
        self.retry_after = retry_after  # optional, from Retry-After header
        super().__init__(f"{service}: {error_code}")

class QuotaExhaustedError(UserError):
    """User's quota/credits with a provider are exhausted."""
    def __init__(self, service: str, top_up_url: str):
        self.service = service
        super().__init__(
            f"{service} quota exhausted",
            suggestion=f"Add credits at {top_up_url}"
        )

class ContentPolicyError(UserError):
    """Provider rejected the content (prompt, image, or audio)."""
    def __init__(self, service: str, content_type: str, detail: str):
        self.service = service
        self.content_type = content_type
        super().__init__(
            f"{service} rejected {content_type}: {detail}",
            suggestion="Adjust the input and try again."
        )
```

---

## 9. Retry Policy

### 9.1 Retry Decision Matrix

| Condition | Action |
|-----------|--------|
| `TransientAPIError` | Retry per backoff schedule (9.2) |
| `UserError` (any subclass) | NEVER retry; surface to user |
| Unknown exception | Log full stack trace; do NOT retry (avoid loops) |
| Total job timeout (e.g., > 1 hour for `generate_song`) | Cancel and mark job failed |

### 9.2 Exponential Backoff with Jitter

```python
# core/retry.py
import asyncio
import random
from typing import Callable, TypeVar, Awaitable

T = TypeVar("T")

class RetryPolicy:
    @staticmethod
    async def with_retry(
        coro_factory: Callable[[], Awaitable[T]],
        max_attempts: int = 3,
        base_delay: float = 1.0,
        max_delay: float = 60.0,
        jitter: bool = True,
    ) -> T:
        """Retry an async operation with exponential backoff.

        Backoff schedule with default params:
            attempt 1: immediate
            attempt 2: ~1 sec delay (1.0 * 2^0)
            attempt 3: ~2 sec delay (1.0 * 2^1)
            (would-be 4): ~4 sec delay (1.0 * 2^2)

        Jitter adds random 0-50% to each delay to avoid thundering herds.
        """
        last_exception = None

        for attempt in range(max_attempts):
            try:
                return await coro_factory()
            except TransientAPIError as e:
                last_exception = e
                if attempt == max_attempts - 1:
                    raise

                # If provider supplied Retry-After, honor it
                if e.retry_after:
                    delay = min(float(e.retry_after), max_delay)
                else:
                    delay = min(base_delay * (2 ** attempt), max_delay)

                if jitter:
                    delay += random.uniform(0, delay * 0.5)

                await asyncio.sleep(delay)
            except (UserError, QuotaExhaustedError, ContentPolicyError):
                raise  # NEVER retry user-actionable errors

        raise last_exception
```

### 9.3 Service-Level Retry Application

Every external API call goes through `RetryPolicy.with_retry`:

```python
async def _generate_with_retry(self, prompt: SubchunkPrompt, refs: list[Path]):
    return await RetryPolicy.with_retry(
        lambda: self.provider.generate_video(prompt, refs),
        max_attempts=3,
        base_delay=2.0,  # video gen is heavier; longer initial delay
    )
```

### 9.4 Job-Level Retry vs. Call-Level Retry

Distinct concerns:

- **Call-level retry** (this section): one failed API call retries 2–3 times
- **Job-level retry** (user action): if the whole job fails, the user can re-trigger via the UI

Call-level retries are silent (logged but not user-facing). Job-level retries are explicit and visible.

---

## 10. Rate Limiting

### 10.1 fal.ai (Most Constrained)

Per Architecture Section 10.2, fal.ai concurrent calls are bounded by `asyncio.Semaphore(settings.falai_max_concurrent)`, default 3. This is the only provider where IRdeo enforces strict client-side throttling.

The semaphore is shared across:
- Video generation calls
- Lip sync calls

so the total concurrent fal.ai workload never exceeds the configured cap. If video gen is using all 3 slots, lip sync waits.

### 10.2 Anthropic / OpenAI / Lalal.ai

These providers have generous rate limits relative to IRdeo's expected load. No client-side throttling needed for MVP. The retry policy handles the occasional 429 transparently.

### 10.3 Tuning Guidance

If users report 429 errors:

1. **fal.ai 429:** Lower `falai_max_concurrent` from 3 → 2 → 1 in Settings
2. **Anthropic 429:** Likely a tier mismatch (user is on free tier with heavy load). Suggest tier upgrade.
3. **OpenAI 429:** Very rare for Whisper at IRdeo's volume. Investigate as anomaly.
4. **Lalal.ai 429:** Daily processing quota; user must wait or upgrade plan.

---

## 11. Cost Tracking

Per Data Model Spec Section 13, every external API call records a cost entry in `cost_log.jsonl`. Provider-specific patterns:

### 11.1 Anthropic Cost Recording

Two entries per call (input and output tokens have different rates):

```python
async def _record_anthropic_cost(self, project_id: str, usage,
                                 operation: str, model: str):
    pricing = PRICING[f"anthropic_{model.replace('-', '_')}"]
    input_cost = usage.input_tokens * pricing["input_per_million_tokens"] / 1_000_000
    output_cost = usage.output_tokens * pricing["output_per_million_tokens"] / 1_000_000

    # Single combined entry with input/output breakdown in metadata
    await self.cost_tracker.record(
        project_id=project_id,
        service="anthropic",
        operation=operation,
        input_units=usage.input_tokens,
        input_unit_type="tokens_input",
        cost_usd=round(input_cost + output_cost, 4),
        metadata={
            "model": model,
            "input_tokens": usage.input_tokens,
            "output_tokens": usage.output_tokens,
            "input_cost_usd": round(input_cost, 4),
            "output_cost_usd": round(output_cost, 4),
        }
    )
```

### 11.2 OpenAI (Whisper) Cost Recording

Single rate per audio second:

```python
await self.cost_tracker.record(
    project_id=project_id,
    service="openai",
    operation="whisper_transcribe",
    input_units=audio_duration_seconds,
    input_unit_type="audio_seconds",
    cost_usd=round(audio_duration_seconds * 0.0001, 4),  # $0.006/min ÷ 60
    metadata={"audio_duration": audio_duration_seconds}
)
```

### 11.3 fal.ai Cost Recording

Per output second; model-specific rates:

```python
async def _record_fal_cost(self, project_id: str, model: str,
                          output_duration: float, operation: str,
                          subchunk_id: str):
    rate = PRICING[f"fal_{model}"]["per_second_of_output"]
    await self.cost_tracker.record(
        project_id=project_id,
        service="fal_ai",
        operation=operation,  # "video_generation" or "lip_sync"
        input_units=output_duration,
        input_unit_type="output_seconds",
        cost_usd=round(output_duration * rate, 4),
        metadata={"model": model, "subchunk_id": subchunk_id}
    )
```

### 11.4 Lalal.ai Cost Recording

Per input second processed (regardless of stem type):

```python
await self.cost_tracker.record(
    project_id=project_id,
    service="lalai",
    operation="vocal_isolation",
    input_units=audio_duration_seconds,
    input_unit_type="input_seconds",
    cost_usd=round(audio_duration_seconds * 0.0019, 4),
    metadata={"engine": engine, "file_id": file_id}
)
```

### 11.5 Cost Reconciliation

Once a month (or on demand), IRdeo can run `irdeo cost-reconcile` which compares its internal cost log against the provider's actual billing (if API access is available). Discrepancies trigger investigation.

Not critical for MVP; the internal log is the user-facing truth.

---

## 12. Sign-Up Instructions Per Provider

These instructions appear in the Settings drawer (UI Spec Section 3.6) and in the Help drawer. Linked to current provider docs since URLs and signup flows change.

### 12.1 Anthropic Claude

1. Visit https://console.anthropic.com/
2. Sign up (Google OAuth or email)
3. Verify email
4. Add a payment method (required even for free tier credit)
5. Go to **Settings → API Keys**
6. Click **Create Key**, name it `IRdeo`
7. Copy the key (starts with `sk-ant-`) — visible ONCE only
8. Paste into IRdeo Settings → Anthropic API Key
9. Click **Test** to validate

**Free tier:** $5 in initial credits, then pay-per-use. Typical IRdeo session: $1–$3 in Claude costs.

### 12.2 OpenAI (Whisper)

1. Visit https://platform.openai.com/
2. Sign up
3. Add payment method (required for API access)
4. Go to **API keys** in left sidebar
5. Click **Create new secret key**, name it `IRdeo-Whisper`
6. Copy the key (starts with `sk-`)
7. Paste into IRdeo Settings → OpenAI API Key
8. Click **Test** to validate

**Pricing:** $0.006 per audio minute. Typical IRdeo session: $0.03–$0.05.

### 12.3 fal.ai

1. Visit https://fal.ai/
2. Sign up (Google, GitHub, or email)
3. Add payment method (top-up balance, pay-as-you-go)
4. Go to **Dashboard → Keys**
5. Click **Add Key**, name it `IRdeo`
6. Copy the key (format: `fal_XXX...` or `id:secret`)
7. Paste into IRdeo Settings → fal.ai API Key
8. Click **Test** to validate

**Pricing:** Pay-per-use, no monthly subscription. Typical IRdeo song generation cost: $15–$25 (varies by chunk count, model selection, lip sync usage).

**Note on credits:** fal.ai typically requires a $5–$10 minimum top-up. Buy enough for at least one full song generation before starting work.

### 12.4 Lalal.ai

1. Visit https://www.lalal.ai/api/
2. Click **Get API access**
3. Sign up
4. Purchase a credit package ($10 minimum, includes ~90 minutes of audio processing)
5. Go to **API → License Keys**
6. Copy your license key
7. Paste into IRdeo Settings → Lalal.ai API Key
8. Click **Test** to validate (will also show remaining credit balance)

**Pricing:** $10 minimum for 90 audio-minutes. Typical IRdeo song: ~0.5 audio-minutes consumed (one vocal isolation per song). One credit pack covers ~180 songs.

---

## 13. SaaS Evolution Path (Phase 2)

### 13.1 What Changes

When IRdeo moves from local MVP to cloud SaaS:

1. **Settings storage:** local Fernet-encrypted file → cloud secrets vault (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager)
2. **Key retrieval:** vault returns key per-request, never cached in plaintext beyond request lifetime
3. **Tenant isolation:** each user has their own vault namespace; cross-tenant access impossible
4. **Audit log:** every key access logged with `user_id`, `timestamp`, `service`, `operation_id`
5. **Rotation policy:** scheduled reminders + forced-rotation policy after suspected compromise events

### 13.2 What Stays the Same

The provider SDKs, error categorization, retry policies, cost tracking, and concurrency control are unchanged. The `SettingsStore` interface is implemented by different backends (LocalSettingsStore vs VaultedSettingsStore) per Architecture Section 13.3.

### 13.3 BYO-Keys Philosophy in SaaS

Even in SaaS, IRdeo NEVER:
- Proxies user requests through a shared IRdeo account
- Marks up provider costs (the user pays providers directly at raw cost)
- Stores user keys in a way that allows IRdeo operators to read them at scale

The vault stores per-user keys; only the per-user processing path retrieves them. IRdeo charges a separate subscription fee for the IRdeo product (smart layer + UI + storage), NOT for API access. This is the architectural insight from BRD discussion — IRdeo is a tool that uses the user's API budget, not a reseller of API capacity.

---

## 14. Open API Integration Questions

These need resolution during build / first real-world testing:

- **AIQ-1** Verify current Anthropic Claude model IDs (`claude-opus-4-7`, etc.) at https://docs.claude.com/en/api/models — update `pricing.py` if model lineup changed
- **AIQ-2** Verify current Whisper model — at present `whisper-1` is the only public model; OpenAI may release newer models in Whisper family
- **AIQ-3** Verify current fal.ai model URLs for Kling, Veo, Sync.so. fal.ai's catalog evolves; some model paths may have changed
- **AIQ-4** Verify current fal.ai concurrency limits at https://fal.ai/docs — adjust `falai_max_concurrent` default if real limit differs from 3
- **AIQ-5** Verify Lalal.ai pricing — currently uses Phoenix and Orion as "top engines" but they may release new ones (e.g., a "Saturn" engine has been hinted at). Test both available engines on representative IRdeo songs (rap with heavy beat, atmospheric ballad) to determine default
- **AIQ-6** Test reference photo URL behavior with fal.ai's Kling integration — confirm temporary upload URLs work and how long they persist within a job
- **AIQ-7** Stress-test the lyrics-audio sanity check with non-English songs (Slovak) to verify the language-skip logic prevents false positives
- **AIQ-8** Investigate Anthropic's batch API support — for non-time-sensitive operations (prompt generation pre-rendering, conversation summarization), batch may offer significant cost savings (~50% off). Defer to future enhancement.
- **AIQ-9** Investigate streaming responses for the conversation LLM — reduces perceived latency in chat. Defer to v1.1 once core integration is proven.
- **AIQ-10** Pre-commit secret-detection thresholds — gitleaks default rules may produce false positives on test fixtures; tune `.gitleaks.toml` after first integration run.

---

## 15. Provenance

This API Integration Specification was derived from:
- Knowledge Base v1.3 — domain rules driving which APIs are needed
- BRD v1.4 — functional requirements per service (Section 8 FRs)
- Architecture v1.2 — service-layer integration patterns, security model, BYO-keys principle
- UI Spec v1.0 — Settings drawer for key entry, validation feedback patterns
- Data Model v1.0 — Settings schema (Section 14), cost tracking schema (Section 13)
- Provider documentation (Anthropic, OpenAI, fal.ai, Lalal.ai)

Every integration pattern is grounded in either an explicit requirement, a provider's API documentation, or a security principle from Architecture Section 2.8 (BYO keys) and Section 18 (security model). Provider-specific details are flagged for verification at integration time in Section 14 (Open API Integration Questions).
