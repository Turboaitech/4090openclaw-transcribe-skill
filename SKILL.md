---
name: local-transcribe
description: Transcribe audio/video from a Google Drive public link using local Whisper on GPU. Use when a user shares a Google Drive link and wants it transcribed.
---

# Local Transcribe (Google Drive → Whisper)

Transcribe audio or video files from **public Google Drive links** using the local `whisper` CLI with GPU acceleration (RTX 4090, 24GB VRAM).

## When to Use

Use this skill when a user sends a message containing a Google Drive link (`drive.google.com`) **and** mentions transcribing, transcription, subtitles, or asks "what does this say/contain."

## Prerequisites

- `gdown` — Google Drive downloader (`pip install gdown`)
- `ffmpeg` — audio conversion
- `whisper` — OpenAI Whisper CLI (`pip install openai-whisper`)
- NVIDIA GPU with CUDA (RTX 4090 recommended)

## IMPORTANT: Live Progress Updates

**You MUST send a message to the user after EVERY step.** This is a single-user system — be fully transparent. The user wants to see exactly what's happening at each stage. Never run multiple steps silently.

---

## Workflow

### Step 1: Extract the Google Drive file ID

Parse the URL to get the file ID:

| URL format | How to extract |
|---|---|
| `https://drive.google.com/file/d/FILE_ID/view?usp=sharing` | Take the segment between `/d/` and `/view` |
| `https://drive.google.com/open?id=FILE_ID` | Take the `id` query parameter |
| `https://drive.usercontent.google.com/download?id=FILE_ID` | Take the `id` query parameter |

**→ Message the user:**
> 🔗 Got it — file ID: `FILE_ID`
> Starting download from Google Drive...

### Step 2: Download the file

```bash
mkdir -p /tmp/transcribe
gdown "https://drive.google.com/uc?id=FILE_ID" -O "/tmp/transcribe/input_audio" --fuzzy
```

Notes:
- The `--fuzzy` flag handles various Google Drive URL formats
- If `gdown` fails with "Access Denied", tell the user to set sharing to "Anyone with the link can view"
- You can also pass the full Drive URL directly: `gdown "FULL_URL" -O "/tmp/transcribe/input_audio" --fuzzy`

After download completes, check the file size:
```bash
ls -lh /tmp/transcribe/input_audio
```

**→ Message the user:**
> ✅ Download complete!
> 📁 Saved to: `/tmp/transcribe/input_audio`
> 📦 File size: XX MB
> Starting audio conversion with ffmpeg...

If download failed:
> ❌ Download failed — the file might not be publicly shared. Set sharing to "Anyone with the link can view" and try again.

### Step 3: Convert to Whisper-compatible format

```bash
ffmpeg -y -i "/tmp/transcribe/input_audio" -vn -ar 16000 -ac 1 -c:a pcm_s16le "/tmp/transcribe/audio.wav"
```

This converts any format (MP3, M4A, OGG, QTA, MP4, MKV, etc.) to 16kHz mono WAV.

After conversion, check the WAV file and get duration:
```bash
ffprobe -v error -show_entries format=duration -of csv=p=0 "/tmp/transcribe/audio.wav"
```

**→ Message the user:**
> ✅ ffmpeg conversion done!
> 🎵 Audio duration: XX minutes XX seconds
> 🔄 Starting Whisper transcription (model: large-v3)... this may take a few minutes.

If ffmpeg failed:
> ❌ ffmpeg failed — the file format isn't supported or the file may be corrupted.

### Step 4: Transcribe with Whisper

```bash
whisper "/tmp/transcribe/audio.wav" --model large-v3 --output_format txt --output_dir "/tmp/transcribe/"
```

- **Model:** `large-v3` (best accuracy, fits in 24GB VRAM)
- **Language:** auto-detected by default. If the user specifies a language, add `--language xx` (e.g., `--language en`, `--language zh`)
- **Output:** `.txt` file in the same directory

**→ Message the user:**
> ✅ Whisper transcription complete!
> 📝 Detected language: XX
> Sending transcript now...

If whisper failed:
> ❌ Whisper failed — [include the error message so the user can see what went wrong]

### Step 5: Read and send the transcript

1. Read the output file: `/tmp/transcribe/audio.txt`
2. If the transcript is **under 4000 characters**: send the full text directly
3. If the transcript is **over 4000 characters**:
   - Send the first chunk (~3500 chars)
   - Follow up with remaining chunks
   - At the end, mention total word count

**→ Message the user (with transcript):**
> 📄 Transcript (XX words):
>
> [transcript text here]

### Step 6: Cleanup

```bash
rm -rf /tmp/transcribe/
```

**→ Message the user:**
> 🧹 Temp files cleaned up. Done!

## Error Handling

At any step, if something fails, **immediately tell the user** what went wrong and what they can do about it:

| Error | What to tell the user |
|---|---|
| `gdown` access denied | "❌ Download failed — the file isn't publicly shared. Set it to 'Anyone with the link can view' in Drive sharing settings." |
| `ffmpeg` fails | "❌ ffmpeg conversion failed — the file format might not be supported or the file is corrupted." |
| `whisper` out of memory | "❌ Whisper ran out of GPU memory — the audio might be too long. Try a shorter clip." |
| `whisper` not found | "❌ Whisper CLI not found on this machine." |

## Example Interaction

**User:** Transcribe this: https://drive.google.com/file/d/1aBcDeFgHiJk/view?usp=sharing

**Agent sends these messages in sequence:**

1. 🔗 Got it — file ID: `1aBcDeFgHiJk`. Starting download from Google Drive...
2. ✅ Download complete! Saved to `/tmp/transcribe/input_audio` (147 MB). Starting ffmpeg conversion...
3. ✅ ffmpeg done! Audio duration: 42 min 15 sec. Starting Whisper (large-v3)... this'll take a few minutes.
4. ✅ Whisper complete! Detected language: English. Sending transcript...
5. 📄 Transcript (5,230 words): [full text]
6. 🧹 Temp files cleaned up. Done!
