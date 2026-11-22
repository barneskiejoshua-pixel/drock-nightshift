# Experiment E-c3ef56ae9e: User-stated facts vanish between chat turns and the agent

- Problem: P-dfe5dfcbee
- Arm: repro
- Base snapshot: 6183480518c8649778ea68595a5e67b5f19fe87e
- Verdict: **not_reproduced** (test writer changed nothing, wanted only tests/test_ns_p_dfe5dfcbee.py; p=false has_lsp=false
2026-11-22T06:38:59.253938Z  INFO plugin discovered name=open-code-review scope=user root=<home>\.claude\plugins\marketplaces\open-code-review\plugins\open-code-review sk)
- Date: 2026-11-22 00:04

## Hypothesis

reproduce User-stated facts vanish between chat turns and the agent

## Measurements

```json
{
 "problem": "P-dfe5dfcbee",
 "ref": "6183480518c8649778ea68595a5e67b5f19fe87e",
 "arms": {},
 "baseline": {
  "ran": 4990,
  "failing": 32,
  "cached": false
 },
 "repro": "test writer changed nothing, wanted only tests/test_ns_p_dfe5dfcbee.py; p=false has_lsp=false\n2026-11-22T06:38:59.253938Z  INFO plugin discovered name=open-code-review scope=user root=<home>\\.claude\\plugins\\marketplaces\\open-code-review\\plugins\\open-code-review sk"
}
```

## Debate (Claude proposes, Grok objects)

```json
{
 "proposal": null,
 "critique": null,
 "rebuttal": null,
 "final": null
}
```

