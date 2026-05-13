# Replicate Cog model — Slideshow Render

FFmpeg-based slideshow video render. Takes audio + images, outputs MP4.

## Inputs

| Field | Type | Default | Description |
|---|---|---|---|
| `audio_url` | string? | — | MP3/WAV URL (optional). Sets video duration |
| `image_urls` | string[] | required | PNG/JPG URLs (1-300) |
| `resolution` | enum | `1920x1080` | `1920x1080` / `1280x720` / `854x480` |
| `fps` | int | 30 | 24-60 |
| `transition` | enum | `none` | `none` / `fade` |
| `per_image_duration` | float | 4.0 | Seconds per image (if no audio) |
| `use_nvenc` | bool | false | Use NVIDIA NVENC (needs GPU) |

## Output

Path to `.mp4` file.

## How it works

1. Downloads `audio_url` (if set) → measures duration via ffprobe.
2. Downloads all `image_urls` to temp folder.
3. Calculates per-image duration: `audio_duration / N` (or uses `per_image_duration`).
4. Builds FFmpeg concat demuxer list.
5. Runs `ffmpeg -f concat -i list.txt -i audio.mp3 -c:v libx264 -c:a aac -shortest out.mp4`.
6. Returns the resulting MP4.

## Setup & deploy

### Prerequisites
- Docker (Replicate Cog runs via Docker)
- Cog CLI: `pip install cog` or [download binary](https://github.com/replicate/cog/releases)
- Replicate account + API token

### Local test

```bash
cd replicate-model
cog predict \
    -i 'image_urls=["https://picsum.photos/1920/1080?random=1", "https://picsum.photos/1920/1080?random=2"]' \
    -i 'audio_url="https://download.samplelib.com/mp3/sample-3s.mp3"' \
    -i 'resolution="1920x1080"' \
    -i 'fps=30'
```

### Push to Replicate

1. Create a model on Replicate (web UI): https://replicate.com/create
   - Give it a name like `your-name/autotube-render`
   - Set visibility to **Private** (free) or **Public**
2. Login & push:
   ```bash
   cog login
   cog push r8.im/your-name/autotube-render
   ```
3. Copy the resulting version hash, e.g.:
   ```
   Pushed: your-name/autotube-render@sha256:abc123def...
   ```
4. In our clone: `Settings → Replicate → Model identifier`:
   ```
   your-name/autotube-render:abc123def...
   ```

### Pricing (Replicate)

On CPU instances (default):
- Cold start: ~30 sec (Docker pull)
- Run time: ~3-5 min for 12-min video (30 scenes, 1080p)
- **Cost: ~$0.10-0.25 per video** at $0.000725/sec CPU

On GPU instances (`use_nvenc=true`):
- Set in cog.yaml: `gpu: true`
- ~30-60 sec per video
- **Cost: ~$0.05-0.15 per video** at $0.000725/sec GPU but ~5x faster

## Troubleshooting

- **"image_urls must not be empty"** — pass at least 1 image URL.
- **FFmpeg fails on download** — make sure URLs are publicly accessible (not behind auth).
- **Out of memory** — reduce resolution to 720p, or split large jobs.
- **Slow** — switch `use_nvenc=true` AND set `gpu: true` in cog.yaml AND push to a GPU model.
