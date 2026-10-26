# CloudCut — Autonomous AI Movie Editor (v3, 100% free core)

Upload 1–120+ clips → AI edits everything on the server → download 16:9 YouTube + 9:16 Shorts MP4, thumbnail, title/description/hashtags.
**Koi paid API/SDK core workflow mein nahi.** Paid providers (Anthropic, Polly, ElevenLabs, AWS) optional hain aur default OFF.

## Phone-only app (no server, no laptop)
`app/` = on-device Flutter app (FFmpegKit + whisper.cpp + ML Kit). Build steps: `app/README.md`. Server mode (`backend/` + `frontend/`) optional hai.

## Free stack
FFmpeg · OpenCV (+ YuNet face & FER+ emotion ONNX) · Whisper (open-source, local) · numpy audio analysis · Piper / espeak-ng TTS ·
FastAPI · Celery + Redis · PostgreSQL (SQLite dev) · MinIO (S3-compatible storage) · Caddy (auto HTTPS) · Flutter · Ollama (optional local LLM)

## What the AI does automatically (code map)
| Step | How (free) | File |
|---|---|---|
| Analyze every clip | ffprobe, OpenCV frame scan (sharpness, motion, exposure, faces, scene cuts), audio RMS/peaks, Whisper words + sentences | `pipeline/analyze.py`, `audio.py`, `vision.py` |
| Remove silent / blurry / dark / duplicate parts | silencedetect + relative-blur runs + perceptual-hash & transcript similarity (best take kept) | `planner.py` |
| Emotions (happy, sad, fear, surprise, excitement) | FER+ ONNX on faces + audio arousal + motion; Haar/heuristic fallback if model files missing | `analyze.fuse_emotion` |
| Important dialogue | sentence score (length, loudness, !/?, vocabulary) → protected from trimming, emphasised in captions | `analyze.rate_sentences` |
| Best hook | best ~5s window by speech/face/emotion/loudness score → cold open | `planner.extract_hook` |
| Story order | shooting order (creation_time → filename numbers → upload order) + 3-act roles; optional LLM re-orders semantically | `planner.clip_order`, `story.py` |
| Cinematic pacing | cut length from emotion × intensity × story role; speech-safe trims | `pacing.py` |
| Transitions | emotion-based (fade/dissolve/slide/wipe/fadeblack/fadewhite) | `pacing.py`, `render.py` |
| Camera motion + FX (only where useful, ≤30%) | slow push-in, pull-out, pan, slow-motion, speed-ramp | `fx.py`, `render.py` |
| Colour grading | exposure fix + mood grade (cinematic / warm / cool / punchy), vignette | `render.GRADES` |
| Animated captions | Whisper word timestamps → ASS karaoke, pop-in, important words emphasised, frame-synced through transitions | `captions.py` |
| Music | **your** royalty-free tracks in `music/<mood>/` (happy, sad, fear, surprise, excitement) | `music.py` |
| Sound effects | your files in `sfx/` (whoosh/impact/riser) else synthesized by FFmpeg; placed on transitions, climax, slow-mo | `sfx.py` |
| Audio balance | per-clip dialogue levelling, music ducked under dialogue & voiceover, SFX ducked, loudnorm −14 LUFS | `render._final_audio` |
| Voiceover only if needed | only when speech < 30% of runtime AND a script exists (your `brief` or LLM) | `story.py`, `voiceover.py` |
| Exports | 16:9 long + 9:16 Shorts (≤59s, best moments, face-follow crop), 1080p/4K | `render.py` |
| Thumbnail + SEO | best face/sharp frame + bold text; keyword-based title/description/hashtags (+Shorts variant) | `thumbnail.py`, `seo.py` |
| Real-time progress, push, one-tap download | polling 3s + local/FCM notification; presigned MP4 | `tasks.py`, Flutter app |

## Run (any Linux server / VPS / home server)
```bash
cp backend/.env.example backend/.env      # JWT_SECRET, MinIO passwords, S3_PUBLIC_ENDPOINT_URL
export DOMAIN=api.example.com S3_DOMAIN=s3.example.com   # DNS A records -> server IP (Caddy auto-HTTPS)
docker compose up -d --build              # first build downloads free ONNX/Piper models
docker compose up -d --scale worker-render=2
```
Music/SFX: MinIO console (`:9001`) ya `mc cp track.mp3 m/cloudcut/music/happy/`. Optional free LLM: `docker compose --profile llm up -d`,
`docker compose exec ollama ollama pull llava:7b && ollama pull llama3.2:3b`, set `LLM_PROVIDER=ollama`.
Free-tier hosts (e.g. Oracle Cloud Always-Free ARM VM) ho sakte hain — specs/availability khud verify karein; 4K render ke liye ≥8 vCPU behtar.
`deploy/terraform` + GitHub Actions = **optional paid AWS** deployment (ECS/RDS/ALB), core ke liye zaroori nahi.

## APK
```bash
cd frontend && flutter create --platforms=android --org com.cloudcut .
# AndroidManifest: INTERNET + POST_NOTIFICATIONS
flutter pub get && flutter build apk --release --dart-define=API_URL=https://api.example.com
```

## Verified vs not verified (honest)
Verified in a sandbox with real FFmpeg/OpenCV on synthetic videos: analysis, silence/blur removal, planner/story/fx/SEO tests, full render
(9:16 1080p, 20s: video 20.03s = audio 20.03s) with transitions, zoom, slow-motion, grade, karaoke captions, music, synthesized SFX, voiceover
mix, and thumbnail generation. **Not verified here:** real Whisper run, ONNX model downloads (URLs in `scripts/download_models.sh` — falls
back if they fail), Piper/espeak voices, Docker build, MinIO/Caddy, Terraform/AWS, Flutter build/APK, real footage with real faces.
Render was slow in the sandbox (~280s for 20s of 1080p) — use a multi-core server.

## Limits
- Without an LLM, "story understanding" is signal-based (emotion, energy, speech, shooting order), not semantic. For unordered clips it cannot know the plot; add a `brief` or enable Ollama.
- Narration without LLM = your `brief` sentences only. Urdu/Hindi voice = espeak-ng (robotic); Piper default voice is English.
- Slow-motion uses frame blending (not optical flow). Emotion without visible faces falls back to audio/motion arousal.
- Redis broker `noeviction`; render workers should not be scale-in killed mid-job.
