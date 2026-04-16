---
name: local-transcribe
description: Transcribe audio/video from a Google Drive public link using local Whisper on GPU. Use when a user shares a Google Drive link and wants it transcribed.
---

# Local Transcribe (Google Drive → Whisper)

Transcribe audio or video files from **public Google Drive links** using the local `whisper` CLI with GPU acceleration (RTX 4090, 24GB VRAM).

## When to Use

Use this skill when a user sends a message containing a Google Drive link (`drive.google.com`). Trigger on ANY of these:

- The link alone with no other text
- The user mentions transcribing, transcription, subtitles, or asks "what does this say/contain"
- The link points to a file with an audio/video extension (.mp3, .wav, .m4a, .ogg, .mp4, .mkv, .avi, .webm, .flac, .qta, etc.)

**When in doubt, assume the user wants transcription** — this is the primary purpose of receiving Drive links.

## Prerequisites

- `gdown` — Google Drive downloader (`pip install gdown`)
- `ffmpeg` — audio conversion
- `whisper` — OpenAI Whisper CLI (`pip install openai-whisper`)
- NVIDIA GPU with CUDA (RTX 4090 recommended)

## CRITICAL RULES

1. **`--device cuda` is MANDATORY.** Whisper defaults to CPU. You MUST always pass `--device cuda`. Never run whisper without it — CPU transcription is 10-20x slower.
2. **Send a message after EVERY step.** This is a single-user system — be fully transparent. The user wants to see exactly what's happening. Never run multiple steps silently.
3. **Ask which model to use** before starting transcription. Don't assume.
4. **Use a unique temp directory per job** to avoid collisions between runs. Use a timestamp: `/tmp/transcribe-YYYYMMDD-HHMMSS/`
5. **ALWAYS clean up temp files** when done — even if a step fails. Never leave files behind.
6. **This is Windows.** Use `C:\tmp\` not `/tmp/`. Whisper names output files after the input filename (e.g., `audio.wav` → `audio.txt`), so keep the input filename consistent as `audio.wav`.

---

## Workflow

### Step 1: Extract the Google Drive file ID & acknowledge

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

Generate a unique working directory using the current timestamp:
```bash
WORKDIR="C:\tmp\transcribe-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$WORKDIR"
gdown "https://drive.google.com/uc?id=FILE_ID" -O "$WORKDIR/input_audio" --fuzzy
```

Notes:
- The `--fuzzy` flag handles various Google Drive URL formats
- If `gdown` fails with "Access Denied", tell the user to set sharing to "Anyone with the link can view"
- You can also pass the full Drive URL directly: `gdown "FULL_URL" -O "$WORKDIR/input_audio" --fuzzy`

After download completes, check the file size:
```bash
ls -lh "$WORKDIR/input_audio"
```

**→ Message the user:**
> ✅ Download complete!
> 📁 Saved to: `C:\tmp\transcribe-XXXXXXXX\input_audio`
> 📦 File size: XX MB
> Starting audio conversion with ffmpeg...

If download failed:
> ❌ Download failed — the file might not be publicly shared. Set sharing to "Anyone with the link can view" and try again.

### Step 3: Convert to Whisper-compatible format

```bash
ffmpeg -y -i "$WORKDIR/input_audio" -vn -ar 16000 -ac 1 -c:a pcm_s16le "$WORKDIR/audio.wav"
```

This converts any format (MP3, M4A, OGG, QTA, MP4, MKV, etc.) to 16kHz mono WAV.

After conversion, get the duration:
```bash
ffprobe -v error -show_entries format=duration -of csv=p=0 "$WORKDIR/audio.wav"
```

**→ Message the user:**
> ✅ ffmpeg conversion done!
> 🎵 Audio duration: XX minutes XX seconds

If ffmpeg failed:
> ❌ ffmpeg failed — the file format isn't supported or the file may be corrupted.

### Step 4: Ask which Whisper model to use

**→ Message the user and WAIT for their reply:**
> Which Whisper model do you want?
>
> 🚀 **turbo** — fastest, good accuracy (~3x realtime on GPU)
> 🎯 **large-v3** — best accuracy, slower (~1x realtime on GPU)
> ⚡ **medium** — balanced speed/accuracy
> 🏃 **small** — fast, lower accuracy
>
> (For a XX-minute file, turbo ≈ ~X min, large-v3 ≈ ~X min)

**Time estimates** (approximate, on RTX 4090):
- **turbo**: ~3x realtime (10 min audio ≈ 3 min processing)
- **large-v3**: ~1x realtime (10 min audio ≈ 10 min processing)
- **medium**: ~5x realtime (10 min audio ≈ 2 min processing)
- **small**: ~10x realtime (10 min audio ≈ 1 min processing)

Calculate estimated time from the audio duration and include it in the message.

### Step 5: Verify GPU is available

Before running whisper, check CUDA:
```bash
python -c "import torch; print('CUDA available:', torch.cuda.is_available()); print('GPU:', torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'NONE')"
```

If CUDA is NOT available:
> ⚠️ CUDA not detected — Whisper will run on CPU which is 10-20x slower. Proceed anyway? (A 30-min file could take 30+ minutes on CPU)

If CUDA is available:
> 🖥️ GPU: NVIDIA RTX 4090 (CUDA). Starting Whisper with `MODEL_NAME` model...

### Step 6: Transcribe with Whisper

```bash
whisper "$WORKDIR/audio.wav" --model MODEL_NAME --device cuda --output_format txt --output_dir "$WORKDIR/"
```

**MANDATORY FLAGS:**
- `--device cuda` — NEVER omit this. Without it, whisper defaults to CPU.
- `--model MODEL_NAME` — use whichever model the user chose
- `--fp16 True` — enabled by default, keeps it for GPU efficiency

If the user specifies a language, add `--language xx` (e.g., `--language en`, `--language zh`). Otherwise let Whisper auto-detect.

**Output file will be:** `$WORKDIR/audio.txt` (whisper names it after the input file `audio.wav` → `audio.txt`)

If the user asks for SRT format later, re-run with `--output_format srt` → produces `$WORKDIR/audio.srt`.

**→ Message the user when complete:**
> ✅ Whisper transcription complete!
> 📝 Detected language: XX
> ⏱️ Processing time: X min X sec
> 📁 Output: `$WORKDIR/audio.txt`
> Sending transcript now...

If whisper failed:
> ❌ Whisper failed — [include the full error message so the user can see what went wrong]

### Step 7: Generate PDF and send the transcript

1. Read the output file: `$WORKDIR/audio.txt`
2. **Verify the file exists first** — if not, check `$WORKDIR/` for any `.txt` files (the output name depends on the input filename)
3. **Generate a PDF** from the transcript:

```bash
python -c "
from fpdf import FPDF
import sys

txt_path = sys.argv[1]
pdf_path = sys.argv[2]

with open(txt_path, 'r', encoding='utf-8') as f:
    text = f.read()

pdf = FPDF()
pdf.add_page()
pdf.add_font('msyh', '', 'C:/Windows/Fonts/msyh.ttc')
pdf.set_font('msyh', size=11)
pdf.multi_cell(0, 7, text)
pdf.output(pdf_path)
" "$WORKDIR/audio.txt" "$WORKDIR/transcript.pdf"
```

> **Note:** If `fpdf2` is not installed: `pip install fpdf2`
> The font `msyh.ttc` (Microsoft YaHei) supports English + Chinese.

4. Send the PDF file to the user as a Telegram document
5. Also send a short text summary (first ~500 words + total word count) as a message

**→ Message the user:**
> 📄 Transcript (XX words) — PDF attached!
>
> **Summary:** [first ~200 words of transcript]...
>
> Full transcript is in the PDF above.

### Step 8: Cleanup

**ALWAYS run cleanup, even if earlier steps failed.** Do not skip this.

```bash
rm -rf "$WORKDIR"
```

**→ Message the user:**
> 🧹 Temp files cleaned up (`$WORKDIR` removed). Done!

## Error Handling

At any step, if something fails, **immediately tell the user** what went wrong with the full error output:

| Error | What to tell the user |
|---|---|
| `gdown` access denied | "❌ Download failed — the file isn't publicly shared. Set it to 'Anyone with the link can view' in Drive sharing settings." |
| `ffmpeg` fails | "❌ ffmpeg conversion failed — the file format might not be supported or the file is corrupted." |
| CUDA not available | "⚠️ GPU not detected — will run on CPU (much slower). Want to proceed or troubleshoot?" |
| `whisper` out of memory | "❌ Whisper ran out of GPU memory. Try the `turbo` or `small` model instead." |
| `whisper` not found | "❌ Whisper CLI not found. Run: `pip install openai-whisper`" |

## Example Interaction

**User:** Transcribe this: https://drive.google.com/file/d/1aBcDeFgHiJk/view?usp=sharing

**Agent sends these messages in sequence:**

1. 🔗 Got it — file ID: `1aBcDeFgHiJk`. Starting download from Google Drive...
2. ✅ Download complete! Saved to `/tmp/transcribe/input_audio` (147 MB). Starting ffmpeg...
3. ✅ ffmpeg done! Audio duration: 42 min 15 sec.
4. Which model? 🚀 turbo (~14 min) | 🎯 large-v3 (~42 min) | ⚡ medium (~8 min) | 🏃 small (~4 min)
5. *(User picks turbo)*
6. 🖥️ GPU: RTX 4090 (CUDA) ✓. Starting Whisper turbo...
7. ✅ Whisper done! Language: English. Time: 13 min 22 sec. Sending transcript...
8. 📄 Transcript (5,230 words): [full text]
9. 🧹 Temp files cleaned up. Done!
