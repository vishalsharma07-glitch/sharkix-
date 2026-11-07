# sharkiix 3.1.1 — 9070xt patch: what changed and how to build it

Target: RX 9070 XT + Ryzen 5 7600X, Ocarina of Time. Goal: remove the 40–100 ms frame-time spikes.

## What the patch changes (and why)

| Spike source found in 3.1.1 | Change |
|---|---|
| A Switch 2 (native) draw whose pipeline is not compiled yet was **built inline on the scheduler thread**, so that frame lasted as long as a full driver compile (`vk_graphics_pipeline.cpp` `ConfigureDraw` → `RunBuild`). Async shaders never applied to native draws. | The build starts immediately on 2 dedicated urgent threads (ahead of the low-priority FIFO worker queue). The draw waits at most a **3 ms budget per frame**; if the pipeline isn't ready, that draw is skipped for this frame only. Pipelines whose draws write buffers/images or use transform feedback (results the game can read back) keep the old behaviour. |
| Pipelines unused for **10 minutes** were destroyed — returning to an area rebuilt all of them mid-game. | 1 hour (configurable, or never). |
| The used-pipeline list was **written to disk on the GPU thread** in the middle of a draw, every 15 s while new pipelines appear (Windows Defender scanning each write makes it worse). | Written on the existing serialization thread; shutdown waits for it, then saves synchronously. |
| Texture cache capped at **5 GiB on every GPU**: Ocarina's village needs ~8 GB, so textures were deleted/re-uploaded every ~20 s. | Up to 3/4 of the device budget on 12 GB+ cards (sharkiix 3.0's measured fix: 26–28 ms → 23–24 ms frames). |

New switches (all optional, defaults shown) — documented in `dist/nx2_switches.txt`:

```
EDEN_NX2_NATIVE_ASYNC=1                 # 0 = old behaviour (freeze while compiling)
EDEN_NX2_NATIVE_ASYNC_BUDGET_US=3000    # per-frame wait budget for compiling pipelines
EDEN_NX2_URGENT_PIPELINE_THREADS=2      # 1-8
EDEN_NX2_EVICT_PIPELINE_FRAMES=216000   # 0 = never evict
```

Requires **Graphics > Advanced > Enable asynchronous shader compilation = ON** (otherwise
native draws build inline exactly as before).

The log reports what it did every 600 frames:
`Native async pipelines: N draws skipped while their pipeline built, M ms waited`.

Verified here: the patch applies cleanly to the 3.1.1 source zip, and every changed translation unit
(`vk_pipeline_cache.cpp`, `vk_graphics_pipeline.cpp`, `vk_rasterizer.cpp`, `vk_texture_cache_base.cpp`
which instantiates the texture cache) passes a g++ 13 C++23 syntax/semantic compile. Not run on a GPU.

## Build option A — GitHub Actions (no local setup, ~1–2 h)

1. Create a new GitHub repository (private is fine) and push this folder to it:
   ```
   git init
   git add -A
   git commit -m "sharkiix 3.1.1 + 9070xt patch"
   git branch -M rx9070xt-perf
   git remote add origin https://github.com/<you>/<repo>.git
   git push -u origin rx9070xt-perf
   ```
2. The push starts **Actions > sharkiix-linux** (it now has a `windows-x86_64` job). Or start it by hand:
   Actions > sharkiix-linux > Run workflow > branch `rx9070xt-perf`.
3. When the `windows-x86_64 binary` job is green, download the artifact
   `lichii-windows-x86_64-<sha>` from the run page. It contains `lichii.exe`, Qt DLLs and the tuned
   `nx2_switches.txt`.

## Build option B — local Windows build (VS Code / terminal)

Install once:
- Visual Studio 2022 Build Tools with "Desktop development with C++" (MSVC v143, Windows SDK)
- Qt 6.8.3, component **MSVC 2022 64-bit** + Qt Charts (Qt online installer)
- Vulkan SDK (or run `tools/windows/install-vulkan-sdk.ps1` in PowerShell)
- CMake ≥ 3.31, Ninja, Git

Then, in an **x64 Native Tools Command Prompt for VS 2022** (so `cl.exe` is on PATH):

```
cd path\to\Lichii-sharkiix-3.1.1-source
cmake -S . -B build\release -G Ninja ^
  -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF -DYUZU_TESTS=OFF ^
  -DYUZU_CMD=ON -DYUZU_ROOM=OFF -DYUZU_ROOM_STANDALONE=OFF ^
  -DYUZU_USE_BUNDLED_QT=OFF -DCMAKE_PREFIX_PATH=C:\Qt\6.8.3\msvc2022_64 ^
  -DYUZU_USE_BUNDLED_SDL3=ON -DENABLE_UPDATE_CHECKER=OFF -DUSE_DISCORD_PRESENCE=OFF ^
  -DLICHII_BUILD_TAG=sharkiix-3.1.1-9070xt -DLICHII_BUILD_PLATFORM=windows-x86_64
cmake --build build\release --target yuzu yuzu-cmd --parallel
C:\Qt\6.8.3\msvc2022_64\bin\windeployqt --release --no-translations build\release\bin\lichii.exe
copy dist\nx2_switches.txt build\release\bin\
```

The first configure downloads dependencies (CPM) from GitHub; the first build takes 30–90 min on a
7600X. Result: `build\release\bin\lichii.exe`.

## After building

- Copy your `Mod Dynamic FPS` folder setup from the 3.1.1 release (or keep it installed in
  `%APPDATA%\eden\load\0400486024DEC000\`).
- Rename `%APPDATA%\eden\nx2_switches.txt` if it exists (it overrides the one next to the exe).
- If anything new looks wrong (an object missing for more than a moment, or a one-time effect
  that never appears): set `EDEN_NX2_NATIVE_ASYNC=0` in `nx2_switches.txt` to compare against
  the old behaviour, or raise `EDEN_NX2_NATIVE_ASYNC_BUDGET_US` to e.g. 8000.

What this does **not** fix: the GPU→CPU round trips for GPU-written command lists ("list GPU
wait" in `EDEN_NX2_GPU_PROFILE`) and compute shaders compiled the first time an effect is seen —
compute dispatches can't be skipped without breaking game state.
