# Building LiteRT-LM on Windows — CI/CD Guide

> **End-to-end instructions for building `litert_lm_main.exe` on Windows,
> suitable for GitHub Actions CI/CD.**

## Overview

LiteRT-LM uses **Bazel 7.x** as its build system. On Windows, this requires
Visual Studio Build Tools (MSVC), Git Bash, and Java.

### Build Output

```
bazel-bin/runtime/engine/litert_lm_main.exe  (~17.6 MB, statically linked)
```

---

## Prerequisites

Install these **before** the build step in your CI runner:

| Tool | Version | Install Command |
|------|---------|----------------|
| **Bazelisk** | latest | `winget install Bazel.Bazelisk` |
| **VS BuildTools** | 2022 | `winget install Microsoft.VisualStudio.2022.BuildTools --custom "--add Microsoft.VisualStudio.Workload.VCTools --includeRecommended --quiet"` |
| **Git for Windows** | latest | `winget install Git.Git` |
| **Python** | 3.13+ | `winget install Python.Python.3.13` |
| **Java** | 17+ | `winget install EclipseAdoptium.Temurin.21.JDK` |
| **7-Zip** | latest | `winget install 7zip.7zip` (for zip packaging) |

### Environment Variables

```powershell
# Required by Bazel on Windows
$env:BAZEL_VC = "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC"
$env:BAZEL_VC_FULL_VERSION = (Get-ChildItem "$env:BAZEL_VC\Tools\MSVC" | Sort-Object Name -Descending | Select-Object -First 1 -ExpandProperty Name)
$env:BAZEL_SH = "C:\PROGRA~1\Git\bin\bash.exe"   # Short path avoids spaces
$env:PATH = "C:\Users\adilh\AppData\Local\Microsoft\WinGet\Links;$env:PATH"  # bazelisk
```

> **Note on `BAZEL_SH`**: The `C:\PROGRA~1\` short (8.3) path is critical —
> Bazel's `.bazelrc` parser splits on spaces, so `"C:\Program Files\Git\..."`
> must use the short form or quotes.

---

## Build Steps

### Step 1: Clone with LFS

```powershell
git clone https://github.com/google-ai-edge/LiteRT-LM.git
cd LiteRT-LM
git checkout <tag>  # e.g. v0.13.0
git lfs pull
```

### Step 2: Cache Dependencies (CI Optimization)

```powershell
# Bazel cache speeds up subsequent builds dramatically
# First build: ~13 minutes. Cached rebuild: ~10 seconds.
$env:BAZEL_DISK_CACHE = "D:\bazel-cache"
```

### Step 3: CPU-Only Build

```powershell
bazelisk --output_base=C:/bzl build //runtime/engine:litert_lm_main --config=windows
```

**Output:** `bazel-bin/runtime/engine/litert_lm_main.exe`

### Step 4: GPU-Enabled Build (Optional)

> **Note:** LiteRT-LM uses **WebGPU** (via Dawn/Direct3D 12) for GPU on
> Windows — not CUDA. See [GPU Support](#gpu-support) below.

```powershell
bazelisk --output_base=C:/bzl build //runtime/engine:litert_lm_main `
    --config=windows `
    --define=litert_link_capi_so=true `
    --define=resolve_symbols_in_exec=false
```

This produces an executable that loads `libLiteRt.dll` and
`libLiteRtWebGpuAccelerator.dll` at runtime (see packaging below).

---

## Packaging for Release

### Layout

```
LiteRT-LM-v0.13.0-Windows-x86_64/
├── bin/
│   ├── litert_lm_main.exe        (17.6 MB)
│   ├── libLiteRt.dll             (10.8 MB, prebuilt)
│   ├── libLiteRtWebGpuAccelerator.dll  (10.6 MB)
│   ├── libwebgpu_dawn.dll        (10.8 MB)
│   ├── libLiteRtTopKWebGpuSampler.dll  (5.7 MB)
│   ├── libGemmaModelConstraintProvider.dll (12.3 MB)
│   ├── dxcompiler.dll            (17.1 MB, for GPU)
│   └── dxil.dll                  (1.4 MB, for GPU)
├── models/                       # Place .litertlm files here
└── README.txt
```

> **IMPORTANT**: All DLLs must be in the **same directory** as the EXE.
> Separating them into a `lib/` subdirectory will cause DLL load failures.

### Packaging Script

```powershell
$version = "v0.13.0"
$pkgDir = "LiteRT-LM-$version-Windows-x86_64"
$binDir = "bazel-bin\runtime\engine"
$prebuiltDir = "prebuilt\windows_x86_64"

# Create structure
New-Item "$pkgDir\bin", "$pkgDir\models" -ItemType Directory -Force | Out-Null

# Copy binary
Copy-Item "$binDir\litert_lm_main.exe" "$pkgDir\bin\" -Force

# Copy prebuilt DLLs (use prebuilt versions, not Bazel-built symlinks)
Copy-Item "$prebuiltDir\libLiteRt.dll" "$pkgDir\bin\" -Force
Copy-Item "$prebuiltDir\libLiteRtWebGpuAccelerator.dll" "$pkgDir\bin\" -Force
Copy-Item "$prebuiltDir\libLiteRtTopKWebGpuSampler.dll" "$pkgDir\bin\" -Force
Copy-Item "$prebuiltDir\libwebgpu_dawn.dll" "$pkgDir\bin\" -Force
Copy-Item "$prebuiltDir\libGemmaModelConstraintProvider.dll" "$pkgDir\bin\" -Force

# Download DXC for GPU support
curl.exe -L -o dxc.zip "https://github.com/microsoft/DirectXShaderCompiler/releases/download/v1.9.2602/dxc_2026_02_20.zip"
python -c "import zipfile; z=zipfile.ZipFile('dxc.zip','r'); [z.extract(n,'.') for n in z.namelist() for f in ['dxil.dll','dxcompiler.dll'] if n.endswith(f) and 'bin/x64' in n.replace('\\','/')]; z.close()"
Move-Item "bin\x64\dxil.dll", "bin\x64\dxcompiler.dll" "$pkgDir\bin\" -Force
Remove-Item "dxc.zip", "bin" -Recurse -Force

# README
Set-Content "$pkgDir\README.txt" @"
LiteRT-LM Windows Binary $version
Quick Start:
  .\bin\litert_lm_main.exe --model_path="models\model.litertlm" --backend=cpu
"@

# Zip it
7z a -tzip "LiteRT-LM-$version-Windows-x86_64.zip" "$pkgDir\*"
```

---

## GitHub Actions Workflow

```yaml
name: Build Windows Binary

on:
  push:
    tags:
      - 'v*'  # Trigger on version tags
  workflow_dispatch:  # Manual trigger

jobs:
  build:
    runs-on: windows-2022
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          lfs: true

      - name: Install Bazelisk
        run: winget install Bazel.Bazelisk -e --accept-source-agreements

      - name: Install VS BuildTools
        run: |
          winget install Microsoft.VisualStudio.2022.BuildTools --silent `
            --custom "--add Microsoft.VisualStudio.Workload.VCTools --includeRecommended --quiet --wait"

      - name: Install Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.13'

      - name: Install Java
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'

      - name: Set build environment
        shell: pwsh
        run: |
          $msvcVer = Get-ChildItem "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Tools\MSVC" `
            | Sort-Object Name -Descending | Select-Object -First 1 -ExpandProperty Name
          echo "BAZEL_VC=C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC" | Out-File -FilePath $env:GITHUB_ENV -Encoding utf8 -Append
          echo "BAZEL_VC_FULL_VERSION=$msvcVer" | Out-File -FilePath $env:GITHUB_ENV -Encoding utf8 -Append
          echo "BAZEL_SH=C:\PROGRA~1\Git\bin\bash.exe" | Out-File -FilePath $env:GITHUB_ENV -Encoding utf8 -Append

      - name: Restore Bazel cache
        uses: actions/cache@v4
        with:
          path: |
            C:\bzl
            ~/.cache/bazel-disk-cache
          key: bazel-${{ runner.os }}-${{ hashFiles('WORKSPACE', '.bazelversion') }}

      - name: Build (CPU-only)
        shell: pwsh
        run: |
          bazelisk --output_base=C:/bzl build //runtime/engine:litert_lm_main --config=windows

      - name: Build (GPU-enabled)
        shell: pwsh
        run: |
          bazelisk --output_base=C:/bzl build //runtime/engine:litert_lm_main `
            --config=windows `
            --define=litert_link_capi_so=true `
            --define=resolve_symbols_in_exec=false

      - name: Package release
        shell: pwsh
        run: |
          $version = "${{ github.ref_name }}"
          # ... packaging script from above ...

      - name: Create Release
        uses: softprops/action-gh-release@v2
        with:
          files: LiteRT-LM-*-Windows-x86_64.zip
          generate_release_notes: true
```

---

## GPU Support

### How GPU Acceleration Works on Windows

LiteRT-LM uses **WebGPU** (via Dawn) on Windows, which in turn uses
**Direct3D 12** under the hood:

```
litert_lm_main.exe  →  libLiteRt.dll  →  libLiteRtWebGpuAccelerator.dll
                                           →  libwebgpu_dawn.dll
                                                →  dxcompiler.dll (HLSL→DXIL)
                                                →  dxil.dll (DXIL validator)
                                                →  D3D12 (Windows built-in)
```

### Prerequisites for GPU

- **NVIDIA GPU** with recent drivers (or Intel Arc/AMD)
- **DirectX Shader Compiler DLLs** (`dxcompiler.dll`, `dxil.dll` v1.9.2602+)
  — download from [Microsoft/DirectXShaderCompiler](https://github.com/microsoft/DirectXShaderCompiler/releases)
- All DLLs must be in the same directory as the executable

### GPU Debugging

```
# Check GPU adapter selection:
--backend=gpu
# Look for log line:
# "Selected adapter: NVIDIA GeForce RTX 5060 Laptop GPU, arch=blackwell"
```

### Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| Access violation (0xC0000005) | ABI mismatch: built `libLiteRt.dll` vs prebuilt WebGPU DLL | Use prebuilt `libLiteRt.dll` from `prebuilt/windows_x86_64/` |
| `dxil.dll: Windows Error 87` | Missing DirectX Shader Compiler | Download DXC v1.9.2602+ alongside the binary |
| `No available engine for backend: GPU_ARTISAN` | GPU_ARTISAN engine not registered | Use `--backend=gpu` instead |
| `WEBGPU_ADAPTER_TYPE_DISCRETE_GPU` not found | No dedicated GPU detected | Falls back to CPU automatically |

---

## CI Runner Requirements

The GitHub Actions `windows-2022` runner has most tools pre-installed:
- Visual Studio 2022 (with MSVC)
- Git for Windows
- Python 3.x
- Java 11+

You only need to install via `winget`:
- `Bazel.Bazelisk` (not pre-installed)

### Disk Space

First build downloads ~2-5 GB of external dependencies (TensorFlow, XLA,
gRPC, etc.). Use `--output_base=C:/bzl` (or a temp drive) to avoid filling
the system drive.

---

## Troubleshooting

### "Unrecognized option: --output_base=..."

`--output_base` is a **startup option**, not a build option. Use:
```powershell
bazelisk --output_base=C:/bzl build //target    # correct
bazelisk build //target --output_base=C:/bzl     # WRONG!
```

### "LNK1181: cannot open input file"

Path too long for MSVC linker. Set a short `--output_base` (e.g. `C:/bzl`).

### "Error applying patch command sed..."

Bazel can't find `bash.exe`. Verify Git Bash is installed and set:
```
$env:BAZEL_SH = "C:\PROGRA~1\Git\bin\bash.exe"
```

### First build takes >30 minutes

Expected. Bazel downloads and compiles TensorFlow, XLA, protobuf, gRPC,
flatbuffers, Abseil, and dozens of other libraries from source.
Subsequent builds are near-instant due to caching.
