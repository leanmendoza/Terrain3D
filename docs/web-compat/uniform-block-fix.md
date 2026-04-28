# WebGL2 uniform block size — analysis and fix

## Symptom

When loading a scene that contains a `Terrain3D` node in a Godot Web export
(Compatibility / WebGL2), the browser console emits:

```
ERROR: Condition "p_ubo_size > uint32_t(Config::get_singleton()->max_uniform_buffer_size)" is true.
       at: update_parameters_internal (drivers/gles3/storage/material_storage.cpp:1096)
ERROR: SceneShaderGLES3: Program linking failed:
Size of uniform block MaterialUniforms in VERTEX shader exceeds GL_MAX_UNIFORM_BLOCK_SIZE (16384)
       at: _display_error_with_code (drivers/gles3/shader_gles3.cpp:259)
ERROR: Method/function failed.
       at: _compile_specialization (drivers/gles3/shader_gles3.cpp:461)
WARNING: shader failed to compile, unable to bind shader.
       at: _version_bind_shader (./drivers/gles3/shader_gles3.h:217)
```

The terrain mesh does not render. Trees, grass, and other non-Terrain3D meshes
render normally because they use materials with much smaller uniform blocks.

The same symptom does not appear in Forward+/Vulkan (Mac/Windows/Linux desktop)
or Mobile, because those backends allow much larger UBOs (typically 64 KB or
more on modern GPUs).

## Why 16384

The WebGL2 spec only requires `GL_MAX_UNIFORM_BLOCK_SIZE = 16 KiB`. Many drivers
on desktop will report a larger value, but **iOS and a number of Android/Intel
WebGL2 implementations cap it at exactly 16384**. Godot's GLES3 backend treats
this as a hard limit and refuses to link.

## Why `MaterialUniforms` overflows

Godot's GLES3 backend places **all** of a spatial shader's `uniform` declarations
into a single block named `MaterialUniforms`, laid out per `std140`.

`std140` rules that bite here:

- A scalar (`int`, `float`) inside an array occupies a **full 16-byte slot** —
  not 4 bytes.
- A `vec2` inside an array also occupies a full 16-byte slot.
- A `vec4` inside an array occupies 16 bytes (no waste).

The Terrain3D main shader (`src/shaders/main.glsl`) declares, among other things:

```glsl
uniform int  _region_map[1024];        // 1024 * 16 = 16384 bytes
uniform vec2 _region_locations[1024];  // 1024 * 16 = 16384 bytes
uniform float _texture_normal_depth_array[32];   // 32 * 16 = 512
uniform float _texture_ao_strength_array[32];    // 32 * 16 = 512
uniform float _texture_ao_affect_array[32];      // 32 * 16 = 512
uniform float _texture_roughness_mod_array[32];  // 32 * 16 = 512
uniform float _texture_uv_scale_array[32];       // 32 * 16 = 512
uniform vec2  _texture_detile_array[32];         // 32 * 16 = 512
uniform vec4  _texture_color_array[32];          // 32 * 16 = 512
```

Just the two region arrays sum to **32 KB**, twice the WebGL2 minimum, before
counting any of the per-texture arrays or the dozen other scalar uniforms.

## What the region arrays mean

`Terrain3DData::REGION_MAP_SIZE = 32` declares a 32×32 grid of region cells
centered on the world origin (cells span `[-16, +15]` on each axis). Each cell
is `region_size` units across — by default 1024 — so a 32×32 grid covers a
32 × 1024 = **32 km square**.

- `_region_map[1024]` is the flat 32×32 grid: at index `(y+16)*32 + (x+16)` it
  stores either `0` (empty) or a 1-based index into `_region_locations`.
- `_region_locations[1024]` is the list of populated cell coordinates as
  `vec2` (one per active region; unused slots stay at the array's default).

The shader `vertex()` and helper functions read these uniforms to translate world
coordinates into the right slice of the region's height/control/color
`sampler2DArray`s.

## Two solution paths

### Option A — smaller grid on Compatibility (this branch)

If the playable area fits in a 16×16 region grid (16 × 1024 = 16 km square,
which is plenty for almost any Godot game), the arrays drop from 1024 to 256
entries each: 256 * 16 = 4 KB per array, 8 KB total. The whole `MaterialUniforms`
block then comfortably fits below 16 KB and the shader links.

The change must be applied in **both** the C++ side and the shader, and they
must agree on the dimension because the index math is `(y + dim/2) * dim + (x + dim/2)`.
A mismatch (e.g. C++ at 32 with shader at 16) silently sends region data to
indices the shader will never read, and the terrain stays invisible — which is
exactly what we observed during diagnosis.

To keep desktop renderers (Forward+, Mobile) at 32×32 and only restrict the
Compatibility renderer, the change is gated:

- **Shader**: `#if CURRENT_RENDERER == RENDERER_COMPATIBILITY` selects 256-entry
  arrays and `_region_map_size = 16`.
- **C++**: `Terrain3DData::REGION_MAP_SIZE` becomes a runtime value that
  inspects `RenderingServer::get_rendering_device()`. When that returns
  `nullptr` the active backend is Compatibility (RenderingDevice is only
  created by Vulkan/D3D12), and we use 16; otherwise 32.

Tradeoff: a project that needs more than a 16 × `region_size` square cannot
target Web/Compatibility. For most games this is non-binding.

### Option B — pack region data into a sampler2D (separate branch)

The 32×32 region map is fundamentally a small 2D lookup table. A `sampler2D` of
size 32×32 RGBA32F (or RGBA32UI) holds the same data without consuming any UBO
budget. The shader replaces array reads with `texelFetch(_region_map_tex, ...)`
calls.

Pros: keeps 32×32 (or any size) on every renderer; fixes the issue at the root
without conditional code.

Cons: more invasive. Requires a new `Terrain3DRegionMapTexture` (or similar)
plumbed from `Terrain3DData::_update_region_map` through `Terrain3DMaterial` to
the shader, plus shader changes in every place that reads `_region_map` or
`_region_locations`.

Explored on the `web-compat-fix-texture-buffer` branch.

## Diagnosis steps for future regressions

If the terrain disappears on Compatibility on a new platform / browser / driver:

1. Browser console — look for `MaterialUniforms`, `GL_MAX_UNIFORM_BLOCK_SIZE`,
   `Program linking failed`. If present, the limit is the issue — find the
   reported limit and compare to the static `std140` size of `MaterialUniforms`.
2. `webgl-report.com` or `chrome://gpu` will show the actual
   `GL_MAX_UNIFORM_BLOCK_SIZE` for the current GPU/driver.
3. If the link succeeds but the terrain is still missing, check for a silent
   index mismatch: the C++ `REGION_MAP_SIZE` and the shader array dimension must
   be the same value — a mismatch produces a clean compile and an invisible
   terrain.
