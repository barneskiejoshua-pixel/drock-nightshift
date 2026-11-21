# Cloak121210/Leafy

- Source: https://github.com/Cloak121210/Leafy (github, repo)
- Found: 2026-11-19
- Status: RESEARCHED (verification ladder: DISCOVERED < RESEARCHED < PROTOTYPED < TESTED < VERIFIED < INTEGRATED)
- Relevance score: 42.7 {"relevance": 24.0, "feasibility": -0.0, "evidence": 8, "novelty": 8.9, "community": 1.8}
- Topics: inference optimization,memory + retrieval,cpu + bandwidth,bitnet + quantization
- License: unknown (README says Apache-2.0, but the

## Problem

Offline Android LLM chat with low battery and thermal cost; not one of DRock's listed open problems.

## Technique

Four runtimes (llama.cpp, Bitnet.cpp, T-MAC, ExecuTorch) behind one Kotlin engine; tree-structured chat; local keyword RAG; ADPF power governor; telemetry that refuses to invent energy numbers; GGUF/PTE/TMAC byte validation before load.

## Foundation

Ternary weights {-1,0,+1} replace multiply with add; LUT matmul avoids dequantization; fixed-weight keyword overlap, not BM25: +2.5 per query term in a passage, +1.8 in the title, best passage per doc, top 2, 300 chars; prompt ceiling = (ctx - reserve - 32)*3 chars clamped 384..32768 with history-turn shedding; ADPF target duration = actual/fraction; energy = elapsed_s*mW/3.6 microWh only when hardware current and voltage exist.

## Compute

Android app, 64-bit arm64-v8a or x86_64, min SDK 24 / target 36, JDK 11; n_gpu_layers=0 so CPU inference; bundled .so files about 42 MB per ABI (two librnllama ~13 MB each, libbitnet 5.2 MB, libtmac 1.9 MB, own JNI 71 KB); user-supplied GGUF/PTE/TMAC weights, no models in repo; ADPF hints only API>=31; energy needs BatteryManager current and voltage.

## Reported

README claims up to 71% CPU energy cut for 1.58-bit i2_s and Eco Leaf under 34 C / -62% power with 2 threads, Balanced 4 threads Q8_0, Turbo 8 threads, contexts 1024/4096/8192. No device, model, baseline, prompt, run count or confidence given. Tests assert different thread counts (Eco 1, Balanced 2, Turbo 2) and ADPF targets 20 ms / 20 ms / 5 ms for a 10 ms step at fractions 0.50/0.50/1.00. Telemetry test uses a fake clock: 60 prompt tokens, 500 ms TTFT, 11 tokens, 20 tok/s decode, energy -1 when milliwatts=0. Repo: 7 stars, 12 commits, v0.9.9.4 / code 25.

## Reproducibility

Source and tests are public (git tree sha 45a7e463..., raw files fetched). Build: ./gradlew assembleDebug. Tests use synthetic GGUF/PTE bytes and a fake clock; no model weights, benchmark logs, dataset, or device results in repo. Default sampler seed -1 in tests. I did not run Gradle or any device test here.

## Drock application

Verification and telemetry: emit -1 or 'unmeasured' instead of a plausible number when no measurement exists; validate artifact bytes before reporting success; prompt-budget ceiling with explicit shed order. None fixes a listed failing test directly.

## Modifications

none; read-only fetch and file reads

## Limitations

Percentages in README are unsourced and stale versus the tests; README calls RAG BM25 but code uses fixed keyword weights; T-MAC with qwen3/type-41 GGUF is skipped and a .tmac test expects NotImplementedError; energy is -1 without hardware current and voltage; metadata.json declares MAJOR_CAPABILITY_SERVER_SIDE_GEMINI_API and .env.example has a commented GEMINI_API_KEY template, both AI Studio scaffold leftovers and not evidence of a live cloud call; no APK, no benchmark data.


Analysis is an agent's reading of the source. Nothing here has been reproduced unless the status says so.
