# YouTube Automation with n8n

Automated YouTube content pipeline: Detect trends → Generate scripts → Create voiceovers → Compile videos → Upload to YouTube.

## 📋 Overview

This project provides a complete, modular n8n workflow suite to automate YouTube video creation from trending topics to polished, upload-ready videos.

**Pipeline Stages:**

```
Phase 1: Trend Detection
    ↓ (Reddit + NewsAPI)
Phase 2: Script Generation
    ↓ (Claude AI)
Phase 3a: Voice Generation
    ↓ (ElevenLabs/Google TTS)
Phase 3b: Video Creation
    ↓ (FFmpeg)
Phase 4: YouTube Upload
    ↓ (YouTube API - Coming Soon)
Published Video
```

---

## 🚀 Quick Start

### Prerequisites

- **n8n** (v1.0+) - [Installation Guide](https://docs.n8n.io/getting-started/installation/)
- **FFmpeg** (for Phase 3b video encoding)
- **Node.js** (v16+) - if running n8n locally
- **Docker** (optional, for containerized n8n)

### 1. Install FFmpeg

**macOS:**
```bash
brew install ffmpeg
```

**Ubuntu/Debian:**
```bash
sudo apt-get update
sudo apt-get install ffmpeg
```

**Windows:**
- Download from [ffmpeg.org](https://ffmpeg.org/download.html)
- Add to system PATH or specify full path in `.env`

**Verify Installation:**
```bash
ffmpeg -version
```

### 2. Configure Environment

```bash
# Copy the template
cp .env.example .env

# Edit with your API keys and paths
nano .env  # or use your preferred editor
```

**Required Environment Variables:**
- `ELEVENLABS_API_KEY` - for voice synthesis
- `GOOGLE_CLOUD_TTS_API_KEY` - fallback TTS provider
- `ANTHROPIC_API_KEY` - for script generation
- `NEWSAPI_KEY` - for trend detection
- `FFMPEG_PATH` - path to FFmpeg binary
- `VIDEO_OUTPUT_DIR` - where to save video files
- `TEMP_DIR` - temporary file storage

See `.env.example` for complete list with descriptions.

### 3. Import Workflows into n8n

**Via n8n UI:**

1. Open n8n: `http://localhost:5678`
2. Go to **Workflows** → **Import**
3. Select workflow JSON file
4. Import in this order (Phase 1 → 4):
   - `workflows/01-trend-detection.json`
   - `workflows/02-script-generation.json`
   - `workflows/03-voice-generation.json`
   - `workflows/04-video-creation.json`

**Via n8n CLI:**

```bash
n8n import:workflow --input workflows/03-voice-generation.json
n8n import:workflow --input workflows/04-video-creation.json
```

### 4. Test Workflows

See [Manual Testing](#manual-testing) section below.

---

## 📁 Project Structure

```
.
├── .env.example                      # Environment variable template
├── README.md                         # This file
├── workflows/
│   ├── 01-trend-detection.json       # Phase 1: Detect trending topics
│   ├── 02-script-generation.json     # Phase 2: Generate video scripts
│   ├── 03-voice-generation.json      # Phase 3a: Synthesize voiceovers
│   └── 04-video-creation.json        # Phase 3b: Create videos from audio
└── docs/                             # (Optional) Detailed documentation
    ├── SETUP.md                      # Detailed setup instructions
    ├── API-KEYS.md                   # How to get each API key
    └── TROUBLESHOOTING.md            # Common issues and solutions
```

---

## 🔧 Workflow Details

### Phase 1: Trend Detection (`01-trend-detection.json`)

**Purpose:** Detect trending topics from Reddit and NewsAPI

**Trigger:** Daily schedule (configurable)

**Output:**
```json
{
  "trends": [
    {
      "title": "Topic Title",
      "description": "Short description",
      "source": "reddit|news",
      "trending_score": 0.95
    }
  ],
  "count": 15,
  "timestamp": "2026-08-18T20:00:00Z"
}
```

**Configuration:**
- Set cron trigger interval in workflow editor
- Requires: `NEWSAPI_KEY`, `REDDIT_USER_AGENT`

---

### Phase 2: Script Generation (`02-script-generation.json`)

**Purpose:** Generate YouTube video scripts from trending topics

**Trigger:** Webhook from Phase 1 or manual test

**Input:**
```json
{
  "trends": [...],
  "style": "educational|entertaining|news",
  "duration_seconds": 300
}
```

**Output:**
```json
{
  "script": "Full video script text...",
  "title": "Video Title",
  "keywords": ["keyword1", "keyword2"],
  "duration_estimate": 285,
  "script_id": "script-1629302400000"
}
```

**Configuration:**
- Requires: `ANTHROPIC_API_KEY` or `OPENAI_API_KEY`
- Adjust prompt in "Generate Script" node for style/tone

---

### Phase 3a: Voice Generation (`03-voice-generation.json`)

**Purpose:** Convert scripts to natural-sounding audio voiceovers

**Trigger:** Webhook from Phase 2 or manual test

**Input:**
```json
{
  "script": "Full video script text...",
  "script_id": "script-1629302400000",
  "language": "en-US",
  "title": "Video Title"
}
```

**Output:**
```json
{
  "audio_url": "https://storage.example.com/audio/script-123-voiceover.mp3",
  "audio_duration_seconds": 245,
  "audio_format": "mp3",
  "language": "en-US",
  "voice_used": "ElevenLabs/Adam",
  "script_id": "script-123",
  "status": "audio_ready"
}
```

**Features:**
- ✅ Primary provider: ElevenLabs (natural voices)
- ✅ Fallback provider: Google Cloud TTS
- ✅ Automatic script chunking (avoids API limits)
- ✅ Subtitle timing data (SRT/VTT format)

**Configuration:**
- `ELEVENLABS_API_KEY` - Required for primary provider
- `ELEVENLABS_VOICE_ID` - Choose from: Adam, Belle, Callum, Charlie, Dorothy, Emily, etc.
- `ELEVENLABS_STABILITY` - Range 0-1 (higher = more consistent)
- `ELEVENLABS_SIMILARITY` - Range 0-1 (higher = better quality)
- `GOOGLE_CLOUD_TTS_API_KEY` - Optional (fallback if ElevenLabs fails)

**Voice IDs (ElevenLabs):**
- `21m00Tcm4TlvDq8ikWAM` - Adam (Male, Narration)
- `EXAVITQu4vr4xnSDxMaL` - Bella (Female, Warm)
- `TtoIDIpyhkDXHfEXzMcf` - Dorothy (Female, Youthful)
- `pFZP5JQG7iQjIQuC4Iy3` - Emily (Female, Gentle)

See [ElevenLabs Voices](https://elevenlabs.io/voice-lab) for complete list.

**Node Breakdown:**
- `Trigger - Receive Script` - Webhook listener
- `Process - Validate Script` - Validate input (50-50K chars)
- `Split - Long Scripts` - Chunk text into 5K char segments
- `ElevenLabs - Generate Voice` - Primary TTS synthesis
- `Google Cloud - TTS (Fallback)` - Alternative if primary fails
- `Extract - Audio Data` - Parse response, handle both binary and base64 formats
- `Store - Save Audio` - Webhook to persist audio metadata
- `Log - Voice Ready` - Log success status

**Error Handling:**
- Falls back to Google Cloud TTS if ElevenLabs fails
- Validates script length (50-50K characters)
- Handles chunking for long scripts
- Retries failed requests (exponential backoff)

---

### Phase 3b: Video Creation (`04-video-creation.json`)

**Purpose:** Compile videos from audio, visuals, and effects using FFmpeg

**Trigger:** Webhook from Phase 3a or manual test

**Input:**
```json
{
  "audio_url": "https://storage.example.com/audio/script-123-voiceover.mp3",
  "script_id": "script-123",
  "audio_duration_seconds": 245,
  "language": "en-US",
  "title": "Video Title"
}
```

**Output:**
```json
{
  "long_form_video": {
    "url": "https://storage.example.com/videos/script-123-long-form.mp4",
    "duration_seconds": 245,
    "resolution": "1920x1080",
    "fps": 30,
    "bitrate": "5000k",
    "codec": "h264",
    "status": "ready"
  },
  "shorts_video": {
    "url": "https://storage.example.com/shorts/script-123-shorts.mp4",
    "duration_seconds": 59,
    "resolution": "1080x1920",
    "status": "ready"
  },
  "overall_status": "video_ready_for_upload"
}
```

**Features:**
- ✅ Two output formats: Long-form (1920x1080) + Shorts (1080x1920)
- ✅ FFmpeg hardware acceleration (CUDA/Metal if available)
- ✅ Quality presets: ultrafast → veryslow (quality vs speed)
- ✅ Multiple codec support (H.264, H.265)
- ✅ Automatic thumbnail generation
- ✅ No real videos uploaded (safe testing)

**Configuration:**
- `FFMPEG_PATH` - Path to FFmpeg binary (required)
- `VIDEO_WIDTH`, `VIDEO_HEIGHT` - Output resolution
- `VIDEO_BITRATE` - Quality (higher = better, slower)
- `FFMPEG_PRESET` - Speed/quality tradeoff: `ultrafast`, `fast`, `medium`, `slow`, `veryslow`
- `SHORTS_DURATION_MAX` - Max Shorts length (default: 60 sec)

**Hardware Requirements:**
- **RAM:** 4GB+ (video encoding is memory-intensive)
- **CPU:** 4+ cores (FFmpeg uses all available)
- **Disk:** 50GB+ temp space (2-3x output file size needed during encoding)
- **GPU:** Optional NVIDIA/AMD (CUDA) or Apple Metal for acceleration

**Performance Tips:**
- Use `fast` preset for quick tests, `medium` for production
- Reduce bitrate for lower file sizes: `3000k` instead of `5000k`
- Enable hardware acceleration if GPU available
- Monitor disk space before large batch encoding

**Node Breakdown:**
- `Trigger - Receive Audio` - Webhook listener
- `Process - Validate Audio` - Validate metadata (duration 30s-1h)
- `Fetch - Background Assets` - Download stock footage (optional)
- `Generate - Visual Timeline` - Build FFmpeg filter graph
- `FFmpeg - Compile Video` - Encode long-form video
- `Create - YouTube Shorts` - Generate vertical format config
- `FFmpeg - Encode Shorts` - Encode shorts version
- `Validate - Video Output` - Verify codec, duration, resolution
- `Store - Save Video` - Persist video metadata (no auto-upload)
- `Log - Video Complete` - Log success status

**File Paths:**
- Long-form: `/tmp/n8n-video/{script_id}-long-form.mp4`
- Shorts: `/tmp/n8n-video/{script_id}-shorts.mp4`
- Temp storage: `${TEMP_DIR}` (ensure writable)
- Output: `${VIDEO_OUTPUT_DIR}` (create if not exists)

**Error Handling:**
- Validates audio exists and is valid (30s-1h duration)
- Verifies FFmpeg binary path before encoding
- Checks disk space before starting
- Gracefully handles encoding timeout (1 hour max)
- Retries with lower quality if initial encoding fails

---

## 🔐 Credentials & API Keys

### Required for Phase 3

**ElevenLabs (Voice Synthesis - Primary)**
- Sign up: https://elevenlabs.io/
- Get API key: https://elevenlabs.io/app/keys
- Pricing: ~$0.30 per 10K characters
- Free tier: 10K characters/month

**Google Cloud Text-to-Speech (Fallback)**
- Project setup: https://console.cloud.google.com/
- Enable API: `Text-to-Speech API`
- Get API key: Create service account or API key
- Pricing: $16 per 1M characters (free tier: 500K/month)

**Unsplash (Stock Photos - Optional)**
- Sign up: https://unsplash.com/
- Create app: https://unsplash.com/developers
- Get API key: Show Access Key
- Pricing: Free (rate limited to 50 requests/hour)

### Environment Variable Setup

```bash
# 1. Copy template
cp .env.example .env

# 2. Add your actual keys (use real values, NOT placeholders)
ELEVENLABS_API_KEY=sk_live_your_actual_key_here
GOOGLE_CLOUD_TTS_API_KEY=your_actual_key_here
UNSPLASH_API_KEY=your_actual_key_here

# 3. Set paths (use absolute paths)
FFMPEG_PATH=/usr/bin/ffmpeg                    # Linux/macOS
FFMPEG_PATH=C:/ffmpeg/bin/ffmpeg.exe           # Windows
VIDEO_OUTPUT_DIR=/home/user/videos
TEMP_DIR=/tmp/n8n-video

# 4. Make sure .env is in .gitignore (don't commit!)
echo ".env" >> .gitignore
git add .gitignore
git commit -m "Add .env to gitignore"
```

---

## 🧪 Manual Testing

### Test Phase 3a: Voice Generation

**Step 1: Open n8n**
```
http://localhost:5678
```

**Step 2: Import Workflow**
- Workflows → Import → Select `workflows/03-voice-generation.json`

**Step 3: Configure Credentials**
- Click workflow → Settings → Credentials
- Add n8n credentials for ElevenLabs API key

**Step 4: Test with Sample Script**
- Find `Trigger - Receive Script` node
- Click "Test" button
- Use this sample input:

```json
{
  "script": "Hello world. This is a test voiceover for YouTube automation. The system successfully converted text to speech using natural language processing and neural networks. Thank you for testing.",
  "script_id": "test-001",
  "title": "Test Voiceover",
  "language": "en-US"
}
```

**Step 5: Check Logs**
- Look for "Voice Generation - Chunk 1/1 Complete"
- Audio duration should be ~10-15 seconds
- Status: "audio_chunk_ready"

**Expected Output:**
```json
{
  "audio_url": "https://storage.example.com/audio/test-001-chunk-0.mp3",
  "script_id": "test-001",
  "chunk_index": 0,
  "audio_provider": "elevenlabs",
  "audio_duration_seconds": 12,
  "status": "audio_chunk_ready"
}
```

**Troubleshooting:**
- ❌ "No API key": Check ELEVENLABS_API_KEY in `.env`
- ❌ "Script too short": Use script > 50 characters
- ❌ "Timeout": ElevenLabs API may be slow (wait 30s)
- ❌ "Falls back to Google Cloud": ElevenLabs failed, ensure API key is valid

---

### Test Phase 3b: Video Creation

**Step 1: Import Workflow**
- Workflows → Import → Select `workflows/04-video-creation.json`

**Step 2: Prepare Test Audio**
- Use audio from Phase 3a test above, or provide:
  - `audio_url`: URL to valid MP3 file
  - `audio_duration_seconds`: 45-120 (good range for testing)

**Step 3: Test with Sample Audio**
- Find `Trigger - Receive Audio` node
- Click "Test" button
- Use this sample input:

```json
{
  "audio_url": "https://storage.example.com/audio/test-001-chunk-0.mp3",
  "script_id": "test-001",
  "audio_duration_seconds": 45,
  "language": "en-US",
  "title": "Test Video"
}
```

**Step 4: Monitor Encoding**
- Check n8n execution logs
- Long-form encoding: 5-30 minutes (depends on duration + bitrate)
- Shorts encoding: 2-10 minutes
- Progress appears in `Log - Video Complete` node

**Step 5: Verify Output**
- Check `${VIDEO_OUTPUT_DIR}`:
  - `test-001-long-form.mp4` (1920x1080)
  - `test-001-shorts.mp4` (1080x1920)
- File sizes: 50-500 MB (depends on duration + bitrate)

**Expected Output:**
```json
{
  "long_form_video": {
    "url": "https://storage.example.com/videos/test-001-long-form.mp4",
    "resolution": "1920x1080",
    "fps": 30,
    "status": "ready"
  },
  "shorts_video": {
    "url": "https://storage.example.com/shorts/test-001-shorts.mp4",
    "resolution": "1080x1920",
    "status": "ready"
  },
  "overall_status": "video_ready_for_upload"
}
```

**Troubleshooting:**
- ❌ "FFmpeg not found": Check FFMPEG_PATH in `.env`
- ❌ "Timeout (3600s)": Video too long or bitrate too high
- ❌ "Disk full": Encoding needs 2-3x output file size temp space
- ❌ "Audio duration invalid": Must be 30-3600 seconds

---

### Test Complete Pipeline (Phase 1→4)

**Full End-to-End Test:**

1. **Phase 1**: Manual execute trend detection
   - Output: List of 10-15 trending topics

2. **Phase 2**: Pass trends to script generation
   - Output: Complete video script

3. **Phase 3a**: Pass script to voice generation
   - Output: MP3 audio file + metadata

4. **Phase 3b**: Pass audio to video creation
   - Output: Long-form + Shorts MP4 files

5. **Phase 4** (future): Pass video metadata to YouTube uploader
   - Output: Published video URL

---

## 📊 Input/Output Formats

### Audio Formats

**Supported Input:**
- MP3 (MPEG-3)
- WAV (Waveform Audio)
- M4A (MPEG-4 Audio)
- OGG (Ogg Vorbis)

**Output Format:**
- MP3 (default, `AUDIO_OUTPUT_FORMAT=mp3`)
- WAV (`AUDIO_OUTPUT_FORMAT=wav`)
- Sample rate: 44100 Hz (standard)
- Bitrate: 128k (good quality), 192k (high quality)

### Video Formats

**Long-Form Output:**
- Format: MP4 (H.264 codec)
- Resolution: 1920x1080 (Full HD)
- Framerate: 30 FPS
- Bitrate: 5000k (adjustable via `VIDEO_BITRATE`)
- Duration: 30 seconds - 15 minutes

**Shorts Output:**
- Format: MP4 (H.264 codec)
- Resolution: 1080x1920 (9:16 aspect, vertical)
- Framerate: 30 FPS
- Bitrate: 3000k (optimized for mobile)
- Duration: Max 60 seconds (auto-trimmed)

**Codec Support:**
- H.264 (libx264) - Default, good compatibility
- H.265 (libx265) - Better compression, needs modern hardware
- VP9 - WebM format (not used in YouTube pipeline)

---

## 🔄 Workflow Integration

### Webhook Connections

Workflows communicate via webhooks (HTTP POST requests):

```
Phase 2 → Phase 3a
├─ POST to: http://localhost:5678/webhook/generate-voiceover
├─ Payload: { script, script_id, language, title }
└─ Response: { audio_url, audio_duration_seconds, ... }

Phase 3a → Phase 3b
├─ POST to: http://localhost:5678/webhook/create-video
├─ Payload: { audio_url, script_id, audio_duration_seconds }
└─ Response: { long_form_video, shorts_video, ... }

Phase 3b → Phase 4 (future)
├─ POST to: http://localhost:5678/webhook/upload-to-youtube
├─ Payload: { video_url, shorts_url, title, script_id }
└─ Response: { youtube_video_id, published_url, ... }
```

### Manual Integration

To trigger workflows manually:

```bash
# Trigger Phase 3a manually
curl -X POST http://localhost:5678/webhook/generate-voiceover \
  -H "Content-Type: application/json" \
  -d '{
    "script": "Your video script here",
    "script_id": "manual-001",
    "title": "Manual Test",
    "language": "en-US"
  }'

# Trigger Phase 3b manually
curl -X POST http://localhost:5678/webhook/create-video \
  -H "Content-Type: application/json" \
  -d '{
    "audio_url": "https://storage.example.com/audio/manual-001.mp3",
    "script_id": "manual-001",
    "audio_duration_seconds": 60,
    "title": "Manual Test"
  }'
```

---

## 📈 Performance & Costs

### Execution Time

| Phase | Duration | Notes |
|-------|----------|-------|
| **1** (Trend Detection) | 30 sec | Parallel API calls |
| **2** (Script Generation) | 30-60 sec | Depends on Claude API speed |
| **3a** (Voice Generation) | 20-120 sec | Linear with script length |
| **3b** (Video Creation) | 5-30 min | Depends on video duration + preset |
| **Total Pipeline** | 10-35 min | For 5-minute video end-to-end |

### Approximate Monthly Costs

| Provider | Usage | Cost |
|----------|-------|------|
| **ElevenLabs** | 500K characters | $15 |
| **Google Cloud TTS** | Backup (0 if ElevenLabs works) | $0 |
| **NewsAPI** | 500 requests | Free (25/day limit) |
| **FFmpeg** | Unlimited | Free (local) |
| **Anthropic Claude** | 1M tokens | $50 |
| **n8n Cloud** | 100 workflows | $30-300/month* |
| **Total** | | ~$95-365/month |

*Self-hosted n8n on VPS is $5-20/month

### Cost Optimization

- ✅ Use free tiers: NewsAPI (25 req/day), Google TTS (500K chars/month)
- ✅ Batch processing: Process multiple trends in one run
- ✅ Lower bitrate: `3000k` instead of `5000k` (saves 40% file size)
- ✅ Self-host n8n: $5-20/month vs $300+/month cloud
- ✅ Cache results: Don't regenerate audio/video if unchanged
- ✅ Monitor usage: Set daily limits in workflows

---

## 🐛 Troubleshooting

### Phase 3a: Voice Generation Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "API key invalid" | Wrong ElevenLabs key | Verify key in dashboard: https://elevenlabs.io/app/keys |
| "Rate limit exceeded" | Too many requests | Add delay between API calls (default: 5s) |
| "Script too short/long" | Character count invalid | Use 50-50K character scripts |
| "Timeout after 60s" | API slow or network issue | Retry; ElevenLabs API can be slow during peak hours |
| "Google TTS fallback fails" | Both providers failed | Check internet connection, API keys |

### Phase 3b: Video Creation Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "FFmpeg not found" | Path incorrect | Run `which ffmpeg` to find correct path, update `.env` |
| "Permission denied" | Directory not writable | `chmod 755 ${VIDEO_OUTPUT_DIR}` |
| "Encoding timeout" | Video too long + high bitrate | Reduce bitrate: `VIDEO_BITRATE=3000k` |
| "Disk full" | Insufficient temp space | Free up space; needs 2-3x output file size |
| "Audio not found" | URL invalid or 404 | Verify audio file exists and is publicly accessible |
| "Invalid codec" | Unsupported FFmpeg build | Reinstall FFmpeg: `brew install ffmpeg` |

### General Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "Workflow won't trigger" | Inactive workflow | Click toggle to activate in n8n UI |
| "Webhooks not working" | N8N_WEBHOOK_URL incorrect | Ensure URL matches n8n server address |
| ".env not loaded" | Restart required | Restart n8n container/process |
| "Out of memory" | Large batch size | Reduce batch: test with 1-2 videos first |

---

## 📚 Additional Resources

### Official Documentation
- [n8n Docs](https://docs.n8n.io/)
- [n8n Webhook Nodes](https://docs.n8n.io/getting-started/core-concepts/executions/#webhooks)
- [FFmpeg Documentation](https://ffmpeg.org/documentation.html)

### API Documentation
- [ElevenLabs API](https://elevenlabs.io/docs)
- [Google Cloud Text-to-Speech](https://cloud.google.com/text-to-speech/docs)
- [Anthropic Claude API](https://docs.anthropic.com/)
- [YouTube API](https://developers.google.com/youtube)

### Workflow Tips
- Use n8n's built-in testing nodes before deploying
- Monitor execution logs for debugging
- Set up error notifications (email/Slack)
- Version control workflows as JSON (already done!)
- Test with small data first, then scale up

---

## 📝 License

This project is provided as-is for educational and personal use.

---

## 🤝 Contributing

Contributions welcome! Please:
1. Test workflows thoroughly before submitting
2. Document any new environment variables
3. Update README.md with changes
4. Keep workflow JSON files committed
5. Never commit `.env` with real secrets

---

**Last Updated:** 2026-08-18  
**Phase Status:** ✅ Phase 1-3 Complete | 🚧 Phase 4 Coming Soon