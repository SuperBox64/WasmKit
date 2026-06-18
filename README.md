# WasmKit

**The web runtime.** A hand-rolled, no-Emscripten JavaScript host that renders WASI / Embedded-Swift wasm games on Canvas2D and fulfils the KitABI `env` imports in the browser.

Run a game in the browser as WebAssembly, without Emscripten. `runtime.js` drives the game loop, renders on Canvas2D (display-p3 wide color on supporting browsers, plus a hidden WebGL2 canvas for GLSL shader passes), mixes audio on the Web Audio API, handles input from DOM events and the Web Gamepad API, speaks via the Web Speech API, and persists state to `localStorage`. No watermarks. No loading screens. No third-party branding. Nothing injected between the game and the player.

**Live demo:** [boss-man.us/play](https://boss-man.us/play)

---

## What Is This

WasmKit is the **browser half** of the SuperBox64 / Boss-Man game stack. Its `runtime.js` loads a `wasm32-wasi` (or Embedded Swift) *reactor* module, supplies the WASI Preview 1 syscalls the binary references, and implements the `include/abi.h` `env` contract on top of Canvas2D, Web Audio, the DOM, the Web Gamepad API, Web Speech, and `localStorage`.

The point of the constellation is **one game source, many targets**. A single SpriteKit codebase (see [UFO-Emoji-Arcade](https://github.com/SuperBox64/UFO-Emoji-Arcade)) compiles three ways — Apple-native, full-Swift wasm, and Embedded-Swift wasm — and the *same* wasm a website serves with this runtime also:

- plays untouched in the [WasmCart](https://github.com/SuperBox64/WasmCart) native console (SDL3 + WAMR), and
- runs as a single-file native binary via [SuperBox64Kit](https://github.com/SuperBox64/SuperBox64Kit)'s SDL3 backend.

`runtime.js` is the **reference behavior** that the native hosts re-implement. There is no Emscripten anywhere in the pipeline: the runtime hand-rolls the WASI Preview 1 syscalls and the entire `env` ABI itself.

### The drop-in story (where WasmKit fits)

The wasm modules WasmKit hosts are produced by [SuperBox64Kit](https://github.com/SuperBox64/SuperBox64Kit) — a Swift reimplementation of Apple's SpriteKit (Box2D physics, an SDL3 backend, a reSVG/nanosvg vector rasterizer) that lets an existing SpriteKit game compile to wasm unchanged. WasmKit owns the *browser side*: it does not emulate SpriteKit; it answers the **KitABI** `env` imports those compiled games call. A header-only **SFML 2.6 shim** (`include/SFML/`) is also vendored here so existing C++ SFML games compile against the same ABI without an external SFML dependency.

---

## Where WasmKit sits in the 5-repo constellation

```
                          ONE game source (SpriteKit)
                          UFO-Emoji-Arcade  ── the flagship demo
                                   │
                 ┌─────────────────┼──────────────────┐
                 │                 │                  │
        Apple-native        full-Swift wasm     Embedded-Swift wasm
        (Xcode/Metal)       wasm32-wasip1        wasm32 (~688 KB)
                 │                 │                  │
                 │                 └────────┬─────────┘
                 │                          │  compiled by
                 │                  ┌────────┴─────────┐
                 │                  │  SuperBox64Kit   │  THE KIT
                 │                  │  SpriteKit + KitABI
                 │                  │  Box2D · SDL3 · reSVG
                 │                  └────────┬─────────┘
                 │                           │ the .wasm cartridge
        ┌────────┴────────┐   ┌──────────────┼───────────────┬────────────────┐
        │  Apple          │   │              │               │                │
        │  SpriteKit      │ ╔═╧════════════╗ │        ┌───────┴──────┐ ┌───────┴───────┐
        │  Metal+UIKit    │ ║  WasmKit     ║ │        │  WasmCart    │ │  Wasm5        │
        └─────────────────┘ ║  runtime.js  ║ │        │  WAMR + AOT  │ │  WKWebView    │
                            ║  Canvas2D    ║◀┘        │  SDL3+Metal  │ │  (this runtime│
                            ║  (browser)   ║          └──────────────┘ │  in a webview)│
                            ╚══════════════╝                          └───────────────┘
                              YOU ARE HERE                  Wasm5 serves THIS runtime.js
                                                            in a WKWebView over localhost
```

| Repo | Role |
|---|---|
| **WasmKit** (this) | **The web runtime** — `runtime.js` + Canvas2D; fulfils KitABI in the browser |
| [SuperBox64Kit](https://github.com/SuperBox64/SuperBox64Kit) | **The kit** — drop-in SpriteKit, KitABI, Box2D, SDL3, reSVG |
| [WasmCart](https://github.com/SuperBox64/WasmCart) | **Native console** — SDL3 shell running carts through WAMR (interpreter + wamrc AOT) |
| [Wasm5](https://github.com/SuperBox64/Wasm5) | **WebView console** — WasmCart's shell, but carts play in a WKWebView running *this* `runtime.js` |
| [UFO-Emoji-Arcade](https://github.com/SuperBox64/UFO-Emoji-Arcade) | **The flagship game/demo** — one Swift source shipped to every target |

---

## Features

- **No-Emscripten wasm loader.** `WebAssembly.instantiateStreaming` with a `fetch` → `arrayBuffer` → `instantiate` fallback for servers that omit `Content-Type: application/wasm`. Supplies WASI Preview 1 imports: `fd_write`/`fd_read`/`fd_close`/`fd_seek`/`fd_readdir`, `environ`/`args` sizes+get, `clock_time_get`, `random_get`, `proc_exit`, `poll_oneoff`.
- **Reactor contract.** Instantiates a module exporting exactly `_initialize`, `boot()`, and `frame(dtMs: f64)` — works identically for C, C++, full-Swift, and Embedded-Swift reactors. Calls `_initialize()` then `boot()` after asset preload, then `frame(dt)` on every `requestAnimationFrame`.
- **Canvas2D renderer (full `env` ABI).** `clear`/`save`/`restore`/`translate`/`scale`/`rotate`, alpha, fill/stroke rect+circle+poly, textured-quad `gfx_draw_image` (src-crop + RGBA tint/alpha), per-target transform/blend save-restore stack, and offscreen render targets (`rt_create`/`rt_image`, `gfx_offscreen_begin`/`end_to_image`/`discard`).
- **display-p3 wide color.** Negotiates a `display-p3` Canvas2D context where supported, emits `color(display-p3 …)` CSS, and falls back to `rgba()` otherwise — no game-side changes needed.
- **Text + emoji.** Canvas2D `fillText` with `txt_width` measurement, `gfx_draw_text` letter-spacing, selectable `textBaseline` (alphabetic/middle/top/bottom), a DPR-aware glyph cache (cap 2048) for crisp+fast emoji, and a curated default-emoji set that appends `U+FE0F` so bare symbols (mountain, tent, …) render in color, not B&W.
- **Blend modes & effects.** Mapped to `globalCompositeOperation` (`add`→`lighter`, `multiply`, `screen`, `alpha`→`source-over`), plus `gfx_set_composite`, `gfx_set_filter`/`clear_filter`, `gfx_set_shadow`/`clear_shadow`, `gfx_draw_shadow_image`, and RGB particle tint on both raster and SVG paths.
- **Resolution-aware SVG.** SVG assets are kept as live `<img>` and rasterized at draw time (`loadSVG`/`drawSVG`/`svgRaster`) with a quantized raster cache to avoid per-frame re-rasterization of scaling sprites. Raster images decode to `ImageBitmap`.
- **WebGL2 GLSL effects.** A hidden `webgl2` canvas (`premultipliedAlpha:false`) backs `gfx_shader_compile`/`release`/`set_uniform_f`/`set_uniform_t`/`draw`, plus `gfx_lighting_draw` (normal-map lighting) and `gfx_warp_draw` (grid-mesh warp), blitted back into the 2D scene.
- **Web Audio mixer.** `snd_by_name`/`from_samples`/`create_pcm` (Float32 PCM for procedural music), `snd_play` (voice handles, volume, loop), `snd_stop`/`set_volume`/`set_pan`/`set_rate`/`status`, `snd_pause_all`/`resume_all`; plus an AVAudioEngine-shaped `eng_*` graph (`player_create`/`release`, `mixer_create`, `node_set_volume`/`pan`, `connect`, `player_schedule_buffer`/`play`/`stop`, `eng_start`/`stop`).
- **Text-to-speech.** Web Speech API `tts_speak`/`cancel` with rate/pitch/volume, preferred/robotic/female voice selection, speech priming, and audio ducking. **TTS is disabled on iOS** (Web Speech only fires in a user gesture there).
- **Input.** Keyboard (`key_pressed` against the SFML 2.6 keycode table), mouse (button/x/y in logical coords), multi-touch (`TouchBegan`/`Moved`/`Ended` in SFML 2.6 event order alongside finger-0 mouse), and an `evt_poll` event queue.
- **Web Gamepad / USB arcade stick.** Up to 4 pads on the W3C Standard layout: `gp_connected`/`button`/`button_value`/`axis`, plus `gp_map_to_keys` to synthesize arrow + Space key events so keyboard games "just work" on a controller.
- **Asset preloader.** Driven by `manifest.json`: fonts via `FontFace` (fetched as **bytes** so `file://` works), images via `ImageBitmap`/SVG, sounds via `AudioContext.decodeAudioData`, JSON texts as strings exposed through `asset_text`. Each asset is registered under its full path, an `assets/`-prefixed path, and its bare basename.
- **Cache-busting.** `CFG.cacheBust` is appended as `?v=…` to the wasm and every asset fetch to defeat stale browser caches.
- **Collision polygons.** `img_polygon_from_alpha` runs Ramer–Douglas–Peucker (`rdpSimplify`) alpha-outline tracing to hand the wasm a simplified collision polygon.
- **Persistence & window control.** `store_get`/`store_set` over `localStorage`; `win_set_title`/`width`/`height`/`request_fullscreen`/`exit_fullscreen` (with a pseudo-fullscreen fallback) and `win_download` via Blob + anchor.
- **Lifecycle.** Background tabs pause audio and the loop. A live FPS + draw-call + per-frame ms HUD overlay (img/txt/rest split, min/max/avg frame, glyph-cache + memory readout) is gated by `SKView.showsFPS` / `showsDrawCount` flags (off by default).
- **Offline bundling.** `scripts/bundle.py` produces a single `local.html` / `bundle.js` with every asset inlined as `data:` URLs, playable from `file:///`.

---

## What Is in This Repo

| Path | What it is |
|---|---|
| `runtime.js` | The entire web runtime (2746 lines) — one `class Runtime`: fetch+instantiate, `wasiImports()`, `envImports()`, asset preload, `_initialize()`→`boot()`, RAF `frame(dt)` loop. Auto-boots on `DOMContentLoaded` from `window.WASMWEB`. |
| `runtime-embedded.js` | Runtime variant reserved for Embedded Swift builds — **byte-identical** to `runtime.js` at this snapshot (same 2746 lines, same contract). Embedded tweaks land here first. |
| `runtime-embedded-min.js` | Terser-minified embedded runtime (~54 KB) that `boss-man.us` and the WebView apps ship. **Generated output committed to the repo** — see the note below. |
| `include/abi.h` | The C ABI header — the documented `import_module("env")` contract. Colors are packed `0xRRGGBBAA` `uint32`; coordinates are logical game pixels. |
| `include/SFML/` | Header-only SFML 2.6 compatibility shim (`Graphics/`, `Window/`, `System/`, `Audio/`) so existing C++ SFML games compile unchanged. |
| `include/WebStore.hpp` | Header for the `localStorage`-backed persistence helper (maps to `store_get`/`store_set`). |
| `src/sfml_web_impl.cpp` | Out-of-line `sf::` definitions (Color constants, `Transform::Identity`, BlendMode globals). Linked when `WASMWEB_SFML=on`. |
| `build.sh` | Build helper for C/C++ games via the WASI SDK (no Emscripten). Defines `wasmweb_build()` and `wasmweb_manifest()`. |
| `shell.html` | Minimal host page template: a `#game` canvas, a `window.WASMWEB` config block, and `<script src=runtime.js>`. |
| `scripts/bundle.py` | Offline bundler: inlines the wasm + all manifest assets as base64 `data:` URLs and ships a `fetch()` shim for `file:///` playback. |
| `example/web_main.cpp` | Toolchain-proof C++ reactor — proves wasm↔JS imports/exports, libc++/STL init, and a RAF Canvas2D loop. |
| `example/build-test.sh` | Builds the toolchain-proof module to `wasm32-wasi` (reactor) with WASI SDK `clang++ -O2`. |
| `example/swift-poc/` | Swift + Box2D(C++) proof-of-concept SwiftPM package (`swift-tools-version 6.0`) compiling a Swift reactor to wasm; `web/index.html` symlinks the top-level `runtime.js`. |
| `cartridge/wasm-cartridge` | Committed arm64 Mach-O executable (~188 KB) + `native-selftest.bmp` — artifacts of the WasmCart native-console cartridge model. |
| `LICENSE` / `NOTICE` | Apache License 2.0 + NOTICE (Copyright 2026 Todd Bruss). |

> **`runtime-embedded-min.js` is committed generated output. There is no minify script in this repo** — `build.sh` only compiles C/C++ wasm; the Terser minify step lives in a consumer repo. **Do not hand-edit the min file**; regenerate it from the consumer's build. (A past ~334-line hand-fork drift caused a long missing-glyph-cache / white-hole / font symptom-chase.)

---

## Build × Runtime Permutations

WasmKit is one host in a matrix where **one game source** runs across several build flavors and several hosts. The browser rows are the ones WasmKit owns (full table lives in [SuperBox64Kit](https://github.com/SuperBox64/SuperBox64Kit) / [UFO-Emoji-Arcade](https://github.com/SuperBox64/UFO-Emoji-Arcade)):

| Build flavor | Host (runtime) | Renderer | Notes |
|---|---|---|---|
| full-Swift `wasm32-wasip1` (SwiftPM) | **`runtime.js`** (this repo) | **Canvas2D** | The primary shippable web build (~4.18 MB for UFO Emoji). `index.html` wires `window.WASMWEB`. |
| Embedded-Swift `wasm32` (raw `swiftc -enable-experimental-feature Embedded`) | **`runtime-embedded-min.js`** | **Canvas2D** | ~688 KB raw, ~6× smaller. Same `runtime.js`, just terser-minified — **never** a hand fork. |
| full-Swift / Embedded `.wasm` cart | [WasmCart](https://github.com/SuperBox64/WasmCart) — WAMR interpreter | SDL3 + Metal | `\0asm` magic → interpreted; universal fallback that always works (~2.3 ms real work/frame). |
| full-Swift / Embedded cart → `.aot` | [WasmCart](https://github.com/SuperBox64/WasmCart) — wamrc AOT | SDL3 + Metal | `\0aot` magic; `--opt-level=0 --bounds-checks=1 --disable-simd`. Native code = 60 fps. |
| full-Swift / Embedded web build (unchanged) | [Wasm5](https://github.com/SuperBox64/Wasm5) — WKWebView | **Canvas2D in a WKWebView** | Serves this runtime over a localhost `http.server` inside a webview; SDL keystrokes forwarded as synthetic DOM `KeyboardEvent`s. |
| Embedded-Swift native (no wasm) | [SuperBox64Kit](https://github.com/SuperBox64/SuperBox64Kit) SDL3 backend | SDL3 + Metal | Single-file binary, assets baked in, no runtime deps. |
| Apple-native (no wasm) | Apple SpriteKit | Metal + UIKit | The original Xcode app — the source of truth. |

---

## Build & Run

### Host a game in a browser

Copy `shell.html`, configure `window.WASMWEB`, and serve `runtime.js` next to it:

```html
<!DOCTYPE html>
<html lang="en"><head>
  <meta charset="utf-8"/>
  <style>
    html,body{margin:0;background:#000;}
    /* canvas width/height and this aspect-ratio MUST match WASMWEB.logical*. */
    #game{width:100vw;aspect-ratio:1184/666;display:block;}
  </style>
</head><body>
  <canvas id="game" width="1184" height="666" tabindex="0"></canvas>
  <script>
    window.WASMWEB = {
      logicalWidth:  1184,     // canvas backing store = fixed logical size
      logicalHeight: 666,      // CSS scales it; mouse/touch arrive in logical px
      wasmUrl:       'game.wasm',
      assetRoot:     'assets',
      title:         'My Game',
      // cacheBust: Date.now()  // stamped per build to defeat stale caches
    };
  </script>
  <script src="runtime.js"></script>
</body></html>
```

Serve over HTTP (the runtime auto-boots on `DOMContentLoaded`):

```bash
python3 -m http.server   # then open the page
```

> `instantiateStreaming` hard-requires `Content-Type: application/wasm`; the runtime falls back to `fetch → arrayBuffer → instantiate` for servers that don't send it.

### Build a C/C++ game (WASI SDK, no Emscripten)

From your game's own build script:

```bash
WASMWEB_OUT=web/game.wasm
WASMWEB_SRC_DIRS=(src)
WASMWEB_INCLUDES=(include)
WASMWEB_SFML=on          # link the sf:: SFML 2.6 shim (off = raw ABI)
source ../WasmKit/build.sh
wasmweb_build
```

`build.sh` variables:

| Variable | Default | Description |
|---|---|---|
| `WASMWEB_OUT` | *required* | Output `.wasm` path |
| `WASMWEB_SRC_DIRS` | `(src)` | Dirs scanned for `*.cpp` / `*.c` |
| `WASMWEB_EXTRA_SRCS` | `()` | Explicit extra source files |
| `WASMWEB_INCLUDES` | `()` | Extra `-I` directories |
| `WASMWEB_DEFINES` | `()` | Extra `-D` defines |
| `WASMWEB_SFML` | `on` | `on` links the `sf::` shim; `off` = raw ABI |
| `WASMWEB_EXCEPTIONS` | `off` | `on` enables C++ exceptions (larger binary) |
| `WASMWEB_ASSETS` | | Assets dir to scan for `manifest.json` |

Under the hood `wasmweb_build` invokes (WASI SDK located via `$WASI_SDK`, default `$HOME/wasi-sdk`; it errors if `$WASI_SDK/bin/clang++` is missing):

```bash
$WASI_SDK/bin/clang++ --target=wasm32-wasip1 \
    --sysroot=$WASI_SDK/share/wasi-sysroot \
    -mexec-model=reactor -std=c++20 -Os -fno-rtti -fno-exceptions \
    <defines> <includes> <srcs> \
    -Wl,--allow-undefined -Wl,-z,stack-size=8388608 -Wl,--strip-all \
    -Wl,--export=_initialize -Wl,--export=boot \
    -Wl,--export=frame -Wl,--export=memory \
    -o web/game.wasm
```

Regenerate `manifest.json` from an assets tree with `wasmweb_manifest <assetsDir> <outFile>`.

### Toolchain-proof example

```bash
# example/build-test.sh — proves wasm<->JS imports/exports + Canvas2D RAF loop
$WASI_SDK/bin/clang++ --target=wasm32-wasi \
    --sysroot=$WASI_SDK/share/wasi-sysroot \
    -mexec-model=reactor -std=c++20 -O2 -fno-exceptions -fno-rtti \
    -Wl,--allow-undefined \
    -Wl,--export=boot -Wl,--export=frame -Wl,--export=key_event \
    -Wl,--export=__heap_base -Wl,--export=memory \
    example/web_main.cpp -o web/boss.wasm
```

### Build a Swift SpriteKit game

Use [SuperBox64Kit](https://github.com/SuperBox64/SuperBox64Kit) as a SwiftPM dependency, then build with the pinned wasm SDK. The output `.wasm` is served with this runtime exactly like a C++ game:

```bash
xcrun --toolchain swift swift build \
    --swift-sdk swift-6.3.2-RELEASE_wasm \
    -c release
```

The Swift + Box2D proof-of-concept (`example/swift-poc/`) links a native `libcbox2d.a` and exports `boot`/`frame`/`_initialize` via linker flags in `Package.swift`:

```bash
swift build   # links native/libcbox2d.a
```

> **Note:** `example/swift-poc/Package.swift` hardcodes an absolute, now-stale path to `libcbox2d.a` under `.../BossMan/wasm-web-kit/…` — it will not build as-is from this repo location.

### Bundle for offline (`file:///`, no server)

```bash
# README form (explicit args):
python3 scripts/bundle.py web/game.wasm web/assets web/index.html local.html

# Actual bundle.py signature: WEB_DIR [WASM_FILENAME] [--asset-root ROOT]
python3 scripts/bundle.py web boss.wasm
```

Produces a single `local.html` / `bundle.js` with every asset inlined as a `data:` URL. Browsers fetch `data:` URLs from a `file://` origin and honor the embedded MIME type — exactly what `instantiateStreaming` wants for the wasm module.

---

## Reactor Contract

The wasm binary is a WASI Preview 1 (or Embedded Swift) **reactor** exporting exactly:

| Export | When called | What it does |
|---|---|---|
| `_initialize` | Once, first | wasi-libc init + global constructors (C / C++ / Swift alike) |
| `boot()` | Once, after asset preload | Create the game scene |
| `frame(dtMs: f64)` | Every `requestAnimationFrame` | Advance and render one frame |
| `memory` | (exported) | Linear memory the host reads/writes strings and arrays through |

Everything else — drawing, audio, input, persistence — is **imported** from the `env` module.

### One frame, from the runtime's side

1. **Instantiate.** `runtime.js` fetches the wasm, provides every `env` + WASI import, and calls `_initialize`.
2. **Preload.** `manifest.json` drives the asset pipeline: images decode to handles/`ImageBitmap`, audio decodes to Web Audio buffers, fonts register through `FontFace` (fetched as bytes, so `file://` works too).
3. **Boot.** `boot()` runs once; the game builds its first scene.
4. **Loop.** Each `requestAnimationFrame`, gamepads are polled into the event queue, then `frame(dt)` runs. The wasm replies with a stream of `gfx_*` calls replayed onto Canvas2D — sprites and atlas sub-rects via `drawImage`, shapes/text natively, offscreen canvases for bake/crop/effect work, and a hidden WebGL2 canvas for `gfx_shader_*` GLSL effects blitted back into the 2D scene.
5. **Audio** plays on a Web Audio graph (`snd_*` voices; `eng_*` AVAudioEngine-shaped player/mixer graph; `tts_*` speech synthesis).
6. **Input** (keyboard, mouse, multi-touch, Web Gamepad) lands in a queue the wasm drains via `evt_poll`; visibility changes pause audio and the loop in background tabs.

---

## KitABI — the `env` import surface

The wasm calls a fixed set of `import_module("env")` functions (declared in `include/abi.h`). Colors are packed `0xRRGGBBAA` `uint32`; coordinates are **logical game pixels**; image/font/sound handles are small ints minted by JS (`0` = not loaded); a "target" is a draw surface handle (`0` = the screen canvas; `>0` = a render texture from `rt_create`).

> **`runtime.js` `envImports()`, not `abi.h`, is the real import surface.** The runtime implements **many more** imports than the header documents — treat the header as the documented core and `envImports()` as authoritative.

### Documented in `include/abi.h`

| Group | Functions |
|---|---|
| Logging | `js_log` |
| Target / transform / blend | `gfx_target`, `gfx_clear`, `gfx_save`, `gfx_restore`, `gfx_translate`, `gfx_scale`, `gfx_rotate`, `gfx_set_alpha`, `gfx_set_blend` |
| Primitives | `gfx_fill_rect`, `gfx_stroke_rect`, `gfx_fill_circle`, `gfx_stroke_circle`, `gfx_fill_poly`, `gfx_stroke_poly`, `gfx_draw_image` |
| Text | `txt_width`, `gfx_draw_text`, `gfx_set_text_baseline` |
| Images / fonts / RTs | `img_by_name`, `img_from_rgba`, `img_width`, `img_height`, `font_by_name`, `asset_text`, `asset_exists`, `rt_create`, `rt_image` |
| Audio | `snd_from_samples`, `snd_by_name`, `snd_create_pcm`, `snd_play`, `snd_stop`, `snd_set_volume`, `snd_status`, `snd_pause_all`, `snd_resume_all` |
| Input | `key_pressed`, `mouse_button`, `mouse_x`, `mouse_y`, `evt_poll` |
| Gamepad | `gp_connected`, `gp_button`, `gp_button_value`, `gp_axis`, `gp_map_to_keys` |
| Window | `win_set_title`, `win_width`, `win_height`, `win_request_fullscreen`, `win_exit_fullscreen`, `win_download` |
| Persistence | `store_get`, `store_set` |

Blend modes (`gfx_set_blend`): `0` alpha, `1` add, `2` multiply, `3` none. Text baseline (`gfx_set_text_baseline`): `0` alphabetic, `1` middle (true vertical centre — what SK `.center` means), `2` top, `3` bottom.

Gamepad buttons follow the W3C Standard layout: `0` A/South, `1` B/East, `2` X/West, `3` Y/North, `4` L1, `5` R1, `6` L2, `7` R2, `8` Select, `9` Start, `10` L3, `11` R3, `12–15` D-pad, `16` Home.

### Additional imports implemented in `runtime.js` (beyond `abi.h`)

| Group | Functions |
|---|---|
| Extra gfx state | `gfx_set_tint`, `gfx_set_line_style`, `gfx_snap_translation`, `gfx_upload_pixels` |
| Offscreen / shadow / filter | `gfx_offscreen_begin`, `gfx_offscreen_end_to_image`, `gfx_offscreen_end_discard`, `gfx_free_image`, `gfx_draw_shadow_image`, `gfx_set_filter`, `gfx_clear_filter`, `gfx_set_shadow`, `gfx_clear_shadow`, `gfx_set_composite` |
| WebGL2 shaders / lighting / warp | `gfx_shader_compile`, `gfx_shader_release`, `gfx_shader_set_uniform_f`, `gfx_shader_set_uniform_t`, `gfx_shader_draw`, `gfx_lighting_draw`, `gfx_warp_draw` |
| Collision | `img_polygon_from_alpha` |
| Audio pan/rate + engine graph | `snd_set_pan`, `snd_set_rate`, `eng_player_create`/`release`, `eng_mixer_create`, `eng_node_set_volume`/`pan`, `eng_connect`, `eng_player_schedule_buffer`/`play`/`stop`, `eng_start`/`stop` |
| Text-to-speech | `tts_speak`, `tts_cancel`, `tts_set_preferred_voices`, `tts_set_robotic_voices`, `tts_set_female_voices` |

---

## JavaScript API

```js
// Config object merged into CFG; runtime auto-boots on DOMContentLoaded.
window.WASMWEB = {
  logicalWidth, logicalHeight,   // fixed canvas backing-store size
  wasmUrl, assetRoot,            // wasm + asset paths (cache-busted via ?v=)
  canvasId,                      // default 'game'
  title,
  cacheBust,                     // appended as ?v=… to wasm + every asset
};

// On DOMContentLoaded the runtime does, effectively:
new Runtime(canvas).load(CFG.wasmUrl);
```

`class Runtime` (in `runtime.js`) public surface: `load(url)`, `preload()`, `discoverAssets()`, `wasiImports()`, `envImports()`, `wireInput()`, `pollGamepads()`, `layout()`, plus internal helpers (`drawFpsOverlay`, `_rasterizeText`, `loadSVG`/`drawSVG`/`svgRaster`, `ensureGL`/`buildShaderFrag`, `ensureAudio`, …).

---

## Display-P3 Wide Color

On Safari, Chrome 104+, and WebKit-based WebViews the runtime negotiates a `display-p3` Canvas2D context automatically. Colors passed through the ABI are emitted as `color(display-p3 …)` CSS, producing more vivid output on wide-gamut displays, with an automatic `rgba()` fallback. No game-side changes needed.

---

## Cross-platform notes

- **The same wasm runs everywhere.** A web build (full-Swift or Embedded) is the *exact* artifact WasmCart loads through WAMR and AOT-compiles, and the artifact Wasm5 plays in a WKWebView. `runtime.js` is the reference behavior the native SDL3 backend re-implements.
- **Logical-size discipline.** The canvas backing store is the fixed logical size; CSS scales it, so mouse/touch events arrive already in logical pixels. `shell.html` warns: the canvas `width`/`height` and the CSS `aspect-ratio` must match `WASMWEB.logical*`.
- **`file://` playback** relies on fonts/assets being fetched as **bytes** (not `url()`), because a `FontFace` `url()` loads outside `window.fetch` and is blocked on `file://`. The `bundle.py` fetch-shim depends on this too.
- **Cache-busting is load-bearing.** Without `CFG.cacheBust`, browsers serve stale `runtime.js` / `*.wasm` / assets by filename — the recurring "I fixed it but nothing changed" trap. The build is meant to stamp the token each time.

---

## Keyboard & input

- **Keyboard** is polled through `key_pressed` against the **SFML 2.6 keycode table**. The JS key map and the C++ `Window/Event.hpp` enum must stay in lockstep with SFML 2.6's fixed values; only the keys the games use are mapped (everything else returns "not pressed"). Web builds key off DOM `KeyboardEvent.code`.
- **Mouse** reports button + `x`/`y` in logical coordinates. **Multi-touch** delivers `TouchBegan`/`Moved`/`Ended` in SFML 2.6 event order, with finger-0 mirrored onto the mouse.
- **Gamepads / USB arcade sticks** (up to 4) report through `gp_*`; `gp_map_to_keys(1)` synthesizes arrow + Space key events so keyboard-only games work on a controller.
- **Wasm5 keyboard forwarding:** because SDL keystrokes break when WebKit is linked, [Wasm5](https://github.com/SuperBox64/Wasm5) injects synthetic DOM `KeyboardEvent`s (`e.code`) that *this* `runtime.js` listens for on `window` `keydown`/`keyup`.

---

## Related repos

- [SuperBox64Kit](https://github.com/SuperBox64/SuperBox64Kit) — **the kit.** Drop-in SpriteKit reimplementation that compiles one game source to wasm; owns SpriteKit emulation, KitABI, Box2D physics, the SDL3 backend, and the reSVG vector rasterizer. Embedded-runtime tweaks land in this repo's `runtime-embedded.js` first.
- [WasmCart](https://github.com/SuperBox64/WasmCart) — **native console.** Embedded-Swift + SDL3 shell that plays `.wasm`/`.aot` carts through WAMR (interpreter + wamrc AOT, ~1.6 MB, no browser). Its SDL3 backend mirrors this repo's `runtime.js` logic.
- [Wasm5](https://github.com/SuperBox64/Wasm5) — **webview console.** WasmCart's shell, but carts play the web build (this runtime) in a WKWebView, with SDL3 keystrokes forwarded as synthetic DOM `KeyboardEvent`s.
- [UFO-Emoji-Arcade](https://github.com/SuperBox64/UFO-Emoji-Arcade) — **the flagship game/demo.** One App Store SpriteKit codebase shipped natively, to the browser via this runtime, and to native consoles, all from the same unchanged sources.
- [Boss-Man](https://github.com/macOS26/Boss-Man) — reference full arcade game; `abi.h` is the "BOSS-MAN web platform ABI" and the `1184×666` `CFG` defaults are Boss-Man's. The live demo at [boss-man.us/play](https://boss-man.us/play) ships `runtime-embedded-min.js`.

---

## Credits & License

**WasmKit, Copyright 2026 Todd Bruss.** This product includes software developed by Todd Bruss.

Licensed under the **Apache License 2.0** — see [LICENSE](LICENSE) and [NOTICE](NOTICE). Apache 2.0 grants an explicit patent license and terminates it on patent litigation, protecting contributors and users from patent ambush.
