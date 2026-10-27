# Lop Engine — Development Rules

Lop Engine is the GPU-first Rust/WASM image-editing engine behind
[Lopsy](https://github.com/theseamusjames/lopsy.art). Lopsy is its main
consumer: it pins a tagged release through a git dependency (its `engine/`
crate re-exports `lop-engine-wasm`). Keep the engine app-agnostic. Don't put
host UI or state concepts in here; expose primitives the host drives.

## Layout

```
crates/
  lop-engine-core/    # Pure Rust, no web dependencies — builds and tests natively
    src/
      blend.rs          # Blend mode math
      brush.rs          # Brush dab geometry
      color.rs          # Color space conversions (also lab.rs, cmyk.rs)
      compress.rs       # RLE / LZ4 compression for pixel blobs
      effect_cache.rs   # Effect-cache invalidation policy
      layer.rs          # Layer descriptor types (GlowDesc, ShadowDesc, etc.)
      pixel_buffer.rs   # crop_to_content_bounds, pixel utilities
      sparse.rs         # Sparse pixel storage (from_sparse, to_sparse)
      flood_fill.rs     # Bucket fill algorithm
      selection.rs      # Selection mask operations
      filters/          # CPU filter implementations
      psd/ dng/ raf/    # PSD read/write, DNG and Fujifilm RAF decoders
    tests/              # Integration and visual tests
  lop-engine-wasm/    # WebGL2 engine, compiled to WASM
    build.rs            # Embeds brushes/*.png as built-in brush tips
    brushes/            # Built-in bitmap brush tips (every PNG is embedded)
    tests/fixtures/     # Fonts used by the text unit tests
    src/
      lib.rs            # #[wasm_bindgen] API surface — engine lifecycle, render
      api/              # The rest of the #[wasm_bindgen] API, by area
      engine.rs         # EngineInner struct: holds all GPU state
      compositor.rs     # Per-frame compositing pipeline
      effect_cache_gpu.rs # Per-layer effect image cache
      layer_manager.rs  # Texture upload/read/duplicate for layers
      brush_gpu.rs      # Brush & eraser dab rendering (GPU)
      clone_stamp_gpu.rs, healing_brush_gpu.rs, smudge_gpu.rs, dodge_burn_gpu.rs, sponge_gpu.rs
      filter_gpu.rs     # GPU filter dispatch (blur, invert, etc.)
      gradient_gpu.rs   # Linear/radial gradient rendering
      shape_gpu.rs      # Shape rasterization (rect, ellipse, polygon)
      selection_gpu.rs  # Selection mask GPU operations
      overlay_renderer.rs # Grid, rulers, marching ants, brush cursor
      text_gpu.rs, glyph_atlas.rs # Text layout (cosmic-text) and glyph atlas
      fonts/            # Bundled fallback fonts (Inter, Noto Sans Symbols 2)
      gpu/
        context.rs      # WebGL2 context wrapper, extension detection
        texture_pool.rs # Texture allocation, RGBA8/RGBA16F, upload/read
        framebuffer.rs  # FBO pool for render-to-texture
        shader.rs       # Shader compilation, program linking
        shaders/        # GLSL sources (see below)
```

### Key GLSL shaders

```
gpu/shaders/
  blend.glsl              # Layer compositing (all blend modes, mask sampling)
  final_blit.glsl         # Composite → screen (viewport transform, EDR passthrough)
  blit.glsl               # Texture-to-texture copy
  mask_expand.glsl        # R8 mask upload → pooled RGBA mask texture
  brush/
    brush_dab_*.glsl      # Brush dab (header/circle/alpha/color/footer), selection-mask aware
    eraser_dab.glsl       # Eraser dab with selection mask support
    clone_stamp.glsl      # Per-dab clone with bounds checking
    dodge_burn.glsl       # Dodge/burn dab
  effects/
    glow.glsl             # Inner/outer glow (distance field based)
    shadow.glsl           # Drop shadow
    stroke_edt.glsl       # Stroke effect (EDT outline)
    color_overlay.glsl    # Color overlay
  filters/
    adjustments.glsl      # Exposure, contrast, highlights, shadows, whites, blacks
    vignette.glsl         # Vignette effect
    gaussian_blur.glsl    # Two-pass Gaussian blur
    box_blur.glsl         # Box blur
    invert.glsl, hue_sat.glsl, sharpen.glsl, noise.glsl, posterize.glsl, threshold.glsl
  gradient/
    gradient_linear.glsl, gradient_radial.glsl
  selection/
    marching_ants.glsl    # Animated selection outline
```

## Commands

```bash
cargo test --workspace                                  # all tests (native)
wasm-pack build crates/lop-engine-wasm --target web     # release WASM package → crates/lop-engine-wasm/pkg
wasm-pack build crates/lop-engine-wasm --target web --dev
```

CI (`.github/workflows/ci.yml`) runs both the tests and the release
wasm-pack build. Run both before pushing.

## Architecture rules

### Compositor pipeline (`compositor.rs`)

Each frame: clear composite FBO → for each visible layer: render
behind-effects (outer glow, drop shadow) → blend layer texture onto
composite (with mask, blend mode, opacity) → blend active stroke texture →
render mask edit overlay → render above-effects (inner glow, stroke) → apply
image adjustments (exposure/contrast/vignette) → final blit to screen with
viewport transform.

When only the viewport (or channel mask) changed, `render` skips all of that
and runs just the final blit (`compositor::present`) from the existing
composite texture: those setters set `needs_present`, not
`needs_recomposite`. Anything that writes the composite texture outside
`composite()` (export, single-layer bake) must set `needs_recomposite`.

### Effect cache

Effect images (glows, shadow, stroke) are cached per layer in the live
compositor (`effect_cache_gpu.rs`, policy in
`lop-engine-core/src/effect_cache.rs`) and replayed until the layer's
content generation, position, opacity, mask state or effect settings
change. The content generation is bumped by `mark_layer_dirty(id)`, so
**every layer- or mask-texture writer must call `mark_layer_dirty` with the
id it wrote**. A writer that doesn't leaves stale effects on screen. Export
and Rasterize Layer Style never use the cache. The live compositor also
scissors each layer blend (`blend_layer_rect_onto_target`) and effect blend
(`EffectOut::clip`) to its rectangle. That is exact because the blend
shader passes the destination through outside the source. Seamless wrap
keeps the full pass.

### GPU snapshots (undo support)

`snapshotLayerGpu()` / `snapshotMaskGpu()` blit a layer or mask texture
into a pooled snapshot texture and return an opaque `u32` handle.
`restoreFromGpuSnapshot()` / `restoreMaskFromGpuSnapshot()` blit it back,
re-acquiring the target texture at the snapshot's size when the dimensions
differ. The `EMPTY_HANDLE` sentinel (`0xFFFFFFFF`) restores as a 1x1
transparent upload. Snapshot textures live in `snapshot_textures` plus a
free list and are freed only by `releaseGpuSnapshot()` /
`clearGpuSnapshots()`. The host owns handle lifetimes, so never free one
implicitly. `liveGpuSnapshotCount()` reports the live count.

### Compressed pixel blobs

`readLayerCompressed()` / `uploadCompressed()` blobs open with a 24-byte
geometry header `[crop_x, crop_y, crop_w, crop_h, full_w, full_h]` (all i32
LE). The crop is local to the texture, not document space. The u16 variant
adds `flags` + `uncompressed_size` for a 32-byte header (flags 0 = raw,
1 = RLE, 2 = LZ4). Changing either layout breaks hosts that store blobs.

### Masks and selection

- `uploadLayerMask()` takes grayscale bytes. They go up as an R8 texture and
  a GPU pass (`mask_expand.glsl`) expands them into the pooled RGBA mask
  texture (`EngineInner::upload_mask_texture`). The selection mask
  (`setSelectionMask()`) uses the same path.
- `blend.glsl` samples the mask during compositing: the mask value
  multiplies layer alpha. When mask edit mode is active, `compositor.rs`
  renders a translucent overlay.
- Brush, eraser and pencil shaders sample the selection mask to constrain
  painting. The `u_hasSelection` uniform controls whether it is checked.

### FP16 / wide gamut

- `texture_pool.rs` detects `EXT_color_buffer_float` and uses `RGBA16F`
  textures when available.
- `final_blit.glsl` passes values through without clamping for EDR headroom.
  The host decides the canvas colour space (Lopsy sets
  `drawingBufferColorSpace = 'display-p3'` on capable displays).

## API surface

- Every JS-callable function is a `#[wasm_bindgen(js_name = "camelCase")]`
  free function taking `&mut Engine` (or `&Engine`) first. Structured
  arguments go in as JSON strings deserialised with serde (see `addLayer`).
- Hosts depend on these names and signatures. Renaming or removing an
  export, or changing its arguments, is a breaking change. Call it out in
  the commit body (`feat!:` / `BREAKING CHANGE:`) and bump the minor
  version while we're below 1.0.

## Code style

- Prefer early returns over nested conditionals. Functions should do one
  thing; split anything much past ~40 lines.
- Rendering hot paths (brush dabs, compositing, pan/zoom) must not allocate
  in tight loops. Pre-allocate and reuse.
- Comments explain *why*, not *what*. No TODO comments; use GitHub issues.
- `cargo fmt` and `clippy` aren't enforced, and the code doesn't conform
  (`cargo fmt --check` reports ~1,500 diffs). Don't reformat code you
  aren't changing. Keep your own hunks clean by comparing the diff count
  against a pristine checkout.
- Pure math and data-structure logic goes in `lop-engine-core` with unit
  tests. `lop-engine-wasm` keeps the GPU plumbing.

## Testing

- Unit tests live next to the code (`#[cfg(test)] mod tests`). Integration
  and visual tests live in `crates/*/tests/`.
- Test the math, not the rendering. GPU code isn't exercised by
  `cargo test`; hosts cover it end to end (Lopsy's Playwright suite).
- Tests that need camera/PSD samples read them from `samples/` at the repo
  root (gitignored) and skip or stay `#[ignore]`d when they're absent.

## Releases

- Version the workspace in `Cargo.toml` (`[workspace.package] version`) and
  tag releases `vX.Y.Z`. Hosts pin a tag.
- To ship an engine change to Lopsy: tag a release here, then bump the tag
  in Lopsy's `engine/Cargo.toml`, run `cargo update -p lop-engine-wasm` in
  `engine/` and rebuild with `npm run wasm:build`.

## Git

- One logical change per commit.
- **Semantic commit messages** ([Conventional Commits](https://www.conventionalcommits.org/)):
  `feat:`, `fix:`, `refactor:`, `perf:`, `test:`, `docs:`, `chore:`, `ci:`.
  Use a scope when it helps: `fix(compositor): correct mask sampling order`.
- Keep the subject line under 72 characters.
