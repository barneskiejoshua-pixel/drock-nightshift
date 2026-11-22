# pavan139/speculative-decoding

- Source: https://github.com/pavan139/speculative-decoding (github, repo)
- Found: 2026-11-19
- Status: RESEARCHED (verification ladder: DISCOVERED < RESEARCHED < PROTOTYPED < TESTED < VERIFIED < INTEGRATED)
- Relevance score: 35.5 {"relevance": 21.5, "feasibility": -0.0, "evidence": 14, "novelty": 0.0, "community": 0.0}
- Topics: memory + retrieval,speculative decoding
- License: MIT

## Problem

A fast draft-and-verify decoder can silently change the output distribution or leave stale KV state after a rejected branch. The repo asks how to get the speed of block verification while keeping the target's exact law and proving the cache rollback is correct.

## Technique

Draft k tokens from q; one target forward returns k+1 conditional rows (including a bonus row); accept token x with min(1, p(x)/q(x)), on rejection draw from normalized max(p-q,0), on full acceptance draw one bonus target token. A pending-token ledger plus crop of the mutable KV cache rolls back to the committed prefix. Correctness is checked three ways: a slow full-prefix serial reference, an independently enumerated finite target law compared path by path, and targeted cache-mutation probes.

## Foundation

Leviathan-Kalai-Matias rejection sampling: accept with min(1,p/q) and sample the residual (p-q)+ so marginals equal p exactly, regardless of q. Enumeration of all positive-probability target paths gives an oracle with no draft model, acceptance ratio or RNG. Causal-attention prefix invariance lets one forward produce every required conditional; logits_to_keep=k+1 avoids computing unused prefix logits. RNG draw order is matched to the serial sampler.

## Compute

Python 3.11+, torch 2.6.0+cpu, transformers 4.57.1 with the [hf] extra. Batch-one unpadded GPT-2 or standard-RoPE Llama, DynamicCache, eager attention, unquantized weights, one physical target forward per verification block. Needs full normalized vocab distributions from both models and an identical token vocabulary; Ollama exposes neither full vocab logits nor a rollback-able KV cache.

## Reported

Stored CPU fixture (results/cpu-tiny-benchmark.json): full-prefix serial median 292.9 ms / 81.95 tok/s, cached serial 204.3 ms / 117.46 tok/s, cached block 109.8 ms / 218.63 tok/s; observed 2.67x ratio of full-prefix to cached-block medians, presented in the README as 2.7x. Conditions: 31,616-parameter target, 6,384-parameter draft, vocab 64, 3 prompt tokens, 24-token cap, draft window 4, float32, one torch thread, seeds 17-21, 2 warmups, 5 repeats, cold caches and prefill included, loading/warmups excluded, random checkpoints deleted. On the four 24-token trials target forwards were 24/24/24/24 serial versus 6/6/5/7 cached block. Correctness record: 40 tests pass, 5,400 random toy trajectories checked exactly, GPT-2/Llama distribution parity to max absolute error 1.49e-8. Author explicitly says this is not a trained-model or GPU speedup result.

## Reproducibility

Code, config and data are in the repo. The raw benchmark JSON keeps per-trial timings, seeds, physical forward/input-token/crop counters and environment fields; bench/time_random_checkpoints.py regenerates the fixture from fixed seeds, though the weights were deleted. Deps pinned in pyproject/requirements-dev.txt, tests run via RUN_ML_SMOKE=1 PYTHONPATH=src python -m unittest discover -s checks -v plus bench/check_sampling.py --require-lint --wheel. Reports retain mutation failure evidence. I did not rerun it. Enumeration is capped at 8 output tokens and an explicit leaf budget, and the author notes finite-sample agreement is evidence, not proof.

## Drock application

Not the engine: DRock runs small Qwen models through Ollama, not HF Transformers with controllable logits and KV state. The transferable part is the verification-gate discipline: keep a slow reference path and an independent oracle, compare traces path by path, and record physical operation counters so a speed or correctness claim cannot come from a shorter output, a different stop rule or a skipped call.

## Modifications

No direct code reuse. Applying the block engine would require per-position logits and KV commit/rollback hooks that Ollama does not expose, plus a draft model sharing the exact vocabulary. For DRock I would take only the pattern: an enumerated action-grammar oracle beside the executor and per-mode counters for calls, artifacts and network attempts; MIT permits copying the harness if wanted.

## Limitations

Batch one, unpadded, eager attention, DynamicCache, standard GPT-2/Llama, unquantized weights; no sliding/static/paged caches, nonstandard RoPE, quantization, multi-request batching, GPU results or fused sampling. Requires identical vocabularies and complete distributions; top-k/truncation must be applied consistently by the backend. Truncation of enumeration to 8 tokens and a leaf budget; float64 arithmetic only. More accepted tokens can coexist with more physical token work. Timing is one fixed-order run on tiny random weights and is sensitive to host scheduling.


Analysis is an agent's reading of the source. Nothing here has been reproduced unless the status says so.
