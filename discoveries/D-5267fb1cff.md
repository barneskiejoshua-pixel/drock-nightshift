# rajveer43/VeloxQuant-MLX

- Source: https://github.com/rajveer43/VeloxQuant-MLX (github, repo)
- Found: 2026-11-19
- Status: RESEARCHED (verification ladder: DISCOVERED < RESEARCHED < PROTOTYPED < TESTED < VERIFIED < INTEGRATED)
- Relevance score: 42.1 {"relevance": 27.0, "feasibility": -0.0, "evidence": 12, "novelty": 0.0, "community": 3.1}
- Topics: memory + retrieval,cpu + bandwidth,bitnet + quantization
- License: MIT

## Problem

The KV cache takes memory during long-context inference with mlx_lm models on Apple Silicon. The library collects 43 KV-cache compression methods behind one KVCacheConfig(method=...) API: 20 quantizers, 16 token eviction or merging methods and 7 hybrids.

## Technique

The default method is turboquant_rvq (veloxquant_mlx/quantizers/turboquant_rvq.py). It is a two-stage residual vector quantizer. Keys are rotated by a seeded random orthogonal matrix, or by a random-sign Hadamard transform when use_hadamard=True. Stage 1 uses a fixed, precomputed Lloyd-Max codebook (Gaussian for d>=64, a beta codebook below that). Stage 2 quantizes the residual with a Laplacian-fit Lloyd-Max codebook. Each stage uses b bits, so 2b bits per coordinate. The indices are bit-packed into uint32 words by a fused Metal kernel (rvq_quant_pack). Inner products are estimated in the rotated space as (y_hat1+y_hat2)·q_rot. The other methods are adaptations of published papers, among them KIVI, KVQuant, SnapKV, StreamingLLM, H2O, TOVA, PyramidKV, MiniCache and ZipCache.

## Foundation

A random orthogonal or Hadamard rotation makes the coordinates close to Gaussian, which is what lets one fixed Lloyd-Max scalar codebook work without calibration. A residual stage then corrects the error of the first stage. The eviction methods drop or merge tokens based on attention scores. The fast paths are Metal compute kernels driven through MLX.

## Compute

Apple Silicon M1 or later, MLX>=0.18, Python>=3.10 and NumPy>=1.26. The fused paths need custom Metal kernels. All reported measurements come from an M4 MacBook Air with 24 GB. There is no CUDA, SYCL, Vulkan or Ollama backend.

## Reported

These are the author's numbers, which I did not reproduce. Table conditions: end-to-end mlx_lm.generate, 200-token prompt, 120-token generation, M4 Air 24 GB. Throughput in tok/s as fp16 / RVQ-1bit (7.5x, keys only) / VecInfer-1bit (16x): Llama-3.2-3B 47.6/46.2/40.2, Llama-3.1-8B 20.5/20.6/19.6, Mistral-7B 23.6/22.8/9.8, Qwen2.5-7B 21.0/20.7/21.5, Qwen3-8B 20.3/19.6/2.4, Falcon3-7B 17.3/21.7/17.0. The Metal quantize kernel is reported at 14.7x faster than the unfused path at S=8192 (228 ms to 15.6 ms), with a range of 6.9-14.7x. Launch overhead is 50-200 us per call, so tiny models can come out slower. Llama-3.2-1B single prefill peak memory: 1402 MB for RVQ with packed keys, 1608 MB for fp16 and 2537 MB for mlx-lm QuantizedKVCache(bits=4). The 729 MB to 12 MB figure is transient scratch memory in one quantize_vq op, not KV-cache size. The 16x figure is VecInfer key bytes only at head_dim=128 and needs user-trained codebooks. The README itself says most quantizers store dequantized fp16 internally, so their ratios are bit-width accounting, not resident memory. Only the eviction and merging methods reduce RSS today. In RVQ the values stay fp16, so 7.5x is a key-only ratio. The turboquant_rvq docstring claims about 0.98 cosine similarity at b=2, d=128, and no measurement backs it. No quality or perplexity numbers were in the README text I read.

## Reproducibility

Code is public and installs with pip install VeloxQuant-MLX. benchmark_scripts/, BENCHMARK_RESULTS.md and tests/non_metal exist, but I did not open them. Seed defaults to 42 and controls only the rotation matrix or Hadamard signs. The codebooks are deterministic and do not depend on the seed. The prompt set behind the throughput table was not identified. Every number comes from one machine and one run of each config, with no variance reported. Nothing can be run on DRock's hardware, which has no Metal.

## Drock application

none directly. DRock runs Qwen in Ollama, which uses llama.cpp, on an Intel Arc A380 and an i5-8400. MLX and Metal do not run there. The KV-cache problem itself does apply: 6 GB VRAM limits context. The practical counterpart already exists in Ollama: OLLAMA_KV_CACHE_TYPE=q8_0 or q4_0 together with OLLAMA_FLASH_ATTENTION=1. That is a config change measured against latency.jsonl, not a port of this library. A later step could be StreamingLLM-style sink+window eviction, but only if DRock ever owns the inference loop.

## Modifications

Using this would mean porting the rotation, Lloyd-Max and residual pipeline to llama.cpp's SYCL or Vulkan backend. That is a large C++/kernel job and outside DRock's scope. The cheap alternative: benchmark the KV types Ollama already ships, f16, q8_0 and q4_0, on the A380. Measure tok/s, peak VRAM and the DRock benchmark pass rate, with before and after numbers. That benchmark is a scored change ShinkaEvolve could explore, but it is a simple config sweep and does not need evolution.

## Limitations

Runs only on Apple Silicon and MLX. Most quantizers do not actually reduce resident memory, by the author's own account. RVQ compresses keys only. The throughput table uses a short 200+120 token context, where the KV cache is not the bottleneck, so it says little about long-context benefit. VecInfer slows Qwen3-8B to 2.4 tok/s. Hardware is one M4 Air, and models above 32B were not measured. 43 methods from a 36-star project with about two active contributors: most are labeled beta or experimental, and the README recommends only 3 as stable. No quality evaluation was found in what I read. The license covers the code, not the papers the methods come from.


Analysis is an agent's reading of the source. Nothing here has been reproduced unless the status says so.
