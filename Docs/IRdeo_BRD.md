# IRdeo — Business Requirements Document
**Version:** 1.3
**Last updated:** May 11, 2026
**Status:** LIVING DOCUMENT — comprehensive formal version
**Companion documents:** `IRdeo_Knowledge_Base.md` (v1.2+)

### Versioning Convention
- **v1.0** — first complete formal version
- **v1.1, v1.2, ...** — substantive revisions and additions
- **v2.0** — major restructure or fundamental approach change

### Changelog
- **v1.3 (May 11, 2026)** — User interface vision corrected. IRdeo's primary user interface is a **local web UI** (browser-based, served by local Python process), NOT a CLI. CLI is internal implementation/dev tooling, not user-facing. Phase 2 SaaS = same web UI, hosted instead of local. Updates: Section 1.3 companion docs (CLI Spec → UI Spec), Section 2.2 (phasing strategy clarified), Section 6.1 (CLI Interface → User Interface), Section 6.2 (SaaS UI clarified as hosted version of same UI), Section 6.4 (deferred items corrected), Section 7 (US-5 revised + new US-5a, US-5b for local web UI), Section 8.11 (FR-11 block rewritten), Section 12.2 (Python dependencies updated with FastAPI/Uvicorn), Section 14.1 (roadmap milestones revised to include web UI build).
- **v1.2 (May 11, 2026)** — Lip sync added as mandatory MVP capability for performer chunks. fal.ai locked as unified generation gateway (video + lip sync, single API key, pay-per-use). Updates: Section 6.1 scope (lip sync added to Video Generation), Section 7 user stories (take selection + automatic lip sync added), Section 8 functional requirements (new FR-8.X entries for lip sync provider integration and take selection workflow), Section 9.6 cost ceiling adjusted to account for lip sync costs, Section 12.1 dependencies (fal.ai added, Sync.so as model provider via fal.ai), Section 13 risks (lip sync quality variance row added), Section 15 open questions revised.
- **v1.1 (May 11, 2026)** — Product renamed from "IronRUST Video Studio" to **IRdeo**. Coined name preserving the "IR" origin (IronRUST) with the "deo" suffix (audio/video/media family). All in-text references updated. Companion document filename reference updated.
- **v1.0 (May 11, 2026)** — First complete formal version. Comprehensive BRD covering MVP and SaaS as two distinct phases. Built on top of Knowledge Base v1.0.

---

## 1. Document Overview

### 1.1 Purpose
This document defines the business requirements for **IRdeo** — an AI-assisted music video generation pipeline. It establishes the problem being solved, the users being served, the scope of the solution, detailed user stories, functional and non-functional requirements, success metrics, constraints, risks, and the two-phase roadmap (MVP → SaaS).

This BRD is intended to be the **source of truth for what the product does and for whom**. It precedes the Architecture Document (what the product is technically composed of) and the Phase Specs (how each component works in detail).

### 1.2 Audience
- **Primary:** The product owner and builder (Rasti) — for personal alignment and to drive build planning
- **Secondary:** Future contributors / collaborators on the project
- **Tertiary:** Potential SaaS investors or partners (relevant in Phase 2)

### 1.3 Related Documents
- **Knowledge Base (`IRdeo_Knowledge_Base.md`)** — Domain expertise loaded into the conversation LLM at runtime. Defines HOW the AI Director thinks. This BRD references it heavily but does not duplicate it.
- **Architecture Document** — To be written. Defines technical components, data flow, deployment topology.
- **Phase Specs** — To be written. One per phase of the conversation flow (Intake, Analysis, Collaboration, Chunking, Generation, Output).
- **API Integration Spec** — To be written. Veo, Kling, Whisper, Claude — what we use, costs, limits, fallbacks.
- **Data Model Spec** — To be written. JSON manifests, song manifest, chunk manifest, segment manifest.
- **UI Spec** — To be written. Local web UI design, page structure, chat panel, take selection, file upload, project management, progress visualization. The user-facing interface specification.
- **Output Spec (Filmora)** — To be written. Filmora `.wfp` investigation + fallback formats.

---

## 2. Executive Summary

### 2.1 The Problem
AI music video generation today is a two-tier market with a missing middle:

- **Consumer tools (e.g., AIVideo.com)** wrap powerful video generation engines (Veo, Kling) behind a primitive prompt-input UX. The engines themselves are capable of cinematic output, but the UX gives users no help in writing the elaborate, structured, lyric-aware prompts those engines need to produce quality output. Result: most users get generic garbage and blame the engine.
- **Professional tools** require deep technical skill, manual prompt engineering, manual chunk management, and significant time investment. Even power users who know what they're doing spend hours on tedious orchestration that could be automated.

There is no tool that combines **smart prompt-engineering automation** with **full creative control and raw output**, priced for individual creators.

### 2.2 The Solution
IRdeo is a pipeline that:
1. **Analyzes** a song (audio, lyrics, structure, vocals, energy)
2. **Collaborates** with the user via an intelligent conversation that asks the right questions in the right order
3. **Generates** engine-ready video prompts in the background — chunk by chunk, segment by segment, continuity-aware
4. **Fires** video API calls directly (bypassing consumer-tool wrappers and their markups)
5. **Outputs** organized raw video clips plus a Filmora project file ready for the user to open and finish manually

The user's creative control is preserved. The tedious work (timestamping, prompt-writing, chunk management, API orchestration, file organization) is automated. Cost per video is 80–95% lower than consumer tools.

### 2.3 Phasing Strategy
- **Phase 1 — MVP (2026):** Build for one user (Rasti) running locally. Local web UI in the browser (FastAPI backend serving a simple HTML/JS frontend on localhost), connected to fal.ai and other APIs. Validate the thesis on his remaining IronRUST tracks and Slovak album work.
- **Phase 2 — SaaS (TBD, when MVP is proven):** Same web UI, hosted in the cloud instead of local. Add multi-tenancy, authentication, billing, cloud storage. Target other power-user creators who know editing but want a hassle-free hosted experience.

The core engine (conversation brain + Knowledge Base + video generation pipeline) AND the user interface are identical between phases. SaaS adds infrastructure (cloud hosting, multi-tenancy, billing), not new features.

---

## 3. Business Context

### 3.1 Origin
This project emerged from production experience on the IronRUST album "The System Burns" — a 13-track political rap album where the creator has been manually building music videos using AIVideo.com (Kling models) plus Filmora editing. Over five videos (Red Lines, Eisenhower's Warning, Truth in Chains, Manufacturing Consent in progress, others), a pattern emerged: the consumer tool's output quality is entirely dependent on the user's manual prompt-writing skill. The engine isn't dumb; the interface is.

The realization in Session 07: AIVideo.com is a thin wrapper around the same engines anyone can call directly via API. What they sell is the wrapper, not the engine. A smart, music-aware, conversation-driven prompt-engineering layer would beat their UX while costing a fraction of their price.

### 3.2 Market Gap
| Segment | Existing Solutions | Gap |
|---------|-------------------|-----|
| Casual users | AIVideo.com, Runway consumer UI, Pika | Bad output without expert prompts |
| Pro creators | Manual API + scripting + editing software | Hours of tedious orchestration |
| **Power-user creators who know editing** | **None** | **No tool combines smart automation with raw output and full control** |

IRdeo targets the third segment — initially with an MVP for the single power user (Rasti), then scaling to others like him.

### 3.3 Strategic Opportunity
- **AI video generation quality is improving rapidly.** Veo 3.1, Kling O3 Pro, Runway Gen-3 all capable of cinematic output today.
- **API access is becoming cheaper and more direct.** Going around consumer-tool wrappers is increasingly viable.
- **The "prompt engineering as a service" pattern** (where the smart layer is the prompt-writing brain, not the model) is emerging across many AI categories — this tool is an instance of that pattern in the music video domain.
- **Filmora and similar editing tools have plateaued on AI integration.** Generating a ready-to-edit Filmora project from analyzed audio is a real gap.

---

## 4. Vision & Strategic Objectives

### 4.1 Vision Statement
**Make AI music video generation feel as effortless as having a conversation with a music video director who understands your song, asks the right questions, and hands you the raw footage ready to edit.**

### 4.2 Strategic Objectives

**Phase 1 (MVP) Objectives:**
- **O1:** Reduce time-to-completed-video by ≥50% compared to current manual workflow (target: from ~6–8 hours per song to ~3 hours)
- **O2:** Reduce cost per video by ≥80% compared to AIVideo.com (target: from ~$125 per video to ~$10–25)
- **O3:** Match or exceed quality of manually-produced videos (subjective assessment by Rasti)
- **O4:** Complete remaining IronRUST tracks (Socialism for the Rich, Masks Slipped Off, The Root, plus video for already-written tracks) using the tool
- **O5:** Validate that the Knowledge Base + Conversation engine approach produces production-quality prompts consistently

**Phase 2 (SaaS) Objectives:**
- **O6:** Acquire N paying power-user creators (specific N TBD when MVP validates)
- **O7:** Maintain ≥70% gross margin on per-video API costs
- **O8:** Achieve net revenue retention ≥100% (users stay and spend more over time)
- **O9:** Build a Knowledge Base that grows organically from real user production experience across genres

---

## 5. Target Users & Personas

### 5.1 Phase 1 Persona: Rasti (Primary MVP User)
- **Background:** Slovak-American music creator, early 50s. Eastern European immigrant perspective informing political/systemic critique work.
- **Skills:** Experienced in Filmora, Suno, AIVideo.com, manual prompt engineering, Python scripting at intermediate level. Comfortable running local Python tools that open a browser interface. Deep editing skill.
- **Projects:** IronRUST political rap album ("The System Burns" — 13 tracks). Slovak atmospheric ballad album under NoNameDOG AI Studio. Personal Slovak gift songs for friends.
- **Tools in use:** Suno (vocals/music), AIVideo.com (current video), Kling models, Filmora (editing), ChatGPT (image generation), Whisper API (transcription), librosa (audio analysis), Python/docx for documentation.
- **Workflow:** Pick Eminem song as cadence template → write lyrics matching that cadence → research extensively via political podcasts → generate vocals with Suno → manually build music video via AIVideo.com → assemble in Filmora → release on YouTube.
- **Pain points:**
  - AIVideo.com is expensive and produces variable quality
  - Writing detailed prompts for 9 chunks per song is tedious
  - Manual timestamp alignment is error-prone
  - No tool understands his album's thematic continuity
  - Whisper output requires constant manual correction
- **Goals:** Complete the remaining IronRUST tracks faster and cheaper. Maintain creative control. Build a tool that could eventually serve other creators like himself.

### 5.2 Phase 2 Persona: Pro Creator (SaaS User)
- **Background:** Independent music creator, podcaster, content producer. Could be in any genre. 25–60 years old.
- **Skills:** Comfortable with editing software (Premiere, DaVinci, Final Cut, Filmora). May or may not be technical beyond that — but does NOT want to write code or manage APIs.
- **Projects:** Single tracks, EPs, full albums. Promotional videos. Lyric videos. Music documentary content.
- **Tools in use:** Their own audio production setup. Their preferred editing software. May currently use Sora, Runway, Pika, Kling consumer UI, or pay video producers.
- **Pain points:**
  - Video production tools are expensive subscription models
  - Consumer AI video tools produce generic output
  - Hiring video producers is cost-prohibitive for indie creators
  - No tool integrates well with their existing editing workflow
- **Goals:** Get high-quality music video footage faster and cheaper without learning to write API prompts or code.

### 5.3 Explicit Non-Personas
The tool is NOT designed for:
- **Beginners with no editing experience** — output is raw chunks requiring editing skill in Filmora or equivalent. KB Section 14 (What This Tool Does NOT Do) confirms: no hand-holding, no auto-transitions, no auto-effects, no final export.
- **Non-music video use cases** — wedding videos, vlogs, corporate content. Knowledge Base is music-specific (genre playbooks, vocal type vocabulary, audio analysis).
- **Live performance / streaming use cases** — the tool generates pre-rendered clips, not real-time video.
- **Users wanting fully automated end-to-end output** — the tool produces raw material for human editing, not finished videos.

---

## 6. Scope

### 6.1 In Scope — MVP (Phase 1)

**Conversation & Knowledge:**
- Knowledge Base loaded into conversation LLM system prompt (per spec in KB Section 1)
- Per-song context (Timestamps File, audio analysis, chunk manifest) loaded into conversation system prompt
- Conversation state persistence between turns (local file or SQLite)
- User question hierarchy per KB Section 4
- KB rule override flow per KB Section 0

**Audio Analysis:**
- MP3 ingestion
- Whisper transcription (with skip-Whisper option for non-English or user-provided lyrics)
- librosa-based audio analysis (BPM, energy curve, beat positions, brightness)
- Claude content analysis for sections, vocal types, mood
- Draft Timestamps File generation in canonical format (KB Section 2)

**Timestamps File Management:**
- Draft → Finalized workflow per KB Section 2
- User-driven correction and confirmation
- Lock and persist finalized file per song
- Whisper corrections log

**Chunk Definition:**
- Auto-propose chunk boundaries from finalized Timestamps File
- User confirmation / override per KB Section 4
- Per-chunk metadata: performer presence, named visuals, mood, style references, model selection
- Chunk manifest persistence (separate file from Timestamps File)

**Reference Management:**
- Per-project reference photo library
- Per-chunk reference photo binding
- Performer continuity defaults per KB Section 5
- Multi-performer handling

**Reference-Informed Research:**
- Pre-baked reference profiles for common artists (Eminem catalog minimum)
- Live web search fallback for unknown references
- Grounding rule per KB Section 6
- Research output used to inform questions, never embedded in final prompt

**Prompt Generation:**
- Per-chunk prompt generation from Knowledge Base rules + user input + Timestamps File + chunk manifest
- Per-segment decomposition (max 8-sec per segment, continuity-aware)
- Genre-playbook-driven prompt construction per KB Section 7
- Performer integration rules per KB Section 8
- Within-chunk continuity rules per KB Section 9
- Audio-to-visual translation vocabulary per KB Section 10

**Video Generation:**
- API integration via **fal.ai** as the unified generation gateway (single API key for video + lip sync)
- Access to multiple video models through fal.ai: Kling 3.0 Motion Control Pro, Kling O3 Pro, Veo 3.x (visuals only — Veo's native audio is discarded), and others as the catalog grows
- Per-chunk model selection (different chunks can use different models)
- Multiple takes per chunk (configurable, default 2–3) — silent video
- Retry logic on API failure
- Generation progress tracking

**Take Selection & Lip Sync:**
- Take selection UI/workflow — present takes per performer chunk to user, user picks the winner
- Automatic lip sync (mandatory, no opt-in) on performer chunks via fal.ai → Sync.so lipsync-2 model
- Vocal track isolation (where possible) before lip sync for best results
- Non-performer chunks skip lip sync entirely

**Output:**
- Organized output directory per song
  - chunks/ folder with subdirectory per chunk
  - Multiple takes per chunk
  - reference_images/ subdirectory
  - prompts/ subdirectory (JSON of all generated prompts)
  - timestamps file (finalized)
  - chunk manifest (JSON)
  - generation log
- Filmora project file generation (or fallback interchange format if `.wfp` not crackable)
- Master `.docx` document with Suno prompt, production notes, album context, lyrics, video prompts, Whisper corrections

**User Interface:**
- Local web UI served by a FastAPI Python backend running on the user's machine
- User opens browser to `localhost:<port>` to interact with the tool
- Chat-style interface as the primary interaction model (AI Director conversation, modeled on the PASC chatbot pattern Rasti previously built)
- Voice input via Whisper API (microphone button, speak instead of type)
- Optional voice output via browser TTS (AI Director speaks responses)
- File upload (MP3, lyrics, reference photos) via drag-and-drop or file picker
- Project sidebar / dashboard — list of song projects, switch between them, resume work
- Take selection UI — visual grid of generated takes per performer chunk, click to select winner
- Real-time progress indicators during long-running generation (chunk N of M, segment N of M, take N of M)
- Inline video/audio playback (preview takes, listen to chunk audio slices)
- Configuration page for API keys and defaults
- Windows-first (Rasti's primary OS), but the local web UI runs cross-platform anywhere Python + a browser are available

**Storage:**
- Local file system
- Project-based directory structure
- No cloud, no remote storage in MVP

### 6.2 Added in SaaS (Phase 2)

Building on the MVP foundation, Phase 2 adds:

**Infrastructure:**
- Hosted web UI (cloud-served version of the same UI built for MVP — same frontend code, just deployed on cloud infrastructure instead of localhost)
- Multi-tenant architecture
- Cloud hosting
- Cloud storage per user (encrypted at rest)
- Database for user accounts, project metadata, billing data
- Background job processing for long-running generation tasks

**User Management:**
- Authentication (OAuth via Google/GitHub/Apple, or email/password)
- User profiles
- Per-user reference photo library
- Per-user project list
- Per-user usage history

**Billing:**
- Payment processor integration (Stripe likely)
- Subscription tiers OR pure pay-per-use model (TBD via market validation)
- Usage tracking per user (LLM tokens, video API costs)
- Cost transparency dashboard
- Billing invoices

**Operational:**
- Logging and monitoring (Datadog, Sentry, or similar)
- Rate limiting per user
- Abuse prevention
- Customer support workflow

**Optional / TBD for SaaS:**
- Collaboration (multiple users on one project)
- Template / preset marketplace
- Public showcase / gallery
- White-label or API offering for resellers

### 6.3 Out of Scope (Both Phases)

Explicit non-goals — these will NOT be built regardless of phase:

- **Real footage / B-roll integration** (KB Section 14) — AI-generated clips only
- **Transitions between chunks** — user handles in editing software
- **Effects (VHS, RGB, overlays, color correction beyond initial grade)** — user handles in editing software
- **Music generation** — user generates audio externally (Suno, etc.) before using this tool
- **Final video export** — tool outputs raw chunks + Filmora project; user opens and exports
- **Real-time / live performance video** — pre-rendered clips only
- **Non-music video use cases** — wedding, corporate, vlog, etc.
- **Beginner-friendly hand-holding UX** — tool assumes editing skill

### 6.4 Out of Scope MVP, Deferred to SaaS

These are MVP non-goals that may become SaaS scope:

- Hosted/cloud web UI (local web UI is MVP, hosted is SaaS)
- Multi-user / authentication
- Payment processing
- Cloud storage
- Collaboration features
- Marketplace / public gallery

---

## 7. User Stories

User stories follow the format: **As a [persona], I want [capability] so that [outcome].**

Stories are grouped by capability area and tagged with phase (MVP / SaaS / BOTH).

### 7.1 Intake & Setup

**US-1 [MVP]** As Rasti, I want to start a new song project from the web UI by clicking "New Song" and entering a song name so I can begin work without manual folder setup.

**US-2 [MVP]** As Rasti, I want to provide my MP3, exact lyrics (optional but recommended), and reference photos at project initialization so the tool has everything it needs to start analysis.

**US-3 [MVP]** As Rasti, I want the tool to ask me the genre at the start of the conversation so it can apply the right visual playbook.

**US-4 [MVP]** As Rasti, I want the tool to explicitly warn me if I skip the exact-lyrics input so I understand the quality tradeoff (Whisper-only timestamps will need significant correction).

**US-5 [MVP]** As Rasti, I want IRdeo to run as a local web app — I start the tool, it opens my browser to a local URL, and I interact with it through a browser interface (chat, file upload, take selection, progress, etc.) — so the experience feels like a polished web product, not a script.

**US-5a [MVP]** As Rasti, I want to speak to the AI Director via my microphone (Whisper-powered voice input) instead of typing every response so the conversation flows naturally during longer interview sessions.

**US-5b [MVP]** As Rasti, I want optional voice output (browser TTS) so the AI Director can speak responses back to me, allowing hands-free conversation when I'm focused on other tasks.

**US-5c [SaaS]** As a Pro Creator, I want to use the same web UI hosted in the cloud (no installation required) so I can use IRdeo from any computer with a browser.

### 7.2 Audio Analysis & Timestamps

**US-6 [BOTH]** As a user, I want Whisper transcription + librosa analysis to run automatically when I provide an MP3 so I don't have to invoke them separately.

**US-7 [BOTH]** As a user, I want to see the draft Timestamps File presented section by section so I can review it before finalization.

**US-8 [BOTH]** As a user, I want to correct Whisper errors against my provided lyrics inline in the conversation so the finalized Timestamps File is accurate.

**US-9 [BOTH]** As a user, I want to confirm or correct AI-detected section boundaries, vocal types, and energy map so I'm in control of the source-of-truth document.

**US-10 [BOTH]** As a user with a non-English song, I want to skip Whisper entirely and do manual timestamping with the tool's help so I don't waste time on unusable Whisper output.

**US-11 [BOTH]** As a user, I want the finalized Timestamps File saved as a markdown file in my project directory so I can reference it outside the tool if needed.

### 7.3 Chunk Definition & Collaboration

**US-12 [BOTH]** As a user, I want the tool to propose chunk boundaries based on my finalized Timestamps File so I have a starting structure without manual work.

**US-13 [BOTH]** As a user, I want to redraw chunk boundaries if I disagree with the AI's proposal so I have final creative control over structure.

**US-14 [BOTH]** As a user, I want the tool to ask me, per chunk: "Performer on screen? Which named visuals? What mood? Any style references?" so the prompt-generation has my creative direction encoded.

**US-15 [BOTH]** As a user, I want to bind specific reference photos to specific chunks (e.g., "use my photo set A for chunks 1, 4, 6" and "use photo set B for chunks 3, 7") so different performers can appear in different sections.

**US-16 [BOTH]** As a user, I want the tool to default to performer continuity (same person across same-vocal-type sections) but let me override per chunk.

**US-17 [BOTH]** As a user, I want to name a reference artist or song (e.g., "feels like Portishead") and have the tool use that knowledge to ask me better questions about my visual vision (NOT to copy the reference into the prompt).

**US-18 [BOTH]** As a user, when I provide a genre the tool doesn't have a pre-built playbook for, I want the tool to build a session-level playbook with me by asking for 2–3 reference songs so I'm not blocked.

### 7.4 Prompt Generation

**US-19 [BOTH]** As a user, I do NOT want to see the auto-generated prompts unless I explicitly ask for them — the tool's job is to write them, not show them to me.

**US-20 [BOTH]** As a user, I want to inspect any generated prompt on demand (e.g., "show me the prompt for chunk 4") so I can audit or learn from the tool's output.

**US-21 [BOTH]** As a user, I want to manually edit a chunk's prompt if I disagree with what the tool generated so I can override the AI Director when needed.

**US-22 [BOTH]** As a user, I want the tool to acknowledge when I'm overriding a KB rule and confirm my intent before proceeding so I don't accidentally bend rules.

### 7.5 Video Generation

**US-23 [BOTH]** As a user, I want to fire video generation for all chunks with a single command (`generate`) and have the tool orchestrate all the API calls.

**US-24 [BOTH]** As a user, I want to specify how many takes per chunk (default 2–3) so I can cherry-pick the best clips later.

**US-25 [BOTH]** As a user, I want to regenerate a specific chunk (e.g., `regenerate --chunk 5 --takes 2`) without re-running the whole song so I don't waste API credits on already-good chunks.

**US-26 [BOTH]** As a user, I want to choose the video generation model per chunk (or accept the tool's recommendation) so I can use Kling Motion Control Pro for performer chunks and Kling O3 Pro or Veo for documentary chunks.

**US-27 [BOTH]** As a user, I want clear progress reporting during generation (e.g., "chunk 3 of 9 — segment 4 of 7 — take 2 of 3") so I know what's happening during long generation runs.

**US-28 [BOTH]** As a user, I want failed API calls to retry automatically (with backoff) and fall back to an alternate provider if available so I'm not blocked by transient failures.

**US-28a [BOTH]** As a user, I want to review all takes for each performer chunk and pick the winning take before lip sync runs so I don't waste API spend on takes I'm going to discard.

**US-28b [BOTH]** As a user, I want lip sync to run automatically on every performer chunk (no toggle, no opt-in) so my performer's mouth always matches the actual song audio — performance without lip sync is broken output.

**US-28c [BOTH]** As a user, I want the tool to skip lip sync entirely on documentary/atmosphere/no-performer chunks so I don't pay for processing that isn't needed.

**US-28d [BOTH]** As a user, I want vocals isolated from the instrumental track before lip sync runs so the lip-sync model gets cleaner input and produces better mouth alignment.

### 7.6 Output & Handoff

**US-29 [BOTH]** As a user, I want all output organized in a predictable directory structure per song so I can find any chunk, any take, any reference photo, any prompt later.

**US-30 [BOTH]** As a user, I want a Filmora project file generated automatically with all clips placed on the timeline in order, audio track attached, and multiple takes stacked on separate video tracks, so I can open Filmora and immediately start editing.

**US-31 [BOTH]** As a user, if the Filmora `.wfp` format proves uncrackable, I want a fallback interchange format (FCP XML / Premiere XML / EDL) generated instead so I can still import into Filmora (or other editing software) with some manual setup.

**US-32 [BOTH]** As a user, I want the master `.docx` document generated automatically at the end of a song (with Suno prompt, production notes, album context, lyrics, video prompts, Whisper corrections) so my song archive is complete.

### 7.7 SaaS-Specific Stories

**US-33 [SaaS]** As a Pro Creator, I want to sign up for the service with my Google account so I don't have to manage another password.

**US-34 [SaaS]** As a Pro Creator, I want a dashboard showing my projects, recent generations, and cost-to-date so I have full visibility into my usage and spend.

**US-35 [SaaS]** As a Pro Creator, I want to upload my reference photo library once to my account and have it available across all my projects.

**US-36 [SaaS]** As a Pro Creator, I want to pay only for what I use (pay-per-generation, no monthly commitment) so I'm not locked into a subscription if I'm between projects.

**US-37 [SaaS]** As a Pro Creator, I want my project data isolated from other users and encrypted at rest so my unreleased music doesn't leak.

**US-38 [SaaS]** As a Pro Creator, I want to download all my project files (chunks, manifest, Filmora project, Timestamps File, master `.docx`) as a single zip so I have full ownership of my work.

---

## 8. Functional Requirements

Numbering: **FR-X.Y** where X is the capability area and Y is the requirement number within that area.

### 8.1 Intake (FR-1.X)
- **FR-1.1** The tool MUST accept an MP3 file as primary input.
- **FR-1.2** The tool MUST accept user-provided lyrics as text input (optional but flagged as recommended).
- **FR-1.3** The tool MUST accept reference photos (file paths or upload) as optional input.
- **FR-1.4** The tool MUST accept genre as a mandatory input before proceeding to analysis.
- **FR-1.5** The tool MUST accept project context (album name, related songs) as optional input.
- **FR-1.6 [SaaS]** The tool MUST support drag-and-drop file upload via web UI.

### 8.2 Analysis (FR-2.X)
- **FR-2.1** The tool MUST run Whisper transcription on provided MP3 (with skip option for non-English / user-lyrics-provided scenarios).
- **FR-2.2** The tool MUST run librosa-based audio analysis producing BPM, energy curve (2-second resolution), beat positions, brightness, and percussive intensity.
- **FR-2.3** The tool MUST run Claude content analysis on transcript to detect sections, vocal types, mood arc.
- **FR-2.4** The tool MUST produce a draft Timestamps File in the canonical format defined in Knowledge Base Section 2.

### 8.3 Timestamps File Management (FR-3.X)
- **FR-3.1** The tool MUST present the draft Timestamps File to the user section by section for confirmation.
- **FR-3.2** The tool MUST allow inline correction of Whisper errors against user-provided lyrics.
- **FR-3.3** The tool MUST log Whisper corrections in the canonical Whisper Corrections block.
- **FR-3.4** The tool MUST allow user to correct section boundaries, vocal types, and energy map.
- **FR-3.5** The tool MUST persist the finalized Timestamps File as a `.md` file in the project directory.
- **FR-3.6** The tool MUST load the finalized Timestamps File into every subsequent conversation API call for that song's session.
- **FR-3.7** The tool MUST allow re-finalization of the Timestamps File mid-session if user makes a correction.

### 8.4 Chunk Definition (FR-4.X)
- **FR-4.1** The tool MUST propose chunk boundaries based on the finalized Timestamps File aligned to musical/lyrical section boundaries.
- **FR-4.2** The tool MUST allow user to confirm, modify, or redraw chunk boundaries.
- **FR-4.3** The tool MUST gather, per chunk: performer presence, named visuals, mood, style references, model selection.
- **FR-4.4** The tool MUST persist chunk definitions in a chunk manifest (JSON) separate from the Timestamps File.

### 8.5 Reference Management (FR-5.X)
- **FR-5.1** The tool MUST maintain a per-project reference photo library.
- **FR-5.2** The tool MUST allow per-chunk binding of reference photos.
- **FR-5.3** The tool MUST default to performer continuity (same photos for same-vocal-type sections) with per-chunk override.
- **FR-5.4** The tool MUST support multiple male performers and multiple female performers per song.
- **FR-5.5 [SaaS]** The tool MUST maintain a per-user reference photo library accessible across all projects.

### 8.6 Reference-Informed Research (FR-6.X)
- **FR-6.1** The tool MUST maintain pre-baked reference profiles for common artists/songs (Eminem catalog minimum at MVP).
- **FR-6.2** The tool MUST fall back to live web search for unknown references.
- **FR-6.3** The tool MUST present research findings to the user and require confirmation before committing to conversation context (per KB Section 6 grounding rule).
- **FR-6.4** The tool MUST NOT embed reference research output directly into final video prompts.

### 8.7 Prompt Generation (FR-7.X)
- **FR-7.1** The tool MUST generate a full chunk-level prompt from KB rules + user input + Timestamps File + chunk manifest.
- **FR-7.2** The tool MUST decompose each chunk into 8-second API segments.
- **FR-7.3** The tool MUST write continuity-aware segment prompts (eyeline, lighting, color grade, setting carry across segments).
- **FR-7.4** The tool MUST apply the appropriate genre playbook (KB Section 7) based on user-selected genre.
- **FR-7.5** The tool MUST apply performer integration rules (KB Section 8) — performer only at strategic moments, never throughout by default.
- **FR-7.6** The tool MUST allow user to inspect generated prompts on demand.
- **FR-7.7** The tool MUST allow user to manually edit generated prompts before generation.

### 8.8 Video Generation (FR-8.X)
- **FR-8.1** The tool MUST integrate with **fal.ai** as the unified generation API gateway for video generation at MVP. fal.ai provides access to Kling 3.0 Motion Control Pro, Kling O3 Pro, Veo 3.x, and other video models via a single API key.
- **FR-8.2** The tool MUST support per-chunk model selection.
- **FR-8.3** The tool MUST support configurable takes per chunk (default 2–3) — silent video only.
- **FR-8.4** The tool MUST support regeneration of specific chunks without re-running the full song.
- **FR-8.5** The tool MUST report generation progress in real time.
- **FR-8.6** The tool MUST retry failed API calls with exponential backoff (configurable max retries).
- **FR-8.7** The tool MUST be architected so video API providers can be swapped via a uniform interface (avoiding fal.ai lock-in if needed).
- **FR-8.8** When using video models with native audio generation (e.g., Veo), the tool MUST discard the generated audio and use only the visual track. User-provided audio drives lip sync separately (FR-8.9 block).

### 8.8a Take Selection & Lip Sync (FR-8.X continued)
- **FR-8.9** The tool MUST present takes for each performer chunk to the user for review and selection BEFORE lip sync runs (cost discipline — avoid lip-syncing discarded takes).
- **FR-8.10** The tool MAY provide a heuristic-based default-take selection if user opts to skip manual review.
- **FR-8.11** The tool MUST automatically run lip sync on every performer chunk's winning take. No per-chunk toggle. No opt-in. Performance implies lip sync per KB Section 8.
- **FR-8.12** The tool MUST integrate lip sync via fal.ai → Sync.so lipsync-2 model (default) with the ability to swap to alternative lip-sync models (Sync.so lipsync-2-pro, sync-3) via configuration.
- **FR-8.13** The tool MUST skip lip sync entirely on chunks without performer presence (documentary, atmosphere, no-performer).
- **FR-8.14** The tool SHOULD isolate vocals from the instrumental track before passing audio to the lip-sync model. If a vocals-only stem is provided by the user (typical with Suno output), use it directly.
- **FR-8.15** The tool MUST output lip-synced video as the final per-chunk artifact for performer chunks — the silent generation is intermediate and not delivered as final output.

### 8.9 Output (FR-9.X)
- **FR-9.1** The tool MUST organize output in a predictable directory structure per song (chunks/, reference_images/, prompts/, timestamps file, chunk manifest, generation log).
- **FR-9.2** The tool MUST generate a Filmora project file with all clips on the timeline, audio attached, takes on separate video tracks.
- **FR-9.3** If Filmora `.wfp` format is not viable, the tool MUST generate a fallback interchange format (FCP XML / Premiere XML / EDL).
- **FR-9.4** The tool MUST generate a master `.docx` document at song completion containing Suno prompt, production notes, album context, lyrics, video prompts, Whisper corrections.
- **FR-9.5 [SaaS]** The tool MUST allow user to download all project files as a single zip.

### 8.10 Conversation & State (FR-10.X)
- **FR-10.1** The tool MUST load the Knowledge Base into every conversation LLM API call as part of the system prompt.
- **FR-10.2** The tool MUST load the per-song context (Timestamps File, chunk manifest, etc.) into every conversation LLM API call when working on a song.
- **FR-10.3** The tool MUST persist conversation state between turns (local file or SQLite for MVP, database for SaaS).
- **FR-10.4** The tool MUST acknowledge KB rule overrides explicitly and confirm user intent before proceeding (per KB Section 0).
- **FR-10.5** The tool MUST log all KB rule overrides for future KB revision analysis.

### 8.11 User Interface (FR-11.X)
- **FR-11.1 [MVP]** The tool MUST provide a local web UI as the primary user interface. The backend runs as a local Python (FastAPI) process; the UI is delivered to the user's default browser at a local URL (e.g., `http://localhost:8000`).
- **FR-11.2 [MVP]** The local web UI MUST support all primary user workflows: project creation, file upload (MP3, lyrics, reference photos), conversation with the AI Director, Timestamps File review and confirmation, chunk definition, take selection, generation progress monitoring, output review, and access to final deliverables.
- **FR-11.3 [MVP]** The local web UI MUST support voice input via Whisper API (microphone button, audio capture, transcription, send to AI Director).
- **FR-11.4 [MVP]** The local web UI SHOULD support voice output via browser TTS as an optional setting.
- **FR-11.5 [MVP]** The tool MUST work on Windows (Rasti's primary OS) but the local web UI MUST be cross-platform (works wherever Python + a modern browser are available).
- **FR-11.6 [MVP]** The local web UI MUST support persistent state — closing the browser tab and reopening it MUST resume the user's session without data loss.
- **FR-11.7 [SaaS]** The same web UI MUST be deployable in a hosted cloud environment for the SaaS phase, with minimal frontend code changes (only the backend hosting and authentication layer differs).
- **FR-11.8 [MVP — internal]** The tool MAY expose a CLI for advanced power-user batch operations, scripting, and developer/debug workflows. The CLI is NOT the primary user-facing interface and is not required for typical use.

### 8.12 SaaS-Specific (FR-12.X)
- **FR-12.1 [SaaS]** The tool MUST support multi-tenant architecture with isolated user data.
- **FR-12.2 [SaaS]** The tool MUST support OAuth authentication (at minimum Google).
- **FR-12.3 [SaaS]** The tool MUST integrate with a payment processor (Stripe likely) for billing.
- **FR-12.4 [SaaS]** The tool MUST track usage per user (LLM tokens, video API costs, generations) and display a cost dashboard.
- **FR-12.5 [SaaS]** The tool MUST encrypt user data at rest.
- **FR-12.6 [SaaS]** The tool MUST support rate limiting per user to prevent abuse.

---

## 9. Non-Functional Requirements

### 9.1 Performance
- **NFR-P1** Audio analysis (Whisper + librosa) MUST complete within 5 minutes for a 5-minute song under typical hardware.
- **NFR-P2** Per-segment video generation typically takes 30–90 seconds (API-bound, not tool-bound). The tool MUST NOT add significant overhead.
- **NFR-P3** Conversation LLM response time MUST be under 10 seconds for typical questions.
- **NFR-P4** Full song workflow (analysis → finalized timestamps → all chunks defined → all takes generated) MUST complete within 90 minutes for a typical 5-minute song with 9 chunks at 3 takes each.

### 9.2 Reliability
- **NFR-R1** The tool MUST handle API failures gracefully (retry, backoff, fallback).
- **NFR-R2** Conversation state MUST persist across tool crashes (recoverable session).
- **NFR-R3** Output file writes MUST be atomic — no partial files leaving inconsistent state.
- **NFR-R4** The tool MUST validate API responses before saving to disk (rejecting corrupt or empty clips).

### 9.3 Security
- **NFR-S1 [MVP]** API keys MUST be stored in a config file with appropriate file permissions (read-only for user, no group/world access).
- **NFR-S2 [MVP]** API keys MUST NEVER appear in logs, prompts, or output files.
- **NFR-S3 [SaaS]** User data MUST be encrypted at rest.
- **NFR-S4 [SaaS]** All client-server communication MUST use HTTPS.
- **NFR-S5 [SaaS]** Reference photos MUST be access-controlled per user.
- **NFR-S6 [SaaS]** Payment data MUST never touch the tool's servers (Stripe-hosted checkout).

### 9.4 Usability
- **NFR-U1 [MVP]** Local web UI MUST follow consistent design language (clear visual hierarchy, predictable component behavior, sensible defaults).
- **NFR-U2 [MVP]** Error messages MUST be clear and actionable (not stack traces).
- **NFR-U3 [MVP]** The tool MUST include built-in `--help` documentation for all commands.
- **NFR-U4 [SaaS]** The web UI MUST be responsive and work on desktop browsers (Chrome, Firefox, Safari, Edge — last 2 major versions).

### 9.5 Maintainability
- **NFR-M1** The architecture MUST be modular with clear interfaces between phases (intake, analysis, conversation, generation, output).
- **NFR-M2** The Knowledge Base MUST be loaded from a file (markdown or YAML) — no recompile needed to update knowledge.
- **NFR-M3** Logs MUST be structured (JSON) for ingestion into analysis tools.
- **NFR-M4** Video generation API providers MUST be swappable without changing core code (uniform interface).

### 9.6 Cost Efficiency
- **NFR-C1** Total API cost per full 5-minute song video (analysis + conversation + video generation with 3 takes + lip sync on performer winning takes) MUST be under $35 at MVP launch. The ~$5 increase from the v1.1 ceiling accounts for added lip sync costs (~$3–5/song typical via fal.ai → Sync.so lipsync-2 at ~60 sec of performer per song).
- **NFR-C2** The tool MUST display estimated cost before firing generation calls so user can decline if it exceeds their budget.
- **NFR-C3** The tool MUST not waste API calls on already-generated content (cache and reuse where applicable).

---

## 10. Success Metrics & KPIs

### 10.1 MVP Success Criteria (must hit ALL to declare MVP a success)

**Quantitative:**
- **M1** — At least 3 IronRUST tracks completed using the tool end-to-end (proves the workflow works for real production)
- **M2** — Time per video reduced by ≥50% compared to manual workflow (measured: time from MP3 + lyrics to ready-to-edit Filmora project)
- **M3** — Cost per video under $30 (measured: total API costs from analysis through final generation)
- **M4** — Generation success rate ≥95% (measured: % of API calls that produce usable clips without need for manual retry beyond automatic retry)
- **M5** — Subjective quality assessment: Rasti rates tool output as "as good or better than my manual workflow output" on completed videos

**Qualitative:**
- **M6** — The tool's prompt-engineering layer consistently produces clips that look intentional (not random)
- **M7** — The conversation flow feels like collaborating with a knowledgeable assistant, not filling out a form
- **M8** — The Knowledge Base accurately represents the production rules and grows organically with each song

### 10.2 SaaS Success Criteria (Phase 2 — measured 6 months post-launch)

These are speculative until MVP validates and we have real market data.

- **S1** — N paying users (specific N TBD based on MVP learnings and market sizing)
- **S2** — Gross margin ≥70% on per-video API costs (i.e., charge ≥3x the API cost)
- **S3** — Net revenue retention ≥100% (users stay and spend more over time)
- **S4** — NPS score ≥30 among power-user creator segment
- **S5** — Average user generates ≥1 video per month (proves recurring use, not one-off curiosity)
- **S6** — Knowledge Base has grown to cover ≥5 genres via the temporary-to-permanent playbook flow

---

## 11. Constraints & Assumptions

### 11.1 Technical Constraints
- **C1** Video generation APIs (Veo, Kling, Runway) have a hard ceiling of 8 seconds per call as of May 2026. Architecture must respect this.
- **C2** Whisper produces unreliable output for non-English vocals and over heavy instrumentation. The tool's strategy must account for this (skip-Whisper option, manual timestamping fallback).
- **C3** Video generation is non-deterministic — same prompt produces different outputs. Multiple takes per chunk are required for cherry-picking.
- **C4** LLM API context window is 200K tokens (Claude). Long conversations may eventually need summarization.
- **C5** Rasti's primary OS is Windows. The local Python backend + web server must work natively on Windows; the browser UI is cross-platform by default.
- **C6** Filmora is the primary editing tool. `.wfp` format may not be cleanly crackable; fallback interchange formats may be required.

### 11.2 Business Constraints
- **C7** MVP must be self-funded — no external investment in Phase 1.
- **C8** Per-song API costs must remain low enough that Rasti can afford to use the tool on his full remaining IronRUST work.
- **C9** No third-party dependencies that require licensing fees beyond per-use API costs at MVP.

### 11.3 Assumptions
- **A1** Video generation API costs will remain stable or decrease over the MVP build period (May 2026 baseline).
- **A2** Veo and/or Kling APIs will remain accessible to direct (non-wrapper) API users.
- **A3** Anthropic's Claude API will continue to support 200K context windows at current pricing.
- **A4** Filmora's project file format will remain a viable target (or fallback formats remain importable into Filmora).
- **A5** Rasti's available time for MVP development is sufficient for the planned scope (TBD validation).
- **A6** The Knowledge Base captured from sessions 04–07 is broadly correct and will hold up under production use; refinements expected but no architectural changes.

---

## 12. Dependencies & Integrations

### 12.1 External Service Dependencies

**Required for MVP:**
- **Anthropic Claude API** (Opus 4.7 or equivalent) — Conversation LLM, content analysis
- **OpenAI Whisper API** (or local Whisper) — Audio transcription
- **fal.ai** — Unified generation API gateway. Single key/billing for: video generation (Kling 3.0 Motion Control Pro, Kling O3 Pro, Veo 3.x) AND lip sync (Sync.so lipsync-2, lipsync-2-pro, sync-3). Pay-per-use, no subscription.

**Required for SaaS:**
- All MVP dependencies, plus:
- **Cloud hosting** (AWS, GCP, or similar)
- **Cloud storage** (S3, GCS, or equivalent)
- **Database** (PostgreSQL likely)
- **Payment processor** (Stripe)
- **Authentication provider** (Auth0 or similar)
- **Monitoring/logging** (Datadog, Sentry, or open-source equivalents)

### 12.2 Python Library Dependencies

**Backend (local web server):**
- `fastapi` — Web framework (REST endpoints + WebSocket for progress streaming)
- `uvicorn` — ASGI server
- `python-multipart` — File upload handling
- `pydantic` — Request/response validation
- `aiofiles` — Async file I/O

**AI / API integration:**
- `openai` — Whisper API client
- `anthropic` — Claude API client
- `fal-client` (or `httpx` against fal.ai REST) — fal.ai integration for video + lip sync
- `langchain` (optional, evaluate vs direct API calls based on complexity)

**Audio / Media:**
- `librosa` — Audio analysis (energy, BPM, beat detection)
- `pydub` or `ffmpeg-python` — Audio slicing / extraction for lip-sync input
- TBD: vocal separation library if needed (Demucs, Spleeter)

**Knowledge / Storage:**
- `chromadb` — Vector store for Knowledge Base embeddings and reference profiles (optional — may load KB directly into system prompt instead, evaluate cost/complexity)
- `sqlite3` (stdlib) or `tinydb` — Lightweight local persistence for project state / conversation history

**Output:**
- `python-docx` — Master `.docx` generation

**Internal tooling (not user-facing):**
- `click` or `typer` — CLI framework for internal dev/debug commands

### 12.3 Frontend Dependencies (Local Web UI)
- Vanilla HTML/CSS/JavaScript at MVP (no heavy framework needed at this scale)
- Optionally: a light reactive framework (Alpine.js, htmx, or similar) if it simplifies the chat + take-selection UI without adding build-step complexity
- Web APIs used: MediaRecorder (voice input), Web Audio (preview), Browser TTS (voice output)
- No Angular / React / Vue at MVP unless complexity demands it

### 12.4 Tool Dependencies (User-Side)
- Filmora (or compatible NLE) — for final editing post-tool
- Suno (or equivalent) — for music/vocal generation pre-tool
- A modern browser (Chrome, Firefox, Safari, Edge — last 2 major versions)
- File system with sufficient storage (per song, ~500MB–2GB for all clips + reference images)

---

## 13. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| **Filmora `.wfp` format not crackable** | Medium | Medium | Build fallback to FCP XML / Premiere XML / EDL (standard interchange formats) |
| **Video/lip sync API pricing changes mid-build** | Medium | High | Architect for API provider swap-ability; track multiple providers via uniform interface |
| **fal.ai becomes unreliable or removes key models** | Low | High | Maintain ability to fall back to direct Sync.so / direct Kling / direct Veo via uniform interface (FR-8.7) |
| **LLM costs scale beyond budget on long sessions** | Medium | Medium | Implement conversation summarization at token threshold |
| **Whisper quality insufficient even for English** | Low | Medium | Strengthen lyrics-required workflow; manual timestamping path |
| **Generated video quality below user expectations** | Medium | High | POC validation (Manufacturing Consent Verse 2 test) before full build; iterate on prompt engineering in KB |
| **Lip sync quality insufficient at lipsync-2 level** | Medium | Medium | Upgrade path to lipsync-2-pro or sync-3 (also via fal.ai, configurable per song). Test on early production songs before committing model. |
| **MVP scope creep delays validation** | Medium | High | Strict scope discipline; defer all SaaS features to Phase 2 |
| **No SaaS demand even after MVP success** | Medium | Low (MVP value stands alone) | SaaS is upside, not required for MVP justification |
| **Reference photo APIs change handling (Kling Motion Control)** | Low | Medium | Track API changes; fal.ai gateway absorbs some breaking changes |
| **Knowledge Base becomes bloated, drives up token costs** | Medium | Low | Discipline rule already in KB; modular loading path planned for v2 |

---

## 14. Roadmap & Phasing

### 14.1 Phase 1 — MVP (Target: 2026)

**Milestone 1 — Foundation (Weeks 1–2)**
- Project structure, config management
- FastAPI backend scaffold + local web server bootstrap (browser auto-opens to localhost on startup)
- Knowledge Base loader
- Local storage layout
- Persistent state model (project list, conversation state)

**Milestone 2 — Web UI Shell + Conversation (Weeks 2–4)**
- Minimal HTML/CSS/JS frontend served by FastAPI
- Chat panel UI (text input + display)
- Project sidebar (create new song, switch projects)
- File upload component (MP3, lyrics, reference photos)
- Claude API conversation flow
- Knowledge Base injection into system prompt
- Conversation persistence (state survives browser close)
- Modeled on the PASC chatbot architecture Rasti previously built

**Milestone 3 — Analysis Pipeline (Weeks 3–5)**
- Whisper integration (text input first; voice input added in M4)
- librosa integration
- Claude content analysis integration
- Draft Timestamps File generation in canonical format
- Timestamps File finalization workflow in the UI (section-by-section review and edit)

**Milestone 4 — Voice + Chunk Definition (Weeks 4–6)**
- Voice input via Whisper (microphone button in UI)
- Optional voice output via browser TTS
- Per-song context injection into LLM
- Chunk definition workflow (UI for proposing/redrawing chunks, gathering per-chunk metadata)
- KB rule override handling

**Milestone 5 — Video Generation + fal.ai (Weeks 6–8)**
- fal.ai integration layer (uniform interface, swappable providers)
- Per-segment prompt generation with continuity rules
- Multi-take generation
- Regeneration support
- Real-time progress streaming to UI (WebSocket or polling)
- Retry logic and failure handling

**Milestone 6 — Take Selection + Lip Sync (Weeks 8–9)**
- Take selection UI (video grid, click winner)
- Lip sync integration via fal.ai → Sync.so
- Vocal isolation pre-step (where needed)
- Final per-chunk artifact assembly

**Milestone 7 — Output & Handoff (Weeks 9–10)**
- Output directory organization
- Filmora project file generation (or fallback format)
- Master `.docx` generation
- Output review UI (preview final chunks before export)
- End-to-end test on a real IronRUST song

**Milestone 8 — Production Validation (Weeks 10–14)**
- Use tool to complete 3+ IronRUST tracks end-to-end
- Iterate on KB based on real production lessons
- Measure success metrics M1–M8
- Declare MVP success or identify gaps for Phase 1.5

### 14.2 Phase 2 — SaaS (Trigger Conditions)

**Phase 2 begins only when:**
- MVP success criteria M1–M8 are met
- Rasti has validated that the tool genuinely beats his manual workflow
- There's at least one other power-user creator willing to beta test
- Capital/time available for SaaS-scale build

**Phase 2 Milestones (TBD — high-level):**
- Multi-tenant architecture
- Web UI
- Authentication
- Payment integration
- Production deployment
- Beta with 3–5 power users
- Public launch

---

## 15. Open Questions

These are unresolved requirements that need answers before or during build:

- **Q1** ~~Which video generation API do we integrate first at MVP — Veo, Kling, or Runway?~~ **RESOLVED v1.2: fal.ai as unified gateway, access to Kling 3.0 Motion Control Pro, Kling O3 Pro, Veo 3.x, and other models via single API key.**
- **Q2** Is the Filmora `.wfp` format crackable? Investigation needed. If not, which fallback format (FCP XML / Premiere XML / EDL) imports cleanest into Filmora?
- **Q3** What's the right pricing model for SaaS — pay-per-generation, subscription tiers, or hybrid? Market research needed.
- **Q4** Should the conversation LLM use Claude (Anthropic) or GPT (OpenAI) or be user-configurable? Decision criteria: prompt-following quality, cost, context window.
- **Q5** How do we handle the long-conversation token-cost problem? Summarization strategy needs design.
- **Q6** Cross-song / album context: should the tool learn from previous songs in the same album (e.g., maintain a project-level memory of past chunk styles, performer choices, etc.)? Possible Phase 1.5 addition.
- **Q7** Cost ceiling per song — $35 is the NFR-C1 ceiling. Acceptable for production, but should we set a stricter MVP-testing ceiling (e.g., $20) with explicit user confirmation for anything above?
- **Q8** Reference photo guidance: what makes a "good" reference photo? Belongs in API integration spec but worth user-facing guidance too.
- **Q9** Lip sync model selection: lipsync-2 (basic, ~$0.05/sec) vs lipsync-2-pro (premium, ~$0.083/sec) vs sync-3 (4K + obstruction detection, ~$0.133/sec). Default to lipsync-2 for MVP cost; test lipsync-2-pro on first production song to assess quality delta. Decision driven by close-up performer footage quality requirement.
- **Q10** Vocal isolation: when user provides MP3 with mixed vocals+instrumental, do we run a vocal separation step (e.g., Demucs, Spleeter) before lip sync, or require user to provide vocals-only stem upfront? Tradeoff: tool complexity vs user friction.

---

## 16. Approval & Sign-off

This BRD is a working document. Approval and sign-off:

- **Product Owner:** Rasti — sign-off on scope, user stories, success criteria
- **Builder:** Rasti (Phase 1) — sign-off on technical feasibility
- **Future SaaS stakeholders:** TBD

**Current status:** v0.1 DRAFT — under review.

---

## 17. Provenance

This BRD was derived from:
- Knowledge Base v1.1 (`IRdeo_Knowledge_Base.md`) — the canonical source of domain rules
- Session 07 (`Manufacturing consent media song`) — original tool concept and architectural realizations
- Session 07 Handoff Document (`IRONRUST_VIDEO_STUDIO_HANDOFF.md`) — first product spec draft
- Sessions 04, 05, 06 — manual video production experience
- Hemick Slovak project — Whisper limitations learning
- Slovak atmospheric album sessions — multi-genre considerations

Every requirement in this document is grounded in real production experience or explicit user vision statements. Nothing is speculative beyond the Phase 2 SaaS section, which is explicitly framed as forward-looking.
