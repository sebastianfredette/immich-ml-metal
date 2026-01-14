# immich-ml-metal

> **⚠️ Important Disclaimer**: I (the repository owner) am not a software developer. This project was architected, designed, and primarily authored by Claude based on my requirements and feedback. While I've tested it in my home environment, please treat this as an **experimental community project** rather than production-ready software. I'm sharing this in hopes it's useful for the Mac community. If it's not helpful for you, please don't feel pressured to use it.

A Metal/ANE-optimized drop-in replacement for [Immich's](https://immich.app/) machine learning service, designed specifically for Apple Silicon Macs. This allows Mac users to run Immich's ML workloads natively on their hardware instead of requiring Docker Desktop.

## What This Does

Immich's standard ML container runs well on NVIDIA, Intel, and AMD GPUs. Recently, the community has had trouble running it natively on Apple's ML framework (particularly after OCR was implemented). This project is a drop-in replacement for Immich-ML that uses the same ML API but uses as many native Apple ML frameworks as possible:

- **CLIP Embeddings**: MLX-accelerated (with open_clip fallback) for image/text search
- **Face Detection**: Apple Vision framework (runs on Neural Engine)
- **Face Recognition**: InsightFace ArcFace with CoreML acceleration
- **OCR**: Apple Vision framework text recognition

## Performance: Why This is Fast

This service uses Apple's MLX framework for CLIP. In my testing, this is significantly faster than alternatives (PyTorch or CPU-only). Due to Metal GPU limitations (AFAIK), inference requests are processed sequentially, but **total throughput is still excellent**:

## Project Status

**⚠️ Alpha Quality - Use at Your Own Risk**

- [x] Phase 0: API contract documentation
- [x] Phase 1: Project setup  
- [x] Phase 2: Stub service (placeholder responses)
- [x] Phase 3: CLIP implementation (MLX + open_clip fallback)
- [x] Phase 4: Face detection (Vision framework)
- [x] Phase 5: Face embeddings (InsightFace + CoreML)
- [x] Phase 6: OCR (Vision framework)
- [x] Phase 7: Basic integration testing with real Immich instance
- [ ] Phase 8: Community testing and validation

**Known Limitations:**
- Only tested in my specific home setup (MacBook Pro M1, macOS 15.2, Immich v1.122.x)
- Not all Immich ML features may be fully compatible
- Memory usage not extensively optimized
- No load testing performed
- Could literally only work on my machine(tm)

## Requirements

- **macOS 13+** (Ventura or later)
- **Apple Silicon Mac** (M1/M2/M3/M4 - Intel Macs not supported)
- **Python 3.11**
- **Immich server** already running (this replaces just the ML service)

## Installation

```bash
# Clone the repository
git clone https://github.com/yourusername/immich-ml-metal.git
cd immich-ml-metal

# Pin mlx-clip to a specific commit (for stability)
# Get the current commit hash:
git ls-remote https://github.com/harperreed/mlx_clip.git HEAD
# Edit requirements.txt and replace the tail end of the mlx_clip.git address with the new hash

# Create and activate virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# First run will download models (~500MB-2GB depending on choices)
# This may take 5-15 minutes
python -m src.main
```

## Configuration

Configure via environment variables or edit `src/config.py`:

| Variable | Default | Description |
|----------|---------|-------------|
| `ML_HOST` | `0.0.0.0` | Bind address |
| `ML_PORT` | `3003` | Port number (must match Immich config) |
| `ML_MODELS_DIR` | `./models` | Model storage directory |
| `ML_CLIP_MODEL` | `ViT-B-32__openai` | CLIP model name |
| `ML_FACE_MODEL` | `buffalo_l` | Face recognition model (buffalo_s/m/l) |
| `ML_FACE_MIN_SCORE` | `0.7` | Face detection confidence threshold |
| `ML_USE_COREML` | `true` | Enable CoreML acceleration |
| `ML_USE_ANE` | `true` | Enable Apple Neural Engine |

### Model Choices

**CLIP Model Mapping**:
- OpenAI CLIP models -> MLX
  - `ViT-B-32__openai`: `mlx-community/clip-vit-base-patch32`
  - `ViT-B-16__openai`: `mlx-community/clip-vit-base-patch16`
  - `ViT-L-14__openai`: `mlx-community/clip-vit-large-patch14`
    
- LAION CLIP models -> MLX
  - `ViT-B-32__laion2b-s34b-b79k`: `mlx-community/clip-vit-base-patch32-laion2b`
  - `ViT-B-32__laion2b_s34b_b79k`: `mlx-community/clip-vit-base-patch32-laion2b`
    
- SigLIP models -> None (uses open_clip fallback)
  - `ViT-B-16-SigLIP__webli`
  - `ViT-B-16-SigLIP2__webli`
  - `ViT-SO400M-16-SigLIP2-384__webli`
    
- Default fallback: `mlx-community/clip-vit-base-patch32`

**Face Models**:
- `buffalo_s`
- `buffalo_m`
- `buffalo_l` - **default**

## Connecting to Immich

In your Immich `docker-compose.yml` or `.env`:

```yaml
# Point to your Mac's IP address (not localhost unless Immich is also native)
MACHINE_LEARNING_URL=http://192.168.1.100:3003
```

## Actually Running the Service

```bash
source .venv/bin/activate
uvicorn src.main:app --host 0.0.0.0 --port 3003 --workers 1
```

### Running as a Service (macOS launchd):

Create `~/Library/LaunchAgents/com.immich.ml.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.immich.ml</string>
    <key>ProgramArguments</key>
    <array>
        <string>/path/to/your/.venv/bin/python</string>
        <string>-m</string>
        <string>src.main</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/path/to/immich-ml-metal</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/immich-ml.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/immich-ml-error.log</string>
</dict>
</plist>
```

Then:
```bash
launchctl load ~/Library/LaunchAgents/com.immich.ml.plist
launchctl start com.immich.ml
```

## Verification

Test the service is working:

```bash
# Health check
curl http://localhost:3003/ping
# Should return: pong

# Service info
curl http://localhost:3003/
# Should return: {"message":"Immich ML"}

# Test CLIP text encoding
curl -X POST http://localhost:3003/predict \
  -F 'entries={"clip":{"textual":{"modelName":"ViT-B-32__openai"}}}' \
  -F 'text=a photo of a cat'
```

In Immich, you should see the ML service connect in the admin logs.

## Contributing

Given this is primarily an AI-assisted project, contributions are **very welcome**, especially from actual developers who can:

- Review and improve the code quality
- Add proper tests
- Verify Immich compatibility  
- Optimize performance
- Add missing features
- Improve documentation

## Support

This is a hobby project with no guarantees. I barely know how to use Git.

## Final Notes

This project exists because I wanted to run Immich ML on my Mac with passable hardware acceleration. **It works for me**, but may not work for everyone. Use at your own risk, and please contribute improvements if you can!

---

**tl;dr**: AI-written Immich ML service for Apple Silicon. Alpha quality. Use with caution. Please help.