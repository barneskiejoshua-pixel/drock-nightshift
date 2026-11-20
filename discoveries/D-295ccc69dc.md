# TensorTonic/llm-optimization-handbook

- Source: https://github.com/TensorTonic/llm-optimization-handbook (github, repo)
- Found: 2026-11-19
- Status: RESEARCHED (verification ladder: DISCOVERED < RESEARCHED < PROTOTYPED < TESTED < VERIFIED < INTEGRATED)
- Relevance score: 46.1 {"relevance": 22.5, "feasibility": -0.0, "evidence": 12, "novelty": 7.7, "community": 3.9}
- Topics: memory + retrieval,speculative decoding,bitnet + quantization
- License: MIT

## Problem

The repo frames LLM inference as slow because a request has two phases that hit different limits: prefill reads the whole prompt in one parallel pass, and decode emits one token at a time while rereading weights and the KV cache. It says most optimization advice is easy to repeat and hard to apply.

## Technique

A 7-stage reading roadmap of 48 topics, not a single algorithm. Stages: inference fundamentals (roofline, memory, sampling, TTFT/TPOT/ITL/throughput, percentiles); quantization (INT8, per-channel, group-wise INT4, block FP8, GPTQ/AWQ, QLoRA); KV cache and attention (MQA, GQA, MLA, RoPE, FlashAttention/online softmax, PagedAttention, prefix caching); speculative decoding (draft-and-verify, acceptance rate, Medusa/EAGLE) and MoE top-k routing; CUDA then Triton kernels and fusion; dynamic/continuous batching, chunked prefill, disaggregation, cache-aware routing, collectives, tensor and expert parallelism; production serving, profiling, and cost under a latency target. The method it states is measure, find the bottleneck, apply the technique for that bottleneck, measure again and check quality.

## Foundation

As the README states it: decode is limited by bytes read per token, so fewer-bit weights are faster and fit a larger model; outlier values can ruin a naive quantizer; speculative decoding uses spare compute to check several drafted tokens in one pass without changing the output distribution; MoE routes each token through a small part of a large model; fused kernels load GPU memory once, reuse on-chip memory, and write once. Roofline, online softmax, and paged KV blocks are topic names with outbound explainers, not derivations in the repo.

## Compute

README prerequisites: Python and a rough knowledge of a transformer; no GPU programming until Stage 5, which is CUDA then Triton. No CPU, GPU, RAM, model, batch size, or runtime is specified. Kernel and multi-GPU topics assume an NVIDIA GPU.

## Reported

No speed, memory, quality, acceptance-rate, or cost measurement is reported, and no hardware or model condition is given. The only quantities are curriculum counts: badge and tables say 48 topics and 7 stages (the seven tables sum to 7+7+9+5+7+8+5=48). Metrics are named (TTFT, TPOT, ITL, throughput, P50/P95/P99) with no values. GitHub API on 2026-11-20: 94 stars, 4 forks, language null, size 54 KB, pushed_at 2026-11-10T21:50:55Z, default branch main. The untrusted blurb said 89 stars.

## Reproducibility

No code, data, or seeds. Recursive tree is LICENSE, README.md, images/banner.svg, images/roadmap.svg. The languages API returned {}. README says practice is Python, CUDA, or Triton checked against tests; those files are not in the repo. Explanations and problems are links to tensortonic.com. There is no core implementation file. Latest commit 3f4366d1 (2026-11-10) only adds guide links.

## Drock application

Only the slow discuss-grounded turn (P-940bb91ac8). The prefill-vs-decode split, weight quantization, KV-cache size, and exact speculative decoding are the pieces that could touch a local Ollama Qwen call. CUDA/Triton, tensor/expert parallelism, continuous batching, and disaggregated serving do not map onto one agent on an i5-8400 and an Arc A380. The failing tests, false-success classes, and prompt assumptions are outside this handbook.

## Modifications

Nothing to port. A trial would be written from the external problem pages, then moved off CUDA/Triton onto Ollama on the Arc A380 or the CPU, with a before/after on the discuss-grounded turn and a check that greedy tokens still match if the verification gate depends on them.

## Limitations

This is a syllabus. It gives no number you can reproduce, no quality delta for INT4/FP8 beyond 'outliers can ruin a naive scheme', and no draft-model acceptance rate. Speculative decoding, as described, needs spare GPU compute during memory-bound decode. A single local turn does not use the batching stage. Intel Arc is not a CUDA/Triton target.


Analysis is an agent's reading of the source. Nothing here has been reproduced unless the status says so.
