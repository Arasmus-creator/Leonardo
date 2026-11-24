# Leonardo local setup report — Radeon RX 9060 XT

**Verified on 2026-11-23 with Xbox package 1.0.2.1.** The Windows runtime reached the Rockstar opening and displayed two distinct loading illustrations. Later, one illustration kept repeating. The session did not reach playable gameplay. This report documents the setup that reached that point.

## Machine and tools

| Component | Configuration used |
|---|---|
| OS and hardware | Windows 11 Pro build 26300, Ryzen 7 5800XT, 32 GB RAM, Radeon RX 9060 XT |
| Graphics | AMD Software 26.11.2, Vulkan driver 2.0.416, Vulkan SDK headers 1.4.363.0 |
| Native build | Visual Studio 2022 Build Tools, MSVC x64 toolset 14.44.35207, CMake 3.31.6, Ninja 1.12.1, SDL 3.4.20 |
| Source and preparation tools | Git for Windows 2.54.0, Python 3.13.9, AMDGPU-capable LLVM for shader disassembly, Vulkan SDK tools for assembly and validation |

The machine already had the toolchain installed. For a new installation, use the project's [Windows prerequisites](https://github.com/Leonardo-Fork/Leonardo/blob/a64d262cdd2032a676451a573a74a22bf706209d/README.md). Select **Desktop development with C++**, Windows SDK 10.0.26100 or newer, and the Visual Studio CMake/Ninja tools. The exact LLVM version from the original shader translation and a clean OS installation were not verified in this run.

## Identify the source before building

The working checkout was `C:\dev\Leonardo`, on local branch `local/loading-art-radeon`. Its base was [PR #23](https://github.com/Leonardo-Fork/Leonardo/pull/23) at **`a64d262cdd2032a676451a573a74a22bf706209d`**. Three additional local source diffs brought in the settings-menu optimization, frame timing, and the opening-video upload fix. Building the base commit alone does not reproduce those additions.

On the configured machine, keep those diffs in place. To reconstruct the source in a **fresh checkout**, these Command Prompt commands fetch the PR and select the tested revision. From there, apply the matching source patch once. Its local backup is outside the repository; another developer needs that patch to reproduce the combined build.

```bat
git clone --recursive https://github.com/Leonardo-Fork/Leonardo.git Leonardo-repro
cd Leonardo-repro
git fetch origin refs/pull/23/head
git switch --detach a64d262cdd2032a676451a573a74a22bf706209d
git submodule update --init --recursive
set "PATCH=%LOCALAPPDATA%\LeonardoRollback\loading-art\menu-video.patch"
git apply --check "%PATCH%" && git apply "%PATCH%"
```

Verified SHA-256 for the source patch: `6f0c67474fe447a4db8aeb48d87cc24ee0d4af6f48d2ddd35b7dc75955959b33`.

## Configure, prepare, build, and launch

1. In the checkout root, create the Git-ignored file `play.local.cmd`. Replace the example package path with the directory that **directly contains** `game_scarlett_master.exe`. On this machine the downloaded package had an extra nested folder. The local output root uses the literal `Documents` path shown below.

   ```bat
   @echo off
   set "LEONIDA_GAME_DIR=D:\path\to\your\readable-package"
   set "LEONIDA_HOME=%USERPROFILE%\Documents\Leonida"
   set "VK_LOADER_LAYERS_DISABLE=~implicit~"
   ```

2. In **Command Prompt**, from the checkout root, load the paths. For a new output directory, prepare the shaders. The earlier run checked 108 package files and translated and validated 26,503 shaders. Subsequent builds on this PC reused the resulting `work\shaders\spirv\index.json`. [Machine report #11](https://github.com/Leonardo-Fork/Leonardo/issues/11)

   ```bat
   call play.local.cmd
   python -I make_game.py --game "%LEONIDA_GAME_DIR%" --check
   python -I make_game.py --game "%LEONIDA_GAME_DIR%" --out "%LEONIDA_HOME%" --stages check,shaders
   ```

3. Generate the BC6H/BC7 texture history from the local package. The commands below produced a 61,216-byte `history.bin` on this PC. The launcher on the tested revision selects that file automatically when `LEONIDA_BCPACK_HISTORY` is unset. Keep the generated tables and package files outside the checkout.

   ```bat
   python -u -I tools\bcpack_history_calib.py "%LEONIDA_GAME_DIR%\data\ttdb" --out "%LEONIDA_HOME%\work\bcpack_history" --limit 32
   python -I bootkit\history_bin.py "%LEONIDA_HOME%\work\bcpack_history" "%LEONIDA_HOME%\work\bcpack_history\history.bin"
   ```

4. Configure, build, and launch from the same checkout. The batch scripts load the x64 Visual Studio environment, build with Ninja in `RelWithDebInfo`, and fetch SDL3. They worked even though ordinary PowerShell could not find CMake. The launcher selects Vulkan and enables the lazy and placeholder pipeline options.

   ```bat
   call rtconf.cmd
   call rtbuild.cmd
   call play.cmd
   ```

## Verify the result and troubleshoot differences

| Checkpoint | Observation on this PC |
|---|---|
| Settings menu | The tester saw below 10 FPS before the buffer-refresh change and above 60 FPS afterward on the window counter. Repeat runs shared a settings overlay and could skip initial setup. [Menu change](https://github.com/SpositoDouglas/Leonardo/commit/a1d70ad555a9b8fee0dfd20a08b991870f5dfc10) |
| Rockstar opening | In the warm-cache comparison with `LEONIDA_FRAME_GAPS=100`, the longest interval during the video was 4,724.7 ms on baseline and 113.9 and 90.2 ms in two corrected runs. The tester observed the mid-video freeze only on baseline and confirmed synchronized audio on both. The fix batched shadow-texture uploads per resource. |
| Loading artwork | Older builds drew a dark panel with a slanted border. With the PR #23 rendering changes and the local table, two distinct illustrations appeared; later one repeated. A native decoder comparison accepted 0/8 sampled UI chunks without the table and 8/8 with it, all provisional. Eight chunks do not imply eight illustrations. |
| Entry to gameplay | Still pending. The last recorded present in the reported run was frame 4,740. A changing frame counter confirms presentation but cannot establish that world loading has completed. |

The following focused checks passed on the combined build. Run them from an **x64 Visual Studio developer prompt** at the checkout root:

```bat
cmake --build runtime\build --target title_tiling_test bindless_sizing_test
ctest --test-dir runtime\build -R "^(title_tiling|bindless_sizing)$" --output-on-failure
```

If the program exits before the opening, first verify the source revision, local diff and the runtime/DLL pair. A previous build on a different PR #8-based branch exited with `0xC0000005`; restoring the executable **and** its matching `gta6rt_tls_host.dll` from the validated video build recovered that setup. For missing artwork, check the table path and whether a stale `LEONIDA_BCPACK_HISTORY` overrides the launcher. Use the local `%LEONIDA_HOME%\last_run.log` for diagnosis. Report aggregate counts and symptoms; keep credentials, game data, raw logs and screenshots out of shared setup reports.
