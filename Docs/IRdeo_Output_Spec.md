# IRdeo — Output Specification
**Version:** 1.0
**Last updated:** May 11, 2026
**Status:** LIVING DOCUMENT — comprehensive output deliverable specification
**Companion documents:** `IRdeo_Knowledge_Base.md` (v1.3+), `IRdeo_BRD.md` (v1.4+), `IRdeo_Architecture.md` (v1.2+), `IRdeo_UI_Spec.md` (v1.0+), `IRdeo_Data_Model.md` (v1.0+), `IRdeo_API_Integration.md` (v1.0+)

### Versioning Convention
- **v1.0** — first complete formal version
- **v1.1, v1.2, ...** — substantive revisions and additions
- **v2.0** — major restructure or fundamental approach change

### Changelog
- **v1.0 (May 11, 2026)** — First complete formal version. Covers all deliverable outputs: per-chunk clip naming and organization, preview MP4 generation via ffmpeg, Filmora project file generation with fallback formats (FCP XML / Premiere XML / EDL), master `.docx` document, prompt pack zip export, manual import workflow. Built on top of KB v1.3, BRD v1.4, Architecture v1.2, UI Spec v1.0, Data Model v1.0, API Integration v1.0.

---

## 1. Document Overview

### 1.1 Purpose
This document specifies every deliverable that IRdeo produces for the user. Where the other docs say HOW the system works, this doc says WHAT THE USER GETS — the artifacts they take to Filmora, the documents they archive, the files they hand to clients.

Output is the moment of truth. If the user finishes a generation run and can't easily open the result in Filmora, all the upstream brilliance was wasted. This spec ensures the final delivery is clean, organized, predictable, and actionable.

### 1.2 Scope
- **In scope:** Output directory structure, clip naming conventions, Filmora project file format and fallbacks, master `.docx` schema, preview MP4 generation (ffmpeg stitching), prompt pack zip export, manual-import workflow when project file generation isn't viable
- **Out of scope:** Internal data persistence (Data Model Spec), generation pipeline (Architecture Spec), user choice between Path A vs Path B (UI Spec)

### 1.3 Audience
- **Primary:** Build implementers wiring OutputService, PromptPackService, PreviewService
- **Secondary:** Rasti validating the output meets his Filmora workflow needs

### 1.4 Output Paths Recap

Per BRD v1.4 and UI Spec v1.0, two output paths exist:

- **Path A — Prompt Pack Export** (secondary, power-user option): zip file with engineered prompts, no video generation performed
- **Path B — Full Generation** (primary path): generated clips + preview + Filmora project + master docx

This spec covers both paths in detail.

### 1.5 Design Principles

**P1 — Files are the deliverable, not the application.** Once IRdeo writes output files, the user owns them. They should be portable, self-explanatory, and openable in standard tools without IRdeo running.

**P2 — Naming conventions are the index.** With predictable flat naming (`01_a_intro.mp4`), a user with no IRdeo running can still figure out what each clip is. No dependency on metadata sidecar files.

**P3 — Manual fallback always works.** If the Filmora project file generation fails for any reason, the user can still build their song manually by importing the flat-named clips. We never produce output the user can't use.

**P4 — Whole-song gates protect users.** Filmora project files and master docx are only generated when ALL chunks are complete. Partial output that pretends to be complete is worse than no output.

**P5 — Idempotent regeneration.** Regenerating output (preview, project file, docx) overwrites cleanly. No accumulating cruft, no version conflicts.

### 1.6 Related Documents
- `IRdeo_Knowledge_Base.md` — Domain rules (KB Section 13 Phase 9 — final output)
- `IRdeo_BRD.md` — Output requirements (FR-9.X)
- `IRdeo_Architecture.md` — OutputService (Section 11), PreviewService (Section 11.4), PromptPackService (Section 11.5)
- `IRdeo_UI_Spec.md` — Output view (View E)
- `IRdeo_Data_Model.md` — Directory structure and file naming (Section 2)

---

## 2. Output Directory Structure (Recap)

Per Data Model Spec Section 2:

```
~/IRdeo_Projects/{project_id}_{song_slug}/
├── chunks/
│   ├── chunk_01/
│   │   ├── 01_a_intro.mp4
│   │   ├── 01_b_intro.mp4
│   │   ├── 01_c_intro.mp4
│   │   └── chunk_metadata.json
│   ├── chunk_02/
│   │   ├── 02_a_verse.mp4
│   │   ├── ...
│   ├── ...
├── output/
│   ├── preview.mp4                          ← Always available after generation
│   ├── {song_slug}.wfp                      ← Filmora project (full song only)
│   ├── {song_slug}.xml                      ← FCP XML fallback
│   ├── {song_slug}.edl                      ← EDL fallback (extreme fallback)
│   ├── {song_slug}_master.docx              ← Master document (full song only)
│   ├── {song_slug}_prompts.zip              ← Path A export (on-demand)
│   └── preview_concat.txt                   ← ffmpeg concat manifest (transient, cleanup after use)
```

Two things to note:

1. **`chunks/` and `output/` are sibling directories.** Clips are the raw output (one per subchunk); `output/` is the user-facing deliverables built FROM those clips.
2. **The user is expected to interact with `output/`.** Power users may navigate into `chunks/` for manual selection; the deliverables in `output/` are the primary interface.

---

## 3. Clip Naming Convention

### 3.1 The Convention

```
{NN}_{letter}_{description_slug}.mp4
```

Where:
- `{NN}` — chunk number, zero-padded to 2 digits (`01`, `02`, ..., `99`)
- `{letter}` — subchunk letter, lowercase (`a`, `b`, `c`, ..., `z`)
- `{description_slug}` — URL-safe lowercase descriptor with underscores (`intro`, `verse_start`, `hook`, etc.)

### 3.2 Examples

```
01_a_intro.mp4              ← Chunk 1, subchunk a, intro description
01_b_intro.mp4              ← Chunk 1, subchunk b (continuation), intro
02_a_verse.mp4              ← Chunk 2, subchunk a, verse
02_b_chomsky.mp4            ← Chunk 2, subchunk b, descriptor reflects content (Chomsky portrait segment)
03_a_filters_diagram.mp4    ← Chunk 3, subchunk a, descriptor reflects content
04_a_hook_open.mp4          ← Chunk 4, subchunk a, hook section
```

### 3.3 Why This Naming Pattern

**Sorts alphabetically into playback order.** Drop the whole chunks directory into Filmora's media bin, select all, sort by name, and they're in the right order. No metadata dependency. No special tooling.

**Identifiable from the filename alone.** Even months later when you've forgotten which chunk was the Chomsky segment, the filename tells you. Helps with troubleshooting, asset management, and collaborating with others.

**Power-user friendly.** Power users can `ffmpeg -f concat` directly, drag-and-drop into any editor, or script automated batch processing. The naming is the affordance.

**Stable across regenerations.** If chunk 4 is regenerated, the new clip overwrites the old `04_a_*.mp4` file. The name stays the same. References from the project file or docx don't break.

### 3.4 Description Slug Sourcing

The `description_slug` comes from the subchunk's `description_slug` field in the chunk manifest (Data Model Spec Section 6.3). It's generated during prompt decomposition — the AI Director picks a 1-3 word descriptor based on what the subchunk's content is. If the AI Director doesn't pick one, fallback is the section slug (e.g., `verse_1`, `hook`).

### 3.5 Edge Cases

**More than 26 subchunks in a chunk:** Use double letters (`aa`, `ab`, `ac`...) following the spreadsheet convention. Realistically a chunk should never need more than ~10 subchunks (max 80 seconds at 8 seconds each), so this is a guardrail, not an expected case.

**More than 99 chunks in a song:** Extremely rare (a song would need to be 99+ minutes). Use triple-digit padding (`001`, `002`, ...). The system detects when chunk count exceeds 99 and auto-adjusts padding at manifest creation time.

**Special characters in descriptor:** Stripped or replaced. Only `[a-z0-9_]` allowed in `description_slug`. Spaces become underscores; everything else dropped.

---

## 4. Per-Chunk Clip Output

### 4.1 Format Specification

Each clip in `chunks/chunk_NN/` is:

- **Container:** MP4 (H.264 video + AAC audio if any)
- **Resolution:** Matches the video generation model's output. Kling typically produces 1080p; Veo can produce 4K
- **Frame rate:** Matches generation output (typically 24fps or 30fps)
- **Duration:** Matches the subchunk's `duration_seconds` ±0.1s tolerance
- **Audio track:** Present for lip-synced clips (the original audio slice baked in); absent for non-performer chunks

### 4.2 Lip Sync In-Place

For performer chunks, the lip-synced version overwrites the silent generation at the same filename. There is NO separate `lipsync.mp4` or `final.mp4` file. The single file at `01_a_intro.mp4` represents:
- The silent generation immediately after video gen
- The lip-synced version immediately after lip sync runs
- The user-facing final clip

This simplification (per BRD v1.4 + Architecture v1.2 take-removal pass) keeps the file system flat and the workflow predictable.

### 4.3 Codec Consistency for Concatenation

ffmpeg concat (used for preview generation in Section 6) requires consistent codecs across all clips to use `-c copy` (no re-encoding, fast). fal.ai's video providers (Kling, Veo) should produce consistent output formats per model. If a project mixes Kling Motion Pro and Veo 3 across different chunks, codec parameters may differ slightly.

**Mitigation:** When the preview build detects codec inconsistency, fall back to re-encoding with `-c:v libx264 -preset fast -crf 23`. Slower but always works. This is invisible to the user — the preview just takes a bit longer to build.

### 4.4 Per-Chunk Metadata Sidecar

Each `chunks/chunk_NN/` directory contains a `chunk_metadata.json` (Data Model Spec Section 10). This is internal state tracking — not part of the user-facing deliverables. Users do NOT need to consult it to use the clips.

---

## 5. Output Subdirectory (`output/`)

This is the user-facing deliverable directory. All artifacts intended for human consumption live here.

### 5.1 Contents Overview

| File | When Generated | Purpose |
|------|---------------|---------|
| `preview.mp4` | After any chunks have clips | Inline preview in browser, validation before Filmora export |
| `{song_slug}.wfp` | Only when ALL chunks complete | Filmora project file (primary format if `.wfp` writable) |
| `{song_slug}.xml` | Only when ALL chunks complete | FCP XML fallback (alternative editing format) |
| `{song_slug}.edl` | Only when ALL chunks complete | EDL extreme fallback (universal but minimal) |
| `{song_slug}_master.docx` | Only when ALL chunks complete | Master archive document |
| `{song_slug}_prompts.zip` | On user request (Path A) | Prompt pack export |
| `preview_concat.txt` | Transient (during preview build) | ffmpeg concat list; deleted after preview build |

### 5.2 Whole-Song Gating

Three artifacts are GATED on full-song completeness: `.wfp`, `.xml`, `.edl`, `master.docx`. If any chunk in the manifest has no clips, these artifacts cannot be generated. The UI shows the export buttons in disabled state with a tooltip explaining why.

Rationale: a Filmora project file referencing missing clips is broken. A master docx claiming completeness when chunks are missing is misleading. Better to be obviously partial (preview only) than misleadingly complete.

### 5.3 Preview Is Always Available

The preview MP4 is NOT gated on completeness. If only 3 of 9 chunks are generated, the preview shows just those 3 chunks. This gives the user immediate visual feedback during incremental work without the gating friction.

---

## 6. Preview MP4 Generation

### 6.1 Purpose

A single MP4 file showing the entire song with all generated clips stitched in sequence, with the original audio overlaid. This is the user's first look at the song-as-music-video before they commit to opening Filmora.

Critical UX function: catches problems EARLY. If chunk 4 is bad, the user sees it in 30 seconds of browser playback rather than 10 minutes into Filmora editing.

### 6.2 Generation Process

Implemented in `PreviewService` (Architecture Section 11.4). The process:

```
Step 1: Gather all clips across all generated chunks in flat-name order
Step 2: Build ffmpeg concat manifest (one line per clip)
Step 3: ffmpeg concat with -c copy (no re-encoding, fast)
Step 4: Overlay the original song audio, replacing any per-clip audio
Step 5: Clean up intermediate files (preview_concat.txt, preview_video.mp4)
```

### 6.3 ffmpeg Commands

**Concat step (clips → video-only intermediate):**

```bash
ffmpeg -y \
  -f concat -safe 0 \
  -i preview_concat.txt \
  -c copy \
  preview_video.mp4
```

The `preview_concat.txt` is a simple list:

```
file '/absolute/path/to/01_a_intro.mp4'
file '/absolute/path/to/01_b_intro.mp4'
file '/absolute/path/to/01_c_intro.mp4'
file '/absolute/path/to/02_a_verse.mp4'
...
```

**Audio overlay step (video-only + original MP3 → final preview):**

```bash
ffmpeg -y \
  -i preview_video.mp4 \
  -i /absolute/path/to/source/original.mp3 \
  -c:v copy \
  -c:a aac \
  -map 0:v:0 -map 1:a:0 \
  -shortest \
  preview.mp4
```

The `-shortest` flag stops output at whichever stream ends first — prevents the preview from running past the last generated clip when partial generation.

### 6.4 Codec Fallback (When `-c copy` Fails)

If clips have inconsistent codecs (rare but possible when mixing Kling and Veo outputs), the `-c copy` step fails. PreviewService catches this and retries with re-encoding:

```bash
ffmpeg -y \
  -f concat -safe 0 \
  -i preview_concat.txt \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac \
  preview_video.mp4
```

Slower (re-encodes everything) but always works. The user sees "Preview building (re-encoding for compatibility)..." instead of "Preview building..."

### 6.5 Performance Expectations

- **Concat with `-c copy`:** seconds for a 5-minute song (just rewrites containers)
- **Concat with re-encoding:** 1-3 minutes for a 5-minute song
- **Audio overlay step:** seconds (only re-encodes audio)
- **Total time for preview:** typically under 30 seconds, occasionally up to 3 minutes if re-encoding triggered

### 6.6 Preview Invalidation

When a chunk is regenerated, the existing preview becomes stale. The project state (`state.json`) tracks `preview_state`: when any chunk regenerates after `preview_state == "complete"`, the state flips to `"stale"`. The UI shows a "Preview is out of date — rebuild?" prompt.

The rebuild is a one-click action; takes seconds to update.

### 6.7 Partial Preview Behavior

When only some chunks have clips, the preview still builds — it just contains whatever clips exist. The audio overlay starts at time 0 and runs to whichever finishes first (preview video or audio file). For a 5-minute song with only chunks 1-3 generated (60 seconds of video), the preview shows the first 60 seconds with audio cutting off at the same point.

This is the right behavior. The preview is honest about what exists.

---

## 7. Filmora Project Generation

### 7.1 The `.wfp` Format Investigation Status

**Status as of v1.0: PENDING INVESTIGATION.**

Filmora's `.wfp` (Wondershare Filmora Project) is a proprietary binary or compressed format. Public documentation does not exist. Reverse-engineering options include:

- Inspecting `.wfp` files produced by Filmora itself (saving an empty project, comparing to a project with one clip, diffing)
- Community-discovered format details (forums, GitHub repos)
- Wondershare's own SDK or API if available (unlikely)

**Build-phase task:** Before implementing the Filmora project writer, an engineer must investigate `.wfp` format viability. Two outcomes:

- **Outcome A — `.wfp` is writable.** Implement `WfpWriter` following discovered format. Primary export uses `.wfp`.
- **Outcome B — `.wfp` is not viable.** Skip `.wfp` entirely; primary export uses FCP XML (Section 7.3). Filmora can import FCP XML with some manual setup.

The architecture supports either outcome without other changes — the `FilmoraWriter` interface in Architecture Section 11.2 is format-agnostic.

### 7.2 What the Filmora Project Must Contain

Regardless of format, the project file MUST contain:

1. **Project metadata:** song title, duration, frame rate, resolution
2. **Audio track:** the original MP3 (or vocals-only stem if user prefers), placed at time 0
3. **Video track:** every generated clip, placed in sequence at correct timestamps matching the chunk manifest
4. **Clip references:** absolute paths to clip files (Filmora can resolve missing media later if files move)
5. **Project save location:** the project file itself lives in `output/`; clips live in `chunks/`

### 7.3 FCP XML Fallback

Final Cut Pro XML (FCPXML) is the industry interchange format. Filmora supports importing FCPXML files (with some manual setup — user opens Filmora, File → Import → Project, selects the `.xml`).

**Schema:** FCPXML 1.10 (current as of v1.0). See https://developer.apple.com/documentation/professional_video_applications/fcpxml_reference for the official schema.

**Structure for IRdeo's output:**

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE fcpxml>
<fcpxml version="1.10">
  <resources>
    <format id="r0" name="FFVideoFormat1080p30"
            frameDuration="1/30s" width="1920" height="1080"/>

    <!-- Audio source -->
    <asset id="r1" name="original.mp3" src="file:///abs/path/source/original.mp3"
           start="0s" duration="285.34s" hasAudio="1"/>

    <!-- Video clips -->
    <asset id="r2" name="01_a_intro" src="file:///abs/path/chunks/chunk_01/01_a_intro.mp4"
           start="0s" duration="8s" hasVideo="1"/>
    <asset id="r3" name="01_b_intro" src="file:///abs/path/chunks/chunk_01/01_b_intro.mp4"
           start="0s" duration="7s" hasVideo="1"/>
    <!-- ... one asset per clip ... -->
  </resources>

  <library>
    <event name="Manufacturing Consent">
      <project name="Manufacturing Consent — IRdeo">
        <sequence format="r0" duration="285.34s">
          <spine>
            <!-- Audio track: original MP3 from time 0 -->
            <asset-clip ref="r1" offset="0s" duration="285.34s" name="original.mp3" audioRole="music"/>

            <!-- Video clips placed at their absolute timestamps -->
            <asset-clip ref="r2" offset="0s" duration="8s" name="01_a_intro"/>
            <asset-clip ref="r3" offset="8s" duration="7s" name="01_b_intro"/>
            <!-- ... -->
          </spine>
        </sequence>
      </project>
    </event>
  </library>
</fcpxml>
```

### 7.4 EDL (Extreme Fallback)

EDL (Edit Decision List) is a simple text format universally supported by editing software. Less expressive than FCPXML but works everywhere.

**Structure for IRdeo's output:**

```
TITLE: Manufacturing Consent
FCM: NON-DROP FRAME

001  AX       V     C        00:00:00:00 00:00:08:00 00:00:00:00 00:00:08:00
* FROM CLIP NAME: 01_a_intro.mp4
* FROM CLIP: /abs/path/chunks/chunk_01/01_a_intro.mp4

002  AX       V     C        00:00:00:00 00:00:07:00 00:00:08:00 00:00:15:00
* FROM CLIP NAME: 01_b_intro.mp4
* FROM CLIP: /abs/path/chunks/chunk_01/01_b_intro.mp4

(... one entry per clip ...)
```

EDL doesn't natively support audio synced to specific clips, so the user imports the EDL and then manually adds the original MP3 as a separate audio track. Marked as "extreme fallback" because it requires the most manual setup.

### 7.5 Format Selection Logic

```python
# services/output/filmora_writer.py

class FilmoraWriter:
    def __init__(self, project: Project, prefer_format: str = "auto"):
        self.project = project
        self.prefer_format = prefer_format

    def write(self, output_path: Path) -> dict:
        """Write the project file in the best available format.

        Returns a dict with 'format' and 'path' keys indicating what was written.
        """
        slug = self.project.song_slug

        # Try formats in order of preference
        if self.prefer_format in ("auto", "wfp") and FormatCapability.wfp_supported():
            try:
                wfp_path = output_path / f"{slug}.wfp"
                self._write_wfp(wfp_path)
                return {"format": "wfp", "path": wfp_path}
            except FormatUnsupportedError:
                pass  # fall through

        if self.prefer_format in ("auto", "fcpxml"):
            xml_path = output_path / f"{slug}.xml"
            self._write_fcpxml(xml_path)
            return {"format": "fcpxml", "path": xml_path}

        # Extreme fallback
        edl_path = output_path / f"{slug}.edl"
        self._write_edl(edl_path)
        return {"format": "edl", "path": edl_path}

    def _write_wfp(self, path: Path):
        """Implement after .wfp investigation completes."""
        if not FormatCapability.wfp_supported():
            raise FormatUnsupportedError("WFP format not yet implemented")
        # Implementation pending build-phase investigation
        raise NotImplementedError("WFP writer pending format reverse-engineering")

    def _write_fcpxml(self, path: Path):
        """Generate FCP XML 1.10."""
        # Build the XML structure per Section 7.3
        doc = self._build_fcpxml_document()
        path.write_bytes(doc.tostring(pretty_print=True, xml_declaration=True))

    def _write_edl(self, path: Path):
        """Generate EDL text format."""
        lines = self._build_edl_lines()
        path.write_text("\n".join(lines), encoding="utf-8")
```

### 7.6 User Preference for Format

The Settings drawer offers a "Filmora Project Format" preference:

- **Auto (recommended):** Try `.wfp` first, fall back to FCP XML, then EDL
- **FCP XML only:** Skip `.wfp` even if supported (user may prefer this if Filmora's `.wfp` import causes issues)
- **EDL only:** Most portable; useful for multi-tool workflows

Default: Auto.

### 7.7 Build-Phase Investigation Plan

Before implementing `_write_wfp`, an engineer must:

1. Create a minimal Filmora project (one audio track, one video clip), save as `.wfp`
2. Open the file in a hex editor — is it binary or text? Compressed?
3. If compressed, try `unzip`, `tar -x`, `gunzip` to see if it's a known archive format
4. If binary, look for header signatures (`PK` = zip, `<?xml` = xml-prefixed, etc.)
5. If text, parse structure
6. Compare files across multiple test projects to identify which bytes encode what
7. Search GitHub for "wfp filmora parser" or "wfp wondershare format" — community implementations may exist
8. If after 1-2 days of investigation no clear path emerges, declare `_write_wfp` not viable and ship with FCP XML as primary

Track this work as: **Build Task BT-Output-1: Filmora .wfp format investigation.**

---

## 8. Master `.docx` Document

### 8.1 Purpose

A self-contained archive document containing everything about the song's IRdeo session:

- Song metadata
- Suno prompt that generated the music (if user recorded it)
- Production notes
- Album context
- Final lyrics
- Full Timestamps File
- Chunk-by-chunk video prompts
- Whisper corrections log
- Generation metadata (models used, costs, dates)

Use cases:
- Archive: years later, the user can find exactly how a song was made
- Sharing: send to collaborators without IRdeo dependency
- Documentation: portfolio piece showing the artist's process
- Backup: human-readable record if IRdeo data is ever lost

### 8.2 Document Structure

```
{song_title} — IRdeo Master Document
Generated by IRdeo v{irdeo_version} on {timestamp}

1. SONG METADATA
   - Title:
   - Artist:
   - Album:
   - Track number:
   - Duration:
   - BPM:
   - Genre:
   - Created:
   - Last modified:

2. SUNO PROMPT
   (User-provided text of the Suno prompt that generated the music. Optional section.)

3. PRODUCTION NOTES
   (Free-form notes the user added during production. Optional section.)

4. ALBUM CONTEXT
   (Context about how this song fits into the larger album/project. Optional section.)

5. FINAL LYRICS
   (User-provided lyrics, if any. Falls back to Whisper transcript if user skipped lyrics.)

6. TIMESTAMPS FILE
   (Full canonical Timestamps File, embedded as code block.)

7. VIDEO PROMPTS PER CHUNK
   7.1 Chunk 01: Intro (spoken) (0:00–0:22)
       Model: Kling Motion Pro
       Mood: intimate, vulnerable
       Style references: Eminem Cleanin' Out My Closet
       Performer present: Yes
       Reference photo set: Performer — Rasti — Main
       
       Chunk-level prompt:
       (full text)
       
       Subchunks:
       01_a (0:00–0:08): (subchunk prompt text)
       01_b (0:08–0:15): (subchunk prompt text)
       01_c (0:15–0:22): (subchunk prompt text)
   
   7.2 Chunk 02: Verse 1 (0:22–1:12)
       (... same structure ...)
   
   (... one section per chunk ...)

8. WHISPER CORRECTIONS
   (List of Whisper transcription errors that the user corrected.)
   - "bloopery" → "blueprint"
   - "I felt this choking" → "Five filters choking"
   (etc.)

9. GENERATION METADATA
   - Total clips generated: 47
   - Total cost: $18.42
   - Generation date: 2026-05-11
   - Models used:
     - Kling Motion Pro: 32 clips
     - Kling O3 Pro: 15 clips
   - Lip sync model: Sync.so lipsync-2 (22 clips)
   - Vocal isolation: Lalal.ai Orion (1 pass)

10. FILE INVENTORY
    Clips:
    - chunks/chunk_01/01_a_intro.mp4 (8.00s, 4.6 MB)
    - chunks/chunk_01/01_b_intro.mp4 (7.00s, 4.0 MB)
    (... full list ...)
    
    Project files:
    - output/manufacturing_consent.wfp (or .xml)
    - output/manufacturing_consent_master.docx (this file)
    - output/preview.mp4
```

### 8.3 Implementation

Per Architecture Section 11.3, `MasterDocWriter` uses `python-docx` to build the document programmatically.

```python
# services/output/docx_writer.py
from docx import Document
from docx.shared import Pt, Inches
from docx.enum.text import WD_ALIGN_PARAGRAPH
from datetime import datetime

class MasterDocWriter:
    def __init__(self, project: Project):
        self.project = project

    def write(self, output_path: Path) -> Path:
        doc = Document()

        # Configure styles
        self._configure_styles(doc)

        # Title
        title = doc.add_heading(f"{self.project.song_title} — IRdeo Master Document", 0)
        title.alignment = WD_ALIGN_PARAGRAPH.CENTER

        # Subtitle
        subtitle = doc.add_paragraph(
            f"Generated by IRdeo v{self.project.irdeo_version} on "
            f"{datetime.utcnow().strftime('%Y-%m-%d %H:%M UTC')}"
        )
        subtitle.alignment = WD_ALIGN_PARAGRAPH.CENTER

        # Sections
        self._add_metadata_section(doc)
        self._add_suno_prompt_section(doc)
        self._add_production_notes_section(doc)
        self._add_album_context_section(doc)
        self._add_lyrics_section(doc)
        self._add_timestamps_section(doc)
        self._add_prompts_section(doc)
        self._add_whisper_corrections_section(doc)
        self._add_generation_metadata_section(doc)
        self._add_file_inventory_section(doc)

        doc.save(str(output_path))
        return output_path

    def _configure_styles(self, doc: Document):
        """Set default fonts and sizes for readability."""
        styles = doc.styles
        normal = styles["Normal"]
        normal.font.name = "Calibri"
        normal.font.size = Pt(11)

    def _add_metadata_section(self, doc: Document):
        doc.add_heading("1. Song Metadata", 1)
        table = doc.add_table(rows=8, cols=2)
        table.style = "Light Grid Accent 1"
        cells = [
            ("Title", self.project.song_title),
            ("Artist", self.project.artist or "Unknown"),
            ("Album", self.project.album_context or "—"),
            ("Track number", str(self.project.track_number or "—")),
            ("Duration", f"{self.project.duration_seconds:.1f} seconds"),
            ("BPM", f"{self.project.bpm:.1f}" if self.project.bpm else "—"),
            ("Genre", self.project.genre),
            ("Created", self.project.created_at[:10]),
        ]
        for i, (label, value) in enumerate(cells):
            table.cell(i, 0).text = label
            table.cell(i, 1).text = str(value)

    def _add_suno_prompt_section(self, doc: Document):
        if not self.project.suno_prompt:
            return  # skip optional section
        doc.add_heading("2. Suno Prompt", 1)
        doc.add_paragraph(self.project.suno_prompt)

    # ... etc. for each section
```

### 8.4 Formatting Considerations

- **Headings:** Numbered (1, 2, 3...) for easy navigation in long documents
- **Monospace blocks:** Use Courier or Consolas for the Timestamps File and prompt text (preserves formatting)
- **Tables:** For metadata and file inventory (better than bullet lists for structured data)
- **Page breaks:** Insert page breaks before major sections (1, 6, 7) for printability
- **Length:** Typical output is 15-30 pages for a 5-minute song with 9 chunks

### 8.5 Idempotent Generation

The master docx is regenerated from scratch each time the user clicks "Generate Master Doc." It does not append or merge with previous versions — the latest project state is the truth.

If the user wants to compare versions, they can save copies manually before regenerating.

---

## 9. Prompt Pack Export (Path A)

### 9.1 Purpose

For users who want IRdeo's smart prompt engineering but don't want IRdeo to fire the generation APIs. Output: a zip file with engineered prompts, ready to paste into AIVideo.com, direct API tools, or any other workflow.

This IS the "smart layer" deliverable in its purest form — IRdeo's value distilled into copyable text.

### 9.2 Zip Contents

```
{song_slug}_prompts.zip
├── README.md                                ← Explains the zip contents
├── timestamps.md                            ← Full canonical Timestamps File
├── 01_intro.txt                             ← Chunk 1 prompts (chunk-level + all subchunks)
├── 02_verse_1.txt                           ← Chunk 2 prompts
├── 03_hook_1.txt                            ← Chunk 3 prompts
├── ...
└── chunk_summary.csv                        ← Tabular summary for spreadsheet users
```

### 9.3 README Contents

```markdown
# {Song Title} — Prompt Pack
Generated by IRdeo on {date}

## Contents
This zip contains engineered video prompts for "{Song Title}".
One text file per chunk, named per the IRdeo flat naming convention.

The prompts are designed to be pasted directly into:
- AIVideo.com (paste the chunk-level prompt)
- Direct API calls to Kling, Veo, or other video generation services
- Other prompt-aware video tools

## Files
- `README.md` — this file
- `timestamps.md` — the canonical Timestamps File for the song
- `{NN}_{section_slug}.txt` — one per chunk, with the engineered prompt
- `chunk_summary.csv` — tabular summary of all chunks for spreadsheet use

## How the Prompts Were Built
These prompts were generated by IRdeo's AI Director from your audio analysis,
your provided lyrics (if any), and your conversation about the visual vision
for each chunk. Genre playbooks, performer integration rules, and continuity
considerations have been applied per IRdeo's Knowledge Base.

## License
You own these prompts. Use them however you want.
```

### 9.4 Per-Chunk Text File Format

```
# Chunk 01 — Intro (spoken)
Time: 0:00 – 0:22 (22.0 seconds)
Vocal type: Male Spoken (close-mic)
Performer present: Yes (full chunk)
Recommended model: Kling Motion Pro

## Chunk-Level Prompt (the full creative direction)
{chunk-level prompt text — the AI Director's full narrative direction for this chunk}

## Subchunk Prompts (8-second slices for API generation)

### Subchunk a (0:00 – 0:08, 8.0s)
{subchunk prompt text}

_Continuity: Establishing shot — sets the visual register for the intro section._

### Subchunk b (0:08 – 0:15, 7.0s)
{subchunk prompt text}

_Continuity: Continue from 01_a. Same room, same lighting. Slight push-in._

### Subchunk c (0:15 – 0:22, 7.0s)
{subchunk prompt text}

_Continuity: Continue from 01_b. Performer eyeline shifts toward camera for first time._
```

### 9.5 chunk_summary.csv Format

For users who prefer spreadsheets:

```csv
chunk_id,section,start,end,duration,vocal_type,performer,model,mood,style_refs
01,Intro (spoken),0.00,22.00,22.00,Male Spoken,yes,kling_motion_pro,"intimate, vulnerable","Eminem Cleanin' Out My Closet"
02,Verse 1,22.00,72.00,50.00,Male Rap,no,kling_o3_pro,"exposed, accusatory",
03,Hook 1,72.00,98.00,26.00,Male Rap,yes,kling_motion_pro,"anthemic, peak","Eminem Lose Yourself"
...
```

### 9.6 Implementation

Per Architecture Section 11.5, `PromptPackService.build_zip` assembles the zip:

```python
# services/prompt_pack_service.py
import zipfile
import csv
from io import StringIO

class PromptPackService:
    async def build_zip(self, project_id: str) -> Path:
        project = self.projects.load(project_id)
        if not project.prompts_generated:
            raise UserError("Prompts not yet generated.",
                          suggestion="Complete the conversation and chunk definition first.")

        output_dir = project.path / "output"
        output_dir.mkdir(exist_ok=True)
        zip_path = output_dir / f"{project.song_slug}_prompts.zip"

        with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as zf:
            zf.writestr("README.md", self._build_readme(project))
            zf.writestr("timestamps.md",
                       (project.path / "timestamps.md").read_text(encoding="utf-8"))

            for chunk in project.chunks:
                chunk_num = chunk.chunk_id.removeprefix("chunk_")
                filename = f"{chunk_num}_{chunk.section_slug}.txt"
                zf.writestr(filename, self._format_chunk_text(project, chunk))

            zf.writestr("chunk_summary.csv", self._build_csv(project))

        return zip_path

    def _build_csv(self, project) -> str:
        buf = StringIO()
        writer = csv.DictWriter(buf, fieldnames=[
            "chunk_id", "section", "start", "end", "duration",
            "vocal_type", "performer", "model", "mood", "style_refs"
        ])
        writer.writeheader()
        for chunk in project.chunks:
            writer.writerow({
                "chunk_id": chunk.chunk_id.removeprefix("chunk_"),
                "section": chunk.section_label,
                "start": f"{chunk.start_seconds:.2f}",
                "end": f"{chunk.end_seconds:.2f}",
                "duration": f"{chunk.duration_seconds:.2f}",
                "vocal_type": chunk.vocal_type,
                "performer": "yes" if chunk.performer_present else "no",
                "model": chunk.model_preference,
                "mood": chunk.mood,
                "style_refs": "; ".join(chunk.style_references),
            })
        return buf.getvalue()
```

### 9.7 Pricing

Path A cost: $0. The prompts are generated as part of the conversation flow (Section 6 of KB Phase 9). Zipping them up consumes no API calls.

This is the "smart layer is free" offer — for users who want the prompts only, IRdeo charges nothing for the prompts themselves. (At SaaS scale, IRdeo might charge a subscription fee for prompt-pack-only access; that's a Phase 2 business model decision, not an MVP cost.)

---

## 10. Manual Import Workflow (Filmora Without Project File)

### 10.1 When This Applies

- `.wfp` writing not yet implemented
- User chose "FCP XML only" or "EDL only" preference and prefers manual setup
- Partial generation: not all chunks complete, so no project file generated
- User wants to do their own thing with the clips

### 10.2 The Workflow

1. **User opens Filmora**
2. **User navigates to the project directory:** `~/IRdeo_Projects/{project_id}_{song_slug}/`
3. **User opens `chunks/` folder**
4. **User sorts files by name** (this gives correct playback order via the flat naming convention)
5. **User selects all `.mp4` files across all chunk subdirectories** (via Cmd+A or batch select)
6. **User drags-and-drops into Filmora's media bin**
7. **User drags clips onto the timeline in order** (Filmora respects the alphabetical sort)
8. **User drags the original `source/original.mp3` onto the audio track**
9. **User aligns audio start to timeline start (0:00)**
10. **User adds transitions, effects, color grading per their workflow**
11. **User exports the final video from Filmora**

### 10.3 Why This Always Works

The flat naming convention IS the project file. No metadata. No fragile path references. Just MP4 files in a sortable directory. If IRdeo's project file generation breaks for any reason, the manual workflow is the same as any other AI clip workflow — drag and drop, align, edit.

### 10.4 UI Affordance

When the user is in Output view and Filmora project file generation isn't viable (any reason), the UI shows a clear "How to import manually into Filmora" section with these steps and an "Open Output Folder" button.

---

## 11. Output Verification Tests

Before considering the output deliverable complete, the OutputService runs verification:

### 11.1 Per-Chunk Verification

For each clip:
- File exists at expected path
- File is non-zero size
- File is parseable as MP4 (ffprobe returns valid metadata)
- Duration matches expected ±0.5s tolerance
- Has video stream (audio stream optional)

### 11.2 Preview Verification

For preview.mp4:
- File exists
- Duration matches sum of all included clip durations ±0.5s
- Has both video and audio streams
- Playable in browser (HTML5 video element compatible)

### 11.3 Project File Verification

For Filmora project (any format):
- File exists and is valid for its format (XML parseable for FCPXML, etc.)
- Every clip reference resolves to an existing file
- Total duration matches song duration

### 11.4 Master Doc Verification

For master.docx:
- File exists and is valid docx (openable by python-docx)
- All required sections present (those flagged as required, optional sections may be empty)
- No placeholder text remaining (`{template_variable}` strings)

### 11.5 Verification Failure Handling

If any verification fails, the OutputService:
1. Logs the failure with detail
2. Surfaces a UserError to the UI
3. Does NOT mark the output state as complete
4. Suggests user actions (regenerate the affected chunk, contact support, etc.)

Better to surface a known broken state than to deliver something the user can't use.

---

## 12. Open Output Questions

Things to resolve during build / first real-world testing:

- **OQ-1** Filmora `.wfp` format viability investigation (see Section 7.7 plan). Outcome determines whether `_write_wfp` is implemented or skipped.
- **OQ-2** FCP XML version — 1.10 is current as of v1.0, but Apple periodically updates the schema. Verify Filmora supports the chosen version before locking.
- **OQ-3** Audio attached to clips vs. separate track in Filmora project — Filmora may handle lip-synced clips differently. Test both approaches (audio baked into clips vs. silent clips with separate audio track) to determine which produces a cleaner editing experience.
- **OQ-4** Codec normalization — should clips be re-encoded to a uniform codec at generation time (instead of fallback re-encoding at preview time)? Tradeoff: extra processing per clip vs. faster preview builds.
- **OQ-5** Master docx page count — 15-30 pages is the expected range, but very long songs (album-length 30-minute productions) may produce 100+ pages. Test with edge cases.
- **OQ-6** Localization — master docx is English-only at v1.0. For Slovak/non-English users, should the boilerplate text (headings, labels) translate based on the song's language? Probably yes; defer to v1.1.
- **OQ-7** PDF export as additional deliverable — some users may want a PDF of the master docx for archive purposes. python-docx → PDF requires LibreOffice or pandoc; adds dependency. Defer unless requested.
- **OQ-8** Direct upload to cloud editors (CapCut, Adobe Premiere Cloud, etc.) — future enhancement, not MVP.

---

## 13. Provenance

This Output Specification was derived from:
- Knowledge Base v1.3 — domain rules (Phase 9 final output requirements)
- BRD v1.4 — output FRs (FR-9.X), user stories (US-29 through US-32), flat naming convention
- Architecture v1.2 — OutputService, PreviewService, PromptPackService implementations
- UI Spec v1.0 — Output view (View E) interactions
- Data Model v1.0 — directory structure (Section 2), clip naming (Section 6.3)
- API Integration v1.0 — confirms ffmpeg is the local stitch tool, not an external API
- FCPXML reference: https://developer.apple.com/documentation/professional_video_applications/fcpxml_reference
- EDL specification: SMPTE 207M (legacy) and CMX 3600 informal standard

Every output decision is grounded in either an explicit requirement, a domain principle, or a user workflow need. Speculative choices (Filmora `.wfp` format, future enhancements) are flagged in Section 12.
