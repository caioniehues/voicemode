# Whisper.cpp with ROCm/HIP GPU Acceleration

**Date:** 2025-12-17
**Status:** Design approved
**Scope:** Personal setup for RDNA3 GPU

## Overview

Build whisper.cpp from source with HIP backend to enable GPU-accelerated speech-to-text on AMD RDNA3 GPUs with ROCm.

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
- `WHISPER_BUILD_SERVER` — builds `whisper-server` binary for VoiceMode

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

## VoiceMode Integration

**Start whisper-server:**
```bash
~/.voicemode/services/whisper.cpp/build/bin/whisper-server \
  --model ~/.voicemode/services/whisper.cpp/models/ggml-small.en.bin \
  --host 127.0.0.1 \
  --port 2022
```

**Configure VoiceMode:**

Option 1 — config file (`~/.voicemode/config/config.yaml`):
```yaml
stt:
  endpoint: http://127.0.0.1:2022
  model: whisper-1
```

Option 2 — environment variable:
```bash
export VOICEMODE_STT_ENDPOINT="http://127.0.0.1:2022"
```

## Verification

**Confirm GPU usage:**
```bash
# Look for in server startup:
# - "ggml_hip_init: found X ROCm devices"
# - "Device 0: AMD Radeon..."

# Monitor during transcription
watch -n 0.5 rocm-smi
```

**Group permissions (if needed):**
```bash
sudo usermod -aG render,video $USER
# Log out and back in
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `hipcc not found` | `sudo pacman -S rocm-hip-sdk` |
| `gfx1100 not supported` | Update ROCm or check `AMDGPU_TARGETS` |
| Server starts but slow | Check `rocm-smi` — may be falling back to CPU |
| Permission denied on GPU | Add user to `render` and `video` groups |

## Performance Expectations (RDNA3)

| Model | Speed |
|-------|-------|
| `small.en` | ~10x realtime (1s audio → ~0.1s) |
| `medium.en` | ~5x realtime |
| CPU fallback | 0.5-1x realtime |
