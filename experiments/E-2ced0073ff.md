# Experiment E-2ced0073ff: assumption: generating reason before the action name produces a better action

- Problem: P-86ab325f96
- Arm: A-grok
- Base snapshot: 57262310da7a00efea38b9916d5298abb414a2c5
- Verdict: **vetoed** (audit: replay failed: ", "The tests for mathutil.py fail. Fix mathutil.py so they pass without changing the tests.", "In the folder <home>\\Desktop run the command: python -m pytest drock/tests/test_native_trace.)
- Date: 2026-11-24 01:20

## Hypothesis

Reordering the ordinary json_schema and GBNF branches to action const, then reason, then params fixes it because the action name is then the first sampled choice and the 20-45 token rationale no longer conditions that distribution.

## Measurements

```json
{
 "files": [
  "gbnf.py",
  "schema.py",
  "tests/test_reliability.py"
 ],
 "diff_lines": 21,
 "target": {
  "rc": 0,
  "failing": [],
  "seconds": 0.8,
  "ran": 3
 },
 "suite": {
  "ran": 5171,
  "failing": 42,
  "new_failures": [],
  "also_fixed": [],
  "seconds": 1397.5,
  "base_seconds": 738.5
 },
 "audit": "audit: replay failed: \", \"The tests for mathutil.py fail. Fix mathutil.py so they pass without changing the tests.\", \"In the folder C:\\\\Users\\\\barne\\\\Desktop run the command: python -m pytest drock/tests/test_native_trace.",
 "verdict": "vetoed",
 "why": "audit: replay failed: \", \"The tests for mathutil.py fail. Fix mathutil.py so they pass without changing the tests.\", \"In the folder C:\\\\Users\\\\barne\\\\Desktop run the command: python -m pytest drock/tests/test_native_trace.",
 "agent_seconds": 208,
 "agent_ok": true
}
```

## Debate (Claude proposes, Grok objects)

```json
{
 "proposal": {
  "root_cause": "schema.json_schema (schema.py:651-663; the cited schema.py:577 is now call_verdict) builds every ordinary branch with properties and required starting reason then action, and gbnf.gbnf (gbnf.py:153-154) emits that same order. Constrained decoding walks schema order, so the sampler writes up to reason_max() free-text characters before the action const. research/search.py:_branch already uses action then reason so the name is sampled at the decision point; ordinary turns do not. tests/test_reliability.py:143 locks the old order.",
  "hypothesis": "Reordering the ordinary json_schema and GBNF branches to action const, then reason, then params fixes it because the action name is then the first sampled choice and the 20-45 token rationale no longer conditions that distribution.",
  "plan": [
   "In schema.json_schema, set props to action const then reason (maxLength reason_max()) and required to [\"action\", \"reason\", *req], leaving param order and constraint merge as they are. Replace the comment that calls a later reason a mere rationalisation.",
   "In gbnf.gbnf, emit the \"action\" key and the name const before the \"reason\" key and the reason rule. Update the module docstring that lists reason then action.",
   "Invert tests/test_reliability.py TestNoEscalation.test_reason_is_generated_before_the_action to expect [\"action\", \"reason\"]. Keep the test.",
   "Do not route ordinary turns through search.action_first_schema and do not change protocol_doc, scaffold.Decoder, or search._branch. Native _one_of already discriminates on the first all-const key, so it follows the schema order.",
   "Run python -m unittest drock.tests.test_ns_p_86ab325f96 and drock.tests.test_reliability.TestNoEscalation.test_reason_is_generated_before_the_action."
  ],
  "files": [
   "schema.py",
   "gbnf.py",
   "tests/test_reliability.py"
  ],
  "risks": [
   "Changing only json_schema or only gbnf fails half of test_ns_p_86ab325f96. Property lists and required must stay identical to search._branch or test_same_property_order_as_search_turns fails.",
   "protocol_doc still exemplifies {\"reason\" first, \"action\" second}. Grammar-off turns and the model's first-token preference still write the reason first. Search turns already live with that mismatch. Editing the prompt is out of this fix.",
   "LiveThought still finds \"reason\" (second key, under the 400-char give-up) but the reason now streams after the action is chosen. scaffold.Decoder, README \u00a74, and the action_first_schema docstring still say the ordinary turn keeps reason first and will tempt a revert."
  ]
 },
 "critique": {
  "objections": [
   "The problem is an untested assumption, and the proposal replaces it with the opposite assumption, also untested. Nothing in the repo shows action-first gives better actions. research/model-internals/parts/part2.txt:231 and part4.txt:776 list action-first versus reason-first as an untested A/B and name the test (split A/B, reflex bench and capbench, 2 runs each). BUILD-PLAN 4.1 says the search-1 A/B is queued, not run. The plan's only check is unit tests on key order, so it gives no accuracy number.",
   "The logprob argument does not apply to ordinary turns. Per-token logprobs are requested only at research/search.py:422, which already uses action_first_schema, and at chat.py:1900 for intent. agent.py never asks for logprobs. On an ordinary turn the flip gives no better measurement and only changes how the action is picked, in a direction nobody has measured.",
   "The reason_max docstring (schema.py:622-626) cites arXiv 2604.02155: the best length for routing reasoning is 8-32 tokens, which is more than zero. On a non-thinking 4B tier, the reason is the only computation done before the action name. Action-first takes that to zero and keeps the cost, because the same 20-45 tokens are still generated after the decision.",
   "The prompt and the grammar would disagree, in more places than the proposal lists. agent.py:33 says 'Write \"reason\" FIRST', agent.py:67 says '\"reason\" first', schema.py:682 gives a reason-first example, and exemplars.py:76 FILLER says 'The reason field comes first'. The proposal names only protocol_doc and leaves all four. On Ollama and llama-server the grammar blocks the \"reason\" key at the first token of every turn, so a 4B model told three times to write reason first is pushed off its preferred path at token 1.",
   "agent.py:36 ('If a result has ok:false, your reason must name the cause from the error; then try a different action') steers the choice only when the reason comes before the action. With action first, the model can pick the action that just failed again before it writes the cause. The proposal has no check for this no-repeat-after-failure path, which is where reason-first is most likely to help.",
   "The branch shape is still kept by hand in three places: schema.json_schema, search._branch and the hard-coded parts list in gbnf.gbnf:153-154. The proposal edits two of them by hand and relies on test_same_property_order_as_search_turns to catch drift. Deriving all three from one builder is cheaper and stops drift by construction.",
   "gbnf._lit(s) returns json.dumps(s). As a GBNF literal that matches s without its quotes. So the keys reason and action, the action const, and string enums come out unquoted, and '\":\"' and '\",\"' match a bare ':' and ','. DROCK_GRAMMAR=gbnf would then produce {reason:\"..\",action:READ_FILE,...}, which is not JSON. I read this from the code and could not run it; this review had no shell. Until that is fixed, reordering the GBNF changes nothing anyone can measure. The target test's substring checks pin the current literal text, so the quoting fix has to go in as its own change.",
   "No receipt records which key order produced a run. The env list at benchmark/run.py:317 has no order flag, so benchmark scores from before and after the merge cannot be told apart, and any shift will be blamed on something else.",
   "README.md:257 and CAPABILITY-STACK.md:173 (8.1) will state the old order as fact after the merge, and so will the action_first_schema docstring at research/search.py:152. Once the order matches, the only difference between the search grammar and the ordinary grammar is that action_first_schema does not drop PLANNER_ONLY (search.py:153). That means USE_SKILL is legal on search turns. The proposal does not notice this."
  ],
  "root_cause_agrees": false,
  "alternative": {
   "hypothesis": "The defect is that the key order is a claim written into three hand-copied grammar builders, with no switch and no measurement behind it. One branch builder in schema.py takes its order from one switch: DROCK_REASON_FIRST=1 gives reason first, and unset gives action first, so the target test passes by default. search._branch and gbnf read their order from that builder, and the prompt line and receipts follow the same switch. The order then cannot drift between json_schema, search and GBNF, the prompt cannot contradict the grammar, and the reflex bench plus capbench A/B can decide whether action-first actually picks better actions before README makes either claim.",
   "plan": [
    "schema.py: add action_branch(name, constraints, action_first=None), the current search._branch body. Order is action, reason, then params when action_first is True, or when it is None and DROCK_REASON_FIRST is not '1'; otherwise reason, action. json_schema becomes [action_branch(n, cons) for n in ACTIONS if not excluded and not PLANNER_ONLY]. Delete the 'only a rationalisation' comment.",
    "research/search.py: make _branch call schema.action_branch(name, cons, action_first=True) and delete the duplicate body. Search turns stay action-first under either switch value, because top_actions needs the action tokens first.",
    "gbnf.py: drop the hard-coded reason/action parts. Walk action_branch(name, cons)['properties'] in order: action emits the key and _lit(name), reason emits the 
```

