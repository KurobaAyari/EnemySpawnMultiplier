# Build from source

Windows x64, Python 3.10+, Git, and MinGW gcc. Fetch LuaJIT with `python -B scripts/bootstrap_dependencies.py` if `tools/src/LuaJIT` is missing, then from `tools/src/LuaJIT/src`:

```bat
mingw32-make XCFLAGS="-DLUAJIT_DISABLE_GC64"
```

Put Git `usr\bin` on PATH so `uname`, `sed` and `[` exist. Set `HD2_GAME_ROOT` if needed, then from the repository root:

```powershell
$env:HD2_LUAJIT = (Resolve-Path 'tools/src/LuaJIT/src/luajit.exe').Path
$env:HD2_GAME_ROOT = 'D:\SteamLibrary\steamapps\common\Helldivers 2'
python -B scripts/build.py
```

The builder verifies the supported EXE and game.dll hashes, compiles the maintained
panel build, runs synthetic memory checks, and inspects the archive in Python. Run
`python -B scripts/build.py` to build
`releases/Enemy-Spawn-Multiplier-Panel-v23.zip` by default (or with
`python -B scripts/build.py panel-menus`). `panel` and `panel-local` remain
available for the existing development profiles. The builder does
not install or launch the game.

This package requires the official Bingus Shared Loader v15 or newer. The archive exposes a plaintext `mods/cowboybingus/enemy_spawn_multiplier` discovery entry and keeps the compiled implementation in `mods/cowboybingus/enemy_spawn_multiplier_impl`. The v15 loader discovers the entry without a coordinator-list edit; the stable entry name also remains compatible with legacy explicit registration.

The MODS settings integration is optional: Mod Options Menu v1.1+ and loader
v18+ provide translated native rows. There is no HD2Runtime dependency.
`tests/test_mod_options.lua` checks batching, synchronization, persistence,
language changes and fallback behavior against the real configuration patch
and Win32 panel. If `../research/ModOptionsMenu` exists (or
`HD2_MOD_OPTIONS_SOURCE` names its checkout), the build also runs
`tests/test_mod_options_upstream.lua` against the actual public menu API and
option model, without activating native game integration.
