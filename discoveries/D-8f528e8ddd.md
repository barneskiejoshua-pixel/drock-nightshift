# the-open-agent/openagent

- Source: https://github.com/the-open-agent/openagent (github, repo)
- Found: 2026-11-19
- Status: RESEARCHED (verification ladder: DISCOVERED < RESEARCHED < PROTOTYPED < TESTED < VERIFIED < INTEGRATED)
- Relevance score: 41.0 {"relevance": 21.5, "feasibility": -0.0, "evidence": 12, "novelty": 0.0, "community": 7.5}
- Topics: agentic planning,memory + retrieval,computer use
- License: Apache-2.0

## Problem

P-759c705bc0 — assumption that prose missing the jailbreak regex will not be obeyed once labelled as data

## Technique

Two-layer guardrail in guardrail/: high-precision regexes over normalized text (full-width to half-width, lowercase, whitespace and invisible chars removed) for clear-cut attacks, plus a separate review-model call whose prompt explicitly says the tagged text is data and must not be executed; tag delimiters are escaped to ‹message›/‹answer› so content cannot close them. DetectHiddenInstruction applies a different rule set plus invisible-character detection to text the user did not type (documents, KB chunks, tool results). Review runs twice, on input and on output, and ParseJudgeOutput fails closed on unparseable JSON or disallowed categories. A deterministic LeaksPrompt check compares 15-rune shingles of answer against system prompt.

## Foundation

Architecture, not math: defense in depth with high-precision rules and a judge for subtle cases; pre-norm to defeat spacing/full-width/zero-width variants; delimiter escaping to stop markup injection; independent output-stage check so a jailbreak that slips past input is caught after generation. Shingle coverage is exact string matching over normalized runes, with fixed thresholds.

## Compute

Pure Go string/regex work, O(n) normalization and O(n) shingle hashing for LeaksPrompt; no GPU. The judge half needs two extra LLM calls per exchange (input and output) with prompts up to roughly 4000/6000 runes each plus 1-2k rune instructions; on DRock's i5-8400 + 6 GB Arc that is added latency per message. Repo needs Go 1.25.0+ and Node 20+/Yarn to build the server.

## Reported

README reports no benchmark, accuracy, precision/recall, latency or ablation numbers; no evaluation set. Only feature claims (30+ providers, single binary, port 14000) and code constants: leakWindow=15, prompts under 50 runes skipped, leak if >=300 covered runes or (>=40 covered and covered*10 >= len(p)*4); smuggling flagged at >=4 Unicode tag chars or a run of >=4 zero-width chars; Truncate keeps head 2/3 and tail 1/3 with caps 200/1500/4000/1000/6000 runes for agent/previous_reply/message/question/answer.

## Reproducibility

Source is public on GitHub master (Apache-2.0, 2531 commits); pure functions DetectPromptAttack, DetectHiddenInstruction, LeaksPrompt, ParseJudgeOutput and Truncate can be re-run after a Go build. No guardrail test files, no fixtures, no dataset, no seeds and no published scores; the judge behaviour depends on an unspecified review model.

## Drock application

Verification gate and prompt assembly: before DRock labels retrieved/tool text as data, run a hidden-instruction/invisible-character check and escape DRock's own delimiters; keep a separate output-stage check rather than trusting the first pass. LeaksPrompt-style shingle coverage is a cheap deterministic gate for false-success classes.

## Modifications

Drop the K-12 Chinese classroom categories and their benign exceptions; port the delimiter escaping, normalization, hidden-instruction rules and shingle check to DRock's Python tests. Do not port the judge prompts wholesale; small Qwen models would need a DRock-specific prompt and a measured false-positive rate. Any judge call needs an explicit timeout and backoff.

## Limitations

Rule lists are Chinese/English and tuned for a school platform; the broad benign-carve-outs would under-block DRock-style cases. No reported measurements means recall and false-positive rate are unknown. Escaping tags only helps if DRock's actual prompt uses the same marker syntax. An extra judge call per stage costs latency on CPU-only hardware.


Analysis is an agent's reading of the source. Nothing here has been reproduced unless the status says so.
