# rajatslakhina/constrained-decoding-kit

- Source: https://github.com/rajatslakhina/constrained-decoding-kit (github, repo)
- Found: 2026-11-19
- Status: RESEARCHED (verification ladder: DISCOVERED < RESEARCHED < PROTOTYPED < TESTED < VERIFIED < INTEGRATED)
- Relevance score: 45.2 {"relevance": 25.0, "feasibility": -0.0, "evidence": 12, "novelty": 8.2, "community": 0.0}
- Topics: tool selection,cpu + bandwidth
- License: MIT

## Problem

An on-device decoder must accept every model string, so prompted JSON still fails in production. The repo builds a Swift decoder that drops any token which would leave a bounded JSON schema, and that is required to finish.

## Technique

JSONSchemaConstraint is a pushdown matcher over one canonical compact encoding (no insignificant whitespace, keys in declaration order). MaskComputer walks a byte trie of the vocabulary and prunes a token at the first rejected byte. ConstrainedDecoder (Decoder.swift) looks up that mask in a 512-entry LRU keyed by the whole frame stack, samples only the allowed ids, and on a mask of size 1 skips the model. The step budget defaults to maxOutputBytes+1 and is clamped to SchemaLimits.maxEncodedBytes+1 (1 MiB + 1). SchemaValidator is a separate recursive-descent checker.

## Foundation

Boundedness: every string, integer, and array has a max, every property is required, and a compiled schema is at most 4096 nodes and 1<<20 encoded bytes, so the language is finite. Each non-EOS token is at least one byte, so after that many tokens only EOS can remain. Mask equality is two algorithms (trie walk and a per-token scan). Sampling renormalises the temperature-scaled logits over the allowed ids; greedy ties take the lowest id; the RNG is SplitMix64. Masked mass is the probability removed from the distribution actually sampled.

## Compute

Swift 6.0+. Package.swift declares iOS 17 and macOS 14. Each model call must return a [Float] of vocabulary size with EOS last. Mask work is a CPU trie walk on a cache miss. There is no GPU mask path. Every published number uses ReferenceTextModel, an n-gram copy model over one fixture string (max 2048 bytes, overlap 8, default noise 0.6), not a neural net and not the Arc A380.

## Reported

README and DecoderTests.testDocumentedSeedSweep: ReferenceTextModel, ticket schema, temperature 1.4, seeds 0..<20, same sampler: SchemaValidator accepts 0/20 unconstrained outputs and 20/20 constrained outputs. testGreedyGoldenOutput, greedy, seed 0: the pinned compact ticket JSON, 41 steps, 7 forced, 34 model calls; masked mass > 0.99 at steps 0 ({"), 18 (high"), 24 (,), 28 (true); steps 3..<13 under 0.06. testConstrainedOutputIsAlwaysSchemaValid: greedy and temperatures 0.7, 1.4, and 3, seeds 0..<15, every output EOS and validator-valid. testTriePruningNeverCostsMoreThanTheNaiveScan asserts the trie never uses more advance calls than the naive scan and is strictly cheaper overall on 4 random-walk seeds; it prints the totals. The README's 88988 vs 149645 is that print from one run, not an asserted constant. README: Linux Swift 6.0.3, 65 tests, 0 failures, 0 warnings. GitHub Actions run 40062720685 on main 2dc1f59 (2026-10-24) concluded success for the Linux Swift 6.0 job and the macOS 15 job (swift test plus iOS Simulator build). The jobs payload does not include the test count. README: no real-model latency or throughput number exists. I did not run swift test.

## Reproducibility

Full Swift package, tests, fixtures, and CI workflow are in the repo. Tag v1.0.0 is commit 258e961; main at read time is 2dc1f59. Seeds and the SplitMix64 first output (seed 0 -> 0xE220A8397B1DCDAF) are pinned. No weights and no external dataset. README says bit-exact results across operating systems are not promised because exp/log come from the platform libm.

## Drock application

DRock already forces one schema-shaped action per turn inside Ollama (agent.py constrain=True) and, on llama-server, with gbnf.py when DROCK_GRAMMAR=gbnf. That grammar emits reason, the action name, then parameters in declaration order, including optional parameters. This Swift masker would apply only if DRock pulled raw logits and masked them in process. On the i5-8400 that is a Python loop of one forward pass per token. The Arc's 6 GB is unused by a CPU [Float] mask.

## Modifications

None for the listed failures. The schema subset rejects optional properties, floats, escapes, and non-ASCII, so a Windows path or a reason containing a quote cannot be encoded. Recording masked mass from the llama.cpp grammar DRock already runs would be a trace field on the current stack, and it would still leave the gate to judge the artifact.

## Limitations

The demo model is an n-gram, and the README says the numbers say nothing about real-model quality or latency. Strings are printable ASCII with no escapes. No floats, no optional properties, no anyOf, no free whitespace, no key reordering. Logits are copied to a CPU array each step. Validity is checked apart from whether the content is the right action: the golden decode's spikes are the model wanting prose, "P1", ".5", and "yes", which the mask replaces. A caller maxTokens below the bound stops with an incomplete document.


Analysis is an agent's reading of the source. Nothing here has been reproduced unless the status says so.
