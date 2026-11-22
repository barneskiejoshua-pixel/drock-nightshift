# dmis-lab/CLAG

- Source: https://github.com/dmis-lab/CLAG (github, repo)
- Found: 2026-11-19
- Status: RESEARCHED (verification ladder: DISCOVERED < RESEARCHED < PROTOTYPED < TESTED < VERIFIED < INTEGRATED)
- Relevance score: 37.1 {"relevance": 26.5, "feasibility": -0.0, "evidence": 8, "novelty": 0.0, "community": 2.6}
- Topics: small-model reasoning,agentic planning,memory + retrieval
- License: MIT (README badge only; LICENSE at repo 

## Problem

Agent memory for small language models. The paper argues that one global retrieval pool lets unrelated memories in. Small models are easily thrown off by irrelevant context, so retrieval and memory rewriting both get worse as the store grows.

## Technique

Each memory is a note with content, keywords, tags, an LLM-written context line and links, as in A-mem. Clusters start with KMeans once 100 notes are buffered (init_n_clusters=3). After that, the SLM routes each new note to one of the 5 nearest centroids, reading a profile for each: a one-sentence summary, 3 one-word tags and 3 snippets of 160 characters. If cosine similarity to the chosen centroid is below 0.1, the note opens a new cluster (per the paper). A cluster over 300 notes is split in two by KMeans. Evolution ('strengthen' / 'update_neighbor') rewrites only the top-5 neighbours inside the routed cluster. At query time, the query embedding picks 5 candidate centroids. The SLM keeps 0 to 10 of them, and cosine top-5 runs only inside those clusters plus linked neighbours. If no cluster is selected, it falls back to global top-5.

## Foundation

Embedding similarity (all-MiniLM-L6-v2, cosine), KMeans with random_state=42 and n_init=10, and centroid distance for the coarse filter. Routing, evolution and cluster selection are all prompted calls to the same backbone SLM. Nothing is trained.

## Compute

One sentence-transformers model, hard-coded to device='cuda', plus the backbone SLM through SGLang (default localhost:30000), Ollama via LiteLLM, or OpenAI. Each add_note makes 2 LLM calls before clusters exist and 3 after (metadata, routing, evolution). Each query adds 1 selection call. Each cluster profile build costs 1 call per cluster, at init and every 100 evolutions. Calls time out at 60 s with max_new_tokens=1000. The paper's backbones are 0.6B to 1.5B models.

## Reported

Paper Table 1, F1 / BLEU-1 on LoCoMo, HotpotQA and BioASQ (the authors rebuilt BioASQ in HotpotQA style with distractors). Qwen3-0.6B: CLAG 20.99/17.88, 15.50/14.33, 22.01/17.23; best baseline 16.05 (GAM), 12.04 (A-mem), 3.61 (A-mem). Llama-3.2-1B-Instruct: CLAG 21.05, 14.20, 10.16; GAM beats it on LoCoMo (22.63 vs 21.05). DeepSeek-R1-Distill-Qwen-1.5B: CLAG 15.46, 9.12, 6.18; best baseline 12.50, 6.45, 4.45. Baselines are RAG, A-mem, MemoryOS and GAM. Absolute scores are low everywhere, and the Qwen3-0.6B BioASQ jump from 3.6 to 22.0 is far larger than any other cell. I read pages 1 to 9 only. The latency table was cut off, so I have no latency numbers and no seed or variance reporting. The README reports no numbers.

## Reproducibility

The code is public: CLAG_memory.py and test_CLAG.py. LoCoMo and HotpotQA ship in data/. BioASQ needs a participant-area download plus run_prepare_bioasq_all.sh. KMeans seed is 42, but LLM calls for evolution and selection pass no temperature, so they run at default sampling, and no run seeds are given. The code has bugs. SimpleEmbeddingRetriever.search is annotated to return dicts but returns a numpy index array. Cluster tags are Python sets, which json.dumps cannot serialize. A force_top3 debug flag bypasses the selector.

## Drock application

The closest fit is exemplars.py:503 ExemplarStore.retrieve. It ranks past traces by goal BM25 plus STEP_WEIGHT times step-signature overlap, with a MIN_OVERLAP floor against one-word coincidences. A cheap, LLM-free version of CLAG's stage 1 would cluster exemplar goals offline and restrict BM25 to the query's nearest clusters. That could cut cross-topic exemplars once the store is large. DRock does not use CLAG's agentic memory-evolution loop, and the loop conflicts with the deterministic-executor design.

## Modifications

Drop all three LLM calls: no routing, no evolution, no SLM cluster selection. Each adds a 4B-model call and a new place for a false success. Keep only centroid filtering, deterministic and seeded. Run the embedder on CPU or Arc rather than CUDA, or cluster on the BM25 token vectors DRock already has. Score it with a script on the existing exemplar retrieval tests and benchmark traces, comparing exemplar precision and pass count before and after (a ShinkaEvolve-style evaluator, not a model's opinion).

## Limitations

The gains are on QA benchmarks with distractors and F1 in the 5 to 22 range. They are not on tool-using agents, and DRock's exemplar store is action traces, not conversation memories. Without seeds or variance, the single-run deltas cannot be checked against noise, which matters given DRock's P-00f0394126 concern about flip counts. Cold start needs 100 notes before any clustering. Cluster profiles regenerate only every 100 evolutions, so they go stale. With sampled LLM routing, the same input can land in different clusters on different runs. GAM beats CLAG on Llama LoCoMo. The code has the type and serialization bugs above and hard-codes CUDA.


Analysis is an agent's reading of the source. Nothing here has been reproduced unless the status says so.
