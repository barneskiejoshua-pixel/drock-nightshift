# EffortlessMetrics/BitNet-rs

- Source: https://github.com/EffortlessMetrics/BitNet-rs (github, repo)
- Found: 2026-11-19
- Status: RESEARCHED (verification ladder: DISCOVERED < RESEARCHED < PROTOTYPED < TESTED < VERIFIED < INTEGRATED)
- Relevance score: 43.7 {"relevance": 30.0, "feasibility": -0.0, "evidence": 12, "novelty": 0.0, "community": 1.7}
- Topics: inference optimization,bitnet + quantization
- License: MIT

## Problem

Rust-native local runner for BitNet 1.58-bit and dense GGUF models. The README (main, pushed 2026-10-31) says the Rust BitNet answer path is still diagnostic.

## Technique

I2_S 2-bit signed weights: four values per byte, default block of 32 with 8 packed bytes plus an f16 scale (crates/bitnet-quantization/src/i2s.rs), plus a QK256 GEMV selected at decode time. One CPU step embeds one token, runs the transformer with a KV cache, and projects logits (decode_one_cpu_token in crates/bitnet-inference/src/cpu_decode.rs). Sampling crate is documented as ChaCha8Rng-seeded.

## Foundation

BitNet b1.58 packing used by Microsoft BitNet.cpp and llama.cpp I2_S GGUF: a per-block scale times 2-bit weights, so the GEMV is a scaled integer dot product. docs/architecture-overview.md states the codebook {-2,-1,1,2} for both the 32-element I2_S layout and the GGML QK256 layout (256 elements, 64 bytes/block). i2s.rs itself calls grouped scales and pack_2bit_values and does not restate that codebook. Activations and logits stay wider (candle tensors in engine.rs).

## Compute

Cargo.toml pins Rust 1.95.0, edition 2024, package version 0.2.1-dev. README CPU build: cargo build --locked -p bitnet-cli --no-default-features --features cpu,full-cli. Official GGUF is 1187801280 bytes plus an external tokenizer.json. Hardware matrix lead CPUs are Core Ultra 7 258V (AVX2) and i5-8250U (AVX2); AVX-512 lane is Ryzen 9 9950X3D; CUDA lane is RTX 5070 Ti. i5-8400 has AVX2, so it is in that CPU class. Arc A380 is not listed (listed Arc parts are A770 16GB and Arc 140V). I2SQuantizer::supports_device returns false for OpenCL. README says roughly 200 crates; architecture-overview.md says 110.

## Reported

README publishes no tokens/s and no accuracy score. It marks I2_S BitNet32 and QK256 CPU paths diagnostic, dense SLM early working, RTX 5070 Ti CUDA execution validated through receipt CUDA-BITNET-009 with coherent answers and speed not established, Metal/OpenCL/OpenVINO/NPU as probe/smoke, and the HTTP server as incomplete. docs/architecture-overview.md states, with no machine, model, date, or receipt, 66+ Melem/s CPU, 200+ Melem/s GPU, and 2GB-parameter models loading in <1.5GB RAM. MODEL-ARTIFACT-007 reports Microsoft BitNet.cpp (commit 01eb415772c342d9f20dc42772f1583ae1e5b102, llama.cpp 1f86f058, llama-cli 3962, gcc 13.2.0 MinGW, Windows, --no-mmap after PrefetchVirtualMemory failed, -ngl 0, -t 8, -c 4096, temp 0, top-k 1, seed 42) on ggml-model-i2_s.gguf SHA256 4221b252fdd5fd25e15847adfeb5ee88886506ba50b8a34548374492884c2162: math_2_plus_2 exact 4, capital_france contains Paris, repeat_colors contains red blue green, say_ok exact OK, yes_no_water starts with No. That report says it is not a Rust CPU, CUDA, Apple, NPU, SLM, server, or speed result. GitHub API: 6 stars, 1 fork, pushed 2026-10-31, default branch main. Page counts were 3 issues and 79 pull requests (API open_issues_count 82).

## Reproducibility

Source is public and Cargo.lock is required by the documented commands. The 5/5 pass names the GGUF SHA256, tokenizer SHA256 e134af98b985517b4f068e3755ae90d4e9cd2d45d328325dc503f1c6b2d06cc7, BitNet.cpp and llama.cpp commits, greedy sampler, seed 42, and corpus ci/quality/bitnet-answer-corpus.yaml. Those passes are the C++ reference runner. The README says the Rust engine has no supported coherent BitNet answers yet. engine.rs exposes BITNET-style env hooks for a seed and a 120s default inference timeout. The 66+ / 200+ Melem/s lines have no run, seed, or machine attached. Many tests are documented as ignored unless hardware or model artifacts are present.

## Drock application

none. DRock generates with small Qwen models in Ollama. This repo is a separate Rust runtime, and its own README says Rust BitNet text is diagnostic. The i5-8400 AVX2 CPU could host the CPU feature flag; the Arc A380 is outside the hardware matrix, and the I2_S quantizer reports OpenCL as unsupported.

## Modifications

A trial would need a Rust 1.95 toolchain, the 1.19 GB Microsoft I2_S GGUF, the external tokenizer.json, the llama-bpe override, and a new serving path in place of Ollama. The repo's own next step, in MODEL-ARTIFACT-007, is a strict Rust CPU answer run against that artifact before any speed claim. Nothing in the tree speaks Qwen tool-calling the way DRock's executor expects.

## Limitations

README: pre-alpha, do not use in production, Rust BitNet answers not supported, throughput qualification is later work, server inference not ready. The answer gate forbids speedup claims before a lane proves answer quality. architecture-overview.md contradicts that with unsourced Melem/s figures and a production-server description. README crate count (about 200) disagrees with the architecture doc (110). Hardware matrix has no i5-8400 and no Arc A380. The 2B I2_S model is not the Qwen models DRock already runs. A 119s discuss-grounded turn is an agent loop, and this repo measures neither that loop nor tokens/s on this machine.


Analysis is an agent's reading of the source. Nothing here has been reproduced unless the status says so.
