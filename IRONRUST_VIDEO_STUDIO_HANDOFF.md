# IRONRUST VIDEO STUDIO — Project Handoff Document

## Executive Summary

Build an AI-powered music video generation pipeline that automates the tedious parts of video creation while keeping creative control with the user. This tool replaces AIVideo.com with something smarter, cheaper, and fully customizable.

**Key Insight:** AIVideo.com is a dumb API wrapper. It doesn't analyze lyrics, doesn't understand context, and generates generic garbage without detailed user prompts. Our tool will be SMART — it collaborates with the user, understands the song, and generates intelligent prompts automatically.

**Proof of Concept:** The "Manufacturing Consent" video was created manually using this exact workflow. Every step can be automated.

---

## The Problem with AIVideo.com

### What AIVideo.com Does:
- Beat detection for timing (basic)
- Generic mood matching (fast = energetic, slow = calm)
- User writes ALL detailed prompts manually
- Stitches chunks with basic transitions
- Applies generic effects (RGB, VHS)

### What AIVideo.com Does NOT Do:
- Read or understand lyrics
- Understand album/song context
- AI-assisted prompt generation
- Intelligent scene suggestions
- Conversation with user to refine vision

### Test Results (May 2026):
- Uploaded "Manufacturing Consent" verse 2 (Netanyahu, CBS, media manipulation content)
- Prompt: "Create a music video for this political rap song"
- No reference images
- **Result:** Generic garbage. Random black guy performer, still images, bad lip sync, nothing related to actual lyrics. Zero intelligence.

**Conclusion:** User's prompts and editing skills made the video good, not AIVideo.com's technology.

---

## The Solution: IronRUST Video Studio

### Core Philosophy
- **Automate the tedious:** API calls, chunk organization, Filmora project generation
- **Keep creative control:** User makes final decisions, AI assists and suggests
- **Smart collaboration:** AI understands the song and asks intelligent questions
- **Pay per use:** No monthly subscriptions, just API costs (~$7-30 per full video)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     IRONRUST VIDEO STUDIO                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐ │
│  │   INTAKE    │───▶│  ANALYSIS   │───▶│   COLLABORATION     │ │
│  │             │    │             │    │                     │ │
│  │ • MP3       │    │ • Whisper   │    │ • AI asks questions │ │
│  │ • Lyrics    │    │ • Librosa   │    │ • User answers      │ │
│  │ • Ref imgs  │    │ • Claude    │    │ • Refine vision     │ │
│  └─────────────┘    └─────────────┘    └─────────────────────┘ │
│                                                 │               │
│                                                 ▼               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐ │
│  │   OUTPUT    │◀───│ GENERATION  │◀───│   SMART CHUNKING    │ │
│  │             │    │             │    │                     │ │
│  │ • Chunks    │    │ • Veo API   │    │ • Optimal segments  │ │
│  │ • .wfp file │    │ • Multiple  │    │ • Intelligent       │ │
│  │ • Filmora   │    │   takes     │    │   prompts per chunk │ │
│  └─────────────┘    └─────────────┘    └─────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Intake

### Inputs
1. **MP3 Audio** (required)
   - The song to generate video for
   - Used for transcription, beat detection, energy analysis

2. **Lyrics** (optional but recommended)
   - Can be auto-generated via Whisper
   - User can provide clean lyrics for accuracy
   - Whisper often garbles names (e.g., "Barry white" → "Bari Weiss")

3. **Reference Images** (optional)
   - Performer photos for character consistency
   - Style references
   - Logo/branding images

4. **Context** (optional)
   - Album name and concept
   - Previous songs in album
   - Overall artistic vision

### Technical Implementation
```python
# intake.py
def intake_song(mp3_path, lyrics_path=None, reference_images=None, context=None):
    song_data = {
        "audio_path": mp3_path,
        "lyrics": load_lyrics(lyrics_path) if lyrics_path else None,
        "reference_images": reference_images or [],
        "context": context or {}
    }
    return song_data
```

---

## Phase 2: Analysis

### Components

#### 2.1 Whisper Transcription
- Transcribe audio to get timestamped lyrics
- Existing script: `C:\projects\GitHub\The_Cure_They_Hid\v2.0\scripts\transcribe_song.py`
- Supports language hints and lyrics hints for accuracy
- Output: Timestamped text segments

#### 2.2 Audio Analysis (Librosa)
- Beat detection
- Energy mapping
- Tempo (BPM)
- Existing script: `analyze_song.py`
- Output: Energy curve, beat timestamps, tempo

#### 2.3 Content Analysis (Claude/Opus API)
- Read and understand lyrics
- Identify themes, names, references
- Detect song structure (intro, verse, hook, bridge, outro)
- Identify mood shifts
- Detect male/female vocals
- Identify genre and style

### Output: Song Analysis Object
```python
{
    "duration": "4:45",
    "bpm": 112,
    "structure": [
        {"type": "intro", "start": "0:00", "end": "0:22", "description": "Spoken intro"},
        {"type": "verse", "start": "0:22", "end": "1:12", "description": "Verse 1 - media trust"},
        {"type": "hook", "start": "1:12", "end": "1:59", "description": "Hook - female vocals"},
        # ... etc
    ],
    "themes": ["media manipulation", "Chomsky", "corporate media", "independent voices"],
    "names_mentioned": ["Netanyahu", "Bari Weiss", "CBS", "TYT", "Grayzone", "Tucker"],
    "vocals": {
        "primary": "male",
        "secondary": "female (high pitch, hook sections)",
        "style": "Eminem-style rapid delivery"
    },
    "mood": "angry, defiant, building to triumphant"
}
```

---

## Phase 3: Collaboration

### The AI Conversation

This is the KEY differentiator. AI doesn't just process — it COLLABORATES.

#### Sample Conversation Flow:

```
AI: I've analyzed "Manufacturing Consent" (4:45, 112 BPM)

    I found:
    - 3 verses, 2 hooks, 1 bridge, intro/outro
    - Male primary vocals (Eminem-style delivery)
    - Female high-pitch vocals on hooks (1:12-1:59, 2:43-3:05)
    - Key themes: Chomsky's 5 filters, CBS story killing, Netanyahu, 
      independent media (Grayzone, Breaking Points, TYT, Tucker)
    
    Let's build your vision. First question:
    
    Should there be a performer in the video?
    [ ] Yes, use my reference photos
    [ ] Yes, AI-generated performer
    [ ] No performer, documentary style
    [ ] Mix (performer in some sections)

USER: Mix - performer in some sections

AI: Got it. Let me show you the sections:
    
    [0:00-0:22]  INTRO (spoken) — Performer here?
    [0:22-1:12]  VERSE 1 — Performer here?
    [1:12-1:59]  HOOK (female) — Performer here?
    [1:59-2:43]  VERSE 2 — Performer here?
    [2:43-3:05]  BRIDGE (female) — Performer here?
    [3:05-3:28]  VERSE 3 (heavy rap) — Performer here?
    [3:28-4:45]  VERSE 3 (continued) + OUTRO — Performer here?

USER: Performer in intro, verse 1, verse 3. Documentary style for verse 2. 
      Female performer for hooks/bridge.

AI: Perfect. For the female performer:
    - Should I generate an AI face?
    - Or do you have reference photos?
    
    For female vocal style, I detected: high pitch, crisp, powerful
    Similar to: Portishead/Beth Gibbons vibe
    
    Does that match your vision?

USER: I have reference photos for female (Olivia Swing inspired look).
      Yes, high pitch crisp delivery is correct.

AI: Great. Now let's discuss visual style for each section:
    
    INTRO [0:00-0:22]: You open with "I used to believe you..."
    The lyrics suggest awakening from media deception.
    
    Suggestions:
    [ ] Dark room, wall of TV screens (CNN, Fox, MSNBC, CBS, ABC, NBC)
    [ ] Performer looking at camera, confessional style
    [ ] Start in darkness, light reveals performer
    
    What resonates?

USER: Dark room with TVs. At the end, I open a door and light floods in.

AI: Love it. That's the visual metaphor for the whole song.
    
    For the 6 TV screens — should they show:
    [ ] Actual network logos
    [ ] Generic "news" imagery
    [ ] Specific anchors (Anderson Cooper, Sean Hannity, etc.)

USER: Actual logos AND specific anchors if possible.

AI: Got it. I'll create prompts for:
    - CNN (Anderson Cooper)
    - Fox (Sean Hannity)  
    - MSNBC (Rachel Maddow)
    - CBS (Norah O'Donnell)
    - ABC (David Muir)
    - NBC (Lester Holt)
    
    [Continues for each section...]
```

### Collaboration Output

After conversation, AI generates:
```python
{
    "sections": [
        {
            "id": "01_intro",
            "start": "0:00",
            "end": "0:22",
            "duration_sec": 22,
            "performer": True,
            "performer_type": "male_reference",
            "visual_concept": "Dark room, 6 TV screens behind performer (CNN, Fox, MSNBC, CBS, ABC, NBC). Cold blue/red glow. Performer confessional. Ends: opens door, light floods in, silhouette.",
            "mood": "somber, awakening",
            "prompt": "[Full detailed prompt for video generation]"
        },
        # ... all sections
    ],
    "reference_images": {
        "male_performer": ["path/to/ref1.jpg", "path/to/ref2.jpg"],
        "female_performer": ["path/to/olivia_ref1.jpg"]
    },
    "global_style": {
        "color_grade": "cold, desaturated, blue shadows",
        "aspect_ratio": "16:9",
        "quality": "cinematic"
    }
}
```

---

## Phase 4: Smart Chunking

### The Challenge
- Video APIs (Veo, Kling) generate max 8-10 seconds per call
- Need to split sections into optimal chunks
- Chunks should align with musical/lyrical boundaries
- Each chunk needs its own detailed prompt

### Smart Chunking Logic

```python
def create_chunks(section, max_duration=8):
    """
    Split a section into optimal chunks for API calls.
    Considers:
    - Max duration (8 sec for Veo)
    - Beat boundaries (don't cut mid-beat)
    - Lyrical boundaries (don't cut mid-word/phrase)
    - Visual continuity needs
    """
    chunks = []
    
    # If section <= max_duration, one chunk
    if section.duration <= max_duration:
        chunks.append(create_chunk(section, 0, section.duration))
    else:
        # Split at optimal points
        split_points = find_optimal_splits(section, max_duration)
        for i, (start, end) in enumerate(split_points):
            chunks.append(create_chunk(section, start, end, chunk_num=i+1))
    
    return chunks
```

### Chunk Prompt Generation

Each chunk gets a detailed prompt built from:
1. Global style settings
2. Section visual concept
3. Specific moment in lyrics
4. Continuity notes (if continuing from previous chunk)
5. Performer presence (yes/no, which performer)
6. Camera directions

```python
chunk_prompt = f"""
Cinematic music video scene, 16:9 aspect ratio.

SETTING: {section.visual_concept}
MOMENT: {chunk.lyrics_excerpt}
MOOD: {section.mood}
PERFORMER: {chunk.performer_instructions}
CAMERA: {chunk.camera_direction}

Style: {global_style.color_grade}, {global_style.quality}
Duration: {chunk.duration} seconds

{chunk.specific_details}
"""
```

---

## Phase 5: Generation

### API Integration

#### Primary: Google Veo 3.1
```python
async def generate_video_veo(prompt, duration=8, model="veo-3.1-lite"):
    response = await veo_client.generate(
        prompt=prompt,
        duration=duration,
        aspect_ratio="16:9",
        resolution="1080p",
        fps=24
    )
    return response.video_url
```

#### Cost Estimates (per second):
| Model | Cost/sec | 8-sec chunk | Full 4:45 video |
|-------|----------|-------------|-----------------|
| Veo 3.1 lite | $0.025 | $0.20 | ~$7 |
| Veo 3.1 fast | $0.05 | $0.40 | ~$14 |
| Veo 3.1 | $0.10 | $0.80 | ~$28 |

#### Alternative APIs (swappable):
- Kling API (if available directly)
- Runway Gen-3
- Replicate.com hosted models

### Multiple Takes

```python
async def generate_chunk_with_takes(chunk, num_takes=3):
    takes = []
    for i in range(num_takes):
        video_path = await generate_video_veo(chunk.prompt)
        takes.append({
            "take": i + 1,
            "path": f"{chunk.id}_take{i+1}.mp4",
            "url": video_path
        })
    return takes
```

### Output Organization

```
/output/manufacturing_consent/
├── chunks/
│   ├── 01_intro/
│   │   ├── 01_intro_take1.mp4
│   │   ├── 01_intro_take2.mp4
│   │   └── 01_intro_take3.mp4
│   ├── 02_verse1_chunk1/
│   │   ├── 02_verse1_chunk1_take1.mp4
│   │   └── ...
│   └── ...
├── reference_images/
│   ├── male_performer/
│   └── female_performer/
├── prompts/
│   └── all_prompts.json
├── project.wfp          # Filmora project file
└── manifest.json        # Full generation metadata
```

---

## Phase 6: Output — Filmora Project Generation

### The .wfp Format

Filmora uses `.wfp` files which are **XML-based**. Structure:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Project>
    <Timeline>
        <Track id="1" type="video">
            <Clip>
                <Source>path/to/01_intro_take1.mp4</Source>
                <Start>0</Start>
                <End>22000</End>  <!-- milliseconds -->
                <Position>0</Position>
            </Clip>
            <Clip>
                <Source>path/to/02_verse1_chunk1_take1.mp4</Source>
                <Start>0</Start>
                <End>25000</End>
                <Position>22000</Position>
            </Clip>
            <!-- ... more clips -->
        </Track>
        <Track id="2" type="video">
            <!-- Take 2 versions stacked here -->
        </Track>
        <Track id="3" type="video">
            <!-- Take 3 versions stacked here -->
        </Track>
        <Track id="4" type="audio">
            <Clip>
                <Source>path/to/song.mp3</Source>
                <Start>0</Start>
                <End>285000</End>
                <Position>0</Position>
            </Clip>
        </Track>
    </Timeline>
</Project>
```

### Generation Script

```python
def generate_filmora_project(manifest, output_path):
    """
    Generate a Filmora .wfp project file with:
    - All chunks placed on timeline in order
    - Multiple takes on stacked tracks
    - Audio track with original song
    """
    
    project = FilmoraProject()
    
    # Track 1: Take 1 (primary)
    # Track 2: Take 2 (comparison)
    # Track 3: Take 3 (comparison)
    # Track 4: Audio
    
    for chunk in manifest.chunks:
        for take_num, take in enumerate(chunk.takes):
            track_id = take_num + 1
            project.add_clip(
                track=track_id,
                source=take.path,
                position=chunk.timeline_position,
                duration=chunk.duration
            )
    
    # Add audio
    project.add_audio(manifest.audio_path)
    
    project.save(output_path)
```

### User Workflow After Generation

1. Run generation: `python cli.py generate --song "manufacturing_consent" --takes 3`
2. Open `output/manufacturing_consent/project.wfp` in Filmora
3. See all chunks on timeline, all takes stacked
4. Compare takes, keep best pieces
5. Add transitions, effects, overlays
6. Export final video

---

## CLI Interface

```bash
# Full workflow
python cli.py new --song "manufacturing_consent" --audio song.mp3 --lyrics lyrics.txt

# Start AI collaboration session
python cli.py collaborate --song "manufacturing_consent"

# Generate all chunks
python cli.py generate --song "manufacturing_consent" --takes 3 --model veo-3.1-lite

# Regenerate specific chunk
python cli.py regenerate --song "manufacturing_consent" --chunk 05 --takes 2

# Export Filmora project
python cli.py export --song "manufacturing_consent" --format filmora

# Quick preview (stitch without Filmora)
python cli.py preview --song "manufacturing_consent" --take 1
```

---

## Project Structure

```
/ironrust-video-studio/
├── config/
│   ├── config.yaml           # API keys, default settings
│   └── models.yaml           # Video model configurations
├── src/
│   ├── intake/
│   │   └── intake.py         # File handling, validation
│   ├── analysis/
│   │   ├── transcribe.py     # Whisper integration
│   │   ├── audio_analysis.py # Librosa integration
│   │   └── content_analysis.py # Claude/Opus integration
│   ├── collaboration/
│   │   └── conversation.py   # AI collaboration logic
│   ├── chunking/
│   │   └── smart_chunker.py  # Optimal chunk splitting
│   ├── generation/
│   │   ├── veo.py            # Google Veo API
│   │   ├── kling.py          # Kling API (if available)
│   │   └── runway.py         # Runway API
│   ├── output/
│   │   ├── filmora_export.py # .wfp generation
│   │   ├── ffmpeg_stitch.py  # FFmpeg stitching
│   │   └── organize.py       # File organization
│   └── cli.py                # Main CLI interface
├── projects/                  # Generated projects stored here
│   └── manufacturing_consent/
├── tests/
├── requirements.txt
└── README.md
```

---

## API Keys Required

1. **OpenAI Whisper API** (or local Whisper)
   - For transcription

2. **Anthropic Claude API** (Opus 4 recommended)
   - For content analysis and collaboration

3. **Google Veo API**
   - For video generation
   - Requires Google AI Studio account + billing

4. **Optional: Other video APIs**
   - Runway
   - Replicate
   - Kling (if direct API available)

---

## Cost Comparison

### AIVideo.com (user's experience):
- Paid: ~$754 total
- Got: 28,569 credits remaining
- Per video: ~5,000 credits = ~$125 equivalent

### IronRUST Video Studio:
- Full 4:45 video (1 take): $7-28
- Full video (3 takes): $21-84
- **Savings: 80-95%**

---

## Proof of Concept

The "Manufacturing Consent" video demonstrates every step:

1. ✅ Whisper transcription with corrections
2. ✅ Librosa audio analysis (112 BPM, energy map)
3. ✅ AI-assisted prompt writing (Claude collaboration)
4. ✅ Chunk-by-chunk generation
5. ✅ Multiple takes, cherry-picking best
6. ✅ Filmora assembly with transitions/overlays
7. ✅ Result: Professional music video

**Everything done manually can be automated.**

---

## Development Phases

### Phase 1: Core Pipeline (Week 1-2)
- [ ] Project structure setup
- [ ] Config management
- [ ] Whisper integration
- [ ] Librosa integration
- [ ] Basic CLI

### Phase 2: AI Collaboration (Week 2-3)
- [ ] Claude API integration
- [ ] Song analysis logic
- [ ] Conversation flow
- [ ] Prompt generation

### Phase 3: Video Generation (Week 3-4)
- [ ] Veo API integration
- [ ] Chunk management
- [ ] Multiple takes
- [ ] File organization

### Phase 4: Output & Polish (Week 4-5)
- [ ] Filmora .wfp generation
- [ ] FFmpeg preview stitching
- [ ] Error handling
- [ ] Documentation

### Phase 5: Future Enhancements
- [ ] Web UI (optional)
- [ ] Additional video APIs
- [ ] Auto-transitions
- [ ] SaaS potential

---

## Key Learnings from Manual Process

### Suno AI (music generation):
- Avoid "poison words": hook, chorus, melodic, haunting
- Use [SPOKEN] or [BRIDGE] instead of [HOOK]
- "Monotone delivery," "spoken-rap," "flat pitch" for rap
- Reference specific artists directly
- Generate multiple versions

### Video Prompts:
- Each generation is INDEPENDENT — never reference "previous versions"
- Include exact timestamps
- Use ONE model per song for consistency
- Performer reference requires 3-4 photos
- Green screen performer shots work well for compositing

### Whisper:
- Unreliable for non-English over heavy instrumentation
- Always provide lyrics hint for accuracy
- Manual correction usually needed for names

---

## Summary

**What We're Building:**
A smart, AI-powered music video generation pipeline that:
1. Understands your song (lyrics, structure, themes)
2. Collaborates with you on the vision
3. Generates intelligent prompts automatically
4. Creates video chunks via API
5. Outputs organized files + Filmora project

**Why It's Better:**
- 10-20x cheaper than AIVideo.com
- Actually understands content (not just generic visuals)
- Full creative control
- No monthly subscription
- Customizable (swap models, adjust workflow)

**Who It's For:**
- Creators who know editing (Filmora, Premiere, etc.)
- Power users who want control + automation
- Initially: personal tool
- Future: potential SaaS product

---

## Next Steps

1. **Start new Claude Code CLI session**
2. **Reference this handoff document**
3. **Begin with Phase 1: Core Pipeline**
4. **Build, test, iterate**

Let's fucking build it. 🔥
