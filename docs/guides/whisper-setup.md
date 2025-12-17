# Whisper Speech-to-Text Setup

Whisper is a local speech recognition service that converts audio to text for VoiceMode using OpenAI's Whisper model. It provides offline STT capabilities with various model sizes to balance speed and accuracy.

## Quick Start

```bash
# Install whisper service with default base model (includes Core ML on Apple Silicon!)
voice-mode whisper install

# Install with a different model
voice-mode whisper install --model large-v3

# Install without any model
voice-mode whisper install --no-model

# List available models and their status
voice-mode whisper models

# Download additional models (with Core ML support on Apple Silicon)
voice-mode whisper model install large-v2

# Start the service
voice-mode whisper start
```

**Apple Silicon Bonus:** On M1/M2/M3/M4 Macs, VoiceMode automatically downloads pre-built Core ML models for 2-3x faster performance. No Xcode or Python dependencies required!

Default endpoint: `http://127.0.0.1:2022/v1`

## Installation Methods

### Automatic Installation (Recommended)

VoiceMode includes an installation tool that sets up Whisper.cpp automatically:

```bash
# Install with default base model (142MB) - good balance of speed and accuracy
voice-mode whisper install

# Install with a specific model
voice-mode whisper install --model small

# Skip Core ML on Apple Silicon (not recommended)
voice-mode whisper install --skip-core-ml

# Install without downloading any model
voice-mode whisper install --no-model
```

This will:
- Clone and build Whisper.cpp with GPU support (if available)
- Download the specified model (default: base)
- **On Apple Silicon:** Automatically download pre-built Core ML models for 2-3x faster performance
- Create a start script with environment variable support
- Set up automatic startup (launchd on macOS, systemd on Linux)

### Manual Installation

#### macOS
```bash
# Install via Homebrew
brew install whisper.cpp

# Download model
mkdir -p ~/.voicemode/models/whisper
cd ~/.voicemode/models/whisper
curl -LO https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-large-v2.bin
```

#### Linux
```bash
# Clone and build whisper.cpp
git clone https://github.com/ggerganov/whisper.cpp
cd whisper.cpp
make

# Download model
mkdir -p ~/.voicemode/models/whisper
cd ~/.voicemode/models/whisper
wget https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-large-v2.bin
```

### Prerequisites

**macOS**:
- Xcode Command Line Tools (`xcode-select --install`) - Only for building whisper.cpp
- Homebrew (https://brew.sh)
- cmake (`brew install cmake`)

**Note for Apple Silicon users:** Core ML models are pre-built and downloaded automatically. No Xcode, PyTorch, or coremltools required!

**Linux**:
- Build essentials (`sudo apt install build-essential` on Ubuntu/Debian)

## Core ML Acceleration (Apple Silicon)

On Apple Silicon Macs (M1/M2/M3/M4), VoiceMode automatically downloads pre-built Core ML models from Hugging Face for 2-3x faster transcription:

- **Automatic:** Core ML models download alongside regular models
- **No Dependencies:** No PyTorch, Xcode, or coremltools needed
- **Pre-built:** Models are pre-compiled and ready to use
- **Performance:** 2-3x faster than Metal acceleration alone

To skip Core ML (not recommended):
```bash
voice-mode whisper model install large-v3 --skip-core-ml
```

## Model Management

### Available Models

| Model | Size | RAM Usage | Accuracy | Speed | Language Support |
|-------|------|-----------|----------|-------|-----------------|
| **tiny** | 39 MB | ~390 MB | Low | Fastest | Multilingual |
| **tiny.en** | 39 MB | ~390 MB | Low | Fastest | English only |
| **base** | 142 MB | ~500 MB | Fair | Fast | Multilingual |
| **base.en** | 142 MB | ~500 MB | Fair | Fast | English only |
| **small** | 466 MB | ~1 GB | Good | Moderate | Multilingual |
| **small.en** | 466 MB | ~1 GB | Good | Moderate | English only |
| **medium** | 1.5 GB | ~2.6 GB | Very Good | Slow | Multilingual |
| **medium.en** | 1.5 GB | ~2.6 GB | Very Good | Slow | English only |
| **large-v1** | 2.9 GB | ~3.9 GB | Excellent | Slower | Multilingual |
| **large-v2** | 2.9 GB | ~3.9 GB | Excellent | Slower | Multilingual (recommended) |
| **large-v3** | 3.1 GB | ~3.9 GB | Best | Slowest | Multilingual |
| **large-v3-turbo** | 1.6 GB | ~2.5 GB | Very Good | Moderate | Multilingual |

### Model Commands

```bash
# List all models with installation status
voice-mode whisper models

# Show/set active model
voice-mode whisper model active
voice-mode whisper model active small.en

# Install models
voice-mode whisper model install                  # Install default (large-v2)
voice-mode whisper model install medium           # Install specific model
voice-mode whisper model install all              # Install all models
voice-mode whisper model install large-v3 --force # Force re-download

# Remove models
voice-mode whisper model remove tiny
voice-mode whisper model remove tiny.en --force   # Skip confirmation
```

Note: After changing the active model, restart the whisper service for changes to take effect.

## Service Configuration

### Environment Variables

Configure in `~/.voicemode/voicemode.env`:

```bash
VOICEMODE_WHISPER_MODEL=large-v2
VOICEMODE_WHISPER_PORT=2022
VOICEMODE_WHISPER_LANGUAGE=auto
VOICEMODE_WHISPER_MODEL_PATH=~/.voicemode/models/whisper
```

### Running the Server

#### OpenAI-Compatible Server Mode

```bash
whisper-server \
  --model models/ggml-large-v2.bin \
  --host 127.0.0.1 \
  --port 2022 \
  --inference-path "/v1/audio/transcriptions" \
  --threads 4 \
  --processors 1 \
  --convert \
  --print-progress
```

Key options:
- `--model`: Path to model file
- `--host`: Server host (default: 127.0.0.1)
- `--port`: Server port (VoiceMode expects 2022)
- `--inference-path`: OpenAI-compatible endpoint path
- `--threads`: Number of threads for processing
- `--processors`: Number of parallel processors
- `--convert`: Convert audio to required format automatically
- `--print-progress`: Show transcription progress

### Service Management

#### macOS (LaunchAgent)

```bash
# Start/stop service
launchctl load ~/Library/LaunchAgents/com.voicemode.whisper.plist
launchctl unload ~/Library/LaunchAgents/com.voicemode.whisper.plist

# Enable/disable at startup
launchctl load -w ~/Library/LaunchAgents/com.voicemode.whisper.plist
launchctl unload -w ~/Library/LaunchAgents/com.voicemode.whisper.plist

# Check status
launchctl list | grep whisper
```

#### Linux (Systemd)

```bash
# Start/stop service
systemctl --user start whisper
systemctl --user stop whisper

# Enable/disable at startup
systemctl --user enable whisper
systemctl --user disable whisper

# Check status and logs
systemctl --user status whisper
journalctl --user -u whisper -f
```

## Hardware Acceleration

### Apple Silicon (CoreML)

CoreML provides 2-3x faster transcription on Apple Silicon Macs:

```bash
# Install with CoreML support (when available)
voice-mode whisper model install base.en --skip-core-ml=false

# Performance comparison
# CPU Only: ~1x baseline
# Metal: ~3-4x faster
# CoreML + Metal: ~8-12x faster
```

**Note**: CoreML support requires full Xcode installation and may be disabled in some versions due to installation complexity.

### GPU Acceleration

The installation tool automatically detects and enables:
- **Mac (Apple Silicon)**: Metal acceleration
- **NVIDIA GPU**: CUDA acceleration
- **AMD GPU**: ROCm/HIP acceleration (see below)
- **CPU**: Optimized CPU builds

### AMD ROCm/HIP Acceleration (Linux)

For AMD GPUs on Linux, you can build whisper.cpp with HIP backend for GPU-accelerated transcription. This provides ~20x+ realtime performance on RDNA2/RDNA3 GPUs.

#### Prerequisites

```bash
# Check ROCm is installed and working
rocminfo | grep "gfx"  # Should show your GPU architecture

# Check HIP compiler
hipcc --version

# Install if needed (Arch/CachyOS)
sudo pacman -S cmake ninja rocm-hip-sdk

# Install if needed (Ubuntu/Debian)
# Follow AMD's ROCm installation guide for your distro
```

#### Build with HIP Backend

```bash
# Clone whisper.cpp
cd ~/.voicemode/services
git clone https://github.com/ggerganov/whisper.cpp.git
cd whisper.cpp

# Create build directory
mkdir build && cd build

# Configure with HIP backend
# Adjust AMDGPU_TARGETS for your GPU:
#   - gfx1100: RX 7900 XTX/XT
#   - gfx1101: RX 7800 XT, 7700 XT
#   - gfx1102: RX 7600
#   - gfx1030: RX 6800/6900 series
#   - gfx1031: RX 6700 XT
cmake .. \
  -DGGML_HIP=ON \
  -DAMDGPU_TARGETS="gfx1100;gfx1101;gfx1102" \
  -DCMAKE_BUILD_TYPE=Release \
  -DWHISPER_BUILD_SERVER=ON

# Build
make -j$(nproc)

# Download model
cd ..
./models/download-ggml-model.sh small.en

# Test (should show ROCm device info)
./build/bin/whisper-cli -m models/ggml-small.en.bin -f samples/jfk.wav
```

#### OpenAI-Compatible Proxy

The whisper.cpp server uses `/inference` endpoint, but VoiceMode expects OpenAI's `/v1/audio/transcriptions`. A small proxy translates between formats.

**Create venv and proxy script:**
```bash
cd ~/.voicemode/services
python -m venv whisper-proxy-venv
./whisper-proxy-venv/bin/pip install aiohttp
```

**Proxy script** (`~/.voicemode/services/whisper-proxy.py`):
```python
#!/usr/bin/env python3
import aiohttp
from aiohttp import web

WHISPER_URL = "http://127.0.0.1:2022"

async def health(request):
    return web.json_response({"status": "ok"})

async def transcriptions(request):
    reader = await request.multipart()
    audio_data = None
    async for part in reader:
        if part.name == "file":
            audio_data = await part.read()
    if not audio_data:
        return web.json_response({"error": "No audio file"}, status=400)

    data = aiohttp.FormData()
    data.add_field("file", audio_data, filename="audio.wav", content_type="audio/wav")
    data.add_field("response_format", "json")
    data.add_field("temperature", "0.0")

    async with aiohttp.ClientSession() as session:
        async with session.post(f"{WHISPER_URL}/inference", data=data) as resp:
            if resp.status != 200:
                return web.json_response({"error": await resp.text()}, status=resp.status)
            result = await resp.json()
            return web.json_response({"text": result.get("text", "").strip()})

async def models(request):
    return web.json_response({"object": "list", "data": [{"id": "whisper-1", "object": "model"}]})

app = web.Application()
app.router.add_get("/health", health)
app.router.add_get("/v1/models", models)
app.router.add_post("/v1/audio/transcriptions", transcriptions)

if __name__ == "__main__":
    web.run_app(app, host="127.0.0.1", port=2023)
```

#### Systemd Services (Linux)

**Whisper server** (`~/.config/systemd/user/voicemode-whisper.service`):
```ini
[Unit]
Description=VoiceMode Whisper STT Server (ROCm)
After=network.target

[Service]
Type=simple
ExecStart=%h/.voicemode/services/whisper.cpp/build/bin/whisper-server \
    --model %h/.voicemode/services/whisper.cpp/models/ggml-small.en.bin \
    --host 127.0.0.1 \
    --port 2022
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

**Proxy service** (`~/.config/systemd/user/voicemode-whisper-proxy.service`):
```ini
[Unit]
Description=VoiceMode Whisper OpenAI-Compatible Proxy
After=voicemode-whisper.service
Requires=voicemode-whisper.service

[Service]
Type=simple
ExecStart=%h/.voicemode/services/whisper-proxy-venv/bin/python %h/.voicemode/services/whisper-proxy.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

**Enable services:**
```bash
systemctl --user daemon-reload
systemctl --user enable voicemode-whisper.service voicemode-whisper-proxy.service
systemctl --user start voicemode-whisper.service voicemode-whisper-proxy.service
```

#### VoiceMode Configuration

Add to shell profile (`~/.bashrc` or `~/.zshrc`):
```bash
export VOICEMODE_STT_BASE_URLS="http://127.0.0.1:2023/v1"
```

Then restart Claude Code to use your local GPU-accelerated Whisper.

#### Performance (RDNA3)

| Model | Speed | Notes |
|-------|-------|-------|
| `small.en` | ~23x realtime | Recommended for most use cases |
| `medium.en` | ~10x realtime | Better accuracy |
| `large-v3` | ~5x realtime | Best accuracy |

Example: 11 seconds of audio transcribed in ~480ms on RX 7800 XT.

#### Troubleshooting ROCm

| Problem | Fix |
|---------|-----|
| `hipcc not found` | Install ROCm SDK for your distro |
| `gfx1100 not supported` | Update ROCm or adjust `AMDGPU_TARGETS` |
| Server starts but slow | Check `rocm-smi` — may be falling back to CPU |
| Permission denied on GPU | `sudo usermod -aG render,video $USER` then re-login |
| Proxy not connecting | Verify whisper-server is running on port 2022 |

## Integration with VoiceMode

VoiceMode automatically detects Whisper when available:

1. **First**: Checks for Whisper.cpp on `http://127.0.0.1:2022/v1`
2. **Fallback**: Uses OpenAI API (requires `OPENAI_API_KEY`)

### Custom Configuration

To use a different endpoint or force Whisper use:

```bash
export STT_BASE_URL=http://127.0.0.1:2022/v1
```

Or in MCP configuration:
```json
"voice-mode": {
  "env": {
    "STT_BASE_URL": "http://127.0.0.1:2022/v1"
  }
}
```

## Fully Local Setup

For completely offline voice processing, combine Whisper with Kokoro:

```bash
export STT_BASE_URL=http://127.0.0.1:2022/v1  # Whisper for STT
export TTS_BASE_URL=http://127.0.0.1:8880/v1  # Kokoro for TTS
export TTS_VOICE=af_sky                       # Kokoro voice
```

## Troubleshooting

### Service Won't Start
- Check if port 2022 is already in use: `lsof -i :2022`
- Verify model file exists at configured path
- Check service logs for error messages

### Poor Transcription Quality
- Try a larger model (base → small → medium → large)
- Ensure audio input quality is good
- Set specific language instead of 'auto' if known

### High CPU Usage
- Use a smaller model for better performance
- Consider English-only models (.en) for English content
- Enable GPU acceleration if available

### Model Installation Issues
- Verify adequate disk space (models range from 39MB to 3GB)
- Check network connectivity to Hugging Face
- Use `--force` flag to re-download corrupted models

## Performance Monitoring

```bash
# Check service status
voice-mode whisper status

# View performance statistics
voice-mode statistics

# Monitor real-time processing
tail -f ~/.voicemode/services/whisper/logs/performance.log

# Test model functionality
voice-mode whisper model test base.en
```

## File Locations

- **Models**: `~/.voicemode/models/whisper/` or `~/.voicemode/services/whisper/models/`
- **Service Config**: `~/.voicemode/services/whisper/config.json`
- **Model Preferences**: `~/.voicemode/whisper-models.txt`
- **Logs**: `~/.voicemode/services/whisper/logs/`
- **LaunchAgent** (macOS): `~/Library/LaunchAgents/com.voicemode.whisper.plist`
- **Systemd Service** (Linux): `~/.config/systemd/user/whisper.service`