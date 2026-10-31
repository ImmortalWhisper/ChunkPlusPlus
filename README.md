# ChunkPlusPlus

Experiments and reviewable patches for faster Horizon chunk generation and rendering.

## Current experiment: reuse terrain density scratch

The Horizon snapshot inspected for this change is `horizon-rust/horizon` revision `1da354b67a509812c4141b67de2affd6de65c342` (the workspace's `meteor-native/target/build-stage-v4` source snapshot).

Noise terrain generation in `crates/horizon-world/src/worldgen/terrain.rs` samples a density volume for each chunk. At a 16×384×16 volume and 32-bit density samples, the temporary buffer is 384 KiB per chunk. The current implementation allocates and releases that buffer for each generation call.

This experiment retains the buffer on each `GeneratorWorker` and resizes it for each volume. The existing `GeneratorWorker::release` path drops it when a streaming worker goes idle. The existing public `fill_noise` API remains available and uses a temporary buffer; the generator worker calls the new scratch-aware function.

The buffer is reset to zero before sampling because some density samplers may leave output elements untouched; this preserves the previous behavior. So the experiment removes repeated allocation/free work, not the buffer initialization, density sampling, or terrain math. Any total speedup depends on allocation being a measurable part of chunk-generation time. It affects locally generated terrain; server-sent multiplayer chunks do not use this path.

## Patch

See [patches/reuse-density-scratch.patch](patches/reuse-density-scratch.patch). Apply it to the matching Horizon source revision, then check the resulting diff before building.

## Measure before claiming a speedup

Use Horizon's existing benchmarks with the same resources, seed, center, thread count, and warmup for both baseline and patched builds:

```sh
cargo run --release -p horizon-world --example chunkgen_bench -- .local/horizon-client/26.3 --threads 8 --size 64 --warmup --json
cargo run --release -p horizon-client --example chunk_load_bench -- .local/horizon-client/26.3 --threads 8 --radius 8 --rounds 3
```

Run multiple baseline and patched samples on an otherwise idle machine. Compare proto chunks/second and the noise-stage time from the first benchmark, plus generated-and-lit columns and mesh checksum/time from the second. First verify that generated chunk dumps/parity are unchanged. Do not claim a performance win from compilation alone.

The benchmark resources are intentionally not included here; prepare them from the user's own Minecraft installation as Horizon documents.
