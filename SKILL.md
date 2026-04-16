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

## Workflow

### Step 1: Extract the Google Drive file ID

Parse the URL to get the file ID:

| URL format | How to extract |
|---|---|
| `https://drive.google.com/file/d/FILE_ID/view?usp=sharing` | Take the segment between `/d/` and `/view` |
| `https://drive.google.com/open?id=FILE_ID` | Take the `id` query parameter |
| `https://drive.usercontent.google.com/download?id=FILE_ID` | Take the `id` query parameter |

### Step 2: Download the file

```bash
mkdir -p /tmp/transcribe
gdown "https://drive.google.com/uc?id=FILE_ID" -O "/tmp/transcribe/input_audio" --fuzzy
```

Notes:
- The `--fuzzy` flag handles various Google Drive URL formats
- If `gdown` fails with "Access Denied", the file is not publicly shared — tell the user to set sharing to "Anyone with the link can view"
- You can also pass the full Drive URL directly: `gdown "FULL_URL" -O "/tmp/transcribe/input_audio" --fuzzy`

### Step 3: Convert to Whisper-compatible format

```bash
ffmpeg -y -i "/tmp/transcribe/input_audio" -vn -ar 16000 -ac 1 -c:a pcm_s16le "/tmp/transcribe/audio.wav"
```

This converts any format (MP3, M4A, OGG, QTA, MP4, MKV, etc.) to 16kHz mono WAV.

### Step 4: Transcribe with Whisper

```bash
whisper "/tmp/transcribe/audio.wav" --model large-v3 --output_format txt --output_dir "/tmp/transcribe/"
```

- **Model:** `large-v3` (best accuracy, fits in 24GB VRAM)
- **Language:** auto-detected by default. If the user specifies a language, add `--language xx` (e.g., `--language en`, `--language zh`)
- **Output:** `.txt` file in the same directory
- This may take several minutes for long files — that's normal

### Step 5: Read and send the transcript

1. Read the output file: `/tmp/transcribe/audio.txt`
2. If the transcript is **under 4000 characters**: reply with the full text directly
3. If the transcript is **over 4000 characters**: 
   - Send a summary of the first ~500 words as a message
   - Mention the total word count
   - Offer to send it in parts if the user wants the full text

### Step 6: Cleanup

```bash
rm -rf /tmp/transcribe/
```

Always clean up temp files after sending the result.

## Error Handling

| Error | What to tell the user |
|---|---|
| `gdown` access denied | "The file isn't publicly shared. Please set it to 'Anyone with the link can view' in Google Drive sharing settings." |
| `ffmpeg` fails | "The file format isn't supported or the file is corrupted." |
| `whisper` out of memory | "The file is too large for GPU memory. Try sending a shorter clip." |
| `whisper` not found | "Whisper is not installed on this machine." |

## Example Interaction

**User:** Transcribe this: https://drive.google.com/file/d/1aBcDeFgHiJk/view?usp=sharing

**Agent:**
1. Extracts file ID: `1aBcDeFgHiJk`
2. Downloads with `gdown`
3. Converts with `ffmpeg`
4. Runs `whisper --model large-v3`
5. Replies: "Here's the transcript: ..."
6. Cleans up `/tmp/transcribe/`
