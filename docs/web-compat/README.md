# Terrain3D — Web / Compatibility renderer notes

Working notes from a debugging + patching session focused on getting Terrain3D to render
on the **Compatibility (OpenGL ES 3.0)** renderer, both on macOS native and on Web (WebGL2).

These docs are scoped to the fork at `git@github.com:leanmendoza/Terrain3D.git`. They
are **not** part of upstream Terrain3D documentation under `doc/` — keep them separate.

## Files

- [setup-and-build.md](setup-and-build.md) — toolchain (Xcode, scons, python, emsdk),
  godot-cpp pinning, scons commands for macOS and Web.
- [web-export-and-test.md](web-export-and-test.md) — Godot CLI export, `serve.py` with
  COOP/COEP headers, Chrome DevTools MCP-driven testing loop.
- [uniform-block-fix.md](uniform-block-fix.md) — the actual bug, root cause analysis,
  and the two solution paths (this branch implements Option A).
- [troubleshooting.md](troubleshooting.md) — symptoms we hit, what they meant, what
  fixed them.

## TL;DR

- On Mac + Compatibility, upstream `main` already renders correctly once the addon
  binary matches the addon GDScript version — i.e. don't mix the v1.0.1 release
  binary with newer GDScript or vice versa.
- On Web + Compatibility, **two independent issues** had to be resolved:
  1. **emscripten ABI**: the extension `.wasm` must be compiled with the **same**
     emsdk that the target Godot's web template was compiled with. Godot 4.6.2 uses
     **emsdk 4.0.20**. Building Terrain3D with emsdk 3.1.64 (the value pinned in
     Terrain3D's CI for Godot 4.4) against a 4.6.2 template caused
     `Uncaught RuntimeError: memory access out of bounds` on first scene load.
  2. **WebGL2 uniform-block size**: WebGL2 only guarantees `GL_MAX_UNIFORM_BLOCK_SIZE
     = 16384 bytes`. The Terrain3D vertex `MaterialUniforms` block holds two arrays
     (`_region_map[1024]` and `_region_locations[1024]`) which alone consume ~32 KB
     under `std140` packing, so the link fails with
     `Size of uniform block MaterialUniforms in VERTEX shader exceeds GL_MAX_UNIFORM_BLOCK_SIZE`.
     Reducing the region grid to 16×16 (256 entries each) drops the contribution to
     ~9 KB and the shader links. This branch (`web-compat-fix`) makes that change
     **renderer-conditional** so Forward+/Mobile keep the larger 32×32 grid.

## Branches

- `web-compat-fix` (this) — Option A: smaller region map on Compatibility only.
- `web-compat-fix-texture-buffer` — Option B (WIP): pack region data into a sampler2D
  so the UBO size is independent of the region count.

## Verified configuration

- macOS 14.6 / Apple Silicon
- Xcode 17 (Apple clang 17.0.0)
- Python 3.14, scons 4.10
- Godot 4.6.2 stable (Compatibility renderer)
- emsdk 4.0.20 (matches Godot 4.6.2 web template)
- Chrome (latest) via Chrome DevTools MCP for headless verification

## Test project

The reproduction was done against a private Godot project (`Trafkin`, a game jam entry).
The relevant invariants for any test project:

- `project.godot` declares `config/features=PackedStringArray("4.6", "GL Compatibility")`
- A scene contains a `Terrain3D` node with at least one populated region
- An export preset of type `Web` with `variant/extensions_support=true` and
  `variant/thread_support=false`

The Terrain3D demo project at `project/Demo.tscn` is a fine substitute.
