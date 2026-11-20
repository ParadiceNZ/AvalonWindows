
# AvalonWindows

A fork of [Avalon](https://github.com/freriadoc/Avalon) specifically fine-tuned for **Windows** on AMD Radeon GPUs.

Contains Windows-specific code paths focusing on doing everything possible to maximise both prompt reading and token generation speed.

(coming soon: a table showing a comparison to current upstream Avalon builds)


## How to compile on Windows (AMD GPU, HIP)

These steps build `llama-server.exe` for an AMD Radeon card with the AMD HIP SDK. 

### 1. Install Visual Studio Build Tools 2022

Note: this is **Visual Studio Build Tools**, not Visual Studio Code. VS Code is an editor and cannot build this.

1. Download **Build Tools for Visual Studio 2022** from https://visualstudio.microsoft.com/downloads/
   (under "Tools for Visual Studio").
2. Run the installer and tick the **Desktop development with C++** workload.
3. In the panel on the right, make sure these are ticked (they normally are by default):
   - MSVC v143 - VS 2022 C++ x64/x86 build tools
   - Windows 11 SDK (or Windows 10 SDK)
   - C++ CMake tools for Windows (this provides CMake and Ninja)
4. Click **Install** and wait for it to finish.

### 2. Install Git

Download and install **Git for Windows** from https://git-scm.com/download/win (the default options are fine).

### 3. Install the AMD HIP SDK

1. Update your graphics driver (AMD Software: Adrenalin Edition) to a recent version.
2. Download the **AMD HIP SDK for Windows** (version 7.2) from AMD's site:
   https://www.amd.com/en/developer/resources/rocm-hub/hip-sdk.html
3. Install it with the default options. It installs to `C:\Program Files\AMD\ROCm\7.2` and sets the `HIP_PATH`
   environment variable.

### 4. Get the source

Open a Command Prompt and run (pick any folder you like instead of `I:\`):

```bat
cd /d I:\
git clone https://github.com/ParadiceNZ/AvalonWindows.git
cd AvalonWindows
git checkout pr/windows-all
```

### 5. Find your GPU's target name

| Card | Target |
|---|---|
| RX 7900 XTX / XT / GRE | `gfx1100` |
| RX 7800 XT / 7700 XT | `gfx1101` |
| RX 7600 / 7600 XT | `gfx1102` |
| RX 6950 XT / 6900 XT / 6800 XT / 6800 | `gfx1030` |
| RX 9070 XT / 9070 | `gfx1201` |
| RX 9060 XT | `gfx1200` |

Other cards: see the processor list at https://llvm.org/docs/AMDGPUUsage.html#processors

### 6. Build

Save this as `build-windows.bat` in the `AvalonWindows` folder, change `GPU_TARGET` to your card's target from
step 5, and double-click it (or run it from a Command Prompt):

```bat
@echo off
rem build-windows.bat [clean] - builds build\bin\llama-server.exe
setlocal
set GPU_TARGET=gfx1101
set HIP_PATH=C:\Program Files\AMD\ROCm\7.2
set VS_DIR=C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools

call "%VS_DIR%\VC\Auxiliary\Build\vcvars64.bat" >nul
rem the HIP SDK's clang must come first on PATH; CMake and Ninja come with the Build Tools
set PATH=%HIP_PATH%\bin;%VS_DIR%\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin;%VS_DIR%\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja;%PATH%

cd /d "%~dp0"
if /i "%~1"=="clean" rmdir /s /q build

cmake -S . -B build -G Ninja -DGGML_HIP=ON -DGPU_TARGETS=%GPU_TARGET% -DCMAKE_BUILD_TYPE=Release ^
      -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ > configure.log 2>&1
if not exist build\build.ninja (
    echo CMake configure FAILED - see configure.log
    findstr /i "error" configure.log
    pause & exit /b 1
)

cmake --build build --target llama-server > build.log 2>&1
if exist build\bin\llama-server.exe (
    echo OK: %~dp0build\bin\llama-server.exe
) else (
    echo Build FAILED - last 40 lines of build.log:
    powershell -NoProfile -Command "Get-Content build.log -Tail 40"
)
pause
```

When it finishes, you should have `build\bin\llama-server.exe`.

### 7. Run it

Download the model as described in the main README, then start the server from the model's folder. 
Here's a known working and optimized command line for a 16GB card on Windows with 64GB+ of RAM (remember to set the paths for "C:\Models" and "I:\AvalonWindows" to wherever you've installed and downloaded)

```bat
set OMP_PLACES=cores
set OMP_PROC_BIND=spread
set GGML_CUDA_MOE_CACHE_MIN_EXPERT_KB=256
set GGML_HIP_COPY_RING=8,16
set GGML_CUDA_MOE_RING_PREFILL=900
set GGML_CUDA_MOE_PIN_DIRECT=1
set GGML_CUDA_HOST_BOUNCE=0

set MODEL_PATH=C:\Models
set AVALON_PATH=I:\AvalonWindows

cd /d %MODEL_PATH%\Qwen3.8-Flash-Next-Avalon-GGUF
"%AVALON_PATH%\build\bin\llama-server" -m Qwen3.8-Flash-Next-Avalon-IQ3KS-00001-of-00033.gguf ^
  -c 262144 -b 16384 -ub 16384 -fa on -ctk q8_0 -ctv q4_0 --jinja --parallel 1 ^
  -md MTP\mtp-Qwen3.8-Flash-Next-shared-IQ3K-IQ4NL-FR65k.gguf --spec-type draft-mtp ^
  --spec-draft-n-max 2 -ubd 256 -ctkd q8_0 -ctvd q8_0
```

Leave out `GGML_CUDA_MOE_PIN_DIRECT=1` if you have less than ~64 GB of RAM, or need memory for other things (it locks ~50GB of memory permanently while it's running)

### Updating

```bat
cd /d I:\AvalonWindows
git pull
build-windows.bat
```

Only the changed files are recompiled. Use `build-windows.bat clean` to start from scratch if a build behaves oddly.

### Troubleshooting

- **"vcvars64.bat is not recognized" / CMake not found**: the Build Tools are installed elsewhere, or without the
  C++ workload. Check that `VS_DIR` in the script matches your install folder.
- **CMake picks the wrong compiler, or "clang is not recognized"**: check that `HIP_PATH` in the script matches your
  HIP SDK folder (`C:\Program Files\AMD\ROCm\<version>`).
- **The build fails**: the errors are in `build.log` (search for `error:`); configure problems are in `configure.log`.
- **`llama-server.exe` will not start, complaining about a missing `amdhip64` DLL**: add
  `C:\Program Files\AMD\ROCm\7.2\bin` to your PATH, or update the graphics driver.



# About Avalon

Avalon is a fork of [llama.cpp](https://github.com/ggml-org/llama.cpp) built for one model, [Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next) (512 experts, 10 active, 48 layers), on one consumer GPU with the experts in system RAM.

<p align="center"><img src="media/avalon-logo.png" alt="Avalon" width="200"></p>

## Credits

- [Avalon](https://github.com/freriadoc/Avalon) - the upstream these changes are based on.
- [llama.cpp](https://github.com/ggml-org/llama.cpp) and the ggml authors: the base.
- [ik_llama.cpp](https://github.com/ikawrakow/ik_llama.cpp) (Iwan Kawrakow): the IQ3_K and IQ3_KS types, the shared MTP head support.
- [Strata](https://github.com/Niko1221/Strata) (Niko1221): the expert ranking format and the preload idea.
- [Unsloth](https://huggingface.co/unsloth): the MTP head.
- [drluoto / Vitronia](https://huggingface.co/drluoto): the quantizations and the FR-Spec token list our GGUF files are built from.
- [Qwen](https://huggingface.co/Qwen): the model.


## License

MIT, as llama.cpp. The model files are under the Qwen Community License 1.0.
