# Lop Engine

A GPU-first image-editing engine for the browser, written in Rust and compiled
to WebAssembly. It drives WebGL 2 directly and keeps every pixel in GPU
textures (RGBA16F where the device supports it). It covers layer compositing
with blend modes, masks and layer effects, brush/eraser/smudge/clone/healing
painting, gradients and shapes, selections, filters and adjustments, text
layout and rendering, and PSD, DNG and Fujifilm RAF import/export.

Lop Engine was extracted from [Lopsy](https://github.com/theseamusjames/lopsy.art),
the browser image editor at [lopsy.art](https://lopsy.art). Lopsy is still its
main consumer, and the git history from Lopsy's `engine-rs/` directory is kept
here.

## Crates

| Crate | What it is |
| --- | --- |
| [`lop-engine-core`](crates/lop-engine-core) | Pure Rust with no web dependencies: blend-mode math, brush dab geometry, colour spaces (sRGB, Display P3, Lab, CMYK), flood fill, selection masks, sparse pixel storage, RLE/LZ4 snapshot compression, CPU filters, and the PSD, DNG and RAF codecs. Builds and tests on any target. |
| [`lop-engine-wasm`](crates/lop-engine-wasm) | The WebGL 2 engine: compositor, layer texture management, GPU brushes, filters, gradients, shapes, selections, overlays (grid, rulers, marching ants), text (cosmic-text + swash + a GPU glyph atlas) and all GLSL shaders. Exposes a `#[wasm_bindgen]` API (`createEngine`, `addLayer`, `uploadLayerPixels`, `setViewport`, `render`, …). |

## Requirements

- Rust (stable) with the `wasm32-unknown-unknown` target:
  `rustup target add wasm32-unknown-unknown`
- [`wasm-pack`](https://rustwasm.github.io/wasm-pack/) to produce the JS
  package

## Building

```bash
# JS + .wasm package in crates/lop-engine-wasm/pkg (lop_engine_wasm.js, .d.ts, _bg.wasm)
wasm-pack build crates/lop-engine-wasm --target web

# Debug build (faster to compile, no wasm-opt)
wasm-pack build crates/lop-engine-wasm --target web --dev
```

## Testing

```bash
cargo test --workspace
```

Some tests compare decoder output against camera samples that aren't in the
repo. Put them in `samples/` at the repo root (gitignored); without them those
tests print a skip message or stay `#[ignore]`d. The visual tests in
`crates/lop-engine-core/tests/` write PNGs to `test-output/` for a human to
review.

## Using it in an app

### Option 1: build the crate directly

Run `wasm-pack build crates/lop-engine-wasm --target web --out-dir <your app>/pkg`
and import the generated module:

```ts
import init, { createEngine, setDocumentSize, setViewport, render } from './pkg/lop_engine_wasm';

await init();
const engine = createEngine(canvas); // HTMLCanvasElement with WebGL 2
setDocumentSize(engine, 1920, 1080);
setViewport(engine, 1, 0, 0, canvas.width, canvas.height);
render(engine);
```

`lop_engine_wasm.d.ts` lists every export with its signature.

### Option 2: depend on it from your own crate (what Lopsy does)

Make a small `cdylib` crate that depends on `lop-engine-wasm` and re-exports
it. That lets you pick the output file name, pin the engine version, set
your own release profile and add exports of your own.

```toml
# Cargo.toml
[package]
name = "my-app-engine"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
lop-engine-wasm = { git = "https://github.com/theseamusjames/lop-engine", tag = "v0.1.0" }

# Profiles only apply from the crate being built, so set them here.
[profile.release]
opt-level = 3
lto = "thin"
codegen-units = 1
panic = "abort"
strip = true

# Rust 1.87+ emits these WASM features by default; wasm-opt must allow them.
[package.metadata.wasm-pack.profile.release]
wasm-opt = ["-O3", "--enable-bulk-memory", "--enable-sign-ext", "--enable-nontrapping-float-to-int", "--enable-mutable-globals"]
```

```rust
// src/lib.rs
pub use lop_engine_wasm::*;
```

Then run `wasm-pack build --target web` in that crate.

To develop the engine and an app side by side, point cargo at a local
checkout instead of the git tag. Put this in the app crate's
`.cargo/config.toml` and keep that file out of version control:

```toml
[patch."https://github.com/theseamusjames/lop-engine"]
lop-engine-wasm = { path = "../../lop-engine/crates/lop-engine-wasm" }
```

## How it works

- **Pixels stay on the GPU.** Layers, masks, the selection and undo
  snapshots are all GPU textures. JS gets pixels back only when it asks for
  them (export, content bounds, compressed blobs).
- **Retained scene, per-frame compositing.** The host pushes layer
  descriptors (`addLayer`, `updateLayer`) and pixel uploads, then calls
  `render`. The compositor clears the composite, blends each visible layer
  with its mask, blend mode, opacity and effects (glows, shadow, stroke,
  colour overlay), applies document adjustments, and blits the result to
  the canvas through the viewport transform. A viewport-only change re-runs
  just that final blit.
- **Effect cache.** Each layer's effect images are cached and replayed until
  its content, position, opacity, mask or effect settings change.
- **Wide gamut.** RGBA16F textures and an unclamped final blit preserve
  Display P3 / EDR headroom when the browser offers it.

[`CLAUDE.md`](CLAUDE.md) documents the architecture and the invariants for
contributors.

## License

MIT with the Commons Clause. See [`LICENSE.md`](LICENSE.md).
