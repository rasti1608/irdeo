# IRdeo — UI Specification
**Version:** 1.0
**Last updated:** May 11, 2026
**Status:** LIVING DOCUMENT — comprehensive UI specification
**Companion documents:** `IRdeo_Knowledge_Base.md` (v1.3+), `IRdeo_BRD.md` (v1.4+), `IRdeo_Architecture.md` (v1.2+)

### Versioning Convention
- **v1.0** — first complete formal version
- **v1.1, v1.2, ...** — substantive revisions and additions
- **v2.0** — major restructure or fundamental approach change

### Changelog
- **v1.0 (May 11, 2026)** — First complete formal version. Defines the local web UI for IRdeo: page layouts, components, interaction patterns, state model, voice input, take-free workflow, two output paths (prompt pack download + full generation). Built on top of KB v1.3, BRD v1.4, Architecture v1.2.

---

## 1. Document Overview

### 1.1 Purpose
This document specifies the user-facing interface of IRdeo: what the user sees, what they can click, what they hear, and how state moves between views. It complements the Architecture Document (which says HOW the system is built) by saying WHAT THE USER EXPERIENCES.

This spec is the source of truth for the frontend implementation. It is also a usability contract: anything outside this spec is either undefined or future scope.

### 1.2 Scope
- **In scope:** Page layouts, component definitions, interaction patterns, state transitions, voice input/output, error display, progress visualization, color/typography baseline, accessibility minimums
- **Out of scope:** Detailed visual design (final colors, typography choices, illustration), brand identity work, marketing site, hosted SaaS interface (covered in Phase 2 spec when written)

### 1.3 Audience
- **Primary:** Frontend implementer (Claude Code CLI, or another agent/human acting on this spec)
- **Secondary:** Rasti, validating that the UX matches his mental model

### 1.4 Design Philosophy
Reference: PASC chatbot pattern. IRdeo's UI takes structural inspiration from that existing tool — chat-first, knowledge-aware, voice-capable, action-button-driven — and extends it for the music video production domain.

**Core principles:**
- **Conversation over forms.** The AI Director interviews the user. Forms appear only when conversation isn't the right shape for the input (e.g., file upload, take-it-all-at-once review screens).
- **One thing at a time.** No multi-step wizard fatigue. The user sees the current phase clearly; future phases reveal as needed.
- **Power-user trust.** Assume the user knows editing software. Don't dumb things down. Don't hide capabilities. Don't add hand-holding tooltips unless asked.
- **Visible progress.** Long-running operations (analysis, generation) show real-time progress with specifics ("subchunk 04_b of 47, model: kling_motion_pro"). Never a vague "Loading...".
- **Reversible actions.** Most actions can be undone, redone, or revised. Destructive actions (delete project, regenerate clip) confirm once before proceeding.

### 1.5 Related Documents
- `IRdeo_Knowledge_Base.md` — Domain rules
- `IRdeo_BRD.md` — Requirements, user stories
- `IRdeo_Architecture.md` — Backend technical architecture
- Data Model Spec (to be written) — JSON schemas
- API Integration Spec (to be written) — fal.ai, Claude, Whisper, Lalal.ai details

---

## 2. Top-Level Information Architecture

### 2.1 Page Map

IRdeo is a single-page application (SPA) with five primary views, plus a global settings drawer and a global help drawer.

```
┌──────────────────────────────────────────────────────────────┐
│                      GLOBAL TOPBAR                           │
│  IRdeo logo  │  Project switcher  │  Settings  │  Help       │
├──────────────────────────────────────────────────────────────┤
│  PROJECT      │                                              │
│  SIDEBAR      │              ACTIVE VIEW                     │
│               │                                              │
│  • New Song   │                                              │
│  • Manifest   │   One of:                                    │
│    Burns/...  │     A. Welcome / Project List                │
│    Slovak/... │     B. Conversation View (chat + inputs)     │
│  • Recent     │     C. Timestamps Review View                │
│               │     D. Chunks Review View                    │
│               │     E. Output View (preview + export)        │
│               │                                              │
│               │                                              │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 The Five Active Views

| View | Purpose | Triggered By |
|------|---------|--------------|
| **A. Welcome / Project List** | Land on app, see projects, create new one | App opens with no active project; user clicks "New Song" or selects existing |
| **B. Conversation** | Chat with AI Director; primary interaction across most of the workflow | Default view once a project is active |
| **C. Timestamps Review** | Review and correct the draft Timestamps File before locking it | AI Director invites user to review after Whisper + analysis complete |
| **D. Chunks Review** | See proposed chunk boundaries, adjust, configure each chunk's settings | AI Director invites user after Timestamps File is finalized |
| **E. Output** | Preview the stitched video, download prompt pack OR export to Filmora | AI Director invites user after generation completes (Path B), OR user clicks "Download Prompt Pack" (Path A) |

### 2.3 Navigation Model

Views are **stacked along the workflow timeline**, not arbitrarily navigable. The user can:
- **Move forward** by completing the current view (AI Director or button)
- **Move backward** by clicking the breadcrumb above the active view (e.g., "← back to Chunks")
- **Switch projects** anytime via the sidebar (project state is persisted)
- **Open Settings or Help** as overlays from the topbar (don't navigate away from active view)

A breadcrumb runs across the top of each view showing where in the workflow the user is:

```
Setup → Timestamps → Chunks → Output
   ✓        ✓          ●        ○
```

Filled circles = completed. Solid circle = current. Open circles = not yet reached.

### 2.4 Persistence Model

Every action persists to disk via the backend. The user can:
- Close the browser tab and reopen later — state resumes exactly where they were
- Switch between projects — each project remembers its own active view
- Refresh the browser — no data loss

The frontend keeps an in-memory snapshot for fast rendering but treats the backend as the source of truth. On any conflict (e.g., another browser tab modified state), the backend wins.

---

## 3. Global Components

### 3.1 Topbar

Fixed at the top of the viewport. Contains:

| Element | Behavior |
|---------|----------|
| **IRdeo logo** (left) | Click → Welcome view |
| **Project switcher** | Dropdown showing current project name + all other projects. Click → switches view to that project |
| **Settings icon** (gear) | Opens Settings drawer (right slide-in) |
| **Help icon** (?) | Opens Help drawer (right slide-in) |

Height: 56px. Stays visible across all views.

### 3.2 Project Sidebar

Fixed on the left, full height below topbar. Width: 240px (collapsible to 56px icon-only).

Contains:
- **"+ New Song" button** at the top, prominent
- **Project list** grouped by album/project context (e.g., "Burns Album" / "Slovak Album" / "Unsorted"). Within each group, songs listed chronologically (newest first) with their state badge (Setup / Conversation / Chunks / Generating / Done)
- **Search field** for fast filtering when project list grows long

Selecting a project loads it into the main panel. The currently-selected project is highlighted (background color, left accent bar).

### 3.3 Chat Panel (used in Conversation View, available as overlay elsewhere)

The core interaction component. Modeled on the PASC chatbot.

Layout:
```
┌────────────────────────────────────────────────────────┐
│                                                        │
│  [AI message]                                          │
│  Got it! Let's start by uploading the MP3 for your    │
│  song. Drag a file in or click below.                 │
│                                                        │
│  [📎 Upload MP3] [📎 Upload Lyrics (optional)]        │
│                                                        │
│  ─────────────────                                     │
│                                                        │
│              [User message →]                          │
│              MP3 uploaded: Manufacturing_Consent.mp3   │
│                                                        │
│  [AI message]                                          │
│  Excellent. Now — what genre is this song? (Political │
│  rap, Slovak atmospheric ballad, etc.)                │
│                                                        │
└────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────┐
│  [type or click microphone]                       [↑] │
│  [🎤] [⌘+Enter]                                       │
└────────────────────────────────────────────────────────┘
```

**Behaviors:**
- User messages right-aligned, AI messages left-aligned
- AI messages can include **action buttons** (e.g., "[📎 Upload MP3]") that trigger structured actions per Architecture Section 4.6
- Action buttons render inline, just below the message text that contains them
- Voice input via microphone icon (Whisper transcribes; transcript appears in input field for confirmation before send)
- Optional TTS voice output toggle (browser speech-synthesis reads AI messages aloud)
- Input field auto-grows up to 6 lines, scrolls beyond
- Markdown-rendered: bold, italic, code blocks, lists. No raw HTML.
- Conversation history persists between sessions (loaded from backend on view enter)

### 3.4 Progress Indicator

Used wherever long-running jobs are visible. Two variants:

**Inline (in chat panel):**
```
🎬 Generating chunk 04 (verse 2)
[████████░░░░░░░] subchunk 04_c of 7 — kling_motion_pro
~2 min remaining
```

**Banner (top of view, sticky):**
```
┌──────────────────────────────────────────────────────────┐
│  ⏳ Generating song — chunk 4 of 9 — 2 min remaining     │
│                                                  [Cancel]│
└──────────────────────────────────────────────────────────┘
```

Updates in real time via WebSocket connection to the JobOrchestrator (Architecture Section 9.4). The banner appears whenever ANY background job is running and disappears on completion.

### 3.5 Cost Display

Sticky bottom-right corner pill, present on all views during/after an active project:

```
💰  $4.27 spent
```

Click → opens a cost breakdown popover:

```
┌──────────────────────────────────────┐
│ Project: Manufacturing Consent       │
├──────────────────────────────────────┤
│ Analysis (Whisper + Claude)   $0.18  │
│ Conversation (Claude Opus)    $0.43  │
│ Video gen (Kling × 47)        $3.20  │
│ Lip sync (Sync × 22 sec)      $1.10  │
│ Vocal isolation (Lalal)       $0.36  │
│ Preview stitch (local)        $0.00  │
│ ──────────────────────────────────── │
│ Total                         $5.27  │
│                                      │
│ Estimated to complete:        $0.00  │
│ (all chunks generated)               │
└──────────────────────────────────────┘
```

Before generation runs, the estimate is shown prominently. After generation, the actual cost shows.

### 3.6 Settings Drawer

Right-slide-in panel, ~480px wide. Opens over the active view (semi-transparent backdrop).

Sections:
- **API Keys** (Anthropic, OpenAI, fal.ai, Lalal.ai) — masked inputs with show/hide toggle, "Test" button per service
- **Default Models** (default video model, default lipsync model, Claude model, Whisper model)
- **Generation Defaults** (falai_max_concurrent slider, default chunk duration target)
- **Vocal Isolation** (provider dropdown, Lalal engine selector, "user supplies vocals-only stem" toggle)
- **Output Directory** (path picker)
- **Voice** (TTS on/off, Whisper language preference)
- **Advanced** (debug log toggle, conversation token budget threshold)

Save button at the bottom. "Test all APIs" runs a quick sanity check (one call to each service) and displays ✓/✗ per service.

### 3.7 Help Drawer

Right-slide-in panel, ~480px wide.

Tabs:
- **Getting Started** — quick walkthrough of the workflow
- **Workflow** — the 9 phases (from KB Section 13), each expandable
- **Knowledge Base** — read-only view of the loaded KB markdown (for transparency)
- **Troubleshooting** — common issues with fixes (ffmpeg missing, API key invalid, generation failed, etc.)
- **About** — version, dependencies, system info, "irdeo doctor" output button

---

## 4. View A — Welcome / Project List

### 4.1 When It Appears
- First time the user launches IRdeo
- After deleting all projects
- User clicks the IRdeo logo with no active project

### 4.2 Layout

```
┌──────────────────────────────────────────────────────────────┐
│                          IRdeo                               │
│        AI Music Video Studio — Local Edition                 │
│                                                              │
│                                                              │
│              ┌──────────────────────────────┐                │
│              │      + Start New Song        │                │
│              └──────────────────────────────┘                │
│                                                              │
│                                                              │
│  Recent Projects                                             │
│  ────────────────────────────────────────────                │
│                                                              │
│  🎵 Manufacturing Consent         (Output / Done) →          │
│     Burns Album • 4:45 • Modified May 10                     │
│                                                              │
│  🎵 Truth in Chains               (Chunks)        →          │
│     Burns Album • 5:12 • Modified May 9                      │
│                                                              │
│  🎵 Sľúbili nám                   (Conversation)  →          │
│     Slovak Album • 3:48 • Modified May 7                     │
│                                                              │
│  [Show all 13 projects]                                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 4.3 Interactions

- **Start New Song** → opens a modal: "What's the song title?" → on submit, creates project and switches to Conversation View
- **Project row click** → loads project and switches to whichever view it left off in (e.g., Manufacturing Consent → Output View; Truth in Chains → Chunks View)
- **Show all** → expanded view with full project list, search, filter by album/state

### 4.4 Empty State

If no projects exist:

```
              ┌──────────────────────────────┐
              │      + Start New Song        │
              └──────────────────────────────┘

   Your music video studio starts with one song.
   IRdeo will guide you through it.

   What you'll need:
     • An MP3 file of your song
     • Exact lyrics (recommended, not required)
     • Reference photos if you'll appear on screen
```

---

## 5. View B — Conversation

The workhorse view. The AI Director and the user collaborate through chat. Most of the workflow happens here.

### 5.1 Layout

```
┌──────────────────────────────────────────────────────────────┐
│  ← Breadcrumb: Setup → Timestamps → Chunks → Output          │
│                  ●        ○          ○         ○             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│                  CHAT PANEL (Section 3.3)                    │
│                                                              │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  [type or click 🎤]                                    [Send]│
└──────────────────────────────────────────────────────────────┘
```

Full-bleed chat. No competing panels. The chat is the work.

### 5.2 Phase-Driven Conversation

The AI Director moves through the phases defined in KB Section 13. The breadcrumb at the top updates as phases complete.

**Phase 1 — Project Setup:**
- AI greets, asks for MP3 upload (action button)
- After upload, asks for genre (free-text or quick-pick chip suggestions: Political Rap / Slovak Atmospheric Ballad / Other)
- Asks for lyrics (action button with file picker + paste option; "Skip" allowed with warning)
- Asks for album/project context (optional, accept-or-skip)

**Phase 2 — Analysis (Background):**
- Inline progress: "🔄 Analyzing audio (Whisper + librosa)..."
- AI surfaces analysis results as a message: "Found 9 sections, BPM 112, energy peaks at 2:10, 3:55. Male and female vocals detected."
- If sanity check triggers (Section 5.4 below), shown as advisory banner in the message

**Phase 3 — Timestamps Finalization:**
- AI invites: "I've drafted the Timestamps File. Want to review?"
- Action button: "[Review Timestamps]" → switches to **View C (Timestamps Review)**
- After user confirms in View C and returns: AI confirms "Timestamps locked. Moving on."

**Phase 4 — Chunk Definition:**
- AI proposes chunk boundaries: "I'm suggesting 9 chunks based on the sections. Want to review?"
- Action button: "[Review Chunks]" → switches to **View D (Chunks Review)**
- Per chunk: AI asks performer presence, named visuals, style references, mood (inline in chat — one chunk at a time, or batch if user prefers)

**Phase 5 — Reference Research (if triggered):**
- AI: "You mentioned wanting it to feel like Eminem's Cleanin' Out My Closet. Let me look that up quickly..."
- Inline progress: "🔍 Researching reference..."
- AI presents findings with sources: "Cleanin' Out My Closet — confessional rap, ~98 BPM, performer-driven with B-roll cuts, cold color grade. Match the confessional energy? Or pull back further?"

**Phase 6 — Prompt Generation (Background):**
- Inline progress: "🔄 Writing prompts for 9 chunks..."
- AI surfaces: "All prompts ready. Want to see any specific chunk's prompt? Or proceed?"

**Phase 7 — Output Path Selection:**
- AI presents the two options as action buttons:
  - **[📦 Download Prompt Pack]** (Path A — secondary, secondary visual weight)
  - **[🎬 Generate Everything]** (Path B — primary, prominent button)
- AI text: "Prompts are ready. You can download them as a zip and use them elsewhere, OR have me run the full generation pipeline. What's the move?"

**Phase 8 — Generation & Lip Sync (Path B, Background):**
- Sticky banner appears (Section 3.4): "Generating song — chunk 4 of 9..."
- Chat remains usable for adjustments ("regenerate chunk 5 with a different mood")
- AI surfaces issues as they arise: "Chunk 6 generation failed twice; retrying with kling_o3_pro fallback."

**Phase 9 — Final Output:**
- AI: "Generation complete. Want to preview before exporting?"
- Action buttons: **[▶️ Preview]** (switches to View E) and **[📁 Open Output Folder]**

### 5.3 Action Button Patterns

Action buttons in the chat are inline elements that trigger structured actions (Architecture Section 4.6). Conventions:

- **Primary action** (the obvious next step): solid color, larger, prominent
- **Secondary action** (alternative path): outlined, smaller, lighter
- **Destructive action** (regenerate, delete): red accent, requires confirmation
- **File upload action**: includes a paperclip icon and accepts drag-and-drop onto the entire chat panel

Buttons render exactly where the AI placed them in the message text. Once clicked, they may transform (e.g., a "Generate" button becomes "Cancel" while the job runs).

### 5.4 Lyrics ↔ Audio Sanity Check Display

When the sanity check (Architecture Section 5.4) triggers a warning, the AI shows it inline:

```
┌──────────────────────────────────────────────────┐
│ ⚠️ Heads up — Lyrics may not match audio         │
│                                                  │
│ I'm having trouble matching your lyrics to this  │
│ audio. Only 18% of words overlap. Is this the    │
│ right combination?                               │
│                                                  │
│ [✓ Yes, proceed]  [↻ Upload different files]    │
└──────────────────────────────────────────────────┘
```

The user can dismiss and proceed, or click to re-upload. Never blocks the workflow.

### 5.5 Voice Input

A microphone icon sits next to the text input. Behaviors:

- **Click to record:** the icon turns red, indicates listening
- **Click again to stop:** the audio is sent to Whisper; transcription appears in the text field for review
- **Edit before sending:** user can correct Whisper errors before submitting
- **Auto-stop:** silence longer than 2 seconds triggers automatic stop (configurable)

Voice input is opt-in per turn — not always-on. The text field remains the default.

### 5.6 Voice Output (TTS)

Optional. Toggled in Settings.

When enabled:
- Browser TTS reads AI messages aloud as they stream in
- A "Stop" button appears next to the playing message
- User can pause / skip per message
- Action buttons in the message are NOT read aloud (just the prose)

Voice output is conversational-friendly but never replaces the visible text. The chat always shows the full conversation regardless of TTS state.

### 5.7 Conversation History Display

When the user enters the Conversation View for a project with prior history:
- The full history loads, scrolled to the bottom
- A "Scroll to top" affordance appears
- Older messages render slightly faded to emphasize the current cursor position
- A timestamp divider appears between conversation sessions (e.g., "─── Resumed May 11, 9:42 AM ───")

If conversation history exceeds the LLM context budget (Architecture Section 4.4), the UI surfaces a notice:
```
ℹ️ Conversation history has been summarized for efficiency.
   Full history is still available — click to expand.
```

---

## 6. View C — Timestamps Review

The user reviews the draft Timestamps File before locking it.

### 6.1 Layout

```
┌──────────────────────────────────────────────────────────────┐
│  ← Back to Conversation                                      │
│  Breadcrumb: Setup → Timestamps → Chunks → Output            │
│                ✓        ●          ○         ○               │
├──────────────────────────────────────────────────────────────┤
│  Timestamps Review — Manufacturing Consent                   │
│                                                              │
│  ┌──────────────┬─────────────────────────────────────────┐  │
│  │ AUDIO PLAYER │                                         │  │
│  │  ▶️ 1:23/4:45│         TIMESTAMPS EDITOR              │  │
│  │  ─────●──────│   (canonical .md format, editable)     │  │
│  │              │                                         │  │
│  │   Waveform   │   ## [INTRO — spoken] (0:00 - 0:22)    │  │
│  │   with       │   [0:00] Yeah…                          │  │
│  │   chunk      │   [0:02] I used to believe you.        │  │
│  │   markers    │   [0:05] Defended you.                  │  │
│  │              │   [0:07] Repeated what you told me...  │  │
│  │              │                                         │  │
│  │              │   ## [VERSE 1] (0:22 - 1:12)           │  │
│  │              │   [0:22] Have you ever sworn the...    │  │
│  │              │   ...                                   │  │
│  └──────────────┴─────────────────────────────────────────┘  │
│                                                              │
│  WHISPER CORRECTIONS                                         │
│  • "bloopery" → "blueprint"                                  │
│  • "I felt this choking" → "Five filters choking"           │
│  • [Add correction]                                          │
│                                                              │
│  [Discard Draft] [Save Draft] [✓ Finalize & Lock]           │
└──────────────────────────────────────────────────────────────┘
```

### 6.2 Components

**Audio Player (left column):**
- Waveform visualization showing the full song
- Click-to-seek anywhere on the waveform
- Section markers overlaid (color-coded: intro / verse / hook / bridge / outro)
- Current playback position scrolls the editor on the right in sync
- Play/pause hotkey: spacebar

**Timestamps Editor (right column):**
- The canonical Timestamps File rendered as live-editable markdown
- Click any timestamp → audio player jumps to that point
- Click any lyric line → highlighted, can be edited inline
- Section headers (## [VERSE 1] etc.) editable inline
- Vocal type sub-markers (Male Rap / Female High Pitch / etc.) are dropdowns
- Energy values shown as inline pills, editable on click

**Whisper Corrections (bottom):**
- Auto-populated as the user edits (system detects changes and logs them)
- Manual additions allowed
- Becomes part of the final Timestamps File when locked

### 6.3 Interactions

- **Edit a lyric inline:** click, type, click away to save (auto-saves draft to backend)
- **Adjust a timestamp:** click the timestamp number, type new value OR drag the marker on the waveform
- **Reassign vocal type:** click sub-marker, dropdown reveals canonical vocal type vocabulary (KB Section 2)
- **Listen to a section:** click section header to play that section in isolation (loops until user stops)

### 6.4 Validation

Before "Finalize & Lock" enables:
- No timestamp gaps that can't be explained (every second of audio accounted for)
- No timestamps out of order
- All section headers have valid types
- All sub-markers (when present) use canonical vocabulary

Invalid states show inline indicators (red underline + tooltip) — user can still save draft, but Finalize button is disabled.

### 6.5 Skip-Whisper Path

When Whisper was skipped (non-English + user lyrics), this view operates differently:
- No Whisper transcript exists; the user starts from their pasted lyrics
- The audio player still shows waveform with detected sections (from librosa)
- The user manually clicks on the waveform at each lyric line's start time
- "Mark current time" button or hotkey (Cmd+M) anchors the next unmarked lyric
- Once all lyrics are anchored, "Finalize & Lock" enables

### 6.6 Buttons

- **Discard Draft:** discards all edits, regenerates from Whisper output. Confirms once.
- **Save Draft:** persists current state without locking. User can return later.
- **Finalize & Lock:** locks the Timestamps File, returns to Conversation View with the breadcrumb advancing.

---

## 7. View D — Chunks Review

The user sees proposed chunk boundaries and configures each chunk's settings.

### 7.1 Layout

```
┌──────────────────────────────────────────────────────────────┐
│  ← Back to Conversation                                      │
│  Breadcrumb: Setup → Timestamps → Chunks → Output            │
│                ✓        ✓          ●         ○               │
├──────────────────────────────────────────────────────────────┤
│  Chunks Review — Manufacturing Consent                       │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │           WAVEFORM with CHUNK DIVIDERS               │    │
│  │  |░░░|████████|██████|████|███████|███|████|██|███|  │    │
│  │   ▲    ▲       ▲      ▲    ▲       ▲   ▲    ▲  ▲   │    │
│  │   01   02      03     04   05      06  07   08 09   │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  Chunks (9)                          [+ Add chunk]           │
│  ────────────────────────────────────────────────────────    │
│                                                              │
│  ┌──────────────────────────────────────────────────┐        │
│  │ 01 ▸ Intro (spoken)             0:00–0:22  (22s) │        │
│  │ ────────────────────────────────────────────────│        │
│  │ Performer:    ○ No   ● Yes   ○ Partial          │        │
│  │ Model:        [Kling Motion Pro     ▼]          │        │
│  │ Mood:         [intimate, vulnerable           ] │        │
│  │ Style ref:    [Eminem confessional           ] │        │
│  │ Visuals:      [add named visuals or notes...]  │        │
│  │ [Show prompt]  [Regenerate]  [Delete]           │        │
│  └──────────────────────────────────────────────────┘        │
│                                                              │
│  ┌──────────────────────────────────────────────────┐        │
│  │ 02 ▸ Verse 1                    0:22–1:12  (50s) │        │
│  │ ────────────────────────────────────────────────│        │
│  │ ... (collapsed)                                  │        │
│  └──────────────────────────────────────────────────┘        │
│                                                              │
│  ... (more chunks, expand on click)                          │
│                                                              │
│  [Save Draft]  [✓ Lock Chunks & Generate Prompts]           │
└──────────────────────────────────────────────────────────────┘
```

### 7.2 Components

**Waveform with Chunk Dividers (top):**
- Same waveform as Timestamps Review but with chunk boundaries overlaid as vertical lines
- Each chunk is numbered and color-coded
- Drag a divider to adjust the boundary between two chunks (snaps to subchunk-friendly timestamps)
- Click a chunk band to scroll to and expand its card below

**Chunk Cards:**
- One card per chunk, vertically stacked
- Collapsed by default (shows just header line: number, section name, time range)
- Click header to expand
- Inside expanded card: all per-chunk configuration controls

**Per-Chunk Controls:**
- **Performer:** radio toggle (No / Yes / Partial — Partial reveals timestamp range picker for performer subchunks)
- **Model:** dropdown of available video models (Kling Motion Pro, Kling O3 Pro, Veo 3.x). Tooltip shows recommendation per chunk type (e.g., "Recommended: Kling Motion Pro for performer chunks")
- **Mood:** free-text, autocomplete suggests common moods
- **Style references:** chip input — user types artist/song name and Enter; chips appear; click to remove
- **Visuals:** free-text area for named real-world visuals, logos, references (e.g., "CBS logo, Bari Weiss photo, $150M raining")
- **Show prompt:** modal showing the engineered prompt for this chunk (read-only or editable with override)
- **Regenerate:** triggers regeneration of just this chunk (only available after prompts are generated)
- **Delete:** removes chunk; adjacent chunks expand to fill the gap. Confirmation required.

### 7.3 Lock Chunks & Generate Prompts

Locking advances to the next phase (prompt generation in background, then output path selection). User returns to Conversation View.

If user wants to skip generating prompts and just download what exists, "Lock Chunks" is the same button — prompts always generate. The Path A vs Path B choice happens AFTER prompt generation in the Conversation View.

---

## 8. View E — Output

After generation (Path B) or prompt-pack export (Path A), the user lands here.

### 8.1 Layout — Path B (Full Generation Complete)

```
┌──────────────────────────────────────────────────────────────┐
│  ← Back to Conversation                                      │
│  Breadcrumb: Setup → Timestamps → Chunks → Output            │
│                ✓        ✓          ✓         ●               │
├──────────────────────────────────────────────────────────────┤
│  Output — Manufacturing Consent                              │
│                                                              │
│  ┌──────────────────────────────────────────────────┐        │
│  │                                                  │        │
│  │             PREVIEW VIDEO PLAYER                 │        │
│  │             (stitched preview.mp4)               │        │
│  │             ▶️ 1:23 / 4:45                       │        │
│  │             ─────────●─────────                  │        │
│  │                                                  │        │
│  │   Chunk markers along the timeline so user can  │        │
│  │   jump to any chunk and see its specific output  │        │
│  └──────────────────────────────────────────────────┘        │
│                                                              │
│  ┌──────────────────────────────────────────────────┐        │
│  │  Generated Clips  (47 subchunks across 9 chunks) │        │
│  │  ────────────────────────────────────────────── │        │
│  │  01_a_intro.mp4    [▶️] [↻ regen]  ✓ lip synced │        │
│  │  01_b_intro.mp4    [▶️] [↻ regen]  ✓ lip synced │        │
│  │  01_c_intro.mp4    [▶️] [↻ regen]                │        │
│  │  02_a_verse.mp4    [▶️] [↻ regen]                │        │
│  │  ...                                             │        │
│  └──────────────────────────────────────────────────┘        │
│                                                              │
│  [📂 Open Output Folder]  [📥 Export Filmora Project]       │
│  [📄 Generate Master Doc]                                    │
│                                                              │
│  ╔══════════════════════════════════════════════════╗        │
│  ║ Power-User Options                               ║        │
│  ║ [📦 Also Download Prompt Pack]                   ║        │
│  ╚══════════════════════════════════════════════════╝        │
└──────────────────────────────────────────────────────────────┘
```

### 8.2 Components

**Preview Video Player (top):**
- Full-bleed video element, plays the ffmpeg-stitched preview.mp4
- Standard playback controls (play/pause, scrub, fullscreen)
- Chunk boundary markers shown along the timeline
- Click a marker → jumps to that chunk's start

**Generated Clips List:**
- One row per subchunk in flat-naming order (01_a, 01_b, 01_c, 02_a, ...)
- Each row: filename, play button (opens inline mini-player), regenerate button (triggers regenerating just that subchunk's parent chunk), lip-sync status indicator
- Clicking the filename copies the absolute path to clipboard (useful for manual Filmora drag-and-drop)

**Primary Action Buttons:**
- **Open Output Folder:** OS-level "reveal in finder/explorer"
- **Export Filmora Project:** generates `.wfp` (or fallback `.xml`) file. Only enabled when ALL chunks complete. Tooltip explains the gating if disabled.
- **Generate Master Doc:** produces the master `.docx`. Only enabled when ALL chunks complete.

**Power-User Options (secondary card):**
- **Also Download Prompt Pack:** even after generation, the user can grab the prompt zip for reference / backup / sharing

### 8.3 Layout — Path A (Prompt Pack Export Only)

When the user chose Path A in the Conversation phase, this view is simpler:

```
┌──────────────────────────────────────────────────────────────┐
│  ← Back to Conversation                                      │
│  Breadcrumb: Setup → Timestamps → Chunks → Output            │
│                ✓        ✓          ✓         ●               │
├──────────────────────────────────────────────────────────────┤
│  Output — Manufacturing Consent                              │
│                                                              │
│  Path A: Prompt Pack Export                                  │
│  No video generation performed.                              │
│                                                              │
│  ┌──────────────────────────────────────────────────┐        │
│  │  📦 manufacturing_consent_prompts.zip (47 KB)   │        │
│  │                                                  │        │
│  │  9 chunk files + timestamps.md + README          │        │
│  │                                                  │        │
│  │  [📥 Download Zip]   [📂 Show in Folder]         │        │
│  └──────────────────────────────────────────────────┘        │
│                                                              │
│  ╔══════════════════════════════════════════════════╗        │
│  ║ Change your mind?                                ║        │
│  ║ [🎬 Run Full Generation Pipeline Instead]       ║        │
│  ╚══════════════════════════════════════════════════╝        │
└──────────────────────────────────────────────────────────────┘
```

User can download the zip and stop, OR pivot to Path B if they want to see what IRdeo would have generated.

### 8.4 Partial Generation State

If the user has generated only some chunks (not the full song):

```
┌──────────────────────────────────────────────────────────────┐
│  ⚠️ Partial Generation — 5 of 9 chunks complete              │
│                                                              │
│  Filmora project and Master Doc are gated until all chunks  │
│  are complete. For now, you can:                            │
│                                                              │
│  • Preview what's been generated (preview.mp4 below)         │
│  • Manually import individual clips into your editor        │
│    using the flat naming convention (01_a_*.mp4, etc.)      │
│  • Generate remaining chunks: [Generate Missing Chunks]      │
└──────────────────────────────────────────────────────────────┘
```

The preview still works, just shorter. Filmora / docx buttons are disabled with explanatory tooltip.

---

## 9. Modals and Overlays

### 9.1 New Song Modal
Triggered by "+ Start New Song". Fields:
- Song title (required)
- Album/project context (optional dropdown — autocompletes from existing albums or "+ New album")
- Genre (optional at creation; will be asked in conversation if skipped)

Buttons: [Cancel] [Create]

### 9.2 Delete Project Confirmation
Triggered from project sidebar context menu. Two-step:
1. "Are you sure you want to delete '{song_title}'? This cannot be undone."
2. Type the song title to confirm.

### 9.3 Show Prompt Modal (from Chunks Review)
Reveals the engineered prompt for a chunk. Tabs:
- **Chunk-level prompt** (the long narrative form)
- **Subchunk prompts** (the 7-8 individual API-ready prompts with continuity notes)

Read-only by default. "Edit" button toggles editability — overrides the AI's prompt for that chunk. Override is preserved in the chunk manifest.

### 9.4 Regenerate Confirmation
Triggered from chunk card or generated clips list. Shows:
- Estimated cost for the regeneration
- Current and proposed prompt diff (if user edited prompt)
- "Use same model" / "Switch model to..." options
- [Cancel] [Regenerate]

### 9.5 Error Modal
For unrecoverable errors that block the workflow:
```
┌──────────────────────────────────────────────────┐
│  ⚠️ fal.ai API key not configured                │
│                                                  │
│  Go to Settings to add your fal.ai API key.     │
│  You can get one at fal.ai/dashboard.           │
│                                                  │
│  [Open Settings]   [Dismiss]                     │
└──────────────────────────────────────────────────┘
```

Tied to the UserError pattern (Architecture Section 15.3).

---

## 10. State Synchronization

### 10.1 Backend Source of Truth
All persistent state lives on the backend. The frontend is a render layer.

### 10.2 Sync Patterns

| Action | Sync Pattern |
|--------|--------------|
| User sends chat message | POST /api/chat → optimistic UI (show message immediately) → reconcile with response |
| User edits Timestamps File | Debounced (1s) PATCH /api/timestamps → backend persists; UI shows "Saved" briefly |
| User adjusts chunk boundary | PATCH /api/chunks → immediate save (snap-on-release) |
| Generation job progress | WebSocket /ws/jobs/{id} → real-time updates → UI re-renders progress |
| User switches project | GET /api/projects/{id} → loads full state → renders the project's last active view |

### 10.3 Optimistic vs Pessimistic Updates

**Optimistic** (UI updates before backend confirms):
- User chat messages (show in chat immediately, backend confirms on response)
- Settings toggles (apply visually, backend syncs)
- Chunk boundary drags (visual feedback during drag, save on release)

**Pessimistic** (UI waits for backend):
- File uploads (show progress, success on confirmation)
- Generation triggers (button shows "Starting..." until job ID returned)
- Project creation (modal stays open until project ID returned)

### 10.4 Conflict Resolution

If the backend returns a different state than the frontend expected (e.g., another browser tab modified the project):
- Frontend shows a non-blocking notice: "This project was updated elsewhere. Reload?"
- "Reload" pulls fresh state and re-renders
- "Dismiss" keeps the local state but disables saves until reload

---

## 11. Voice Input/Output Detail

### 11.1 Voice Input Flow

1. User clicks microphone icon
2. Browser requests mic permission (first time only)
3. UI shows recording state: red mic icon, level meter, "Listening..."
4. User speaks; level meter responds
5. Silence detection (default 2 sec) auto-stops, OR user clicks mic icon again
6. Audio sent to backend `/api/voice` endpoint
7. Whisper transcribes; transcript returned
8. Transcript populates the text input field, NOT auto-sent
9. User reviews transcription, can edit, clicks Send

This 2-step pattern (transcribe → review → send) prevents Whisper errors from causing wrong messages.

### 11.2 Voice Output Flow

When TTS is enabled in Settings:
- Each new AI message triggers browser SpeechSynthesis
- Voice plays inline; visual indicator shows "Speaking..."
- User can click the speaker icon next to any past message to replay
- Stop button cancels playback
- Voice selection (Settings) lets user pick among system voices
- Markdown formatting is stripped before TTS (no "asterisk asterisk" reading)

### 11.3 Accessibility

- All voice features have keyboard alternatives
- TTS is opt-in, not default-on
- Visual transcription always available regardless of TTS state
- Screen readers can read the full chat conversation linearly

---

## 12. Color, Typography, Layout Baseline

The visual design is intentionally minimal at MVP. Final design polish can happen later.

### 12.1 Color Palette (MVP defaults)

```
Backgrounds:
  --bg-primary:   #0F1115 (near-black, easy on eyes for long sessions)
  --bg-secondary: #1A1D24 (sidebar, cards)
  --bg-tertiary:  #252830 (input fields, hover states)

Text:
  --text-primary:   #E8E8E8 (main copy)
  --text-secondary: #A0A0A0 (metadata, secondary info)
  --text-muted:     #6B6B6B (timestamps, hints)

Accents:
  --accent-primary: #FF6B35 (IRdeo orange; CTAs, active states)
  --accent-success: #4ADE80 (✓ statuses)
  --accent-warning: #FBBF24 (warnings, advisory)
  --accent-error:   #EF4444 (errors, destructive)
  --accent-info:    #60A5FA (info banners, neutral statuses)

Borders / Dividers:
  --border-subtle:  rgba(255,255,255,0.08)
  --border-strong:  rgba(255,255,255,0.16)
```

Dark mode is default. Light mode can be added later.

### 12.2 Typography

- **System font stack:** `-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif`
- **Monospace:** `"SF Mono", Monaco, "Courier New", monospace` for code, timestamps, file paths
- **Base size:** 14px
- **Line height:** 1.5 for body, 1.3 for headings
- **Heading scale:** 24 / 20 / 18 / 16 / 14 / 12

### 12.3 Spacing

- 4px base unit
- Common values: 4, 8, 12, 16, 24, 32, 48, 64
- Layouts use generous whitespace; this is a creative tool, not a dashboard

### 12.4 Component Anatomy

**Button:**
- Padding: 8px 16px (small), 12px 24px (medium), 16px 32px (large)
- Border-radius: 6px
- Transition: 150ms ease for hover/active states

**Input:**
- Padding: 12px 16px
- Border: 1px solid var(--border-subtle); focus → var(--accent-primary)
- Border-radius: 6px

**Card:**
- Background: var(--bg-secondary)
- Padding: 24px
- Border-radius: 12px
- Subtle shadow on hover

---

## 13. Accessibility Minimums

- **Keyboard navigation:** Tab through all interactive elements in logical order; Enter activates buttons; Esc closes modals
- **Focus indicators:** Visible focus ring on all interactive elements (browser default + custom enhancement)
- **Color contrast:** WCAG AA minimum on all text against background
- **Screen reader labels:** All icons have aria-labels; complex components (waveform, chunk cards) have aria-describedby
- **No motion-required interactions:** Critical actions don't require precise mouse motion (drag-to-redraw chunks has keyboard equivalent: arrow keys nudge boundaries)
- **Voice input not required:** Every voice feature has a text equivalent

Full WCAG AAA compliance is not an MVP goal but should not be made impossible by MVP decisions.

---

## 14. Frontend Technical Approach

### 14.1 Stack

Per Architecture Section 12.3:
- **HTML + CSS + Vanilla JS** at MVP (no heavy framework)
- **Optional:** Alpine.js or htmx for reactive state if vanilla becomes unwieldy
- **No build step** required for MVP — files served directly by FastAPI

### 14.2 File Organization

```
irdeo/frontend/
├── index.html              (main shell, includes all CSS/JS)
├── style.css               (single stylesheet — split later if grows)
├── app.js                  (main controller, view router)
├── components/
│   ├── chat.js
│   ├── timestamps_editor.js
│   ├── chunks_review.js
│   ├── audio_player.js
│   ├── progress_banner.js
│   ├── settings_drawer.js
│   └── help_drawer.js
├── lib/
│   ├── api_client.js       (wraps fetch calls to backend)
│   ├── ws_client.js        (WebSocket helper for job progress)
│   ├── markdown.js         (lightweight MD renderer for chat)
│   └── audio_record.js     (MediaRecorder wrapper for voice input)
└── assets/
    ├── logo.svg
    └── icons/  (SVG icons for buttons)
```

### 14.3 State Management

A simple state-store pattern. No Redux, no MobX.

```javascript
// app.js
const state = {
  activeProjectId: null,
  activeView: 'welcome',
  projects: [],
  currentProject: null,  // full project data when loaded
  activeJob: null,       // currently-running background job
  settings: null,
  conversation: [],
};

const listeners = [];
function setState(patch) {
  Object.assign(state, patch);
  listeners.forEach(fn => fn(state));
}
function subscribe(fn) { listeners.push(fn); }
```

Components subscribe to state changes and re-render their affected portions. This is intentionally minimal — keeps the codebase debuggable.

### 14.4 API Client

```javascript
// lib/api_client.js
const api = {
  async getProjects() {
    return (await fetch('/api/projects')).json();
  },
  async createProject(data) {
    return (await fetch('/api/projects', {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify(data),
    })).json();
  },
  async chat(projectId, message) {
    return (await fetch('/api/chat', {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({project_id: projectId, message}),
    })).json();
  },
  // ... etc
};
```

### 14.5 WebSocket Client

```javascript
// lib/ws_client.js
function subscribeToJob(jobId, onProgress, onComplete, onError) {
  const ws = new WebSocket(`ws://localhost:8000/ws/jobs/${jobId}`);
  ws.onmessage = (event) => {
    const evt = JSON.parse(event.data);
    if (evt.event_type === 'progress') onProgress(evt);
    if (evt.event_type === 'complete') { onComplete(evt); ws.close(); }
    if (evt.event_type === 'error') { onError(evt); ws.close(); }
  };
  return ws;
}
```

Used by the progress banner (Section 3.4) to render live updates.

---

## 15. Open UI Questions

These need resolution during build but don't block the spec:

- **UQ-1** Should the chat panel auto-scroll to the latest message, or hold position when user is scrolling history? Probably auto-scroll only when user is already at bottom.
- **UQ-2** Should there be a "command palette" (Cmd+K) for power-user shortcuts? Probably MVP-future, not MVP-essential.
- **UQ-3** How does the user re-open a song that's in mid-conversation? Project sidebar click → loads conversation history → resumes at last AI message? Confirmed approach.
- **UQ-4** Should chunk cards in Chunks Review be drag-reorderable? Probably not — chunk order is locked to the timeline; reordering means redrawing boundaries, which is already supported.
- **UQ-5** Confidence threshold for warning display in lyrics-audio sanity check — same 25% as backend, or adjustable per user? Use backend default; not adjustable in MVP UI.
- **UQ-6** When generation fails (e.g., fal.ai 5xx errors after all retries), how is the user notified? Inline AI message + sticky error banner + log link? Confirmed approach.
- **UQ-7** Should the preview video player support setting in/out points (clip the preview)? Probably MVP-future. Filmora is where editing happens.
- **UQ-8** What's the UX for re-running prompt generation on an existing project (e.g., KB has been updated)? Probably an "Regenerate all prompts" button in Settings or in the Chunks View.

---

## 16. Provenance

This UI Specification was derived from:
- Knowledge Base v1.3 — domain rules and conversation flow phases
- BRD v1.4 — user stories, scope, two-path output model
- Architecture v1.2 — backend components, API routes, state model, JobOrchestrator pattern
- PASC AI Assistant (`AI_Assistant_Technical_Architecture.md`) — proven reference UI pattern: chat-first, voice-capable, action-button-driven
- Direct user input during spec process — local web UI vision (BRD v1.3), prompt pack as power-user export (BRD v1.4), take selection removal (BRD v1.4)

Every UI decision is grounded in either an explicit requirement, a domain rule, or a usability principle named in Section 1.4. Speculative choices are flagged in Section 15 (Open UI Questions).
