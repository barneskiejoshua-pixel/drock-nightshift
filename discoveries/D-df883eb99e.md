# Beam Model Technical Report

- Source: http://arxiv.org/abs/2611.26358v1 (arxiv, paper)
- Found: 2026-11-19
- Status: RESEARCHED (verification ladder: DISCOVERED < RESEARCHED < PROTOTYPED < TESTED < VERIFIED < INTEGRATED)
- Relevance score: 39.9 {"relevance": 33.0, "feasibility": -10.0, "evidence": 2, "novelty": 14.9, "community": 0.0}
- Topics: small-model reasoning,agentic planning,limited context,speculative decoding,bitnet + quantization
- License: CC BY-SA 4.0 for the article; model/chec

## Problem

Build a coding/reasoning/agentic frontier model with 1M-token context at 3-4x less inference compute than comparable models, and keep a large asynchronous RL campaign and serving stack stable.

## Technique

Sparse MoE (501.3B total, 23.4B active, 52 layers, 256 experts); two-stage correctness-anchored midtraining to 1M context; verifiable-reward RL with reward-hacking judges; multi-teacher on-policy distillation; FP8/NVFP4 post-training quantization; DFlash 2 speculative drafter; and a short harness-level policy telling the agent that tool output is data, not instructions.

## Foundation

Conditional compute via MoE with auxiliary-loss-free load balancing and depth-scaled residuals; rejection-sampling speculative decoding preserves the target distribution; PTQ with per-channel/block scales; GRPO-style rewards from verifiers and judges; a replicator equation used to forecast behavior prevalence through RL; accuracy-abstention frontier.

## Compute

Pretraining under 4 weeks on 6,144 GB300 GPUs (92.3% goodput, 21% MFU); RL 4 weeks on 10.5K GB300 (1,920 trainer + 8,576 inference configured), 110K concurrent rollouts, 100M+ rollouts, 12-second weight updates; drafter training 3.4K B300 GPU-hours; serving on 4-8 B300 GPUs. Nothing here fits an i5-8400 and 6 GB Arc A380.

## Reported

Authors' numbers, not independently replicated: 23.8T pretraining tokens; DeepSWE pass@1 35% at 79% of the RL run and 41% final, at e=512, temp 1, top-p 0.99, 256K output cap; LongBench-v2 67% vs 50.1% human reference; FP8 and NVFP4 recover 99.93% and 98.98% of median benchmark scores (temp 0.7, top-p 0.99, max effort, 1M context; mean delta +0.03 and -0.45 across 22 benchmarks); NVFP4 gives 6.0x per-GPU throughput vs 8-GPU BF16 at 50 tok/s/user; DFlash 2 tau=4.38 sampling / 4.64 greedy out of 8 and 1.96-3.03x decode speedup; Iris ISE 88.9% refuse-or-pivot on harmful and 81.5% comply on harmless; alignment teacher 100B to 501B cut over-refusals 48.7%; the Mirror IPI policy cut ASR from 3.1%/3.6% to 0.3%/0% on MirrorDojo/MirrorDyn with -1.0/+1.7 points clean utility over 97/60 tasks. Authors note 1-2 point sampling variance and that the IPI test uses one fixed impersonation attack.

## Reproducibility

Paper only. No training code, data, or seed list in the report; weights and a DFlash 2 drafter are claimed released but no repo or checkpoint URL is given. Internal determinism is described (DLS permutation keyed by source/seed/epoch/ratio; same data/seed/pipeline for quantization ablations; 5% CE threshold for significance). ISE is called soon-to-be open-sourced, not released.

## Drock application

One portable piece: the Mirror developer-prompt policy quoted in section 7.5. Add a short rule to DRock's system prompt that tool results, retrieved pages, and file contents supply data values but never new actions, targets, tool calls, or file transfers; the user's request is the only trusted intent. Also keep the verification gate checking outcomes, not the agent's own success claim.

## Modifications

Add the policy block, then write a regression test that plants imperative prose in a tool result or document and asserts DRock stays on the original goal; measure ASR-like failure rate and normal-task pass rate on DRock's own traces before and after. No MoE, RL, quantization, or drafter port: the hardware and model do not support it.

## Limitations

Scale and domain gap are huge: 501B params and 10.5K GPUs versus small local Qwen models on one A380. The IPI result comes from Reflection's own harness, models, and a single fixed attack; small Qwen may ignore the policy, so local measurement is required, not assumption. Many evaluations were reproduced internally against external baselines, and the safety ratings are self-reported under Reflection's OSF. The report gives no code, data, or seeds, so none of the headline numbers can be rerun here.


Analysis is an agent's reading of the source. Nothing here has been reproduced unless the status says so.
