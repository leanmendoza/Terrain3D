# Setup and build

## Prerequisites

- macOS with full Xcode (`xcode-select -p` should point to `/Applications/Xcode.app/...`,
  not the command-line tools shim — emsdk needs the full SDK headers).
- `scons >= 4.0`, `python >= 3.8`. SCons 4.10 + Python 3.14 verified.
- `git`.

## Clone with submodule

```bash
git clone --recurse-submodules --shallow-submodules \
  git@github.com:leanmendoza/Terrain3D.git
cd Terrain3D
```

The `godot-cpp` submodule is pinned by Terrain3D's `main`. As of writing it points to
master HEAD, which corresponds to **Godot 4.6.x** development. There is no
`godot-4.6-stable` tag yet on godot-cpp; master is the right ref for Godot 4.6.x.

If you need a different Godot version, see upstream `doc/docs/building_from_source.md`
for the procedure to point the submodule at a matching `godot-cpp` tag/commit.

## Building for macOS

No emsdk required for native targets.

```bash
scons platform=macos arch=arm64 target=template_debug -j$(sysctl -n hw.ncpu)
# → project/addons/terrain_3d/bin/libterrain.macos.debug.framework/libterrain.macos.debug

scons platform=macos arch=arm64 target=template_release
```

Add `dev_build=yes` to keep symbols for lldb.

## Building for Web

### Pick the matching emsdk

The wasm produced here is loaded by Godot's web template. Their emscripten ABIs must
match (struct layouts, calling conventions). **Mismatch produces silent corruption
that surfaces as `Uncaught RuntimeError: memory access out of bounds` deep in scene
loading**, not at extension init.

| Godot version | emsdk |
|---|---|
| 4.4.x         | 3.1.64 (per Terrain3D CI: `.github/actions/base-deps/action.yml`) |
| 4.6.x         | **4.0.20** (read from `Build configuration: Emscripten X.Y.Z` in browser console) |

To check what your Godot uses, open any web export in the browser, look at console.

### Install emsdk

```bash
git clone https://github.com/emscripten-core/emsdk.git ~/emsdk
cd ~/emsdk
./emsdk install 4.0.20
./emsdk activate 4.0.20
```

### Build

Source the env in the **same** shell as scons (each Bash call is a fresh shell):

```bash
cd Terrain3D
source ~/emsdk/emsdk_env.sh
scons platform=web target=template_debug threads=no debug_symbols=no -j$(sysctl -n hw.ncpu)
# → project/addons/terrain_3d/bin/libterrain.web.debug.wasm32.wasm
```

Flags worth knowing:

- `threads=no` — must match the Godot export preset's `variant/thread_support` value.
  Single-threaded is more compatible across browsers/iOS.
- `debug_symbols=no` — keeps the wasm under ~2.5 MB; debug build still has GDScript-side
  errors with stack traces, just no C++ symbols.

## Wiring a local build into a downstream Godot project

Terrain3D ships precompiled binaries inside its addon at
`project/addons/terrain_3d/bin/`. A downstream project copies (or vendors) that whole
folder into its own `addons/`. To iterate quickly without copying every build:

```bash
cd <downstream-project>/addons/terrain_3d/bin/libterrain.macos.debug.framework
mv libterrain.macos.debug libterrain.macos.debug.shipped   # back up the shipped one
ln -s /path/to/Terrain3D/project/addons/terrain_3d/bin/libterrain.macos.debug.framework/libterrain.macos.debug \
      libterrain.macos.debug
```

Same idea for `libterrain.web.debug.wasm32.wasm`. Now `scons` rebuilds and the
downstream project picks up the change on next launch (no copy).

## Keeping addon GDScript in sync with the binary

The `addons/terrain_3d/` shipped with a release contains both the C++ binary **and**
a snapshot of the GDScript editor plugin (asset dock, menus, importer, etc.). The
GDScript calls into the C++ bound API.

If you build a newer Terrain3D `main` (1.1-dev) but leave the v1.0.1 GDScript in place,
you will see errors like:

```
SCRIPT ERROR: Parse Error: Invalid argument for "pack_image()" function:
              argument 3 should be "Image" but is "bool".
              at: GDScript::reload (res://addons/terrain_3d/menu/channel_packer.gd:451)
SCRIPT ERROR: Compile Error: Failed to compile depended scripts.
```

This is **API drift**, not a shader bug. Sync the GDScript to the same revision as
the binary:

```bash
rsync -av --delete --exclude='bin/' \
  /path/to/Terrain3D/project/addons/terrain_3d/ \
  <downstream-project>/addons/terrain_3d/
```

`--exclude='bin/'` preserves the symlinks/binaries set up in the previous step.
