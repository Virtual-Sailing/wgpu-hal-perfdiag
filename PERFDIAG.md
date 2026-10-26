# Performance diagnostics changes

This repository contains the published `wgpu-hal` 27.0.4 package plus two Metal
observers. The initial source import is a separate Git commit, so subsequent
source changes can be compared directly with the upstream release. The original
crate name and version are retained for Cargo overrides. `publish = false`
prevents attempting to upload the fork under the upstream registry name.

The upstream revision and SHA-256 of the published `.crate` archive are recorded
in `package.metadata.perfdiag` in `Cargo.toml`. The standalone manifest is based
on that published package: `wgpu-types`, `naga`, and other unmodified dependencies
come from crates.io rather than sibling paths in the upstream wgpu workspace.

## Metal observers

- `src/metal/mod.rs` adds `Surface::set_presented_handler` and installs an
  `addPresentedHandler` block on each drawable before either presentation path.
  It reports `MTLDrawable.presentedTime`, without retaining the drawable or
  waiting for presentation. `src/metal/surface.rs` initializes the handler state.
- `src/metal/command.rs` adds `CommandEncoder::on_next_render_pass_completed`.
  Its one-shot observer attaches to the next render pass's native command buffer
  and reads `GPUStartTime` / `GPUEndTime` after successful completion. Pending
  observers are cleared when encoding is discarded or the encoder is reset.
  These are command-buffer execution timings, not display timestamps.

The command-buffer observer preserves SailSim's workaround for stale
end-of-fragment timestamp-query samples reported during an Apple M4 offscreen
smoke check. wgpu internally splits passes and transitions into different native
command buffers; observing the current buffer before or after a pass can time an
empty buffer instead. This workaround is not a general requirement for Metal.

Handlers run on Metal callback threads and must be brief, non-blocking, and must
not panic. The observers do not submit commands or wait for GPU/display completion.
No DirectX 12, Vulkan, GL, or other backend source is customized.

## Consumption and validation

Applications replace `wgpu-hal` in their root `[patch.crates-io]` table using this
repository's HTTPS Git URL and a full commit `rev`. Cargo patch tables in
non-root dependencies are ignored. Using the matching `egui-wgpu-perfdiag` fork
therefore requires both overrides in the application's manifest.

CI runs the existing library tests on macOS with `metal` and on Windows with
`dx12`. Run the macOS job locally with:

```sh
cargo test --lib --locked --no-default-features --features metal
```

On Windows, substitute `dx12` for `metal`. These library tests do not validate
actual display callbacks or timings on a GPU. SailSim integration validation
should also build the application, run its tests, and run `--render-smoke` on
supported hardware. Commit the updated lockfile when intentionally changing
resolved dependencies or the upstream release.
