# Whisper.cpp with ROCm/HIP GPU Acceleration

**Date:** 2025-12-17
**Status:** Implemented
**Scope:** Personal setup for RDNA3 GPU (RX 7800 XT)

## Overview

Build whisper.cpp from source with HIP backend to enable GPU-accelerated speech-to-text on AMD RDNA3 GPUs with ROCm. Includes an OpenAI-compatible API proxy for VoiceMode integration.

## Architecture

```
Claude Code → VoiceMode MCP → whisper-proxy (port 2023)
                                    ↓
                              whisper-server (port 2022, ROCm GPU)
```

**Components:**
1. `whisper-server` — Native whisper.cpp server with HIP/ROCm, uses `/inference` endpoint
2. `whisper-proxy` — Python aiohttp proxy translating OpenAI API format to whisper.cpp format
3. systemd user services for auto-start

## Prerequisites

**System requirements:**
- ROCm installed system-wide
- HIP compiler (`hipcc`) in PATH
- CMake 3.21+

**Verification:**
```bash
# Check ROCm is working
rocminfo | grep "gfx"  # Should show gfx1100/gfx1101/gfx1102 for RDNA3

# Check HIP compiler
hipcc --version

# Check CMake version
cmake --version
```

**Install if needed (Arch/CachyOS):**
```bash
sudo pacman -S cmake ninja rocm-hip-sdk
```

**Model choice:** `ggml-small.en` — good accuracy/speed balance for RDNA3 GPUs with plenty of VRAM.

## Build Process

```bash
# Clone whisper.cpp
cd ~/.voicemode/services
git clone https://github.com/ggerganov/whisper.cpp.git
cd whisper.cpp

# Create build directory
mkdir build && cd build

# Configure with HIP backend for RDNA3
cmake .. \
  -DGGML_HIP=ON \
  -DAMDGPU_TARGETS="gfx1100;gfx1101;gfx1102" \
  -DCMAKE_BUILD_TYPE=Release \
  -DWHISPER_BUILD_SERVER=ON

# Build
make -j$(nproc)
```

**CMake flags explained:**
- `GGML_HIP=ON` — enables ROCm/HIP backend
- `AMDGPU_TARGETS` — targets RDNA3 architectures:
  - gfx1100: RX 7900 XTX/XT
  - gfx1101: RX 7800 XT, 7700 XT
  - gfx1102: RX 7600
- `WHISPER_BUILD_SERVER` — builds `whisper-server` binary

**Download model:**
```bash
cd ~/.voicemode/services/whisper.cpp
./models/download-ggml-model.sh small.en
```

**Test build:**
```bash
./build/bin/whisper-cli -m models/ggml-small.en.bin -f samples/jfk.wav
# Should show GPU device info and transcribe the sample
```

## OpenAI-Compatible Proxy

The whisper.cpp server uses `/inference` endpoint, but VoiceMode expects OpenAI's `/v1/audio/transcriptions`. A small Python proxy translates between formats.

**Create venv and install dependencies:**
```bash
cd ~/.voicemode/services
python -m venv whisper-proxy-venv
./whisper-proxy-venv/bin/pip install aiohttp
```

**Proxy script** (`~/.voicemode/services/whisper-proxy.py`):
```python
#!/usr/bin/env python3
"""
Simple proxy to make whisper.cpp server OpenAI API compatible.
Translates /v1/audio/transcriptions -> /inference
"""

import asyncio
import aiohttp
from aiohttp import web

WHISPER_URL = "http://127.0.0.1:2022"

async def health(request):
    return web.json_response({"status": "ok"})

async def transcriptions(request):
    """Handle OpenAI-compatible /v1/audio/transcriptions endpoint"""
    reader = await request.multipart()

    audio_data = None
    response_format = "json"

    async for part in reader:
        if part.name == "file":
            audio_data = await part.read()
        elif part.name == "response_format":
            response_format = (await part.read()).decode()

    if not audio_data:
        return web.json_response({"error": "No audio file provided"}, status=400)

    data = aiohttp.FormData()
    data.add_field("file", audio_data, filename="audio.wav", content_type="audio/wav")
    data.add_field("response_format", "json")
    data.add_field("temperature", "0.0")

    async with aiohttp.ClientSession() as session:
        async with session.post(f"{WHISPER_URL}/inference", data=data) as resp:
            if resp.status != 200:
                error_text = await resp.text()
                return web.json_response({"error": error_text}, status=resp.status)

            result = await resp.json()
            text = result.get("text", "").strip()
            return web.json_response({"text": text})

async def models(request):
    """Return fake models list for compatibility"""
    return web.json_response({
        "object": "list",
        "data": [{"id": "whisper-1", "object": "model"}]
    })

app = web.Application()
app.router.add_get("/health", health)
app.router.add_get("/v1/models", models)
app.router.add_post("/v1/audio/transcriptions", transcriptions)

if __name__ == "__main__":
    print("Whisper proxy starting on http://127.0.0.1:2023")
    print("Forwarding to whisper.cpp at http://127.0.0.1:2022")
    web.run_app(app, host="127.0.0.1", port=2023)
```

## Systemd Services

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

## VoiceMode Configuration

Add to shell profile (`~/.bashrc` or `~/.zshrc`):
```bash
# VoiceMode: Use local Whisper with ROCm GPU
export VOICEMODE_STT_BASE_URLS="http://127.0.0.1:2023/v1"
```

Then restart Claude Code or source the profile.

## Verification

**Check services:**
```bash
systemctl --user status voicemode-whisper
systemctl --user status voicemode-whisper-proxy
```

**Check endpoints:**
```bash
curl http://127.0.0.1:2022/health  # whisper-server
curl http://127.0.0.1:2023/health  # proxy
```

**Confirm GPU usage (in service logs):**
```bash
journalctl --user -u voicemode-whisper -f
# Look for:
# - "ggml_cuda_init: found X ROCm devices"
# - "Device 0: AMD Radeon RX 7800 XT, gfx1101"
# - "using ROCm0 backend"
```

**Monitor GPU during transcription:**
```bash
watch -n 0.5 rocm-smi
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `hipcc not found` | `sudo pacman -S rocm-hip-sdk` |
| `gfx1100 not supported` | Update ROCm or check `AMDGPU_TARGETS` |
| Server starts but slow | Check `rocm-smi` — may be falling back to CPU |
| Permission denied on GPU | `sudo usermod -aG render,video $USER` then re-login |
| Proxy not connecting | Check whisper-server is running on port 2022 |
| VoiceMode not using local | Verify `VOICEMODE_STT_BASE_URLS` is set, restart Claude Code |

## Performance Results (RX 7800 XT)

Tested with JFK sample (11 seconds of audio):

| Metric | Value |
|--------|-------|
| Total time | 480ms |
| Encode time | 93ms |
| Speed | ~23x realtime |
| Model | ggml-small.en (487 MB on GPU) |

## File Locations

```
~/.voicemode/services/
├── whisper.cpp/                    # whisper.cpp source and build
│   ├── build/bin/whisper-server   # GPU-accelerated server
│   └── models/ggml-small.en.bin   # Model file
├── whisper-proxy.py               # OpenAI API proxy
└── whisper-proxy-venv/            # Python venv for proxy

~/.config/systemd/user/
├── voicemode-whisper.service       # Whisper server service
└── voicemode-whisper-proxy.service # Proxy service
```
