# Quantizing Diffusion LLMs under Parallel Decoding: Commitment Risk and the Parallelism Tax

- Source: http://arxiv.org/abs/2611.30587v1 (arxiv, paper)
- Found: 2026-11-24
- Status: RESEARCHED (verification ladder: DISCOVERED < RESEARCHED < PROTOTYPED < TESTED < VERIFIED < INTEGRATED)
- Relevance score: 38.9 {"relevance": 24.0, "feasibility": -0.0, "evidence": 0, "novelty": 14.9, "community": 0.0}
- Topics: routing + confidence,bitnet + quantization
- License: unknown; paper is arXiv.org perpetual no

## Problem

W4A4 post-training quantization of diffusion LLMs has been calibrated and scored under fixed-step samplers with no cache. Deployed Fast-dLLM decoding caches context and, in one step, commits every masked position whose confidence is at least a threshold, or forces the single most confident position when none clears it. Those commits are never revised. The paper says this decoder decides both where quantization hurts accuracy and how many forward passes a response takes.

## Technique

DECAL (DEcoder-aligned Commitment-risk cALibration) reweights FlatQuant layer-wise hidden-state MSE by a position prior from full-precision trajectories of the deployed threshold decoder, collected with no cache. Flip score pi is a Gaussian-CDF model of token-gap and, on forced commits, placement-gap failure; reach a grows with how many later passes still read a returned token; risk r is the mean of pi times a; weights are an affine map with coefficient of variation c. A commit temperature T<1 is applied only on cache-reuse steps: softmax(z/T) is used for the threshold test and for which position is forced, while the written token stays argmax(z). T_match is the largest T in (0,1] whose mean forward-pass count is at most the FP16 count, fit from forward passes only.

## Foundation

Threshold decoding is a margin test: at tau=0.9, two-candidate confidence needs a top-logit gap of log(tau/(1-tau))=2.20 nats. Quantization is modeled as shrinking those gaps (Delta_q approx kappa Delta) while mostly keeping the argmax, which raises the effective threshold and adds forward passes (the parallelism tax). Forced commits are the low-margin decisions the threshold would have refused. Proposition 1: for T in (0,1], content is invariant and the number of commits is monotone in T. Calibration is FlatQuant reconstruction with FAIR-Calib-style position weights, not a new quantizer.

## Compute

LLaDA-8B-Instruct and Dream-v0-Base-7B, W4A4. Prior from about 640 5-shot GSM8K training prompts under the deployed rule, no cache: 2.8 GPU-hours on Dream (153 forward passes per prompt); LLaDA about half that. Calibration: 128 WikiText-2 sequences, length 1024, last K=256 as the window. Decode: Fast-dLLM, dual cache, tau=0.9, block 32, 256 generated tokens, on Fudan CFFF. GPU model is not stated. Main accuracy tables use T=1; temperature is a separate sweep. Not runnable on DRock's i5-8400 and Arc A380 6GB.

## Reported

Conditions unless noted: W4A4, Fast-dLLM, dual cache, tau=0.9. FlatQuant, FAIR-Calib, and DECAL are mean±std over 3 calibration runs; FP16 and QuaRot are one run. Tasks: GSM8K 5-shot flexible exact match, HumanEval 0-shot pass@1, MBPP 3-shot pass@1, MATH 4-shot Minerva, 200 problems per subject. Table 1 accuracy % LLaDA FP16 78.70/36.59/36.40/30.79 avg 45.62; QuaRot 74.30/33.54/32.20/26.86 avg 41.73; FlatQuant 75.44±1.07/35.57±3.97/35.80±2.43/28.21±0.22 avg 43.76; FAIR-Calib 75.84±0.88/35.77±1.86/36.12±1.41/28.43±2.29 avg 44.04; DECAL 76.65±0.46/36.21±0.93/36.08±1.06/28.67±0.24 avg 44.40. Dream FP16 73.16/53.05/51.00/30.79 avg 52.00; QuaRot 53.30/23.78/37.80/17.64 avg 33.13; FlatQuant 66.34±1.05/40.85±3.66/47.40±1.39/23.57±0.97 avg 44.54; FAIR-Calib 66.47±1.47/40.88±4.25/46.33±2.19/23.43±0.86 avg 44.28; DECAL 67.27±1.08/41.81±2.11/46.20±1.78/24.60±0.72 avg 44.97. Paper: QuaRot loses 3.9 avg points on LLaDA and 18.9 on Dream; DECAL best avg and best GSM8K, HumanEval, and MATH; MBPP within one std; DECAL std lower than FAIR-Calib on all 8 model-task pairs (examples 0.46 vs 0.88 LLaDA GSM8K, 2.11 vs 4.25 Dream HumanEval, 0.24 vs 2.29 LLaDA MATH). No cache, one run: LLaDA avgs FP16 48.09, QuaRot 43.71, FlatQuant 45.03, FAIR-Calib 46.24, DECAL 46.34; Dream 51.86, 35.71, 43.37, 44.98, 47.52. FlatQuant forced commits are 16.5% (LLaDA) and 29.2% (Dream) of commits but 97.1% and 88.8% of permanent token flips (99.5% and 96.7% with placement flips); per-commit flip 6.0% and 9.8% forced vs 0.04% and 0.13% threshold. Tax up to 13% extra passes on LLaDA QuaRot (0.1% to 13.3% across settings). Paired width loss 13.4% vs end-to-end 14.6% on 400 GSM8K questions. Where FP would commit and W4A4 withholds, same token in 97.9% of cases vs 68.3% over all masked positions. QuaRot LLaDA top-2 gap shrinks 26% with logit std unchanged. T_match=0.873 returns QuaRot LLaDA to FP16 pass count; accuracy moves by less than 0.6 points, inside the paper's single-run noise ±2.4. Dream T_match on 50 GSM8K prompts removes 2-7% taxes with accuracy changes below 0.6 points. Hyperparameters: LLaDA (epsilon,c)=(0.01,0.30) with FAIR-Calib window scale; Dream (1,0.15) with unit-mean window weights. c set once per model on the GSM8K test set, then frozen. Cache-vs-no-cache prior on LLaDA GSM8K: 76.45±0.24 vs 76.65±0.46, three runs. Appendix: repairing only forced commits fixes about nine in ten broken answers; those steps are 44% of LLaDA passes and 68% of Dream passes, so a second FP pass there is not the proposed fix. Polestar: DECAL 75.76 vs FlatQuant 74.68 GSM8K; QuaRot tax 15.0%. Fast-dLLM v2-7B: DECAL +1.1 GSM8K and +0.9 MATH over FlatQuant.

## Reproducibility

No code, data release, or numeric seeds in the v1 paper. Reproducibility statement says code for the prior, weighted calibration, commit temperature, and evaluation will be released. Setup, equations, and appendices are in the paper. Models are public (LLaDA-8B-Instruct, Dream-v0-Base-7B); calibration text is WikiText-2; prior prompts are 5-shot GSM8K training. Three calibration runs under dual cache; no-cache tables are one run. c is chosen on the GSM8K test set. A search on 2026-11-24 found no repository for arXiv:2611.30587.

## Drock application

none

## Modifications

None inside DRock. DECAL is a calibration weight on FlatQuant for 7-8B masked diffusion models, and the temperature is a change to a confidence-threshold commit rule. DRock samples small autoregressive Qwen models through Ollama with a deterministic executor and a verification gate. It has no diffusion canvas, no threshold commit, no KV-cache block decoder, and no W4A4 calibration loop.

## Limitations

Reported: the prior is tied to block length and threshold; one decoder family and one bit width in the main study; product pi*a is within 0.1 avg points of the stronger single factor; shuffled weights do not explain the gain; temperature below T_match is an explicit accuracy-cost trade. Opinion: the cached accuracy edge over FAIR-Calib is small next to run std (LLaDA avg 44.40 vs 44.04; several tasks overlap), and c was set on the GSM8K test set that supplies the headline GSM8K number. No released code means the tables cannot be re-run from this source. The Gaussian flip model is a ranking score, not a calibrated probability.


Analysis is an agent's reading of the source. Nothing here has been reproduced unless the status says so.
