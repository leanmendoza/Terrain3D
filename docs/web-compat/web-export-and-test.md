# Web export and test loop

## Web export from the CLI

```bash
"/Applications/Godot 4.6.2.app/Contents/MacOS/Godot" \
  --headless \
  --path <downstream-project> \
  --export-debug "Web" <downstream-project>/build/index.html
```

`"Web"` is the preset name from `export_presets.cfg`. The path argument is the output
HTML; Godot writes `index.html`, `index.wasm`, `index.side.wasm`, `index.pck`, and
copies any GDExtension wasm referenced by `.gdextension` files (e.g.
`libterrain.web.debug.wasm32.wasm`).

The export will fail with a signal 11 backtrace through `terrain_3d_init` if either:

- the GDExtension binary is from a different emscripten than the host Godot
  (in which case the editor itself loads it for module discovery and crashes), or
- the build tree is in a half-clean state (SCons sometimes leaves stale `.o`
  files when toggling header constants — when in doubt, `scons --clean` before
  rebuilding).

## Static server with cross-origin isolation

Browsers refuse to load Godot's web export from a server that doesn't send
`Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy:
require-corp`. SharedArrayBuffer (used by Godot's threading) requires this even
when the export is single-threaded — better to have the headers either way.

`serve.py` (place at the root of the downstream project, alongside `build/`):

```python
import http.server
import socketserver
import sys
from pathlib import Path

PORT = int(sys.argv[1]) if len(sys.argv) > 1 else 8000
DIRECTORY = Path(__file__).parent / "build"


class COIHandler(http.server.SimpleHTTPRequestHandler):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, directory=str(DIRECTORY), **kwargs)

    def end_headers(self):
        self.send_header("Cross-Origin-Opener-Policy", "same-origin")
        self.send_header("Cross-Origin-Embedder-Policy", "require-corp")
        self.send_header("Cross-Origin-Resource-Policy", "same-origin")
        self.send_header("Cache-Control", "no-store")
        super().end_headers()


class ReusableTCPServer(socketserver.ThreadingTCPServer):
    allow_reuse_address = True
    daemon_threads = True


with ReusableTCPServer(("127.0.0.1", PORT), COIHandler) as httpd:
    print(f"Serving {DIRECTORY} at http://127.0.0.1:{PORT} (cross-origin isolated)")
    httpd.serve_forever()
```

Run with `python3 serve.py 8000`. `Cache-Control: no-store` matters when iterating
on builds — Chrome will otherwise serve a stale `.wasm`.

## Editor smoke test for the Compatibility renderer

A minimal test harness for the **macOS native** Compatibility path (used to verify
that nothing regressed locally before exporting for Web):

```bash
#!/usr/bin/env bash
# tools/test.sh
set -euo pipefail

DRIVER="opengl3"   # opengl3 = Compatibility, vulkan = Forward+
MODE="edit"        # edit | run | smoke
SCENE="res://scenes/forest.tscn"   # any scene with a Terrain3D node
GODOT="/Applications/Godot 4.6.2.app/Contents/MacOS/Godot"
PROJECT="$(cd "$(dirname "$0")/.." && pwd)/godot"
LOGDIR="$(cd "$(dirname "$0")/.." && pwd)/logs"
mkdir -p "$LOGDIR"

# parse simple flags ...
# (full version in fictional-succotash/tools/test.sh)

ARGS=(--path "$PROJECT" --rendering-driver "$DRIVER" --verbose)
[[ "$MODE" == "edit" ]] && ARGS+=(--editor "$SCENE")
[[ "$MODE" == "smoke" ]] && ARGS+=(--editor "$SCENE")

LOG="$LOGDIR/${MODE}-${DRIVER}-$(date +%Y%m%d-%H%M%S).log"

if [[ "$MODE" == "smoke" ]]; then
  ( "$GODOT" "${ARGS[@]}" 2>&1 | tee "$LOG" ) &
  PID=$!; sleep 12
  pkill -TERM -P "$PID" 2>/dev/null || true; kill -TERM "$PID" 2>/dev/null || true
  wait "$PID" 2>/dev/null || true
else
  "$GODOT" "${ARGS[@]}" 2>&1 | tee "$LOG"
fi
```

`--rendering-driver opengl3` forces the Compatibility codepath even if `project.godot`
declares Forward+ — useful for A/B comparison. `--verbose` produces the shader and
extension load trace.

## Browser test loop with Chrome DevTools MCP

The full headless verification cycle used during this debugging:

1. Build and export.
2. Reload the served page (`mcp__chrome-devtools__navigate_page` type=`reload`,
   `ignoreCache: true`).
3. Wait for `Build configuration` (the Godot init log line) to appear in the
   console.
4. Take a screenshot to see the menu.
5. Send the click as a focused-element keypress: the canvas is the only DOM
   element, so dispatching `Enter` on the focused canvas triggers Godot's default
   button handler.
6. Capture `list_console_messages` filtered by `error`.

Things to grep the console for:

- `Build configuration: Emscripten X.Y.Z` — confirms ABI match with your build.
- `Uncaught RuntimeError: memory access out of bounds` — emscripten ABI mismatch
  or genuine wasm memory bug.
- `Size of uniform block ... exceeds GL_MAX_UNIFORM_BLOCK_SIZE (16384)` — the
  uniform block size issue addressed by this branch.
- `Program linking failed` / `_compile_specialization` / `_version_bind_shader` —
  any of Godot's GLES3 pipeline failures.
- `WARNING: Terrain3DTextureAsset#... has no mipmaps` — benign asset-side warning,
  ignore.

The screenshot from a healthy run should show the terrain mesh with splat
textures applied at expected camera height; an unhealthy run typically shows the
multi-mesh foliage and trees floating without ground beneath them.
