# artalis-io/bitnet.c

- Source: https://github.com/artalis-io/bitnet.c (github, repo)
- Found: 2026-11-19
- Status: RESEARCHED (verification ladder: DISCOVERED < RESEARCHED < PROTOTYPED < TESTED < VERIFIED < INTEGRATED)
- Relevance score: 56.0 {"relevance": 39.0, "feasibility": -0.0, "evidence": 14, "novelty": 0.0, "community": 3.0}
- Topics: inference optimization,memory + retrieval,cpu + bandwidth,bitnet + quantization
- License: MIT

## Problem

DRock's discuss-grounded turns are slow (median 119 s) on an i5-8400 with AVX2, 32 GB RAM, and an Intel Arc A380. bitnet.c is a C11 GGUF inference engine. It does not speak to DRock's failing contract tests or false-success classes.

## Technique

CPU-first GGUF decode with scalar, NEON/SDOT, AVX2, AVX512 BW/VNNI, and WASM SIMD kernels; MoE experts loaded by mmap, pread, or madvise, with a pinned LRU expert cache; TurboQuant KV at 2, 3, or 4 bits so GQA scores packed keys. src/transformer.c routes each token to a BF16 path, a GPU-resident forward, or a CPU layer loop. src/turboquant.c rotates with a fast Walsh-Hadamard transform, quantizes coordinates to hardcoded Lloyd-Max centroids, and adds a 1-bit QJL residual on keys. src/transformer/kv.c stores FP32, FP16, or TurboQuant rows.

## Foundation

Standard transformer decode (RMSNorm, RoPE, GQA or SSM, dense FFN or MoE) plus the TurboQuant construction cited in include/turboquant.h as ICLR 2026 / arXiv 2504.19874: a random diagonal and Walsh-Hadamard rotation, scalar Lloyd-Max quantizers for a Gaussian scaled by 1/sqrt(d), and a 1-bit quantized Johnson-Lindenstrauss residual so the key inner product is unbiased. The paper abstract reports near-optimal distortion, within about a 2.7 factor of its lower bound, and KV-cache quality neutrality at 3.5 bits per channel with marginal degradation at 2.5 bits. That abstract does not state an 8.9x memory figure.

## Compute

A CPU build is C11 plus libc, libm, and pthreads. The i5-8400 can run the AVX2 kernels and cannot run AVX512. Optional backends are macOS Metal, wgpu-native WebGPU, and CUDA. src/gpu_cuda.cu is 1,792,711 bytes. third_party holds cutlass, exllamav3, flash-attention, flashinfer, and pytorch. Weights are external GGUF files. No SYCL, Level Zero, or Arc backend is named. The CUDA checkpoint that has a machine attached used an RTX PRO 6000 Blackwell Max-Q, driver 595.104.02, CUDA toolkit 13.4.92, GCC 13.3, SM120a.

## Reported

GitHub repo description: 'TurboQuant 3-bit KV compression (8.9x less memory per session)', with no machine or model. AGENTS.md, on --kv-tq: 2, 3, or 4 bits, recommended 3, '8.9x per-session KV compression' and '~9x more concurrent sessions'; '35B MoE + 64K context = 8.4 GB/session (vs 26 GB with FP32 KV)' together with --pread --cache-mb 2048. No command, hash, or log is attached to 8.9x, 8.4 GB, or 26 GB. 26/8.4 is about 3.1x for the whole session, a different ratio from 8.9x. src/turboquant.c states the d=128, 3-bit layouts: key 68 bytes (48 index + 16 QJL + 2 residual-norm + 2 vec-norm), value 50 bytes. README states no tok/s and no memory ratio, and says docs/benchmarks.md numbers are local checkpoints. That file is 831,615 bytes. From its opening CUDA section: Qwen3.8 27B Q4_K_XL, 128 prompt tokens from docs/inference.md, 8 host threads, FP16 KV, flash off, warm cache, final-token logits, session reset excluded. CUDA batch, 5 runs, median 218.188 ms, 586.649 tok/s. Tokenwise prompt control, 3 runs, median 2266.818 ms, 56.467 tok/s. The text calls this a batching benefit, not a new kernel or vLLM parity. Same doc: strict 8-prompt CUDA vs llama.cpp, 24 tokens each, context 512, F32 KV, flash off, 8 threads, 192/192 continuation IDs. Later in the same file a longer prompt is 21/24 IDs; CPU chunked 33/9 is 15/16 IDs with a 0.0140915-logit margin; batched CPU on an exact 33-token prefix is 5/8 IDs; tokenwise CPU on that prefix is 8/8; one CUDA 33/9 Compute Sanitizer run is 16/16 IDs and zero memory errors. Those checks cite /tmp logs on the author's machine.

## Reproducibility

Public C source, MIT, default branch main, pushed 2026-11-18. AGENTS.md says unit tests use synthetic data and need no model; e2e and coherence need a local GGUF that is not in the repo. bn_tq_init takes a uint64 seed and fills signs from a local PRNG. Benchmark commands and some binary SHA256s are written in docs/benchmarks.md. The parity logs named there are /tmp paths, not committed artifacts. I did not compile or run it.

## Drock application

Only as a possible Ollama replacement on the AVX2 CPU path, if discuss-grounded time is decode or long-context KV on the i5-8400. It does not change DRock's executor or verification gate. The Arc A380 is not a documented target.

## Modifications

A fair trial on this PC would be the CPU AVX2 build only, the same small Qwen GGUF Ollama already serves, and one timed discuss-grounded prompt against Ollama. CUDA, AVX512, and Metal do not apply. WebGPU on the A380 would be an adapter check, separate from the measured CUDA run.

## Limitations

The README says CUDA is an architectural target and is not implemented. The tree and docs/benchmarks.md contain a CUDA backend and CUDA timings. The GitHub description says zero-dependency. CLAUDE.md limits that to CPU builds (libc, libm, pthread). GPU builds pull the third_party trees above. The 8.9x line is a slogan in the description and AGENTS.md, not a row in the benchmark table. From the layout comment, d=128 3-bit K+V is 118 bytes against 1024 bytes of FP32 K+V, about 8.7x on paper, and that is arithmetic from the comment, not a measured session. CPU greedy tokens still diverge from llama.cpp in the same benchmark write-up. The timed 586.649 tok/s figure is 128-token prefill of a 27B model on a Blackwell workstation GPU. One session is not thread-safe. WASM is limited by wasm32 memory. The i5-8400 has no AVX-512, and a 6 GB Arc is outside the named backends.


Analysis is an agent's reading of the source. Nothing here has been reproduced unless the status says so.
