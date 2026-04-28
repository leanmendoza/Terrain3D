# Option B — Pack the region map into a sampler2D

Status: **design only**. Not implemented on this branch.

## Goal

Eliminate the WebGL2 `MaterialUniforms` overflow without restricting the world
size on Compatibility. Where Option A (other branch) reduces the region grid
from 32×32 to 16×16 on Compatibility, Option B keeps any grid size by moving
the lookup tables out of the uniform block and into a small 2D texture that
the shader samples.

## Background

See [uniform-block-fix.md](uniform-block-fix.md) for why
`MaterialUniforms` overflows on WebGL2. The two offending uniforms are:

```glsl
uniform int  _region_map[1024];        // 16384 bytes under std140
uniform vec2 _region_locations[1024];  // 16384 bytes under std140
```

Together they consume ~32 KB inside a block whose WebGL2-mandated maximum is
16 KB. Reducing the count helps; moving them out of the UBO entirely is the
clean fix.

## Approach

Pack the same data into one or two small textures:

- **`_region_map_tex`** — `R32I` (or `R8` if region_id < 256), size
  `region_map_size × region_map_size` (32×32 by default = 1024 texels). Each
  texel holds the 0-or-1-based region_id.

- **`_region_locations_tex`** — `RG32I` (or `RG16I`), size 1×N where N is the
  active region count, holding `(loc.x, loc.y)`. Or if 32×32 fits the same
  storage envelope, use a fixed 32×32 texture and look up by region_id.

Either way the per-frame texture upload is at most a few KB and replaces UBO
pressure with a couple of `texelFetch` calls per fragment.

## Shader changes (sketch)

```glsl
// Replace these:
//   uniform int  _region_map[1024];
//   uniform vec2 _region_locations[1024];
// With:
uniform highp isampler2D _region_map_tex;        // R32I, size = (DIM, DIM)
uniform highp isampler2D _region_locations_tex;  // RG32I, size = (count, 1)

// Helper (replaces direct array reads):
int region_id_at(ivec2 pos) {
    return texelFetch(_region_map_tex, pos, 0).r;
}

ivec2 region_loc_at(int region_id) {
    // region_id is 1-based; convert to 0-based texture coord
    return texelFetch(_region_locations_tex, ivec2(region_id - 1, 0), 0).rg;
}
```

Then `get_index_coord`/`get_index_uv` read via these helpers instead of
indexing the UBO arrays.

## C++ side

New owned RIDs in `Terrain3DMaterial` (or `Terrain3DData`):

- `_region_map_tex_rid: RID` — created once via
  `RS->texture_2d_create(image)` with `Image::FORMAT_R32I` (or `R8` if range
  allows).
- `_region_locations_tex_rid: RID` — same idea.

Refresh path (where `_update_uniforms` currently calls
`material_set_param(p_material, "_region_map", region_map)`):

1. Build a `Ref<Image>` of the right format from the existing
   `PackedInt32Array region_map`.
2. `RS->texture_replace(_region_map_tex_rid, RS->texture_2d_create(image))`
   (or `texture_set_data` equivalent).
3. `RS->material_set_param(p_material, "_region_map_tex", _region_map_tex_rid)`.

Same flow for `_region_locations`.

## Things to check while implementing

- **Integer sampler support on WebGL2**: WebGL2 supports `isampler2D` with
  formats like `R32I`, `RG32I`, `R8I`, `RG8I`. Verify the chosen format works
  on real browsers. The shader-on-web prior art (Westhoff's blog) ran into
  Godot not exposing `usampler2D` on its web template — `isampler2D` may need
  the same workaround using `floatBitsToInt()` / `intBitsToFloat()` and a
  `RG32F` storage texture as a workaround.
- **Texture creation timing**: `RenderingServer` must be ready when the
  textures are first allocated. Lazy-create on first `_update_uniforms` call,
  same place we currently push the array uniforms.
- **Lifecycle**: free RIDs in `~Terrain3DMaterial` (or wherever the existing
  generated textures are freed — see `_generated_height_maps` etc.).
- **Renderer-conditional or unconditional?** Keeping the texture lookup on
  Forward+/Mobile too is simpler (no `#if`) but costs a `texelFetch` per
  fragment that those backends don't need today. Prior art in Terrain3D
  (heightmaps, color maps) is already texture-based, so this is consistent.

## Files to touch

- `src/shaders/main.glsl` — uniforms + `get_index_coord` + `get_index_uv` +
  any other shader that reads `_region_map` / `_region_locations` (also check
  `displacement_buffer.glsl`, `gpu_depth.glsl`).
- `src/terrain_3d_material.h/.cpp` — add `_region_map_tex` and
  `_region_locations_tex` members; update `_update_uniforms`; free in
  destructor.
- `src/terrain_3d_data.h/.cpp` — `get_region_map_size()` can stay at the old
  static constant (32×32) since the UBO is no longer the bottleneck.
- `docs/web-compat/uniform-block-fix.md` — link to this doc and clarify which
  branch implements which option.

## Out of scope here

The other `[32]`-sized uniforms (`_texture_*_array`) also live in
`MaterialUniforms` but their combined footprint is small (~3 KB). They do not
need to move unless we ever add a 4th 32-element array.

## Testing

Same loop as Option A — build mac + web, export, serve, open in browser,
inspect console for shader errors, take a screenshot of the terrain. The
expected result: terrain renders on Compatibility with the **default 32×32
grid** intact, and no `GL_MAX_UNIFORM_BLOCK_SIZE` errors.
