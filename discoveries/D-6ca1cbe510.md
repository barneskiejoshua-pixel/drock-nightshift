# wybe-labs/npurun

- Source: https://github.com/wybe-labs/npurun (github, repo)
- Found: 2026-11-19
- Status: RESEARCHED (verification ladder: DISCOVERED < RESEARCHED < PROTOTYPED < TESTED < VERIFIED < INTEGRATED)
- Relevance score: 45.7 {"relevance": 30.0, "feasibility": -0.0, "evidence": 12, "novelty": 2.2, "community": 1.6}
- Topics: inference optimization,bitnet + quantization
- License: MIT

## Problem

Ternary (1.58-bit) GGUF matrix multiplies on a Ryzen AI laptop leave the XDNA 2 NPU idle while the CPU is memory-bandwidth bound.

## Technique

A ggml CPU extra buffer (src/npu/npu.cpp) that keeps a row slice of each PQ2_0 or PTQ1_0 matrix on an AMD XDNA 2 NPU and the rest on the CPU, and runs both at once. One prebuilt shape-agnostic program (tgemv.xclbin) on all 32 AI Engine cores; per-call DMA streams; q/k/v, qkv/gate, and ffn gate/up fused into one launch (max 3 matrices).

## Foundation

Weights are {-1,0,+1} with an fp16 scale per group of 128. pack.py stores five ternary codes per byte as base-3 (1.75 bits/weight) plus those scales. Activations are int8 per 128 with scale amax/127. npu.cpp sets NPU rows Mn to floor(split * M / 1024) * 1024, default split 0.75, so the NPU DRAM stream and the CPU DRAM stream overlap. ROW_MULTIPLE is 8 columns * 4 rows * 32.

## Compute

Windows 11 x64, AMD Ryzen AI 300/400/Max (XDNA 2), AMD NPU driver with XRT 2.19 or newer. Guide: 16 GB RAM for the 6.7 GB 27B file, 32 GB recommended. CPU share is built for Zen 5 AVX-512; the guide says that without AVX-512 the CPU share is more than twice as slow. IRON/MLIR-AIE is only needed to rebuild the kernel. Does not use a GPU. An i5-8400 has no XDNA NPU and no AVX-512; an Intel Arc A380 is not a target.

## Reported

Author figures, same table in README, docs/guide.md, and docs/releases/v0.1.0.md. Bonsai 2 27B PQ2_0 on a Ryzen AI 9 465. Guide adds: 32 GB LPDDR5X, background load, about ±15% run to run; CPU-only column is 12 threads; NPU+CPU column is 6 threads; prompt column is 64 tokens. Decode 3.3–3.9 tok/s CPU-only vs 4.1–5.2 tok/s NPU+CPU. Prompt 10.1 tok/s vs 12.2–13.8 tok/s. Guide also says NPU reads memory at about 35–44 GB/s and the CPU at up to about 70 GB/s; on 8B and below the CPU alone is often as fast or faster for decode; on PTQ1_0 the NPU path is 'several times faster' because the CPU has no optimized kernel for that format. Model table: 27B 'output identical to the CPU path'; 8B 'perplexity equal to the CPU path'; widths tested up to 32768. How-it-works and AGENTS.md say match is within quantization noise, and npurun-test must print worst cosine > 0.999 vs a double-precision reference. No perplexity number, no sample size, no raw log is in the repo.

## Reproducibility

Code is public. llama.cpp is a submodule pinned at 87268f775d74cf8f7ffc6c22a95684aa55995533; patches/llama.cpp.patch is applied at configure. tgemv.xclbin is committed (1,479,892 bytes) and embedded. AGENTS.md gives the speed check: llama-bench -p 64 -n 16 -t 6 -mmp 0 --poll 100 against GGML_NPU=0 in the same session, and says background load makes single runs unreliable. Correctness needs the NPU (npurun-test) or llama-perplexity with vs without GGML_NPU. pack.py and kernel/npurun/emulate.py are host-side references. The GGUF is a separate Hugging Face download. No seeds and no committed bench output.

## Drock application

none

## Modifications

The backend cannot be pointed at DRock's Ollama/Qwen path or at the Vulkan llama.cpp bundle. A useful variant would be a different ternary GEMV for the Arc A380 or for the i5-8400, plus PQ2_0 weights for the 0.6B–4B Qwen tiers. This repo has neither.

## Limitations

Windows only. Non-ternary GGUF (Q4/Q8/F16) stays on the CPU. NPU opens only when a ternary weight exists; open failure falls back to CPU. Mn==0 skips the NPU, which is what the 1024-row rounding does when 0.75*M is under 1024. If weights are under 4 GiB, GGML_NPU_CPU_COPY defaults on and prompt batches (ntok>1) skip the NPU and run on the CPU copy. 'Identical' and 'within quantization noise' are both in the docs, with no published perplexity. Their ±15% caveat is wider than the gap between 3.9 and 4.1 tok/s, so the decode ranges are not a controlled speedup. Third-party: llama.cpp MIT, TxnEncoding.h Apache-2.0 WITH LLVM-exception, XRT Apache-2.0 downloaded at build and not redistributed.


Analysis is an agent's reading of the source. Nothing here has been reproduced unless the status says so.
