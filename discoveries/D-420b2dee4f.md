# Draft Calibration for Speculative Decoding: Accelerating LLM Serving by Matching Draft and Target Distributions

- Source: http://arxiv.org/abs/2611.33111v1 (arxiv, paper)
- Found: 2026-11-25
- Status: RESEARCHED (verification ladder: DISCOVERED < RESEARCHED < PROTOTYPED < TESTED < VERIFIED < INTEGRATED)
- Relevance score: 41.9 {"relevance": 27.0, "feasibility": -0.0, "evidence": 0, "novelty": 14.9, "community": 0.0}
- Topics: routing + confidence,speculative decoding
- License: CC BY 4.0 (stated on the arXiv HTML pape

## Problem

Speculative decoding loses speed when the draft distribution q is flatter than the target p. For chain drafting, acceptance is 1 - 0.5*||p-q||_1, so a systematic gap costs accepted tokens per verification round.

## Technique

Post-hoc draft calibration: lower the draft temperature Td while keeping target Tt=1, and use the same calibrated q in drafting and rejection sampling. Also tested Dirichlet scaling (top-K block plus scalar tail), token-wise temperature heads, and an offline trajectory-replay screen that recomputes p and q from cached final hidden states.

## Foundation

Leviathan/Chen rejection sampling: P(accept)=sum_x min(p(x),q(x))=1-||p-q||_1/2. Temperature softmax(z/T) sharpens q without changing its ranking; Dirichlet scaling is softmax(W log q + b) with W=(1/T)I recovering temperature scaling. Calibration leaves the target distribution unchanged when q is used consistently.

## Compute

No extra model computation per token: one scalar Td. Replay stores BF16 final hidden states at eligible positions and reapplies LM heads; paper reports >1M replay checks in GPU minutes. Online serving used two NVIDIA GH200 GPUs, one TP=1 server per GPU, concurrency 16/64/128, 2,048-token cap, 3 seeds.

## Reported

Underconfidence: top-1 draft-target agreement ~70%+ and >40% in top-16, but draft top-1 probability is lower and entropy higher. Accepted length grid Td in {0,0.2,...,1.2}: ~5-10% gain (Qwen3.6-35B-A3B/DFlash 5.275 to 5.773); DSpark 3-9%; Td in [0.4,0.6] works across tasks. Online held-out table, 2,082 requests, 3 seeds: GPT-OSS-120B/EAGLE-3 CC16 accepted length 3.102 to 3.432 (+10.64%), throughput 760.9 to 836.2 tok/s (+9.92%); CC128 throughput gain +5.13%. GPT-OSS/DFlash accepted-length +9.25-9.27%, throughput +8.98-9.87%. Qwen3.6/DSpark accepted length +8.82-9.51%, throughput +12.04%/+22.08%/+19.53% at CC16/64/128. Replay Lpos: Td=1 gives 3.0739/3.6345/4.0731; temperature validation 3.5085/4.0751/4.4825; diagonal Dirichlet ties temperature, full-matrix is worse; token oracle reaches 4.0598/4.7628/4.9247 but learned heads stay near or below scalar temperature. Replay vs online mean absolute relative error 1.83%-2.64%, max up to 8.72%.

## Reproducibility

No code or data repository is linked for this paper. It names public HF checkpoints (Table 5), frozen DeepSpec prompt JSONL at revision 005e03b8, 8 benchmarks, 40/10/50 request split (1,660/413/2,082), base seed 980406 with request-index offsets, 3 online seeds, sweep grids and Adam settings in Appendix A.3. The replay pipeline, calibrated heads and trained Dirichlet checkpoints are described but not published, so exact re-running needs rebuilding those pieces.

## Drock application

none

## Modifications

The paper only rescales draft logits, with no retraining of the target or draft. DRock's spec path is llama-server --spec-type ngram-simple|ngram-mod|ngram-cache|ngram-map-k|ngram-map-k4v (server.py:976-1029), which drafts by copying tokens already in context. There is no draft distribution q to sharpen and no draft temperature knob, so applying this would first mean adding a learned drafter with separate logits and rejection sampling.

## Limitations

Post-hoc only; training-based calibration is left as future work. Gains come from models far larger than DRock's tiers and from two-GH200 multi-request serving with continuous batching, not a single-user 6 GB Arc setup. The 10% figure is accepted length, and throughput gain varies with concurrency (5.13% at CC128 for EAGLE-3). Replay is a proxy on fixed Td=1 contexts: it does not reproduce every temperature ranking or exact online gain. Token-wise oracle room is large but learned heads do not capture it.


Analysis is an agent's reading of the source. Nothing here has been reproduced unless the status says so.
