# 4090 OpenClaw Transcribe Skill

OpenClaw skill for transcribing audio/video files from Google Drive links using local [OpenAI Whisper](https://github.com/openai/whisper) with GPU acceleration.

## How it works

1. User sends a public Google Drive link to the bot on Telegram
2. Agent downloads the file using `gdown`
3. Converts to WAV with `ffmpeg`
4. Transcribes locally with `whisper --model large-v3` on GPU
5. Replies with the transcript text

## Requirements

- **GPU:** NVIDIA RTX 4090 (24GB VRAM) or similar
- **Python packages:** `openai-whisper`, `gdown`
- **System:** `ffmpeg`
- **OpenClaw:** Gateway with Telegram channel configured

## Installation

1. Copy `SKILL.md` to `~/.openclaw/workspace/skills/local-transcribe/`
2. Install dependencies:
   ```bash
   pip install openai-whisper gdown
   ```
3. Restart the gateway:
   ```bash
   openclaw gateway restart
   ```

## Usage

Send a message to your OpenClaw bot on Telegram:

> Transcribe this: https://drive.google.com/file/d/YOUR_FILE_ID/view?usp=sharing

The file must be set to **"Anyone with the link can view"** in Google Drive sharing settings.

## Supported formats

Any audio/video format that `ffmpeg` can read: MP3, WAV, M4A, OGG, MP4, MKV, AVI, QTA, WEBM, FLAC, and more.

## License

MIT
