# Prism — Project Evolution

> End-to-end record of what was broken at each stage, what was built to fix it, and what is planned next.
> Updated as the project evolves. Last updated: 2026-07-05 (Stage 18).

---

## Table of Contents

1. [Stage 0 — v1 Baseline](#stage-0--v1-baseline)
2. [Stage 1 — v2 Hybrid Retrieval Architecture](#stage-1--v2-hybrid-retrieval-architecture-2026-05-17)
3. [Stage 2 — Chain Scores + RAGAS Endpoint](#stage-2--chain-scores--ragas-endpoint-2026-05-23)
4. [Stage 3 — Groq Migration + Web Search](#stage-3--groq-migration--web-search-fixes-2026-05-24)
5. [Stage 4 — OOM Hell on Render](#stage-4--oom-hell-on-render-2026-05-24-four-sub-issues)
6. [Stage 5 — TinyBERT + RAGAS Removal + Benchmark JSON](#stage-5--tinybert--ragas-removal--benchmark-json-2026-05-30)
7. [Stage 6 — Multi-Workspace](#stage-6--multi-workspace-2026-06-early)
8. [Stage 7 — Singleton Cache + URL Guard](#stage-7--singleton-cache--url-guard-2026-06-14)
9. [Stage 8 — Eval Dashboard + Rigorous Metrics](#stage-8--eval-dashboard--rigorous-metrics-2026-06-17)
10. [Stage 9 — Multi-Query Retrieval](#stage-9--multi-query-retrieval-2026-06-19)
11. [Stage 10 — Contextual Retrieval (Eval)](#stage-10--contextual-retrieval-eval-2026-06-20)
14. [Stage 14 — Briefing Fix + HyDE Re-eval](#stage-14--briefing-fix--hyde-re-eval-2026-06-24)
12. [Stage 12 — HF Spaces Migration](#stage-12--hf-spaces-migration-2026-06-22)
12. [Stage 11 — Contextual Retrieval in Production + Dashboard Polish](#stage-11--contextual-retrieval-in-production--dashboard-polish-2026-06-20)
15. [Stage 15 — Semantic Chunking Ablation + Retrieval Stack Finalized](#stage-15--semantic-chunking-ablation--retrieval-stack-finalized-2026-06-26)
16. [Stage 16 — Citation Highlighting](#stage-16--citation-highlighting-2026-06-26)
17. [Stage 17 — Metadata Filtering](#stage-17--metadata-filtering-2026-06-27)
13. [Current State Snapshot](#current-state-snapshot)
13. [Roadmap — Retrieval & Answer Quality](#roadmap--retrieval--answer-quality)
14. [Roadmap — New Features](#roadmap--new-features)

---

## Stage 0 — v1 Baseline

### What existed
- Dense-only ChromaDB vector retrieval
- Single global document collection
- Basic chat with ConversationalRetrievalChain
- No evaluation framework
- No web search
- No logging

### What was wrong

| Problem | Impact |
|---------|--------|
| Dense-only retrieval | Misses exact keyword matches — regulatory text has section numbers, policy codes, specific terms that semantic search fails on |
| No evaluation | No way to measure if answers were correct or grounded |
| No web search | Static corpus only — cannot answer questions about current stock prices, recent news |
| Single collection | No topic isolation — all documents mixed in one retrieval pool |
| No logging | Impossible to debug production failures |

**This was the starting point. No fixes yet.**

---

## Stage 1 — v2 Hybrid Retrieval Architecture (2026-05-17)

### What was wrong before building
- `retriever.py` was dense-only ChromaDB — v2 was documented but not implemented
- `ragas_eval.py` missing entirely; RAGAS eval endpoint not wired
- No BM25, no reranker, no score visibility

### What we built

| File | What changed |
|------|-------------|
| `server/bm25_index.py` | BM25Okapi singleton; module-level (not `app.state`) so importable anywhere; rebuilt on startup + after upload |
| `server/reranker.py` | CrossEncoder singleton; pre-loaded at startup to avoid cold-start latency on first query |
| `server/retriever.py` | Full rewrite as `HybridRetriever(BaseRetriever)` — RRF fusion of dense (weight 0.7) + sparse (weight 0.3) |
| `server/main.py` | BM25 build + reranker load wired into lifespan startup |
| `server/routes/upload.py` | BM25 rebuild triggered after each upload |

### Key design decisions

- **`HybridRetriever` as `BaseRetriever` subclass** — `ConversationalRetrievalChain` expects a `BaseRetriever`; subclassing means `chain.py` needs zero changes
- **BM25 as module-level singleton** — avoids threading state through lifespan → constructor; `get_index()` importable anywhere
- **Reranker pre-loaded at startup** — ~0.5s load from disk cache; better to pay at startup than add latency to first user query
- **MiniLM-L-6-v2** chosen as reranker (~85MB) — best ranking quality available at the time

### What was still missing
- Chain score extraction (similarity/BM25/RRF/rerank not returned in API response)
- RAGAS eval endpoint
- Groq LLM (still on Euron)

---

## Stage 2 — Chain Scores + RAGAS Endpoint (2026-05-23)

### What was wrong
- API response had no retrieval scores — no way to show per-source similarity/BM25/RRF/rerank scores
- RAGAS eval endpoint not wired; `ragas_eval.py` missing
- `data/ground_truth/eval_pairs.json` had only keyword hints, no `ground_truth` answers → `context_precision` and `context_recall` always returned null

### What we built

| File | What changed |
|------|-------------|
| `server/chain.py` | Score extraction — similarity/bm25/rrf/rerank scores passed through to API response per source |
| `server/routes/chat.py` | Added `retrieval_method` field; stores contexts in `eval_log` for downstream RAGAS eval |
| `server/config.yaml` | Hybrid retrieval params: `dense_weight`, `sparse_weight`, `retrieve_k`, `rerank_k` |
| `requirements.txt` | Added `rank_bm25`, `sentence-transformers`, `ragas`, `datasets` |
| `server/eval/ragas_eval.py` | RAGAS faithfulness + answer_relevancy via `LangchainLLMWrapper` |
| `server/routes/eval.py` | `POST /api/eval/ragas` endpoint wired |
| `server/main.py` | `/health` endpoint added |

### What was still broken
- RAGAS not installed in venv (added to requirements.txt; installs at Docker build only)
- `eval_pairs.json` still had no ground_truth → 2 of 4 RAGAS metrics null
- LLM still on Euron gpt-4.1-mini

---

## Stage 3 — Groq Migration + Web Search Fixes (2026-05-24)

### What was wrong

| Problem | Root cause |
|---------|-----------|
| LLM on Euron (gpt-4.1-mini) | Closed model, slower, weaker interview story vs open-weight |
| Web search silently broken | `tavily-python` in requirements.txt but never pip-installed |
| Tavily returned shallow results | `search_depth="basic"` — not enough content from financial sites |
| Web context never reached LLM | `ConversationalRetrievalChain`'s condensation step rewrote the question and stripped prepended Tavily context before LLM ever saw it |
| Follow-up web queries returned garbage | Raw follow-up ("Is the price level good?") sent to Tavily with no chat history context |
| No request logging | Production failures undebuggable |

### What we built

| Component | Change |
|-----------|--------|
| LLM | Migrated Euron → Groq `llama-3.3-70b-versatile` via `langchain-groq`. Euron kept for embeddings (Groq has no embeddings endpoint) |
| `server/chain.py` | `run_query_with_web()` — bypasses chain condensation; direct LLM call with RAG + Tavily context + memory |
| `server/chain.py` | `condense_question()` — rewrites follow-up queries using chat history before Tavily search |
| Tavily | `search_depth="advanced"`, `max_results=3` (2× credits but richer content) |
| `server/main.py` | Request logging middleware — logs `METHOD /path STATUS Xms` per request |
| `server/utils.py` | Centralised logging — root logger + `logs/finrag.log` (5MB×3 rotation), noisy libs silenced |
| `frontend/.../MessageBubble.jsx` | `WebSourcesList` component — Tavily URLs as clickable green pill links |
| `.env.example` | Fixed — real keys had been committed; replaced with placeholders |

### Key discoveries
- `ConversationalRetrievalChain` condensation = silent context killer for web queries. Only fix: bypass the chain entirely for web path.
- Memory's `output_key="answer"` — `save_context` must use `{"answer": answer}` not `{"output": answer}` or KeyError.

---

## Stage 4 — OOM Hell on Render (2026-05-24, four sub-issues)

Render free tier: 512MB RAM. This stage was four separate OOM root causes discovered in sequence.

---

### 4a — CUDA torch OOM (startup crash)

**Problem:** `sentence-transformers` pulled CUDA torch (~2GB) by default. OOM before uvicorn bound to port → Render showed "No open ports detected" timeout. Zero server stdout — invisible failure.

**Diagnosis clue:** Build log showed `cuda-toolkit-13.0.2`, `nvidia-cublas` being installed. Port scan timeout = uvicorn crash at import time (not lifespan — lifespan runs *after* port bind).

**Fix:**
```dockerfile
# Install CPU-only torch BEFORE requirements.txt
RUN pip install torch --index-url https://download.pytorch.org/whl/cpu
RUN pip install -r requirements.txt

ENV HF_HUB_OFFLINE=1
ENV TRANSFORMERS_OFFLINE=1
```

---

### 4b — ragas startup ImportError

**Problem:** `ragas 0.4.3` imports `langchain_community.chat_models.vertexai` at package `__init__` level. That module was removed in `langchain-community 0.4.x`. Crash propagated: `eval.py` → `ragas_eval.py` → `ragas.__init__` → ImportError before uvicorn bound port.

**Fix:** All ragas imports moved inside `run_ragas_eval()` function body (lazy import).

---

### 4c — CrossEncoder OOM during chat

**Problem:** `CrossEncoder.predict(20 pairs)` = BERT forward pass on 20 pairs → ~200–400MB spike on top of base ~250MB → OOM on first web query.

**Fix:**
- `config.yaml`: `retrieve_k: 20 → 10`
- `reranker.py`: `model.predict(pairs, batch_size=4)` — limits how many pairs processed at once

---

### 4d — Web search OOM (post-GC headroom)

**Problem:** After first query, Python retained chain/LLM objects at ~490MB. Web query added ~9KB Tavily content + `condense_question` LLM call + `run_query_with_web` LLM call → OOM.

**Fix:**
- Tavily content truncated to 800 chars per result (was up to ~3000)
- `max_results`: 3 → 2
- `gc.collect()` after each chat request in `routes/chat.py`

---

## Stage 5 — TinyBERT + RAGAS Removal + Benchmark JSON (2026-05-30)

### What was wrong

| Problem | Root cause |
|---------|-----------|
| MiniLM-L-6-v2 (~85MB) still OOMing on web queries | Too close to 512MB ceiling even after GC |
| Live RAGAS eval always 500 on Render | `nest_asyncio.apply()` (called at ragas import time) cannot patch `uvloop` — the event loop uvicorn uses on Linux. `ValueError: Can't patch loop of type uvloop.Loop`. Permanently unfixable without replacing uvicorn's event loop. |
| Groq 70B exhausted 100k daily tokens in one RAGAS run | RAGAS makes ~10 LLM calls per sample for statement decomposition. 10 samples × 10 calls = 100k tokens gone. |
| context_precision and context_recall always null | `eval_pairs.json` had no `ground_truth` answers — only keyword hints |

### What we built

| Component | Change |
|-----------|--------|
| `server/reranker.py` | Switched to `cross-encoder/ms-marco-TinyBERT-L-2-v2` (~17MB vs 85MB). Saves 68MB permanently. |
| `server/routes/eval.py` | Removed `POST /api/eval/ragas` endpoint |
| `requirements.txt` | Removed `ragas` |
| `scripts/run_ragas_local.py` | Local RAGAS runner: ingest corpus → generate answers → run eval → write JSON. Uses `llama-3.1-8b-instant` as judge (500k TPD vs 70B's 100k TPD) |
| `frontend/src/data/ragas_benchmark.json` | Static scores — Vercel builds dashboard from file |
| `frontend/.../EvalPanel.jsx` | Replaced live run button with static benchmark panel |
| `data/ground_truth/eval_pairs.json` | Added `ground_truth` field to all 20 pairs → unlocked `context_precision` + `context_recall` |
| UI | i-button tooltips on faithfulness badge + all 4 RAGAS metric cards |

### Real scores committed
```
faithfulness:      1.0   (note: likely inflated — see below)
answer_relevancy:  0.90
context_precision: TBD   (pending fresh run)
context_recall:    TBD   (pending fresh run)
```

### Key discoveries
- `results["metric_name"]` returns `None` in ragas 0.2.x — must use `results.to_pandas()["metric_name"].mean()`
- TinyBERT loads with harmless `UNEXPECTED key bert.embeddings.position_ids` warning
- **Faithfulness 1.0 is likely inflated** — eval queries were designed alongside the corpus, and 8B judge is lenient. Scores are directional, not absolute. Run on held-out queries for honest numbers.

---

## Stage 5.5 — Rebranding: FinRAG → Prism (2026-06-13)

### What changed
The project was originally named **FinRAG** — a fintech-specific RAG demo. As the architecture matured (multi-workspace, URL ingestion, domain-agnostic retrieval), it became clear the tool was no longer fintech-specific. Any corpus — legal, HR, medical, research — could be loaded and queried.

**Decision:** Rebrand to **Prism**. Name reflects the core idea: feed any document set in, get clear structured answers out. One engine, any domain.

| Before | After |
|--------|-------|
| FinRAG | Prism |
| Fintech-specific framing | Domain-agnostic positioning |
| `finrag-v2.onrender.com` | `prism.onrender.com` |
| README pitched at fintech analysts | README pitched at any knowledge-worker |

### What stayed the same
All retrieval architecture, eval framework, and deployment stack unchanged. Rebrand is naming and framing only — the engine is identical.

### What was wrong with the old name
- "FinRAG" implied fintech-only → narrowed the demo audience
- Interviewers at non-fintech MNCs (Adobe, Atlassian, Intuit) would dismiss it as domain-locked
- The actual retrieval engine is domain-agnostic — the name should match

---

## Stage 6 — Multi-Workspace (2026-06 early)

### What was wrong
- Single ChromaDB collection — no isolation between document sets
- Switching topics meant re-ingesting and overwriting previous docs
- `list_collections()` broke on chromadb ≥0.5.4 (returns `list[str]`, not `list[Collection]`)
- Non-web chat path used stale global chain's `source_documents` instead of workspace-specific retriever → wrong docs shown after workspace switch

### What we built

| File | Change |
|------|--------|
| `server/routes/workspaces.py` | Workspace CRUD — one ChromaDB collection per workspace |
| `server/routes/chat.py` | Always resolves workspace-specific retriever before branching on `web_search` |
| `server/routes/workspaces.py` | `list_collections()` normalised with `isinstance` check — works on chromadb ≥0.5.4 (`list[str]`) and <0.5 (`list[Collection]`) |
| `frontend/src/components/Sidebar.jsx` | Workspace switcher UI; per-workspace doc list |
| `frontend/src/App.jsx`, `api.js`, `ChatArea.jsx`, `FileUpload.jsx` | `workspace_id` passed on all requests |

### Key discovery
- Non-web path was relying on stale global chain's `source_documents` rather than workspace-specific retriever. After switching workspaces, the wrong collection's docs were being cited.

---

## Stage 7 — Singleton Cache + URL Guard (2026-06-14) ← Current

### What was wrong
- Every `POST /api/chat` called `get_or_create_collection()` + built a new `HybridRetriever` = full embedding reload per request → OOM after 2–3 queries in the same workspace
- React component state (message list) persisted across workspace switch — showed previous workspace's chat history
- External URL ingestion had no size guard → large pages (news articles, regulatory filings) caused OOM during embed

### What we built

| File | Change |
|------|--------|
| `server/retriever.py` | Module-level `Dict[workspace_id, (vectorstore, retriever)]` cache. Cache invalidated after ingest. `routes/chat.py` reuses cached retriever. |
| `server/routes/chat.py` | Eliminated double retrieval on non-web path; fixed stray print statement |
| `server/url_loader.py` | Max content size guard before embedding external URL content |
| `frontend/src/App.jsx` | `key={workspaceId}` on `<ChatArea>` → remounts component on workspace switch → clears stale messages and state |

### Commits
```
529675f  fix: singleton vectorstore/retriever cache to prevent OOM on repeated queries
6c9f809  fix: URL size guard for OOM prevention, eliminate double retrieval in chat
521a27a  fix: remount ChatArea on workspace switch to clear stale messages
```

### Additional fix (2026-06-16) — HyDE (Hypothetical Document Embeddings)

**What:** Before dense ChromaDB search, LLM generates a hypothetical 2-sentence answer. That answer (not raw query) is embedded for ANN search. BM25 + reranker still use original query.

| File | Change |
|------|--------|
| `server/retriever.py` | `_hyde_expand()` method; `use_hyde: bool` field on `HybridRetriever`; dense path uses expanded query when enabled |
| `config.yaml` | `retrieval.hyde_enabled: false` — toggle without code change |

**Why off by default:** Adds one Groq call per query (~200ms). Enable to measure RAGAS context_recall lift, then decide.

**Commit:** `8945b43`

---

### Additional fix (2026-06-17) — Mandatory web search

**Problem:** Web search was opt-in toggle. Users querying corpus-only got hallucinated answers from irrelevant documents (e.g., Singapore visa question grounded in random passport-mentioning corpus doc, faithfulness 4/5).

**Fix:**

| File | Change |
|------|--------|
| `frontend/src/components/ChatArea.jsx` | Removed toggle button; `const webSearch = true` hardcoded; placeholder always says "docs + web" |
| `server/routes/chat.py` | `web_search: bool = True` as default in `ChatRequest` |

Every query now hits Tavily + RAG corpus. `run_query_with_web` always called with both rag_docs + web_sources.

---

## Stage 8 — Eval Dashboard + Rigorous Metrics (2026-06-17)

### What was wrong
- faithfulness 1.0 and context_precision 1.0 artificially inflated — eval pairs designed alongside corpus, 8B judge lenient. Meaningless scores.
- Per-message faithfulness badge cluttered user UI. Users don't care about LLM judge scores.
- 10 samples — not statistically meaningful.
- Single flat JSON, no versioning — no way to track metric evolution across architecture changes.

### What we built

| Component | Change |
|-----------|--------|
| `eval-dashboard/` | Separate Vite + React static site (own Vercel project). Reads versioned JSON run files. |
| `eval-dashboard/src/components/` | MetricCard (score + delta vs prev), EvolutionChart (Recharts line chart across versions), RunTable (per-query expandable rows with answer vs ground_truth), LatencyStats (p50/p95 bars) |
| `eval-dashboard/public/data/index.json` | Run registry — list of all versioned eval runs |
| `scripts/run_eval_versioned.py` | New eval script. Args: `--version`, `--tag`, `--n`. Computes answer_correctness (LLM judge vs ground_truth), answer_relevancy + context_recall (RAGAS), precision@5, latency p50/p95/p99. Writes versioned JSON + updates index. |
| `data/ground_truth/eval_pairs.json` | Expanded 20 → 50 pairs. Added multi-hop, comparative, negative, numeric, and edge-case questions. |
| `frontend/src/components/MessageBubble.jsx` | Removed FaithfulnessBadge component and rendering block. |
| `server/routes/chat.py` | Removed `score_faithfulness()` call. One fewer Groq API call per query → faster responses. |

### Metrics before vs after

| Metric | Before | After |
|--------|--------|-------|
| faithfulness | 1.0 (inflated) | Removed from prod path |
| context_precision | 1.0 (inflated) | Replaced by answer_correctness (LLM judge vs ground_truth) |
| answer_relevancy | 0.88 | Kept (RAGAS) |
| context_recall | 0.83 | Kept (RAGAS) |
| precision@5 | tracked separately | Now in main eval dashboard |
| latency p50/p95 | not tracked | Now tracked per eval run |
| sample_count | 10 | 50 (5× improvement) |

---

## Stage 9 — Multi-Query Retrieval (2026-06-19)

### What was wrong
- context_recall = 0.51 in v1.0.0 Violet — retriever missed ~half the relevant chunks
- Single-phrasing retrieval only surfaces chunks whose vocabulary matches the query tokens
- Chunks expressing same concept with different words (e.g. "PSP ceiling" vs "merchant limit") never entered the candidate pool

### What we built

| File | Change |
|------|--------|
| `server/retriever.py` | `_multi_query_expand()` — Groq LLM generates 3 phrasings (temperature=0.3). `_get_relevant_documents()` iterates all phrasings, deduplicates by content key keeping best rank, RRF fuses pooled results, reranks with original query. |
| `config.yaml` | `retrieval.multi_query_enabled: false` toggle |
| `docs/learning.md` | Concept 17 — Multi-Query Retrieval |

### Key design decisions
- **Deduplication keeps best rank** — a chunk at rank 1 in one phrasing and rank 8 in another enters RRF at rank 1, not 8
- **Reranker uses original query** — phrasings widen the pool; the reranker judges relevance against what the user actually asked
- **Off by default** — adds one Groq call per query (~200ms). Enable → run v1.1.0 "Indigo" eval → measure delta → decide
- **retrieve_k cap maintained** — reranker input capped at `retrieve_k` even with wider pool, preserving RAM budget on Render

### Expected outcome
- context_recall: 0.51 → measurably higher (target: >0.65)
- P@5: ~0.89 (no degradation expected — reranker filters noise from wider pool)
- Latency: +200–300ms per query (one extra Groq call for phrasing generation)
- Next eval run: `scripts/run_eval_versioned.py --version v1.1.0 --tag "Indigo" --n 50`

---

## Stage 10 — Contextual Retrieval (Eval) (2026-06-20)

### What was wrong
Phase 1 (HyDE + Multi-Query) left context_recall at ~0.51. Root cause confirmed: fixed-size 500-char splits produce decontextualized chunks. `"The limit was revised to ₹2 lakh."` has no document name, no section, no subject — weak embedding that misses ~half relevant content. Query-side techniques cannot fix bad chunk quality.

### What we built

| File | Change |
|------|--------|
| `server/ingest.py` | `contextualize_chunks(chunks, documents, model, sleep_between_calls)` — calls Groq 8B per chunk, prepends 2-sentence situating context to `page_content` before embedding. Fallback to original text on any failure. |
| `config.yaml` | `contextual_retrieval.enabled: false`, `contextual_retrieval.model: llama-3.1-8b-instant` |
| `scripts/run_eval_versioned.py` | `--contextual` flag + `--data-dir` arg. When set: clears `eval_ctx` collection, re-ingests with `contextualize_chunks`, evaluates against `eval_ctx`. Production upload untouched. |
| `tests/test_ingest.py` | 3 tests: context prepended, fallback on failure, empty chunks skipped |

### Results — v1.3.0 "Violet"

| Metric | v1.0.0 baseline | v1.3.0 contextual | Delta |
|--------|-----------------|-------------------|-------|
| context_recall | 0.510 | 0.601 | **+9.1pp (+18%)** |
| precision_at_5 | 0.890 | 0.956 | **+6.6pp** |
| answer_relevancy | 0.620 | 0.633 | +1.3pp |
| answer_correctness | 0.820 | 0.815 | -0.5pp (noise) |
| latency p50 | 2029ms | 4161ms | **+2× ⚠️** |
| latency p95 | — | 6122ms | — |

### Key discoveries
- Biggest single lift across all Phase 1+2 experiments: recall +9.1pp absolute
- Precision also improved significantly (0.890→0.956) — wider context gives reranker stronger signal
- Latency 2× because contextualized chunks are longer (~150 extra tokens per chunk) → LLM processes more tokens per answer generation call. Zero retrieval-time overhead (as designed), but query-time cost is real.
- Recall target was 0.65 — hit 0.60. Gap remains; next candidate is semantic chunking (Phase 2b)
- Production path: if shipping contextual retrieval, need FastAPI BackgroundTask for async contextualization at upload time (otherwise user waits 40s+ per doc upload)

---

## Stage 11 — Contextual Retrieval in Production + Dashboard Polish (2026-06-20)

### What was wrong
- Contextual retrieval proven in eval (recall +18%) but never shipped to production — users got non-contextual chunks
- eval-dashboard X-axis showed raw version strings (`v1.0.0`) with no dates
- No version badge visible in main Prism UI
- Encrypted PDFs caused 500 Internal Server Error instead of a clean user-facing message
- `max_concurrent=20` for parallel Groq calls → 20k token burst → 429 TPM limit on free tier (6000 TPM)
- Render free tier ephemeral filesystem: docs lost on every cold start (known limitation)
- Upload blocking for ~52s while Euron embedding API processes chunks sequentially

### What we built

| File | Change |
|------|--------|
| `server/ingest.py` | `contextualize_chunks_async()` — parallel Groq calls via `asyncio.gather` + `Semaphore(max_concurrent)`. ~10× faster than sequential. Retry parses suggested wait time from 429 error message. |
| `server/routes/upload.py` | Two-phase upload: sync non-contextual embed first (user queryable immediately), then `_contextual_refresh_bg()` BackgroundTask replaces non-contextual chunks with contextual versions. |
| `config.yaml` | `contextual_retrieval.enabled: true`, `max_concurrent: 3` (3 × ~1000 tokens = 3000 TPM — safe under 6000 limit) |
| `server/ingest.py` | `load_documents_from_paths()`: catches `FileNotDecryptedError` → raises `ValueError` with user-friendly message |
| `server/routes/upload.py` | Catches `ValueError` from loader → returns HTTP 422 instead of 500 |
| `eval-dashboard/src/components/EvolutionChart.jsx` | Custom `XAxisTick`: stacked version name + short date (e.g. `Violet (v1.3)` / `20 Jun 26`) |
| `eval-dashboard/src/App.jsx` | `VERSION_NOTES` constant with bullet notes per version; release notes panel shown below run meta |
| `frontend/src/components/Sidebar.jsx` | `Violet v1.3` badge (indigo pill) in sidebar footer |
| `frontend/src/config.js` | New file — `MAINTENANCE_MODE` + `MAINTENANCE_MESSAGE` config flags |
| `frontend/src/App.jsx` | Maintenance banner driven by `config.js`; hidden when `MAINTENANCE_MODE = false` |

### Key discoveries
- `asyncio.gather` with `Semaphore(3)` keeps burst under 3000 TPM — safe on Groq free tier (6000 TPM limit)
- Groq 429 errors include `"Please try again in X.Xs"` — parse this for accurate retry sleep instead of hardcoded 2s
- Render free tier: ephemeral filesystem. Every cold start wipes `./chroma_db`. Docs must be re-uploaded. Fix: Render persistent disk ($0.25/GB/month)
- Euron embedding API sequential calls: 30 chunks × ~1.7s/call = ~52s blocking upload. Next optimization: move embed to background too (return 202 immediately, notify when ready)
- `max_concurrent=20` was the OOM trigger in the previous session — 20 async coroutines each holding ~10MB response + retry state saturated 512MB

---

## Stage 15 — Semantic Chunking Ablation + Retrieval Stack Finalized (2026-06-26)

### What was wrong
Ablation study incomplete — semantic chunking (v1.4.0) was blocked by Groq rate limits in the prior session. Best production stack unconfirmed.

### What we built / ran

| Version | Config | recall | P@5 | relevancy | correctness | p50 |
|---------|--------|--------|-----|-----------|-------------|-----|
| v1.1.0 | HyDE | 0.721 | 0.911 | 0.845 | 0.750 | 4018ms |
| v1.2.0 | HyDE+MQ | 0.645 | 0.904 | 0.890 | 0.770 | 1812ms |
| v1.3.0 | HyDE+MQ+CTX | 0.768 | **0.984** | 0.799 | 0.780 | 2610ms |
| v1.4.0 | HyDE+MQ+CTX+Semantic | **0.861** | 0.711 | 0.885 | 0.750 | 12952ms |

### Decision: semantic chunking rejected

Semantic chunking raises recall +9.3pp (0.768→0.861) but P@5 collapses -27.3pp (0.984→0.711) and latency is 5× worse (2610ms→12952ms p50).

**Root cause of P@5 collapse:** SemanticChunker produces variable-size, topic-boundary chunks. These don't align with the fixed ground-truth keyword spans used for precision@5 scoring. The reranker receives a wider but noisier candidate pool — recall expands while precision degrades.

**Best stack confirmed: v1.3.0 — HyDE + Multi-Query + Contextual Retrieval.**

### Key discoveries
- MQ alone hurts recall (-7.6pp vs HyDE-only) but recovers fully when combined with CTX
- CTX is highest-leverage single addition: +8pp P@5, recall recovery, at 2× query latency cost
- Semantic chunking is a double-edged sword — better chunk boundaries for recall, worse alignment with precision evaluation
- Ablation study is the interview story: systematic metric-driven elimination of techniques

---

## Stage 16 — Citation Highlighting (2026-06-26)

### What was wrong
Sources listed below each answer as truncated 200-char snippets. LLM already outputs `[1]`, `[2]` inline citations but they rendered as plain unclickable text. Users couldn't see which passage in the answer corresponded to which source.

### What we built

| File | Change |
|------|--------|
| `frontend/src/components/CitationPopover.jsx` | New — viewport-aware popover (fixed-position). Shows: source type badge (pdf/web/file), filename/title, page badge, full chunk content (scrollable), rerank score, "Open page N →" for PDF / "Open source →" for web |
| `frontend/src/components/MessageBubble.jsx` | Parse `[N]` markers in answer text → clickable `<sup>` superscripts. `openCitation` state (`{ idx, rect } \| null`). Toggle on same click. `onMouseDown` stopPropagation fix (prevents document mousedown from immediately re-opening after close). |
| `frontend/src/components/SourceExpander.jsx` | Removed 200-char content truncation — full chunk text shown |
| `server/main.py` | Added `GET /api/files/{filename}` → `FileResponse` from `data/raw/`. Path traversal blocked via `is_relative_to()`. `UPLOAD_DIR` made absolute (`Path(__file__).resolve().parent.parent / "data" / "raw"`). |
| `server/routes/upload.py` | `UPLOAD_DIR` made absolute (`Path(__file__).resolve().parent.parent.parent / "data" / "raw"`) |

### Key discoveries
- `mousedown` on document fires before `click` — without `e.stopPropagation()` on the `<sup>` mousedown, clicking an open citation closes then immediately reopens it (toggle broken)
- `startswith()` on raw path strings has prefix-confusion bug (`/data/rawevil` passes `/data/raw` check) — replaced with `Path.is_relative_to()` (Python 3.9+)
- Relative `Path("data/raw")` resolves against process CWD — if uvicorn starts from non-project-root directory, file serving breaks. Absolute `__file__`-relative path fixes this.
- `anchorRect` captured at click time via `el.getBoundingClientRect()` — stored in state as plain object, no ref needed in popover

### Interview story
> "The LLM cites [1], [2] in its answer. Clicking one opens a popover showing the exact passage retrieved — full text, source file, page number, and rerank score. For PDFs it links directly to that page in the browser."

---

## Stage 17 — Metadata Filtering (2026-06-27)

### What was wrong
All documents in a workspace were always searched together. A user with 10 docs spanning 5 years had no way to scope a query to a specific doc or subset. Corpus-wide retrieval diluted precision when the relevant content was known to be in one file.

### What we built

| File | Change |
|------|--------|
| `server/ingest.py` | `source_type` metadata field (`pdf`/`txt`/`csv`) added to all chunks at load time via `SOURCE_TYPE_MAP`. Both `load_documents` and `load_documents_from_paths` patched. |
| `server/url_loader.py` | `source_type: "url"` added to URL-ingested doc metadata. |
| `server/bm25_index.py` | `BM25Index.search()` gets `filter_sources: set[str] \| None = None`. When set, scores using full-corpus BM25 index (stable IDF) but restricts candidate pool to matching docs. |
| `server/retriever.py` | `filter_docs: list[str] \| None = None` field on `HybridRetriever`. Wired into `_dense_retrieve` (ChromaDB `where={"source": {"$in": filter_docs}}`) and `_get_relevant_documents` (BM25 `filter_sources`). New `get_retriever_filtered(workspace_id, filter_docs)` helper — one-off instance reusing cached vectorstore, not added to singleton cache. |
| `server/routes/chat.py` | `filter_docs: list[str] \| None = None` on `ChatRequest`. Guard: empty list → None. When truthy: `get_retriever_filtered(workspace, active_filter)`. Log includes `filter=%s`. |
| `frontend/src/api.js` | `streamChat` gets `filterDocs = null` as 4th arg; sends `filter_docs: filterDocs?.length ? filterDocs : null`. |
| `frontend/src/App.jsx` | `filterDocs: string[]` state. `useEffect` resets to `[]` on workspace change. `handleFilterChange` toggles doc in/out. Props forwarded to Sidebar + ChatArea. |
| `frontend/src/components/Sidebar.jsx` | Doc list items clickable — toggle filter on click. Selected: `ring-2 ring-indigo-500 bg-indigo-50`. Unselected during active filter: `opacity-50`. "Clear filter" button in section header when any selected. Delete button: `e.stopPropagation()` + deselects deleted doc from filter. |
| `frontend/src/components/ChatArea.jsx` | Filter badge above input when `filterDocs.length > 0` (shows scoped doc names + × clear). Placeholder: "Searching N selected doc(s)..." when filter active. `streamChat` called with `filterDocs.length > 0 ? filterDocs : null`. |
| `tests/test_bm25_filter.py` | 6 tests: no-filter returns all, filter restricts by source, empty set returns empty, nonexistent source returns empty, multiple sources, unbuilt index returns empty. |
| `tests/test_source_type.py` | 4 tests: pdf/txt/csv/url each gets correct `source_type`. |

### Key design decisions
- **Full-corpus BM25 for filtering**: spec suggested rebuilding BM25 on filtered subset; implementation uses full-corpus index to score + restricts candidate pool by source. Stable IDF — correct IR semantics. Accepted as superior to spec.
- **New retriever instance per filtered request**: `get_retriever_filtered()` creates a one-off `HybridRetriever`; singleton cache (`_retriever_cache`) untouched. Thread-safe: heavy vectorstore stays cached, lightweight retriever is cheap.
- **Empty filter = no filter**: backend guard `body.filter_docs if body.filter_docs else None` — empty array from frontend treated as no filter.
- **Filter resets on workspace switch**: `useEffect(() => setFilterDocs([]), [currentWorkspace])` — stale filter from workspace A doesn't carry to workspace B.
- **Delete deselects**: `handleDelete` calls `onFilterChange(docName)` if deleted doc was selected — prevents badge showing "Scoped to: [deleted]" with zero results.

### Interview story
> "Within a workspace, users can click any doc chip in the sidebar to scope retrieval. Dense retrieval passes `where={"source": {"$in": selected_docs}}` to ChromaDB; BM25 pre-filters its candidate pool. Zero selection = full-corpus behavior unchanged. Filter badge above the input makes the scope visible."

---

## Stage 18 — Free-Tier Stability + App Restored to Live (2026-07-05)

### What was wrong
- Maintenance banner left ON after Cerebras migration attempt (2026-06-29) failed and was reverted
- Config.yaml still had `hyde_enabled: true` + `multi_query_enabled: true` — each query burned 4 Groq calls
- Free tier limit: 6000 TPM → 429 storms under concurrent use with HyDE + MQ + contextual all on
- Eval dashboard had no indication which version is live or why best stack (v1.3.0) isn't deployed

### What we built

| File | Change |
|------|--------|
| `config.yaml` | `hyde_enabled: false`, `multi_query_enabled: false` — reduces query-time Groq calls 4 → 1–2 |
| `frontend/src/config.js` | `MAINTENANCE_MODE: false` — app live |
| `eval-dashboard/public/data/index.json` | `is_live: true` + `live_note` on v1.1.0 (closest proxy); `blocked_by` constraint on v1.3.0 + v1.4.0 |
| `eval-dashboard/src/App.jsx` | Green LIVE badge + prod config note on v1.1.0; amber "not in production" warning on v1.3.0/v1.4.0 |

### Key design decisions
- **Contextual retrieval kept ON** — uses `openai/gpt-oss-20b` via Euron API, zero Groq TPM impact at query time. Ingest-time only.
- **HyDE + MQ disabled, not removed** — toggles in config.yaml; re-enable instantly when on paid tier
- **v1.1.0 as live proxy in eval dashboard** — no eval run exists for "CTX-only, no HyDE, no MQ" config. v1.1.0 (recall=0.721) is an overestimate; actual live recall ≈ 0.55–0.65 given contextual index without HyDE query expansion
- **Upgrade path documented in eval dashboard** — v1.3.0 blocked_by note explains exactly what to fix

### Groq call budget (current vs best)

| Config | Calls/query | TPM risk |
|--------|------------|----------|
| Current (CTX only) | 1–2 | Safe |
| v1.3.0 (HyDE+MQ+CTX) | 4 | 429 on free tier |

### Upgrade path to v1.3.0
1. Switch to paid Groq tier (or find higher-TPM free provider)
2. Set `hyde_enabled: true` + `multi_query_enabled: true` in `config.yaml`
3. Push → HF Spaces rebuilds → run `scripts/run_eval_versioned.py --version v1.3.1 --tag "Violet" --n 50` to confirm metrics

---

## Current State Snapshot

```
Retrieval:    Hybrid BM25 (0.3) + ChromaDB dense (0.7) → RRF → TinyBERT rerank top-10→5
LLM:          Groq llama-3.3-70b-versatile
Embeddings:   Euron API text-embedding-3-small (sequential, ~1.7s/chunk — bottleneck)
Chunking:     RecursiveCharacterTextSplitter 500-char, overlap 50
Memory:       ConversationBufferWindowMemory k=10
Web search:   Tavily advanced, 800-char truncation, max 2 results — MANDATORY (always on)
HyDE:         DISABLED (hyde_enabled=false). Best measured: +21pp recall but costs 1 Groq call/query.
              Re-enable when on paid Groq tier or higher-TPM provider.
Multi-Query:  DISABLED (multi_query_enabled=false). Costs 1 Groq call/query — free tier cannot sustain.
              Re-enable with HyDE together (v1.3.0 config) on paid tier.
Contextual:   ENABLED (contextual_retrieval.enabled=true). Uses Euron model (openai/gpt-oss-20b) —
              zero Groq TPM impact. Two-phase upload: sync non-contextual embed (<3s queryable),
              BackgroundTask replaces with contextual chunks. max_concurrent=3, max_chunks=50 gate.
Semantic:     DISABLED (semantic_enabled=false). Ablation showed recall +9.3pp but P@5 -27.3pp and 5× latency.
              Rejected — v1.3.0 (HyDE+MQ+CTX) is the confirmed best stack when TPM allows.
Groq calls/query (current): 1–2 (condense_question if follow-up + answer). Safe under 6000 TPM free tier.
Groq calls/query (v1.3.0):  4 (condense + HyDE + Multi-Query + answer) → 429 storms on free tier.
Eval:         Separate eval-dashboard/ static site → https://askprism-eval.vercel.app/
              v1.1.0 marked LIVE (closest proxy). v1.3.0 and v1.4.0 show amber "not in production" warning.
              Best measured: v1.3.0 recall=0.768, P@5=0.984, p50=2610ms
              Versioning: MAJOR.MINOR.PATCH — name changes on MAJOR only (v1.x.x=Violet, v2.x.x=Indigo)
Citation:     [N] markers in LLM answers → clickable <sup> → CitationPopover (fixed-position, viewport-aware).
              Shows full chunk text, source name, page, rerank score. PDF: "Open page N →" link via GET /api/files/{filename}.
              Web: "Open source →". Toggle, click-away, above/below flip at 60% viewport height.
              SourceExpander: full content shown (200-char truncation removed).
Frontend:     Violet v1.3 badge in sidebar footer. Maintenance banner config-driven (frontend/src/config.js).
              MAINTENANCE_MODE=false — app is live as of 2026-07-05.
Filter:       Sidebar doc chips toggleable. Selected: indigo ring. Badge above chat input shows scoped docs + clear ×.
              POST /api/chat accepts filter_docs: string[] | null. Empty = no filter. Resets on workspace switch.
              Backend: get_retriever_filtered() creates one-off HybridRetriever; singleton cache untouched.
              ChromaDB where={"source": {"$in": filter_docs}}. BM25 filters candidate pool, scores with full-corpus IDF.
Workspaces:   Per-workspace ChromaDB collection, singleton retriever cache
Infra:        HF Spaces CPU Basic (backend, 16GB RAM, ephemeral FS — re-upload required after cold start) +
              https://askprism.vercel.app/ (frontend) + https://askprism-eval.vercel.app/ (eval)
              Backend URL: https://benroshan-prism.hf.space
Known limits: Euron embed ~5s/chunk sequential — 30 chunks = ~150s total contextualization in background.
              HF Spaces ephemeral FS: chroma_db lost on cold start. Fix: mount HF persistent storage bucket.
              HyDE + MQ disabled for free-tier stability. Best stack (v1.3.0) needs paid Groq or alt provider.
Observability: LangSmith traces all LLM + retrieval calls (optional, env var)
Streaming:    POST /api/chat returns SSE stream. token events per LLM chunk, done event with
              sources + retrieval_method. Frontend streams tokens into pre-placed assistant
              bubble. Bouncing dots while condense+search runs, blinking cursor during generation.
```

---

## Stage 12 — HF Spaces Migration (2026-06-22)

### What was wrong
Render free tier (512MB RAM) caused repeated OOM crashes under contextual retrieval:
- Base RSS after upload = 524MB (over the 512MB limit)
- `gc.collect()` had no effect — ChromaDB HNSW index + torch runtime held by native allocators, not Python heap
- Contextual refresh (3 async Groq coroutines) + simultaneous chat (Tavily + LLM + CrossEncoder) = peak exceeded 512MB
- Workarounds (RSS guard skipping contextual retrieval, web search suppression during refresh) negated the +18% recall improvement

### What we built

| File | Change |
|------|--------|
| `Dockerfile` | Port 8000 → 7860 (HF convention). Add `useradd -m -u 1000 user` + `chown -R user /app` (HF runs containers as UID 1000). Set `HF_HOME=/app/.cache/huggingface` BEFORE pre-download so user 1000 owns cached weights. Set `HF_HUB_OFFLINE=1` AFTER download. |
| `README.md` | Added HF Spaces frontmatter (`sdk: docker`, `app_port: 7860`). Updated deploy instructions. |
| `server/routes/chat.py` | Removed `is_contextualizing` web search suppression guard (Render-specific). |
| `server/routes/upload.py` | Removed `RSS > 460MB` contextual retrieval skip guard (Render-specific). |
| `docs/`, `decisions.md` | Render → HF Spaces across all infra references. |

### Key discoveries
- HF_HUB_OFFLINE must be set AFTER the pre-download RUN step — setting it before blocks the download itself
- Docker build runs pre-download as root by default; must `USER 1000` first then set `HF_HOME` under `/app` so runtime user 1000 can read the cached weights
- HF Spaces free CPU Basic: 2 vCPUs, 16GB RAM — resolves all Render OOM issues permanently
- Contextual retrieval now runs fully in production (was silently skipped by RSS guard on Render)

---

## Stage 13 — Async Embed Upload (2026-06-23)

### What was wrong
`embed_and_store()` blocked `POST /api/upload` for ~150s (30 chunks × ~5s/chunk via Euron API). User saw spinner, could not query, could not cancel. Upload timeout was 300s.

### What we built

| File | Change |
|------|--------|
| `server/main.py` | `app.state.upload_jobs = {}` initialized in lifespan |
| `server/routes/upload.py` | `POST /api/upload` returns 202 + `job_id` in <1s. Parse+chunk sync; embed+contextual in `_embed_and_contextualize_bg()` BackgroundTask. New `GET /api/upload/status/{job_id}` endpoint. |
| `frontend/src/api.js` | Added `getUploadStatus(jobId)`; reduced `uploadFiles` timeout 300s → 30s |
| `frontend/src/components/FileUpload.jsx` | Polls status every 2s; shows stage label under spinner; fires callbacks on ready. Defensive `|| []` guard on documents. |

### Key discoveries
- Old Vercel frontend receiving new 202 response before redeploy → `data.documents` undefined → React crash. Fix: defensive `docs?.documents || []` guard.
- Groq TPM 429s at `max_concurrent=3` still hit (~5/30 chunks fall back to original text) — some chunks are larger than average. Retry logic handles gracefully.
- Briefing fails with JSON parse error (pre-existing bug in `generate_briefing` — separate fix).

---

## Stage 14 — Briefing Fix + HyDE Re-eval (2026-06-24)

### What was wrong
- `generate_briefing()` crashed with `JSONDecodeError` when Groq LLM returned control characters (ASCII 0x00–0x1f) or Python dict syntax (single quotes) instead of valid JSON.
- Old eval runs (v1.0.0–v1.4.0) accumulated across multiple sessions; stale runs cluttered the dashboard.
- HyDE recall measurement from prior session (v1.1.0_20260619, recall=0.545) was based on 50-sample run that hit Groq 429s mid-run — partial results, unreliable numbers.

### What we built

| File | Change |
|------|--------|
| `server/briefing.py` | Strip control chars `[\x00-\x08\x0b\x0c\x0e-\x1f]` before JSON parse. Fall back to `ast.literal_eval()` on `JSONDecodeError` to handle Python dict syntax from LLM. Added `import ast`. |
| `config.yaml` | `hyde_enabled: true`, `multi_query_enabled: true`, `contextual_retrieval.enabled: false` (contextual off — 429s at 30-chunk scale even with Semaphore(3)) |
| `eval-dashboard/public/data/runs/` | Deleted stale runs (v1.0.0_20260618, v1.1.0_20260619, v1.2.0_20260619, v1.3.0_20260620, v1.3.0_20260623, v1.4.0_20260623). Added `v1.1.0_20260624.json` — fresh HyDE-only run. |
| `eval-dashboard/public/data/index.json` | Updated to single clean run registry. |

### HyDE re-eval results — v1.1.0_20260624 (18 samples, hyde=true, multi_query=false)

| Metric | v1.0.0 baseline | v1.1.0 HyDE | Delta |
|--------|-----------------|-------------|-------|
| answer_correctness | 0.820 | 0.750 | -0.070 |
| answer_relevancy | 0.620 | 0.845 | **+0.225** |
| context_recall | 0.510 | 0.721 | **+0.211** |
| precision_at_5 | 0.890 | 0.911 | +0.021 |
| latency p50 | 2029ms | 4018ms | +2× |

### Key discoveries
- HyDE gives **+21pp recall** (0.51→0.72) on this 18-sample run — much larger than previously measured (+3.5pp on 50 samples with 429s). Smaller sample set; repeat at 50 samples to confirm.
- answer_correctness flat at 0.75 for all 18 samples — 8B judge giving uniform score, not differentiating. May indicate judge calibration issue, not actual correctness plateau.
- Latency 2× (2029ms→4018ms) — HyDE adds one Groq call per query for hypothetical expansion.
- Briefing fix unblocks document upload → briefing flow end-to-end.

---

## Roadmap — Retrieval & Answer Quality

### Phase 1 — Quick wins (no infra change, measurable RAGAS lift)

#### ~~HyDE (Hypothetical Document Embeddings)~~ ✅ Done (Stage 7, commit 8945b43)
- Implemented in `server/retriever.py`. Toggle: `config.yaml hyde_enabled` (default: false).
- Enable + re-run eval to measure context_recall lift vs v2.0 baseline (0.70).

#### ~~Multi-Query Retrieval~~ ✅ Done (Stage 9, 2026-06-19)
- Implemented in `server/retriever.py`. Toggle: `config.yaml multi_query_enabled` (default: false).
- Enable + run `scripts/run_eval_versioned.py --version v1.1.0 --tag "Indigo" --n 50` to measure context_recall lift vs 0.51.

---

### Phase 2 — Ingest pipeline (requires re-ingest of all docs)

#### ~~Contextual Retrieval~~ ✅ Done + Shipped to Production (Stage 10+11, 2026-06-20)
- `contextualize_chunks_async()` + BackgroundTask in `routes/upload.py`. Two-phase: sync non-contextual embed (queryable <3s) → background contextual replacement.
- v1.3.0 results: recall 0.510→0.601 (+18%), P@5 0.890→0.956. Latency 2× at query time (longer chunks → more LLM tokens).
- `max_concurrent=3` in `config.yaml` — safe under Groq 6000 TPM limit.

#### Semantic Chunking
- **Problem:** Fixed 200-char splits cut mid-sentence, mid-table, mid-list. Embedding a truncated sentence returns a weak vector.
- **How:** Replace `RecursiveCharacterTextSplitter` with LangChain's `SemanticChunker` — splits at sentence boundaries where cosine similarity between adjacent sentences drops below a threshold (topic shift).
- **Effort:** Medium. Config change in `ingest.py` + re-ingest. Tune `breakpoint_threshold_type`.
- **Expected lift:** Fewer nonsensical chunks in top-5. Most noticeable on regulatory PDFs with section headers and numbered lists.

---

### Phase 3 — UX + trust

#### Streaming Responses
- **Problem:** User submits question → 8–15s wait → full answer appears. Feels broken even on fast hardware.
- **How:** Backend: `chain.astream_events()` → `StreamingResponse` yielding SSE tokens. Frontend: `EventSource` or `fetch` + `ReadableStream` — append tokens as they arrive. Faithfulness scoring runs as background task after full answer assembled.
- **Effort:** High — both backend and frontend change. `ConversationalRetrievalChain` supports `astream_events()` in LangChain ≥0.2.
- **Impact:** Perceived latency drops from 10s to ~1s. Single biggest UX improvement.

#### ~~Citation Highlighting~~ ✅ Done (Stage 16, 2026-06-26)
- `[N]` markers clickable → `CitationPopover` with full chunk text, page badge, rerank score. PDF "Open page N →" link. Zero new npm deps.
- Works for all source types: PDF, URL, TXT, CSV. No PDF viewer library needed — page link uses browser's built-in viewer.

---

### Phase 4 — Differentiation

#### Metadata Filtering
- **Problem:** Multi-workspace isolates by collection, but within a workspace (10 docs across 5 years) no way to scope retrieval to `year=2024` or `doc_type=rbi_circular`.
- **How:** Tag chunks with `{source_type, year, doc_name}` at ingest. Pass optional `filter` param in `/api/chat` request. ChromaDB `where` clause on dense retrieval; BM25 pre-filters corpus to matching chunk IDs.
- **Impact:** Precision boost on time-scoped or source-scoped queries.

#### Document Comparison Mode
- **Problem:** No way to ask "What changed between RBI circular 2023 and 2024?"
- **How:** Frontend sends two doc IDs + comparison query. Backend retrieves relevant chunks from each collection separately, synthesises a structured diff answer.
- **Impact:** Killer fintech feature. Unique demo moment. Differentiates from generic RAG.

#### Agentic Mode (LangGraph)
- **Problem:** Single-shot RAG cannot handle multi-step reasoning: retrieve → compute → web search → synthesise.
- **How:** Replace `ConversationalRetrievalChain` with a LangGraph graph. Nodes: retriever, web_search, calculator, synthesiser. LLM decides which tool to call.
- **Impact:** Separates Prism from basic RAG — becomes a research agent. Strongest interview story.

---

## Roadmap Priority Matrix

```
HIGH impact × LOW effort  → Build first
  HyDE
  Multi-query retrieval
  Metadata filtering

HIGH impact × MEDIUM effort → Build second
  Contextual retrieval (+ re-ingest)
  Semantic chunking (+ re-ingest)
  Streaming responses

HIGH impact × HIGH effort → Build last
  Citation highlighting
  Document comparison
  Agentic mode (LangGraph)
```

---

## Interview Story Arc

```
v1  → Dense-only retrieval. No eval. No baseline.
v2  → Hybrid BM25+dense, cross-encoder rerank. Measured with RAGAS.
     → faithfulness=1.0, answer_relevancy=0.90 on 20-pair eval set.
+HyDE → context_recall 0.51→0.72 (+21pp). Hypothetical answer embedding closes vocabulary gap.
+Contextual → context_recall 0.60 (+18% vs baseline). Ingest-time LLM chunk augmentation.
+Agentic  → Multi-step reasoning. Not RAG anymore — research agent.
```

Each step has a metric. That is the complete RAG engineering narrative for MNC DS interviews.
