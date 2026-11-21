# 1bit-MONSTER/engine

- Source: https://github.com/1bit-MONSTER/engine (github, repo)
- Found: 2026-11-19
- Status: RESEARCHED (verification ladder: DISCOVERED < RESEARCHED < PROTOTYPED < TESTED < VERIFIED < INTEGRATED)
- Relevance score: 39.5 {"relevance": 25.0, "feasibility": -0.0, "evidence": 12, "novelty": 0.5, "community": 2.1}
- Topics: inference optimization,bitnet + quantization
- License: Apache-2.0

## Problem

Run local LLMs on AMD Strix Halo (Ryzen AI MAX+ 395) under Linux. The engine serves GGUF models on the Radeon 8060S iGPU (gfx1151) and on the XDNA 2 NPU, behind Lemonade Server's OpenAI-compatible API.

## Technique

One binary with its own HIP kernels for the HRX GPU path and its own NPU runtime for XDNA 2. It adds MTP (multi-token prediction) decoding and ternary BitNet b1.58 quants (TQ1_0/TQ2_0). The README also lists MoE, ONNX, LoRA and zero-copy directories, which I did not read.

## Foundation

Architecture and systems work: HIP kernels written for gfx1151, an NPU runtime for AMD XDNA 2, MTP speculative-style decoding, and 1.58-bit ternary weights. No new math is claimed.

## Compute

AMD Ryzen AI MAX+ 395 with Radeon 8060S iGPU and XDNA 2 NPU, Linux (Ubuntu with Lemonade). The OS image needs a USB stick of 16 GB or more. Nothing targets Intel Arc or Windows.

## Reported

Qwen3.8-27B decodes at 36 tok/s with MTP on their kernels (milestone 2026-10-21, Strix Halo hardware gate). The NPU figure is 82.03 tok/s from candidate 19, a software gate only; the README says that candidate has not been booted or installed on hardware. Kernel ops passed 1959/1959, and VMTEST-RELEASE gave 10 pass / 9 skip / 0 fail. The README gives no batch size, context length or quant for any of these numbers.

## Reproducibility

Source is public with CMake and submodules. The release image is not downloadable: the Hugging Face repo stays private until v2026.44. The fast HRX kernels and the NPU runtime are proprietary binaries, so a source build has no NPU engine and runs on public kernels only, and the headline numbers cannot be checked from source. The README names no seeds or eval data. docs/models.md was not accessible to me.

## Drock application

none

## Modifications

DRock could only borrow ideas, not code. One idea is MTP or speculative decoding for the Qwen models, but Ollama would have to expose it on the Arc A380. Another is trying ternary BitNet-class models in 6 GB of VRAM. Both are separate projects that need their own measurements; this repo does not supply them for DRock's hardware.

## Limitations

Hardware lock-in to gfx1151 and XDNA 2 on Linux, while DRock runs an Intel Arc A380 on Windows 11. The best numbers depend on closed binaries. One headline number is from a software gate that never ran on hardware, and no number comes with its conditions. The project is small: 10 stars, last pushed 2026-11-20. I did not open the core implementation files, only the README and file tree.


Analysis is an agent's reading of the source. Nothing here has been reproduced unless the status says so.
