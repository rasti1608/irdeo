# IRdeo — Data Model Specification
**Version:** 1.0
**Last updated:** May 11, 2026
**Status:** LIVING DOCUMENT — comprehensive data model with concrete schemas
**Companion documents:** `IRdeo_Knowledge_Base.md` (v1.3+), `IRdeo_BRD.md` (v1.4+), `IRdeo_Architecture.md` (v1.2+), `IRdeo_UI_Spec.md` (v1.0+)

### Versioning Convention
- **v1.0** — first complete formal version
- **v1.1, v1.2, ...** — substantive revisions and additions
- **v2.0** — major restructure or fundamental approach change

### Changelog
- **v1.0 (May 11, 2026)** — First complete formal version. Defines all JSON schemas, Python dataclasses, persistence formats, validation rules, and migration patterns. Built on top of KB v1.3, BRD v1.4, Architecture v1.2, UI Spec v1.0.

---

## 1. Document Overview

### 1.1 Purpose
This document is the source of truth for all data structures in IRdeo. Every JSON file written to disk, every API request and response shape, every in-memory dataclass — all defined here. The Architecture Document specifies which components own which data; this document specifies the exact shape of that data.

### 1.2 Scope
- **In scope:** All persistent file formats (JSON, JSONL, markdown), in-memory Python dataclasses, API request/response schemas, validation rules, migration patterns between schema versions
- **Out of scope:** Database schemas (SaaS Phase 2; relevant when we move beyond local file persistence), encryption details (Architecture Section 13 covers it), wire-format encoding choices (covered in API Integration Spec when written)

### 1.3 Audience
- **Primary:** Build implementers (CLI agents, human developers)
- **Secondary:** Future contributors needing to extend the data model

### 1.4 Design Principles

**P1 — File system is the source of truth.** Each project is a self-contained directory. JSON files are human-readable. State can be inspected, backed up, and emailed without tooling.

**P2 — Schema versioning is explicit.** Every persistent file declares its `schema_version`. Migrations are forward-only and lossless where possible.

**P3 — Schemas are forgiving on read, strict on write.** Reading tolerates extra fields (for forward compatibility); writing produces only spec-compliant output.

**P4 — Identifiers are stable.** Once a chunk_id or subchunk_id is assigned, it never changes — even when the user reorders, edits, or regenerates. Tools, links, and external references stay valid.

**P5 — Timestamps are ISO 8601 UTC.** Every datetime field uses `YYYY-MM-DDTHH:MM:SS.sssZ` format. No timezone-naive datetimes ever persisted.

**P6 — Currency is USD float, rounded to 4 decimals.** Cost tracking precision matters at fractional-cent granularity since per-call costs can be $0.0019 (Lalal.ai).

**P7 — Paths are relative within the project directory.** Stored paths are like `chunks/chunk_01/01_a_intro.mp4`, NOT absolute. This lets users move project folders without breaking references.

### 1.5 Related Documents
- `IRdeo_Knowledge_Base.md` — Domain rules (Timestamps File canonical format defined in KB Section 2)
- `IRdeo_BRD.md` — Requirements driving these schemas
- `IRdeo_Architecture.md` — Components that own each schema
- `IRdeo_UI_Spec.md` — UI surfaces that read/write these schemas

---

## 2. Top-Level Project Directory Structure

(Reference: Architecture Section 11.1)

```
~/IRdeo_Projects/                      ← root configured via Settings
├── {project_id}_{song_slug}/          ← one directory per project
│   ├── state.json                     ← top-level project state (see Section 3)
│   ├── source/
│   │   ├── original.mp3
│   │   ├── lyrics.txt                 ← if user provided
│   │   ├── vocals_only.wav            ← after Lalal.ai isolation, if applicable
│   │   └── upload_metadata.json       ← see Section 4
│   ├── reference_images/
│   │   ├── {photo_set_id}/
│   │   │   ├── photo_1.jpg
│   │   │   ├── photo_2.jpg
│   │   │   └── metadata.json          ← see Section 5
│   ├── timestamps.md                  ← canonical Timestamps File (KB Section 2 format)
│   ├── chunks.json                    ← chunk manifest (Section 6)
│   ├── analysis.json                  ← audio analysis output (Section 7)
│   ├── conversation.jsonl             ← conversation history (Section 8)
│   ├── prompts/
│   │   ├── 01_chunk_prompt.txt        ← human-readable chunk-level prompt
│   │   ├── 01_a.json                  ← subchunk prompt + metadata (Section 9)
│   │   ├── 01_b.json
│   │   ├── ...
│   ├── chunks/
│   │   ├── chunk_01/
│   │   │   ├── 01_a_intro.mp4
│   │   │   ├── 01_b_intro.mp4
│   │   │   ├── chunk_metadata.json    ← per-chunk runtime metadata (Section 10)
│   │   ├── ...
│   ├── output/
│   │   ├── preview.mp4
│   │   ├── preview_concat.txt         ← ffmpeg concat manifest (transient)
│   │   ├── {song_slug}.wfp            ← Filmora project (if generated)
│   │   ├── {song_slug}.xml            ← FCP XML fallback
│   │   ├── {song_slug}_master.docx
│   │   └── {song_slug}_prompts.zip    ← Path A export
│   ├── jobs/
│   │   ├── {job_id}.json              ← job records (Section 11)
│   ├── generation_log.jsonl           ← event log (Section 12)
│   └── cost_log.jsonl                 ← cost tracking (Section 13)
```

Global (not per-project):

```
~/.irdeo/
├── settings.json                      ← encrypted with Fernet (Section 14)
├── .key                               ← Fernet encryption key (0600 perms)
├── logs/
│   └── irdeo.log                      ← rolling structured logs
└── projects_index.json                ← global project registry (Section 15)
```

---

## 3. Project State (`state.json`)

Top-level state for a single project. Read on every view load, updated by services as the project advances.

### 3.1 Schema

```json
{
  "schema_version": 1,
  "project_id": "p_8f3a2b1c",
  "song_slug": "manufacturing_consent",
  "song_title": "Manufacturing Consent",
  "artist": "IronRUST",
  "album_context": "The System Burns",
  "track_number": 7,
  "genre": "Political Rap",
  "created_at": "2026-05-10T14:23:18.421Z",
  "last_modified": "2026-05-11T09:14:02.117Z",

  "irdeo_version": "0.1.0",
  "kb_version": "1.3",

  "active_view": "conversation",

  "states": {
    "intake_state": "complete",
    "analysis_state": "complete",
    "timestamps_state": "finalized",
    "chunks_state": "locked",
    "prompts_state": "generated",
    "generation_state": "in_progress",
    "lipsync_state": "not_started",
    "preview_state": "not_started",
    "export_state": "not_started"
  },

  "selected_path": "B",

  "timestamps_finalized_at": "2026-05-10T15:01:33.500Z",
  "chunks_locked_at": "2026-05-10T15:42:17.823Z",
  "prompts_generated_at": "2026-05-10T15:58:44.012Z",

  "skip_whisper": false,
  "user_supplies_vocals_only": false,

  "user_lyrics_provided": true,
  "user_lyrics_path": "source/lyrics.txt",

  "mp3_path": "source/original.mp3",
  "vocals_only_path": "source/vocals_only.wav",

  "duration_seconds": 285.34,
  "language": "en",

  "active_job_id": "j_2c9e1d4f"
}
```

### 3.2 Field Definitions

| Field | Type | Description |
|-------|------|-------------|
| `schema_version` | int | Schema version (currently 1) |
| `project_id` | str | Stable UUID-derived ID. Format: `p_` + 8 hex chars |
| `song_slug` | str | URL/filename-safe lowercase with underscores |
| `song_title` | str | User-provided display name |
| `artist` | str \| null | Artist name (defaults to "Unknown" if user skipped) |
| `album_context` | str \| null | Album/project grouping |
| `track_number` | int \| null | Track position within album |
| `genre` | str | Genre name; drives Section 7 playbook lookup |
| `created_at` | ISO datetime | Project creation timestamp |
| `last_modified` | ISO datetime | Most recent state.json write |
| `irdeo_version` | str | IRdeo version that created/last touched this project |
| `kb_version` | str | KB version at last modification (informational) |
| `active_view` | enum | Last view the user was on: `welcome`, `conversation`, `timestamps`, `chunks`, `output` |
| `states` | object | Per-phase state machine (see Section 3.3) |
| `selected_path` | enum \| null | Output path choice: `A` (prompt pack) or `B` (full generation) or `null` (not yet chosen) |
| `timestamps_finalized_at` | ISO datetime \| null | When user locked Timestamps File |
| `chunks_locked_at` | ISO datetime \| null | When user locked chunk definitions |
| `prompts_generated_at` | ISO datetime \| null | When prompts were generated |
| `skip_whisper` | bool | True if user opted to skip Whisper |
| `user_supplies_vocals_only` | bool | True if user uploaded vocals-only stem (Lalal isolation skipped) |
| `user_lyrics_provided` | bool | True if user uploaded/pasted lyrics |
| `user_lyrics_path` | relative path \| null | Path within project dir |
| `mp3_path` | relative path | Path to original MP3 within project dir |
| `vocals_only_path` | relative path \| null | Path to vocals stem (provided or generated by Lalal) |
| `duration_seconds` | float | Song duration from librosa |
| `language` | str | Whisper-detected language code (e.g., "en", "sk") |
| `active_job_id` | str \| null | If a background job is currently running, its ID |

### 3.3 Phase State Machine

Each `states.*` field uses these enum values:

| Value | Meaning |
|-------|---------|
| `not_started` | Phase hasn't begun |
| `in_progress` | Phase is actively running (associated job in progress) |
| `awaiting_user` | Phase needs user input/confirmation before continuing |
| `complete` / `finalized` / `locked` / `generated` | Phase finished (terminology varies per phase) |
| `failed` | Phase failed; user needs to retry or fix |
| `cancelled` | User cancelled the phase |

Specific per-phase terminology:

- `intake_state`: `not_started` → `awaiting_user` → `complete`
- `analysis_state`: `not_started` → `in_progress` → `complete` | `failed`
- `timestamps_state`: `not_started` → `draft` → `finalized`
- `chunks_state`: `not_started` → `proposed` → `locked`
- `prompts_state`: `not_started` → `generating` → `generated`
- `generation_state`: `not_started` → `in_progress` → `complete` | `partial` | `failed`
- `lipsync_state`: `not_started` → `in_progress` → `complete` (only applies if performer chunks exist)
- `preview_state`: `not_started` → `building` → `complete` | `stale`
- `export_state`: `not_started` → `building` → `complete`

`preview_state` may become `stale` if any chunk is regenerated after the last preview build.

### 3.4 Validation Rules

- `project_id` must match `^p_[a-f0-9]{8}$`
- `song_slug` must match `^[a-z0-9_]+$`
- `selected_path` must be `null` until `prompts_state == "generated"`
- `generation_state == "in_progress"` implies `active_job_id` is set
- All ISO datetime fields validate as UTC (must end in `Z` or `+00:00`)

---

## 4. Upload Metadata (`source/upload_metadata.json`)

Records the original uploaded files (for audit and re-validation).

### 4.1 Schema

```json
{
  "schema_version": 1,
  "mp3": {
    "original_filename": "Manufacturing_Consent_v3_FINAL.mp3",
    "stored_filename": "original.mp3",
    "size_bytes": 6842150,
    "sha256": "a1b2c3d4e5f6...",
    "uploaded_at": "2026-05-10T14:23:45.102Z",
    "mime_type": "audio/mpeg"
  },
  "lyrics": {
    "original_filename": "manufacturing_consent_lyrics.txt",
    "stored_filename": "lyrics.txt",
    "size_bytes": 4827,
    "sha256": "b2c3d4e5f6a7...",
    "uploaded_at": "2026-05-10T14:24:12.330Z",
    "mime_type": "text/plain"
  },
  "vocals_stem": null
}
```

### 4.2 Field Definitions

Each file slot (`mp3`, `lyrics`, `vocals_stem`) is either `null` or an object with:
- `original_filename` — what the user named it
- `stored_filename` — what IRdeo renamed it to inside the project directory
- `size_bytes` — file size at upload
- `sha256` — content hash (for tamper detection / dedup)
- `uploaded_at` — ISO datetime
- `mime_type` — detected MIME type

---

## 5. Reference Image Sets (`reference_images/{set_id}/metadata.json`)

Each reference photo set has its own subdirectory with metadata.

### 5.1 Schema

```json
{
  "schema_version": 1,
  "photo_set_id": "rs_a4b2c1d8",
  "label": "Performer — Rasti — Main",
  "performer_role": "male_primary",
  "created_at": "2026-05-10T14:31:22.418Z",
  "photos": [
    {
      "filename": "photo_1.jpg",
      "original_filename": "rasti_front.jpg",
      "size_bytes": 245680,
      "sha256": "c1d2e3f4...",
      "dimensions": [1024, 1024],
      "uploaded_at": "2026-05-10T14:31:22.418Z"
    },
    {
      "filename": "photo_2.jpg",
      "original_filename": "rasti_side.jpg",
      "size_bytes": 198420,
      "sha256": "d1e2f3g4...",
      "dimensions": [1024, 1024],
      "uploaded_at": "2026-05-10T14:31:25.802Z"
    }
  ]
}
```

### 5.2 Field Definitions

| Field | Type | Description |
|-------|------|-------------|
| `photo_set_id` | str | Format: `rs_` + 8 hex chars |
| `label` | str | Human-readable name; shown in UI |
| `performer_role` | enum | `male_primary`, `male_secondary`, `female_primary`, `female_secondary`, `other` |
| `photos` | array | List of photo file descriptors |
| `photos[].dimensions` | [int, int] | [width, height] in pixels |

### 5.3 Validation Rules

- At least one photo per set
- Maximum 4 photos per set (typical API limit for character reference)
- All photos JPG or PNG
- All photos at least 512×512 px (recommendation, not enforced)

---

## 6. Chunk Manifest (`chunks.json`)

The source of truth for chunk-level metadata. Built during Chunks Review (UI Spec View D). Read by every downstream phase.

### 6.1 Schema

```json
{
  "schema_version": 1,
  "project_id": "p_8f3a2b1c",
  "version": 3,
  "last_modified": "2026-05-10T15:42:17.823Z",
  "chunks": [
    {
      "chunk_id": "chunk_01",
      "sequence_index": 0,
      "section_label": "Intro (spoken)",
      "section_slug": "intro",
      "start_seconds": 0.00,
      "end_seconds": 22.00,
      "duration_seconds": 22.00,
      "vocal_type": "Male Spoken",
      "vocal_modifiers": ["close-mic"],
      "performer_present": true,
      "performer_subchunks": null,
      "reference_photo_set_id": "rs_a4b2c1d8",
      "mood": "intimate, vulnerable",
      "style_references": ["Eminem Cleanin' Out My Closet"],
      "named_visuals": "performer alone in dim room, looking at TV",
      "color_grade_override": null,
      "model_preference": "kling_motion_pro",
      "model_lipsync": "lipsync-2",
      "user_notes": "Set the confessional tone here",
      "subchunks": [
        {
          "subchunk_id": "01_a",
          "subchunk_letter": "a",
          "sequence_index": 0,
          "start_seconds": 0.00,
          "end_seconds": 8.00,
          "duration_seconds": 8.00,
          "description_slug": "intro_open"
        },
        {
          "subchunk_id": "01_b",
          "subchunk_letter": "b",
          "sequence_index": 1,
          "start_seconds": 8.00,
          "end_seconds": 15.00,
          "duration_seconds": 7.00,
          "description_slug": "intro_mid"
        },
        {
          "subchunk_id": "01_c",
          "subchunk_letter": "c",
          "sequence_index": 2,
          "start_seconds": 15.00,
          "end_seconds": 22.00,
          "duration_seconds": 7.00,
          "description_slug": "intro_close"
        }
      ]
    },
    {
      "chunk_id": "chunk_02",
      "sequence_index": 1,
      "section_label": "Verse 1",
      "section_slug": "verse_1",
      "start_seconds": 22.00,
      "end_seconds": 72.00,
      "duration_seconds": 50.00,
      "vocal_type": "Male Rap",
      "vocal_modifiers": [],
      "performer_present": false,
      "performer_subchunks": null,
      "reference_photo_set_id": null,
      "mood": "exposed, accusatory",
      "style_references": [],
      "named_visuals": "Chomsky portrait, 5 filters animated diagram, newsroom B-roll",
      "color_grade_override": "cold_desaturated",
      "model_preference": "kling_o3_pro",
      "model_lipsync": null,
      "user_notes": "Documentary feel, no performer",
      "subchunks": [
        {"subchunk_id": "02_a", "subchunk_letter": "a", "sequence_index": 0, "start_seconds": 22.0, "end_seconds": 30.0, "duration_seconds": 8.0, "description_slug": "verse1_open"},
        {"subchunk_id": "02_b", "subchunk_letter": "b", "sequence_index": 1, "start_seconds": 30.0, "end_seconds": 38.0, "duration_seconds": 8.0, "description_slug": "verse1_chomsky"},
        {"subchunk_id": "02_c", "subchunk_letter": "c", "sequence_index": 2, "start_seconds": 38.0, "end_seconds": 46.0, "duration_seconds": 8.0, "description_slug": "verse1_filters"},
        {"subchunk_id": "02_d", "subchunk_letter": "d", "sequence_index": 3, "start_seconds": 46.0, "end_seconds": 54.0, "duration_seconds": 8.0, "description_slug": "verse1_ads"},
        {"subchunk_id": "02_e", "subchunk_letter": "e", "sequence_index": 4, "start_seconds": 54.0, "end_seconds": 62.0, "duration_seconds": 8.0, "description_slug": "verse1_access"},
        {"subchunk_id": "02_f", "subchunk_letter": "f", "sequence_index": 5, "start_seconds": 62.0, "end_seconds": 70.0, "duration_seconds": 8.0, "description_slug": "verse1_fear"},
        {"subchunk_id": "02_g", "subchunk_letter": "g", "sequence_index": 6, "start_seconds": 70.0, "end_seconds": 72.0, "duration_seconds": 2.0, "description_slug": "verse1_bridge"}
      ]
    }
  ]
}
```

### 6.2 Field Definitions — Chunk

| Field | Type | Description |
|-------|------|-------------|
| `chunk_id` | str | Stable ID; format `chunk_NN` (zero-padded) |
| `sequence_index` | int | 0-based position in song |
| `section_label` | str | Human-readable section name (e.g., "Verse 1") |
| `section_slug` | str | URL/filename-safe version (e.g., "verse_1") |
| `start_seconds` | float | Absolute timestamp in song |
| `end_seconds` | float | Absolute timestamp in song |
| `duration_seconds` | float | `end - start` |
| `vocal_type` | str | From canonical vocabulary (KB Section 2) |
| `vocal_modifiers` | array[str] | E.g., `["reverb", "close-mic"]` |
| `performer_present` | bool | Whether performer appears on screen in this chunk |
| `performer_subchunks` | array[str] \| null | If partial, list of subchunk_ids where performer appears; null means all-or-none per `performer_present` |
| `reference_photo_set_id` | str \| null | Binding to reference image set |
| `mood` | str | Free-text mood description |
| `style_references` | array[str] | Artist/song names referenced |
| `named_visuals` | str | Free-text named visuals/logos/people |
| `color_grade_override` | str \| null | E.g., "cold_desaturated"; null uses genre playbook default |
| `model_preference` | enum | Video generation model for this chunk |
| `model_lipsync` | enum \| null | Lip sync model; null when `performer_present=false` |
| `user_notes` | str | Free-form notes for the AI Director / future reference |
| `subchunks` | array | Subchunk definitions (Section 6.3) |

### 6.3 Field Definitions — Subchunk

| Field | Type | Description |
|-------|------|-------------|
| `subchunk_id` | str | Format: `NN_X` (e.g., `01_a`); used in filenames |
| `subchunk_letter` | str | Single lowercase letter (`a`, `b`, ...) |
| `sequence_index` | int | 0-based position within the chunk |
| `start_seconds` | float | Absolute timestamp in song |
| `end_seconds` | float | Absolute timestamp in song |
| `duration_seconds` | float | Must be ≤ 8.0 |
| `description_slug` | str | URL-safe descriptor (e.g., "intro_open"); used in clip filename |

### 6.4 Validation Rules

- `chunk_id` matches `^chunk_\d{2,}$`
- `subchunk_id` matches `^\d{2,}_[a-z]$`
- Chunks ordered by `sequence_index` ascending
- Adjacent chunks have no gap: `chunks[i].end_seconds == chunks[i+1].start_seconds`
- First chunk starts at 0; last chunk ends at song duration
- Within a chunk, subchunks ordered by `sequence_index`, no gaps, contiguous, sum of durations equals chunk duration
- `duration_seconds ≤ 8.0` for every subchunk
- `vocal_type` must be from KB Section 2 canonical vocabulary
- `model_preference` must be a supported model (see Settings)
- If `performer_present=true` and chunk type is video gen, `reference_photo_set_id` strongly recommended (UI warns if null)
- `performer_subchunks` only allowed when `performer_present=true` (means "partial" mode)
- `version` increments on every write
- Filename derived from subchunk: `{NN}_{letter}_{description_slug}.mp4` (e.g., `01_a_intro_open.mp4`)

---

## 7. Audio Analysis (`analysis.json`)

Output of the AnalysisService (Architecture Section 5). Read by chunk proposal and prompt generation.

### 7.1 Schema

```json
{
  "schema_version": 1,
  "project_id": "p_8f3a2b1c",
  "generated_at": "2026-05-10T14:25:48.301Z",
  "analyzer_versions": {
    "whisper": "whisper-1",
    "librosa": "0.10.1",
    "claude_classification": "claude-opus-4-7"
  },

  "transcript": {
    "language": "en",
    "language_confidence": 0.998,
    "segments": [
      {
        "start": 0.00,
        "end": 1.84,
        "text": "Yeah..."
      },
      {
        "start": 2.10,
        "end": 5.32,
        "text": "I used to believe you."
      }
    ],
    "full_text": "Yeah... I used to believe you. Defended you..."
  },

  "features": {
    "bpm": 112.0,
    "duration_seconds": 285.34,
    "energy_curve": [3.2, 3.4, 3.5, 8.5, 8.7, 8.9, ...],
    "energy_curve_resolution_seconds": 2.0,
    "beat_positions": [0.534, 1.069, 1.605, 2.139, ...],
    "brightness_curve": [1247.3, 1450.8, ...],
    "onset_strength": [0.12, 0.34, 0.89, ...],
    "rms_peak": 0.412,
    "rms_mean": 0.187
  },

  "classification": {
    "detected_sections": [
      {
        "label": "Intro (spoken)",
        "start_seconds": 0.0,
        "end_seconds": 22.0,
        "section_type": "intro",
        "vocal_type": "Male Spoken",
        "vocal_modifiers": ["close-mic"],
        "energy_level": "low",
        "energy_numeric": 3.5
      },
      {
        "label": "Verse 1",
        "start_seconds": 22.0,
        "end_seconds": 72.0,
        "section_type": "verse",
        "vocal_type": "Male Rap",
        "vocal_modifiers": [],
        "energy_level": "high",
        "energy_numeric": 8.5
      }
    ],
    "mood_arc": "low → high → peak → high → medium → peak → drop",
    "detected_genre_hint": "Political Rap",
    "primary_voice": "male",
    "has_female_sections": true,
    "has_spoken_sections": true,
    "estimated_chunk_count": 9
  },

  "sanity_check": {
    "ran": true,
    "skipped": false,
    "skip_reason": null,
    "overlap_ratio": 0.87,
    "threshold": 0.25,
    "should_warn": false,
    "message": null,
    "user_lyrics_token_count": 412,
    "whisper_token_count": 389
  }
}
```

### 7.2 Field Definitions

Most are self-explanatory; notable ones:

- `transcript.segments` — Whisper raw output (segment-level, not word-level)
- `transcript.language_confidence` — Whisper's confidence in the detected language
- `features.energy_curve` — RMS values at 2-second intervals
- `features.beat_positions` — seconds at each detected beat
- `features.brightness_curve` — spectral centroid values, same resolution as energy
- `classification.detected_sections` — AI-classified sections with vocal type assignment
- `classification.mood_arc` — high-level mood progression description
- `sanity_check` — per Architecture Section 5.4

### 7.3 Validation Rules

- `transcript.segments` ordered by `start` ascending, non-overlapping
- `features.energy_curve.length * features.energy_curve_resolution_seconds ≈ features.duration_seconds`
- `classification.detected_sections` cover the full song (no gaps from 0 to duration)
- `sanity_check.overlap_ratio ∈ [0.0, 1.0]`
- `sanity_check.should_warn` and `sanity_check.message` are consistent (both set or both null)

### 7.4 Skip-Whisper Variant

When `state.skip_whisper == true`, the `transcript` section is synthesized from user lyrics:

```json
{
  "transcript": {
    "language": "sk",
    "language_confidence": null,
    "segments": [],
    "full_text": "(user-provided lyrics text)",
    "source": "user_provided"
  }
}
```

`features` and `classification` still run normally (librosa doesn't care about language).

---

## 8. Conversation History (`conversation.jsonl`)

Per-project conversation history. One turn per line. Append-only. Loaded into LLM context every turn (Architecture Section 4.3).

### 8.1 Format

JSONL — one JSON object per line. Each object is a ConversationTurn:

```jsonl
{"turn_id": "t_a1", "role": "user", "content": "Let's start with this song", "timestamp": "2026-05-10T14:23:18.421Z", "metadata": {}}
{"turn_id": "t_a2", "role": "assistant", "content": "Got it! Let's start by uploading the MP3...", "timestamp": "2026-05-10T14:23:19.011Z", "metadata": {"actions": [{"type": "upload_file", "label": "Upload MP3", "target": "mp3"}]}}
{"turn_id": "t_a3", "role": "user", "content": "MP3 uploaded: Manufacturing_Consent.mp3", "timestamp": "2026-05-10T14:23:45.102Z", "metadata": {"file_uploaded": "source/original.mp3"}}
```

### 8.2 Schema (Per Turn)

```python
@dataclass
class ConversationTurn:
    turn_id: str         # format: t_ + 8 hex chars
    role: Literal["user", "assistant", "system"]
    content: str         # the message text (markdown allowed in assistant turns)
    timestamp: str       # ISO datetime
    metadata: dict       # turn-specific metadata (actions, file refs, etc.)
```

### 8.3 Metadata Conventions

The `metadata` field is a flexible dict. Common keys:

| Key | Type | Used By | Description |
|-----|------|---------|-------------|
| `actions` | array | assistant | Structured action buttons (see Section 8.4) |
| `file_uploaded` | str | user | Path to file the user uploaded in this turn |
| `voice_input` | bool | user | True if message came from voice transcription |
| `phase` | str | assistant | KB Section 13 phase identifier (e.g., "phase_4_chunks") |
| `summarized_from` | array[str] | system | If this is a summary turn, list of turn_ids it replaces |
| `model_used` | str | assistant | Which LLM produced this turn (for audit) |

### 8.4 Action Schema

When the assistant message includes action buttons, they're in `metadata.actions`:

```json
{
  "actions": [
    {
      "type": "upload_file",
      "label": "Upload MP3",
      "target": "mp3",
      "primary": true
    },
    {
      "type": "review_timestamps",
      "label": "Review Timestamps",
      "primary": false
    },
    {
      "type": "regenerate_chunk",
      "label": "Regenerate chunk 4",
      "params": {"chunk_id": "chunk_04"},
      "destructive": true,
      "requires_confirmation": true
    }
  ]
}
```

Action types correspond to Architecture Section 4.6 ACTION_TYPES map.

### 8.5 Validation Rules

- `turn_id` matches `^t_[a-f0-9]{8}$`
- Each `turn_id` unique within the file
- Turns ordered chronologically by `timestamp` (append-only)
- `role` is one of `user`, `assistant`, `system`
- `system` turns are reserved for conversation summaries

### 8.6 Summarization Turn Pattern

When conversation exceeds context budget (Architecture Section 4.4), older turns get summarized into a single `system` turn:

```json
{
  "turn_id": "t_sum01",
  "role": "system",
  "content": "[CONVERSATION SUMMARY of turns t_a1 through t_a47]: User uploaded MP3 and lyrics for 'Manufacturing Consent'. Set genre to Political Rap. Confirmed 9 sections in Timestamps File with minor Whisper corrections. Defined 9 chunks; chunks 1, 4, 7 have performer present using rs_a4b2c1d8 photo set. Style references: Eminem Cleanin' Out My Closet for chunk 1.",
  "timestamp": "2026-05-11T08:12:00.000Z",
  "metadata": {
    "summarized_from": ["t_a1", "t_a2", ..., "t_a47"],
    "model_used": "claude-opus-4-7"
  }
}
```

After this, the original turns can optionally be archived to `conversation_archive.jsonl` (keeping `conversation.jsonl` lean).

---

## 9. Subchunk Prompt Files (`prompts/{subchunk_id}.json`)

One file per subchunk. Stores the full prompt + metadata generated by PromptGenService (Architecture Section 8).

### 9.1 Schema

```json
{
  "schema_version": 1,
  "subchunk_id": "01_a",
  "chunk_id": "chunk_01",
  "subchunk_letter": "a",
  "sequence_index": 0,
  "start_seconds": 0.00,
  "end_seconds": 8.00,
  "duration_seconds": 8.00,
  "description_slug": "intro_open",

  "prompt_text": "Dark, intimate room. Performer (male, 40s) sits alone watching a TV showing news. Soft warm key light from the TV illuminates his face. He looks contemplative, vulnerable. Slow push-in over 8 seconds. Cinematic 4K, shallow depth of field, color grade: cold blue shadows with warm TV light contrast.",

  "continuity_carry_from": null,
  "continuity_notes": "Establishing shot — sets the visual register for the intro section. Performer eyeline on TV, not camera.",

  "model": "kling_motion_pro",
  "reference_photo_set_id": "rs_a4b2c1d8",
  "reference_photo_filenames": ["photo_1.jpg", "photo_2.jpg"],

  "expected_visual_summary": "Performer watching TV, dim room, contemplative mood",

  "generated_at": "2026-05-10T15:58:44.012Z",
  "generator_model": "claude-opus-4-7",
  "user_edited": false,
  "user_edit_history": []
}
```

### 9.2 Field Definitions

Largely overlap with SubchunkPrompt dataclass from Architecture Section 8.3. Additions:

| Field | Type | Description |
|-------|------|-------------|
| `reference_photo_filenames` | array[str] | Specific filenames within the photo set used for this subchunk |
| `generator_model` | str | Which LLM wrote this prompt |
| `user_edited` | bool | True if user has overridden the AI's prompt |
| `user_edit_history` | array | Each prior version stored as `{prompt_text, edited_at, edited_by}` |

### 9.3 Chunk-Level Prompt File (`prompts/{NN}_chunk_prompt.txt`)

Companion file per chunk — the longer narrative prompt before decomposition. Plain text (markdown):

```markdown
# Chunk 01: Intro (spoken)
Time: 0.00 – 22.00s | Duration: 22.0s
Vocal Type: Male Spoken (close-mic)
Performer: Yes (full chunk)
Model: kling_motion_pro

## Mood
Intimate, vulnerable, confessional. The performer is alone with himself,
acknowledging he used to believe — setting the entire song's confessional
tone.

## Visual Direction
A dark room, performer sitting watching a TV showing news. Warm TV light
on his face. The world is small, intimate. He's not addressing anyone yet;
he's processing.

## Continuity Plan
Across the 3 subchunks (01_a, 01_b, 01_c), maintain:
- Same room, same lighting setup
- Slow push-in / dolly over the full 22 seconds
- Performer eyeline on TV, looking away from camera until the final beat

## Style References (background context)
- Eminem "Cleanin' Out My Closet" — opening confessional energy
- Cold color grade with warm spot accents
```

### 9.4 Validation Rules

- `subchunk_id` matches the parent chunk manifest entry
- `prompt_text` non-empty, less than 50000 chars
- `reference_photo_set_id` is null OR matches an existing photo set
- `user_edited == true` implies `user_edit_history` is non-empty
- `continuity_carry_from` is null for the first subchunk of a chunk, otherwise the immediate prior subchunk_id

---

## 10. Per-Chunk Runtime Metadata (`chunks/chunk_XX/chunk_metadata.json`)

Tracks generation runtime state per chunk. Updated during/after generation.

### 10.1 Schema

```json
{
  "schema_version": 1,
  "chunk_id": "chunk_01",
  "generation_status": "complete",
  "generation_started_at": "2026-05-11T09:02:11.402Z",
  "generation_completed_at": "2026-05-11T09:08:43.917Z",
  "regeneration_count": 0,

  "subchunks": [
    {
      "subchunk_id": "01_a",
      "clip_filename": "01_a_intro_open.mp4",
      "status": "generated",
      "model_used": "kling_motion_pro",
      "api_provider": "fal.ai",
      "api_call_id": "fal_jobs_2c8d4e",
      "generated_at": "2026-05-11T09:03:24.108Z",
      "duration_seconds": 8.00,
      "file_size_bytes": 4823910,
      "cost_usd": 0.32,
      "retries": 0,
      "lipsync_applied": true,
      "lipsync_model": "lipsync-2",
      "lipsync_cost_usd": 0.40,
      "lipsync_applied_at": "2026-05-11T09:06:11.502Z"
    },
    {
      "subchunk_id": "01_b",
      "clip_filename": "01_b_intro_mid.mp4",
      "status": "generated",
      "model_used": "kling_motion_pro",
      "api_provider": "fal.ai",
      "api_call_id": "fal_jobs_2c8d5f",
      "generated_at": "2026-05-11T09:04:01.224Z",
      "duration_seconds": 7.00,
      "file_size_bytes": 4231580,
      "cost_usd": 0.28,
      "retries": 1,
      "lipsync_applied": true,
      "lipsync_model": "lipsync-2",
      "lipsync_cost_usd": 0.35,
      "lipsync_applied_at": "2026-05-11T09:06:48.221Z"
    }
  ]
}
```

### 10.2 Field Definitions

| Field | Type | Description |
|-------|------|-------------|
| `generation_status` | enum | `not_started`, `in_progress`, `complete`, `partial`, `failed` |
| `regeneration_count` | int | How many times user has regenerated this chunk |
| `subchunks[].status` | enum | `not_started`, `generating`, `generated`, `failed`, `lipsyncing` |
| `subchunks[].api_call_id` | str \| null | Provider's job/request ID for traceability |
| `subchunks[].cost_usd` | float | Actual cost (from cost log) |
| `subchunks[].retries` | int | Number of retries before success (or final failure) |
| `subchunks[].lipsync_applied` | bool | True if lip sync was run on this subchunk |
| `subchunks[].lipsync_cost_usd` | float \| null | Lip sync cost (null if not applied) |

### 10.3 Validation Rules

- `generation_status == "complete"` requires all subchunks have `status == "generated"`
- `lipsync_applied == true` requires `lipsync_model` and `lipsync_cost_usd` set
- If parent chunk has `performer_present == false`, all subchunks have `lipsync_applied == false`
- `regeneration_count >= 0`; incremented atomically when user triggers regeneration

---

## 11. Job Records (`jobs/{job_id}.json`)

Per-job state for the JobOrchestrator (Architecture Section 9).

### 11.1 Schema

```json
{
  "schema_version": 1,
  "id": "j_2c9e1d4f",
  "kind": "generate_song",
  "project_id": "p_8f3a2b1c",
  "status": "running",
  "progress": 0.42,
  "current_step": "Generating subchunk 04_b — kling_motion_pro",
  "total_steps": 47,
  "completed_steps": 20,
  "created_at": "2026-05-11T09:02:00.000Z",
  "started_at": "2026-05-11T09:02:00.105Z",
  "completed_at": null,
  "error": null,
  "result": null,
  "metadata": {
    "chunks_to_generate": ["chunk_01", "chunk_02", "chunk_03", "..."],
    "model_selection": {
      "chunk_01": "kling_motion_pro",
      "chunk_02": "kling_o3_pro"
    }
  }
}
```

### 11.2 Field Definitions

(See Architecture Section 9.2 for Job/JobKind/JobStatus enums.)

| Field | Type | Description |
|-------|------|-------------|
| `id` | str | Format: `j_` + 8 hex chars |
| `kind` | enum | One of: `analysis`, `generate_chunk`, `generate_song`, `regenerate_chunk`, `build_preview`, `export_prompt_pack`, `export` |
| `status` | enum | `pending`, `running`, `succeeded`, `failed`, `cancelled` |
| `progress` | float | 0.0 to 1.0 |
| `current_step` | str | Human-readable current step |
| `error` | str \| null | Error message if `status == "failed"` |
| `result` | object \| null | Job-specific result data; structure varies by `kind` |
| `metadata` | object | Job-specific input parameters |

### 11.3 Result Shapes Per Job Kind

```python
# generate_song result
{
  "chunks_generated": ["chunk_01", "chunk_02", ...],
  "total_clips_generated": 47,
  "total_cost_usd": 18.42,
  "duration_seconds": 1843
}

# build_preview result
{
  "preview_path": "output/preview.mp4",
  "size_bytes": 24513820,
  "duration_seconds": 285.34
}

# export_prompt_pack result
{
  "zip_path": "output/manufacturing_consent_prompts.zip",
  "size_bytes": 47230,
  "chunk_count": 9
}

# export result
{
  "filmora_path": "output/manufacturing_consent.wfp",
  "docx_path": "output/manufacturing_consent_master.docx",
  "format": "wfp"  // or "fcp_xml" if fallback
}

# regenerate_chunk result
{
  "chunk_id": "chunk_04",
  "subchunks_regenerated": ["04_a", "04_b", "04_c"],
  "cost_usd": 1.20
}
```

### 11.4 Validation Rules

- Status transitions: `pending → running → (succeeded | failed | cancelled)`
- `status == "succeeded"` requires `result` non-null
- `status == "failed"` requires `error` non-null
- `progress` matches `completed_steps / total_steps` when `total_steps > 0`
- Orphaned `running` jobs found on startup are auto-marked `failed` (Architecture Section 15.4)

---

## 12. Generation Event Log (`generation_log.jsonl`)

Append-only structured event log for the entire generation pipeline. One event per line.

### 12.1 Format

JSONL:

```jsonl
{"timestamp": "2026-05-11T09:02:00.105Z", "event_type": "job_started", "job_id": "j_2c9e1d4f", "project_id": "p_8f3a2b1c", "kind": "generate_song"}
{"timestamp": "2026-05-11T09:02:01.220Z", "event_type": "subchunk_started", "project_id": "p_8f3a2b1c", "chunk_id": "chunk_01", "subchunk_id": "01_a", "model": "kling_motion_pro"}
{"timestamp": "2026-05-11T09:03:24.108Z", "event_type": "subchunk_completed", "project_id": "p_8f3a2b1c", "chunk_id": "chunk_01", "subchunk_id": "01_a", "duration_ms": 82888, "cost_usd": 0.32}
{"timestamp": "2026-05-11T09:05:11.402Z", "event_type": "lipsync_started", "project_id": "p_8f3a2b1c", "chunk_id": "chunk_01", "subchunk_id": "01_a", "model": "lipsync-2"}
{"timestamp": "2026-05-11T09:06:11.502Z", "event_type": "lipsync_completed", "project_id": "p_8f3a2b1c", "chunk_id": "chunk_01", "subchunk_id": "01_a", "duration_ms": 60100, "cost_usd": 0.40}
{"timestamp": "2026-05-11T09:06:48.221Z", "event_type": "subchunk_retry", "project_id": "p_8f3a2b1c", "chunk_id": "chunk_01", "subchunk_id": "01_b", "retry_count": 1, "reason": "RateLimitError"}
{"timestamp": "2026-05-11T09:08:43.917Z", "event_type": "job_completed", "job_id": "j_2c9e1d4f", "duration_ms": 403812, "total_cost_usd": 18.42}
```

### 12.2 Event Types

| Event | Description | Common Fields |
|-------|-------------|---------------|
| `job_started` | A background job began | `job_id`, `kind` |
| `job_completed` | Job finished successfully | `job_id`, `duration_ms`, `total_cost_usd` |
| `job_failed` | Job failed | `job_id`, `error` |
| `job_cancelled` | User cancelled | `job_id` |
| `analysis_started` | AnalysisService began | `project_id` |
| `analysis_completed` | AnalysisService done | `project_id`, `duration_ms`, `cost_usd` |
| `whisper_request` | Whisper API call started | `project_id`, `audio_duration_seconds` |
| `whisper_response` | Whisper returned | `project_id`, `language`, `duration_ms`, `cost_usd` |
| `subchunk_started` | Video generation began | `subchunk_id`, `model` |
| `subchunk_completed` | Video generation done | `subchunk_id`, `duration_ms`, `cost_usd` |
| `subchunk_failed` | Generation failed after retries | `subchunk_id`, `error` |
| `subchunk_retry` | Retrying after transient failure | `subchunk_id`, `retry_count`, `reason` |
| `lipsync_started` | Lip sync began | `subchunk_id`, `model` |
| `lipsync_completed` | Lip sync done | `subchunk_id`, `duration_ms`, `cost_usd` |
| `vocal_isolation_started` | Lalal.ai call began | `project_id` |
| `vocal_isolation_completed` | Lalal.ai returned | `project_id`, `duration_ms`, `cost_usd` |
| `preview_built` | ffmpeg stitching completed | `project_id`, `duration_ms` |
| `regeneration_triggered` | User triggered chunk regen | `chunk_id`, `reason` |
| `kb_override_used` | User overrode a KB rule | `project_id`, `rule`, `override` |
| `sanity_check_warning` | Lyrics-audio mismatch detected | `project_id`, `overlap_ratio` |

### 12.3 Validation Rules

- All events have `timestamp` and `event_type` at minimum
- Events ordered chronologically (append-only)
- File rotates when size exceeds 10 MB (older file archived as `generation_log.YYYY-MM-DD.jsonl`)

---

## 13. Cost Tracking (`cost_log.jsonl`)

Per-call cost records. Drives the cost dashboard (UI Spec Section 3.5).

### 13.1 Format

JSONL:

```jsonl
{"timestamp": "2026-05-10T14:25:30.421Z", "project_id": "p_8f3a2b1c", "service": "openai", "operation": "whisper_transcribe", "input_units": 285.34, "input_unit_type": "audio_seconds", "cost_usd": 0.0286, "metadata": {"audio_duration": 285.34}}
{"timestamp": "2026-05-10T14:25:48.301Z", "project_id": "p_8f3a2b1c", "service": "anthropic", "operation": "claude_content_classification", "input_units": 4232, "input_unit_type": "tokens_input", "cost_usd": 0.0635, "metadata": {"model": "claude-opus-4-7", "output_tokens": 1240, "output_cost_usd": 0.0930}}
{"timestamp": "2026-05-11T09:03:24.108Z", "project_id": "p_8f3a2b1c", "service": "fal_ai", "operation": "video_generation", "input_units": 8.0, "input_unit_type": "output_seconds", "cost_usd": 0.32, "metadata": {"model": "kling_motion_pro", "subchunk_id": "01_a"}}
{"timestamp": "2026-05-11T09:06:11.502Z", "project_id": "p_8f3a2b1c", "service": "fal_ai", "operation": "lip_sync", "input_units": 8.0, "input_unit_type": "output_seconds", "cost_usd": 0.40, "metadata": {"model": "lipsync-2", "subchunk_id": "01_a"}}
{"timestamp": "2026-05-11T08:55:12.012Z", "project_id": "p_8f3a2b1c", "service": "lalai", "operation": "vocal_isolation", "input_units": 285.34, "input_unit_type": "input_seconds", "cost_usd": 0.5421, "metadata": {"engine": "orion"}}
```

### 13.2 Schema (Per Entry)

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | ISO datetime | When the call completed |
| `project_id` | str | Which project incurred the cost |
| `service` | enum | `openai`, `anthropic`, `fal_ai`, `lalai` |
| `operation` | str | Specific operation (e.g., `whisper_transcribe`, `claude_chat`, `video_generation`, `lip_sync`, `vocal_isolation`) |
| `input_units` | float | How much of the input unit was consumed |
| `input_unit_type` | enum | `tokens_input`, `tokens_output`, `audio_seconds`, `output_seconds`, `input_seconds`, `requests` |
| `cost_usd` | float | USD cost rounded to 4 decimals |
| `metadata` | object | Service-/operation-specific details |

### 13.3 Validation Rules

- `cost_usd >= 0.0`
- For LLM calls, separate input and output token entries (or one combined entry with both costs in metadata)
- File rotates monthly (`cost_log.YYYY-MM.jsonl`)

### 13.4 Aggregation

The cost dashboard derives totals by reading and summing entries. No precomputed aggregates stored (avoids stale-cache problems). For very long-running projects, a daily-rollup file can be added in v1.x:

```jsonl
{"date": "2026-05-10", "project_id": "p_8f3a2b1c", "service_totals": {"openai": 0.0286, "anthropic": 1.42, "fal_ai": 15.30, "lalai": 0.54}, "total_usd": 17.29}
```

---

## 14. Global Settings (`~/.irdeo/settings.json`)

Stored encrypted via Fernet (Architecture Section 13.1). After decryption, this schema:

### 14.1 Schema

```json
{
  "schema_version": 1,

  "anthropic_api_key": "sk-ant-...",
  "openai_api_key": "sk-...",
  "falai_api_key": "fal_...",
  "lalai_api_key": "lalai_...",

  "default_video_model": "kling_motion_pro",
  "default_lipsync_model": "lipsync-2",
  "claude_model": "claude-opus-4-7",
  "whisper_model": "whisper-1",

  "falai_max_concurrent": 3,

  "vocal_isolation_provider": "lalai",
  "lalai_engine": "orion",
  "user_supplies_vocals_only": false,

  "output_directory": "/Users/rasti/IRdeo_Projects",
  "ffmpeg_path": "ffmpeg",

  "voice": {
    "tts_enabled": false,
    "tts_voice": "system_default",
    "whisper_language_preference": "auto"
  },

  "advanced": {
    "debug_logging": false,
    "conversation_token_budget_threshold": 120000,
    "auto_open_browser_on_start": true,
    "server_port": 8000
  }
}
```

### 14.2 Field Definitions

| Field | Type | Description |
|-------|------|-------------|
| `*_api_key` | str | API key for each provider |
| `default_video_model` | enum | Default model for new chunks |
| `default_lipsync_model` | enum | Default for new chunks |
| `falai_max_concurrent` | int | Semaphore limit (Architecture Section 10.2) |
| `vocal_isolation_provider` | enum | `lalai`, `passthrough`, `replicate` (future) |
| `lalai_engine` | enum | `phoenix`, `orion` |
| `user_supplies_vocals_only` | bool | Skip vocal isolation entirely |
| `output_directory` | absolute path | Root directory for projects |
| `ffmpeg_path` | str | Binary name or absolute path |
| `voice.tts_enabled` | bool | TTS opt-in |
| `voice.tts_voice` | str | Browser voice identifier |
| `voice.whisper_language_preference` | str | "auto" or language code |
| `advanced.conversation_token_budget_threshold` | int | When to trigger summarization |
| `advanced.server_port` | int | FastAPI bind port |

### 14.3 Validation Rules

- API key fields validate against expected prefixes (`sk-ant-`, `sk-`, `fal_`, etc.) when set
- `falai_max_concurrent` between 1 and 10
- `advanced.server_port` between 1024 and 65535
- `output_directory` exists and is writable (validated on save)

### 14.4 Migration on First Run

When the settings file doesn't exist, IRdeo writes a default settings object with all API keys set to `null`. The user must populate keys via the UI before any external calls work.

---

## 15. Global Projects Index (`~/.irdeo/projects_index.json`)

Quick-load registry of all projects. Lets the UI populate the project sidebar without scanning the entire output directory.

### 15.1 Schema

```json
{
  "schema_version": 1,
  "last_updated": "2026-05-11T09:14:02.117Z",
  "projects": [
    {
      "project_id": "p_8f3a2b1c",
      "song_slug": "manufacturing_consent",
      "song_title": "Manufacturing Consent",
      "album_context": "The System Burns",
      "current_state": "generating",
      "created_at": "2026-05-10T14:23:18.421Z",
      "last_modified": "2026-05-11T09:14:02.117Z",
      "project_path": "/Users/rasti/IRdeo_Projects/p_8f3a2b1c_manufacturing_consent"
    },
    {
      "project_id": "p_a1b2c3d4",
      "song_slug": "truth_in_chains",
      "song_title": "Truth in Chains",
      "album_context": "The System Burns",
      "current_state": "chunks",
      "created_at": "2026-05-09T10:14:33.012Z",
      "last_modified": "2026-05-09T18:42:18.117Z",
      "project_path": "/Users/rasti/IRdeo_Projects/p_a1b2c3d4_truth_in_chains"
    }
  ]
}
```

### 15.2 Field Definitions

`projects` is an array of lightweight project summaries (NOT the full state). Updated on:
- Project creation
- Project deletion
- State transitions that affect `current_state`

`current_state` is a high-level enum derived from the project's `state.json`:
- `setup` — intake not complete
- `conversation` — actively in conversation
- `timestamps` — timestamps review
- `chunks` — chunks review
- `generating` — generation job active or generation incomplete
- `complete` — full song generated and exported
- `failed` — last operation failed
- `archived` — soft-deleted

### 15.3 Reconciliation

If the index gets out of sync with actual project directories (e.g., user moves folders manually), `irdeo doctor` rebuilds it by scanning the output directory.

---

## 16. API Request/Response Schemas

Selected schemas for backend HTTP endpoints. Full set documented in API Integration Spec (separate doc).

### 16.1 POST /api/projects

Create a new project.

Request:
```json
{
  "song_title": "Manufacturing Consent",
  "album_context": "The System Burns",
  "genre": "Political Rap"
}
```

Response (200):
```json
{
  "project_id": "p_8f3a2b1c",
  "song_slug": "manufacturing_consent",
  "project_path": "/Users/rasti/IRdeo_Projects/p_8f3a2b1c_manufacturing_consent",
  "state": { ... full state.json ... }
}
```

### 16.2 POST /api/chat

Send a chat message.

Request:
```json
{
  "project_id": "p_8f3a2b1c",
  "message": "Let's make chunk 4 more documentary, no performer."
}
```

Response (200):
```json
{
  "turn_id": "t_b1e8c2a4",
  "reply": "Got it. Removing performer from chunk 4...",
  "actions": [
    { "type": "review_chunks", "label": "Review chunks", "primary": true }
  ],
  "metadata": { "phase": "phase_4_chunks", "model_used": "claude-opus-4-7" }
}
```

### 16.3 POST /api/upload

Multipart file upload.

Request: `multipart/form-data` with fields:
- `project_id` (str)
- `file_type` (`mp3` | `lyrics` | `reference_photo` | `vocals_stem`)
- `file` (binary)
- `photo_set_id` (str, only if `file_type == reference_photo`)

Response (200):
```json
{
  "stored_filename": "original.mp3",
  "stored_path": "source/original.mp3",
  "size_bytes": 6842150,
  "sha256": "a1b2c3d4..."
}
```

### 16.4 POST /api/generate/start

Start the generation job (Path B).

Request:
```json
{
  "project_id": "p_8f3a2b1c",
  "confirm_cost_above": 50.0
}
```

Response (200):
```json
{
  "job_id": "j_2c9e1d4f",
  "estimated_cost_usd": 18.42,
  "estimated_duration_seconds": 1800,
  "subchunks_to_generate": 47,
  "lipsync_subchunks": 22
}
```

Response (400) if cost exceeds threshold:
```json
{
  "error": "estimated_cost_exceeds_threshold",
  "estimated_cost_usd": 75.20,
  "threshold_usd": 50.0,
  "message": "Re-submit with confirm_cost_above >= 75.20 to proceed"
}
```

### 16.5 WS /ws/jobs/{job_id}

WebSocket stream of job events.

Each message:
```json
{
  "job_id": "j_2c9e1d4f",
  "timestamp": "2026-05-11T09:03:24.108Z",
  "event_type": "progress",
  "message": "Generated subchunk 04_b",
  "data": {
    "completed_steps": 20,
    "total_steps": 47,
    "current_step": "Generating subchunk 04_c — kling_motion_pro",
    "progress": 0.426
  }
}
```

`event_type` values: `progress`, `log`, `error`, `complete`.

---

## 17. Python Dataclass Reference

Canonical Python dataclass shapes for the core domain objects. Implementers should use these as the authoritative type signatures.

### 17.1 Project Models

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Literal, Optional
from pathlib import Path

# Phase state enums
PhaseState = Literal[
    "not_started", "in_progress", "awaiting_user",
    "draft", "finalized", "locked", "proposed", "generating",
    "generated", "complete", "partial", "failed", "cancelled",
    "building", "stale",
]

@dataclass
class ProjectStates:
    intake_state: PhaseState = "not_started"
    analysis_state: PhaseState = "not_started"
    timestamps_state: PhaseState = "not_started"
    chunks_state: PhaseState = "not_started"
    prompts_state: PhaseState = "not_started"
    generation_state: PhaseState = "not_started"
    lipsync_state: PhaseState = "not_started"
    preview_state: PhaseState = "not_started"
    export_state: PhaseState = "not_started"

@dataclass
class ProjectState:
    schema_version: int = 1
    project_id: str = ""
    song_slug: str = ""
    song_title: str = ""
    artist: Optional[str] = None
    album_context: Optional[str] = None
    track_number: Optional[int] = None
    genre: str = ""
    created_at: str = ""
    last_modified: str = ""
    irdeo_version: str = ""
    kb_version: str = ""
    active_view: Literal["welcome", "conversation", "timestamps", "chunks", "output"] = "welcome"
    states: ProjectStates = field(default_factory=ProjectStates)
    selected_path: Optional[Literal["A", "B"]] = None
    timestamps_finalized_at: Optional[str] = None
    chunks_locked_at: Optional[str] = None
    prompts_generated_at: Optional[str] = None
    skip_whisper: bool = False
    user_supplies_vocals_only: bool = False
    user_lyrics_provided: bool = False
    user_lyrics_path: Optional[str] = None
    mp3_path: str = ""
    vocals_only_path: Optional[str] = None
    duration_seconds: float = 0.0
    language: str = "en"
    active_job_id: Optional[str] = None
```

### 17.2 Chunk Models

```python
VocalType = Literal[
    "Male Rap", "Male Heavy Rap", "Male Sharp Rap", "Male Spoken",
    "Male Singing", "Male Singing Low", "Male Whisper", "Male Background",
    "Female Rap", "Female High Pitch", "Female Solo", "Female Singing",
    "Female Spoken", "Female Whisper", "Female Background",
    "Male + Female Duet", "Male + Female Call-Response",
    "Group Vocal", "Spoken", "Instrumental", "Sample",
]

VideoModel = Literal["kling_motion_pro", "kling_o3_pro", "veo_3", "veo_3_fast"]
LipsyncModel = Literal["lipsync-2", "lipsync-2-pro", "sync-3"]

@dataclass
class Subchunk:
    subchunk_id: str
    subchunk_letter: str
    sequence_index: int
    start_seconds: float
    end_seconds: float
    duration_seconds: float
    description_slug: str

@dataclass
class ChunkDefinition:
    chunk_id: str
    sequence_index: int
    section_label: str
    section_slug: str
    start_seconds: float
    end_seconds: float
    duration_seconds: float
    vocal_type: VocalType
    vocal_modifiers: list[str] = field(default_factory=list)
    performer_present: bool = False
    performer_subchunks: Optional[list[str]] = None
    reference_photo_set_id: Optional[str] = None
    mood: str = ""
    style_references: list[str] = field(default_factory=list)
    named_visuals: str = ""
    color_grade_override: Optional[str] = None
    model_preference: VideoModel = "kling_motion_pro"
    model_lipsync: Optional[LipsyncModel] = None
    user_notes: str = ""
    subchunks: list[Subchunk] = field(default_factory=list)

@dataclass
class ChunkManifest:
    schema_version: int = 1
    project_id: str = ""
    version: int = 1
    last_modified: str = ""
    chunks: list[ChunkDefinition] = field(default_factory=list)
```

### 17.3 Subchunk Prompt Model

```python
@dataclass
class SubchunkPrompt:
    schema_version: int = 1
    subchunk_id: str = ""
    chunk_id: str = ""
    subchunk_letter: str = ""
    sequence_index: int = 0
    start_seconds: float = 0.0
    end_seconds: float = 0.0
    duration_seconds: float = 0.0
    description_slug: str = ""
    prompt_text: str = ""
    continuity_carry_from: Optional[str] = None
    continuity_notes: str = ""
    model: VideoModel = "kling_motion_pro"
    reference_photo_set_id: Optional[str] = None
    reference_photo_filenames: list[str] = field(default_factory=list)
    expected_visual_summary: str = ""
    generated_at: str = ""
    generator_model: str = ""
    user_edited: bool = False
    user_edit_history: list[dict] = field(default_factory=list)
```

### 17.4 Job Model

```python
JobStatus = Literal["pending", "running", "succeeded", "failed", "cancelled"]
JobKind = Literal[
    "analysis", "generate_chunk", "generate_song", "regenerate_chunk",
    "build_preview", "export_prompt_pack", "export",
]

@dataclass
class Job:
    schema_version: int = 1
    id: str = ""
    kind: JobKind = "analysis"
    project_id: str = ""
    status: JobStatus = "pending"
    progress: float = 0.0
    current_step: str = ""
    total_steps: int = 0
    completed_steps: int = 0
    created_at: str = ""
    started_at: Optional[str] = None
    completed_at: Optional[str] = None
    error: Optional[str] = None
    result: Optional[dict] = None
    metadata: dict = field(default_factory=dict)
```

### 17.5 Cost Tracking

```python
@dataclass
class CostEntry:
    timestamp: str
    project_id: str
    service: Literal["openai", "anthropic", "fal_ai", "lalai"]
    operation: str
    input_units: float
    input_unit_type: Literal[
        "tokens_input", "tokens_output", "audio_seconds",
        "output_seconds", "input_seconds", "requests",
    ]
    cost_usd: float
    metadata: dict = field(default_factory=dict)
```

---

## 18. Schema Versioning & Migrations

### 18.1 Versioning Strategy

Every persistent file declares `schema_version` as the first field. When IRdeo reads a file:

```python
def load_with_migration(path: Path, target_schema_version: int):
    raw = json.loads(path.read_text())
    current = raw.get("schema_version", 1)
    while current < target_schema_version:
        raw = MIGRATIONS[current](raw)
        current += 1
    return raw
```

### 18.2 Migration Function Conventions

A migration function takes raw dict, returns raw dict at the next version:

```python
def migrate_chunk_manifest_v1_to_v2(raw: dict) -> dict:
    """v1 → v2: rename 'segments' to 'subchunks'."""
    raw["schema_version"] = 2
    for chunk in raw.get("chunks", []):
        if "segments" in chunk:
            chunk["subchunks"] = chunk.pop("segments")
    return raw
```

Each migration is **forward-only**, **idempotent**, and **lossless where possible** (data that can't be migrated cleanly is preserved as `_legacy_<field_name>`).

### 18.3 Registry

```python
MIGRATIONS = {
    "chunk_manifest": {
        1: migrate_chunk_manifest_v1_to_v2,  # example for future use
    },
    "project_state": {},  # none yet
    "settings": {},
    # ...
}
```

### 18.4 Cross-File Migrations

When a schema change affects multiple files (e.g., renaming a field that appears in both `state.json` and `chunks.json`), migrations run in a defined order per project. `irdeo migrate {project_id}` runs all pending migrations atomically (with backup).

---

## 19. File Integrity & Atomicity

### 19.1 Atomic Writes

All file writes use the **write-temp-then-rename** pattern:

```python
def atomic_write(path: Path, content: str | bytes) -> None:
    tmp = path.with_suffix(path.suffix + ".tmp")
    if isinstance(content, str):
        tmp.write_text(content, encoding="utf-8")
    else:
        tmp.write_bytes(content)
    tmp.replace(path)  # atomic on POSIX; near-atomic on Windows
```

This prevents partial-write states from corrupting state.

### 19.2 JSONL Append Atomicity

JSONL files (conversation, generation log, cost log) use append-mode writes with line buffering:

```python
def append_jsonl(path: Path, obj: dict) -> None:
    line = json.dumps(obj) + "\n"
    with path.open("a", encoding="utf-8") as f:
        f.write(line)
        f.flush()
```

A crash mid-write may leave a partial line at the end of the file. Readers must tolerate this:

```python
def read_jsonl_tolerant(path: Path) -> list[dict]:
    if not path.exists():
        return []
    out = []
    with path.open("r", encoding="utf-8") as f:
        for line in f:
            try:
                out.append(json.loads(line))
            except json.JSONDecodeError:
                pass  # skip partial last line
    return out
```

### 19.3 Backups Before Destructive Operations

Before destructive operations (migration, file replacement after regeneration), IRdeo writes a backup:

```python
def backup_before_modify(path: Path) -> Path:
    backup_dir = path.parent / ".backups"
    backup_dir.mkdir(exist_ok=True)
    timestamp = datetime.utcnow().strftime("%Y%m%dT%H%M%S")
    backup_path = backup_dir / f"{path.name}.{timestamp}.bak"
    shutil.copy2(path, backup_path)
    return backup_path
```

Backups retained for 30 days, then cleaned up via `irdeo doctor`.

---

## 20. Open Data Model Questions

Things to resolve as build progresses:

- **DMQ-1** Should ID prefixes be longer (e.g., `proj_` instead of `p_`) for self-documenting clarity? Current short prefixes save horizontal space; arguable both ways.
- **DMQ-2** Should `conversation.jsonl` get an index for fast jumping to specific turns? Probably not needed for MVP (linear scan is fast at expected sizes).
- **DMQ-3** Compression for large JSON files (audio analysis can be ~5MB)? Maybe gzip if disk pressure becomes a concern. Plain JSON for MVP.
- **DMQ-4** Schema migration testing — fixture corpus of "legacy" projects for migration validation? Add to test suite once first migration is needed.
- **DMQ-5** Should reference_photo binaries also have separate metadata files, OR just sit in the directory with one combined `metadata.json` per set? Currently combined per Section 5 — simpler, sufficient.
- **DMQ-6** Cost log granularity — should every Claude API call get its own entry, or aggregated per conversation? Currently per-call for accuracy; revisit if file sizes become a problem.
- **DMQ-7** Should the chunk manifest store the AI-generated subchunk definitions inline, OR reference separate subchunk files? Currently inline (Section 6) — single source of truth, simpler reads.
- **DMQ-8** Settings file size implications when API keys are encrypted — Fernet's overhead is fixed; not a concern.
- **DMQ-9** Concurrency — what if two browser tabs of the same project both try to write `state.json`? Last-write-wins for MVP; SaaS will need optimistic locking.

---

## 21. Provenance

This Data Model Specification was derived from:
- Knowledge Base v1.3 — domain rules driving structure (vocal vocabulary, conversation phases, Timestamps File canonical format)
- BRD v1.4 — functional requirements (FR-3.X timestamps, FR-4.X chunks, FR-7.X prompts, FR-9.X output)
- Architecture v1.2 — service ownership of each data store, JSON shapes loosely referenced in code samples
- UI Spec v1.0 — UI surfaces that need specific data fields
- PASC AI Assistant — reference pattern for JSON-based conversation state and structured-action metadata
- Manufacturing Consent Timestamps File — concrete example of the Timestamps File canonical format

Every schema in this document is grounded in either an explicit requirement, an existing service contract, or a UI display need. Speculative schemas (DMQ-* items) are flagged in Section 20.
