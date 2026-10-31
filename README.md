# ChunkPlusPlus

Experiments and reviewable changes for faster Horizon chunk generation and rendering.

## Density buffer reuse experiment

The workspace snapshot inspected for this change is the Horizon source under `meteor-native/target/build-stage-v4`. Its generation pipeline is in `crates/horizon-world/src/worldgen/generator.rs`; terrain filling is in `crates/horizon-world/src/worldgen/terrain.rs`.

Noise terrain generation samples a 16×384×16 density volume by default. At 32-bit density samples, that temporary buffer is 384 KiB per chunk. The previous implementation allocated and released it for every chunk.

The workspace source now keeps a density buffer on each `GeneratorWorker`, resizes and zeros it for each volume, and releases it in the existing worker idle cleanup. Zeroing is required because some density samplers can leave output elements untouched. The existing public `fill_noise` API remains available and delegates through temporary scratch; the generator worker uses the scratch-aware function.

This removes repeated buffer allocation/free work while preserving the zero-initialized sampler input. It does not reduce density sampling or terrain math, so any total speedup depends on allocation being a measurable part of chunk-generation time. This affects locally generated terrain; multiplayer chunks sent by a server bypass this path.

`patches/reuse-density-scratch.patch` is a review artifact for the source change. Its application against the full Horizon checkout has not been validated; inspect it against the target revision before applying.

## How to measure

Use Horizon's existing benchmarks with matching resources, seed, center, thread count, and warmup for baseline and changed builds:

```sh
cargo run --release -p horizon-world --example chunkgen_bench -- .local/horizon-client/26.3 --threads 8 --size 64 --warmup --json
cargo run --release -p horizon-client --example chunk_load_bench -- .local/horizon-client/26.3 --threads 8 --radius 8 --rounds 3
```

Compare repeated proto chunks/second and noise-stage time, plus generated-and-lit columns and mesh time/checksum. Verify chunk parity first. A performance win remains unproven until those A/B runs are completed.

The benchmark resources are not included; prepare them from the user's own Minecraft installation as Horizon documents.
