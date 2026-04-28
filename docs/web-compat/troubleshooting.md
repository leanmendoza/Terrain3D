# Troubleshooting log

Symptoms hit during this session and what each one meant.

## `SCRIPT ERROR: Invalid argument for "pack_image()" function: argument 3 should be "Image" but is "bool"`

**Cause**: GDScript bundled with `addons/terrain_3d/` does not match the C++
binary's bound API. Typically caused by upgrading the binary (or building from
`main`) without syncing the GDScript side.

**Fix**: rsync the addon GDScripts from the matching Terrain3D revision into
the downstream project's `addons/terrain_3d/`, excluding `bin/` so the locally
built binary stays. See [setup-and-build.md](setup-and-build.md#keeping-addon-gdscript-in-sync-with-the-binary).

## `SCRIPT ERROR: Invalid call. Nonexistent function 'get_texture' in base 'Terrain3DAssets'`

**Cause**: same as above — API drift. `get_texture` was renamed/removed in a
newer revision but the editor plugin GDScript is still calling it.

## `Uncaught RuntimeError: memory access out of bounds` (in browser)

**Cause**: emscripten ABI mismatch between the Terrain3D `.wasm` and Godot's
web template. Crashes typically don't happen at extension-init time; they
surface deep into scene loading, which makes this misleading.

**Fix**: rebuild the extension with the emsdk version that matches the Godot
release. Read `Build configuration: Emscripten X.Y.Z` from the browser console
to find the value.

## Same backtrace through `terrain_3d_init` on Mac, repeating

```
[2] terrain_3d_init + 1164236
[3] terrain_3d_init + 1164236
[4] terrain_3d_init + 1145384
[5] terrain_3d_init + 1108860
[6] terrain_3d_init + 84
```

**Cause**: SCons incremental build leaving stale `.o` files when constants in a
header are toggled (e.g. flipping `REGION_MAP_SIZE` between 16 and 32). The
linked binary mixes object files compiled against inconsistent layouts.

**Fix**:
```bash
scons --clean platform=macos arch=arm64 target=template_debug
rm -rf project/addons/terrain_3d/bin/libterrain.macos.debug.framework
scons platform=macos arch=arm64 target=template_debug -j$(sysctl -n hw.ncpu)
```

A regular incremental build after the clean is fine.

## `Size of uniform block MaterialUniforms in VERTEX shader exceeds GL_MAX_UNIFORM_BLOCK_SIZE (16384)`

**Cause**: the Terrain3D shader's region arrays (`_region_map[1024]`,
`_region_locations[1024]`) push the WebGL2 vertex UBO past the spec-mandated
16 KB minimum. See [uniform-block-fix.md](uniform-block-fix.md).

**Fix**: this branch (Option A) — region grid is 16×16 on Compatibility, 32×32
elsewhere.

## Terrain still invisible after the link succeeds

**Cause**: the shader array dimension must match `Terrain3DData::REGION_MAP_SIZE`
exactly. If C++ writes to index `528` (region (0,0) under a 32×32 grid) but the
shader array only has 256 slots, the data lives at indices the shader never
reads. The compile is clean, the terrain is invisible.

**Fix**: keep the two values in lockstep. Option A does this via the renderer
check; Option B sidesteps it entirely by using a texture lookup.

## `WARNING: Terrain3DTextureAsset#XXXX:set_albedo_texture: Albedo texture 'NAME' has no mipmaps`

**Cause**: import setting on the texture in the downstream project. Not a
Terrain3D bug.

**Fix**: open the texture in Godot's Import panel, enable Mipmaps, reimport.

## `WARNING: Terrain3DAssets#XXXX:_update_texture_files: Texture ID N normal is saved in the scene. Save it as a file and link it.`

**Cause**: a texture asset is embedded in the scene/resource instead of being a
separate file. Functional warning — the texture works but takes up scene
memory.

**Fix**: in the Terrain3D asset dock, save the embedded texture to its own
`.tres` and re-link.

## Browser served a stale `.wasm` after rebuild

**Cause**: missing `Cache-Control: no-store` on the dev server, or Chrome's
disk cache.

**Fix**: `serve.py` here already sets `Cache-Control: no-store`. Use
`navigate_page` with `ignoreCache: true` (or DevTools Network → Disable cache)
to be sure.

## Headless export crashes in `terrain_3d_init`

If `godot --headless --export-debug Web ...` crashes with the same
`terrain_3d_init` backtrace as the editor smoke test, the cause is the same
stale-build issue. Clean and rebuild the **macOS** binary (the host running the
export uses it for module discovery), not just the web one.
