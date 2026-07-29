# Prism — Code Walkthrough

Reading order to understand the whole system, ranked by how the request actually flows. Each file's job in one line, key functions with line numbers, and what to read next. Pair with `docs/architecture.md` (diagrams) and `docs/learning.md` (concept deep-dives) — this file is the map between the two: "here's the concept, here's exactly where it lives in code."

---

## Where to start

Read in this order — it follows one document's journey from upload to answered query:

1. `server/main.py` — app boot, what state exists, routes wired
2. `server/ingest.py` — parse + chunk + contextualize
3. `server/retriever.py` — the hybrid retrieval engine (the core of this project)
4. `server/bm25_index.py` + `server/reranker.py` — the two retrieval helpers `retriever.py` calls
5. `server/chain.py` — how retrieved chunks + web results become a streamed LLM answer
6. `server/routes/upload.py` then `server/routes/chat.py` — the two HTTP entry points that glue everything above together
7. `frontend/src/App.jsx` → `FileUpload.jsx` / `ChatArea.jsx` — the UI driving those two endpoints

Everything else (`eval/`, `routes/workspaces.py`, `bm25_index.py` internals) is support machinery — useful once the main loop makes sense.

---

## 1. `server/main.py` (106 lines) — App Entry Point

FastAPI app construction, startup sequence, route registration. Read this first because it declares every piece of shared state (`app.state.*`) that the rest of the backend reads and writes.

| Lines | What |
|---|---|
| 24-49 | `lifespan()` — runs once at boot. Creates `app.state.memory`, `.chain`, `.retriever` (all `None` until first upload), `.eval_log` (list), `.is_contextualizing` (bool flag other routes check), `.upload_jobs` (dict — job_id → status, the async-upload state). Pre-loads the cross-encoder reranker (`load_reranker()`, line 36) so first query isn't slow. If a workspace already has documents on disk, rebuilds BM25 + retriever + chain immediately (lines 38-43) — otherwise chain stays `None` until first upload. |
| 54-59 | CORS — regex allows any `*.vercel.app` + any `localhost:*` port |
| 62-74 | Request logging middleware — every request logged as `METHOD path STATUS Xms` |
| 77-82 | Route registration — `chat`, `eval`, `upload`, `workspaces` routers mounted under `/api` |
| 90-99 | `serve_file()` — serves uploaded originals for citation "Open page N" links. `target.is_relative_to(upload_root)` (line 95) blocks path traversal — see `docs/learning.md` for why `startswith()` would have been wrong here |

**Read next:** `server/ingest.py` (what happens when a file is uploaded) or `server/retriever.py` if you already know how docs get in and want the retrieval engine.

---

## 2. `server/ingest.py` (360 lines) — Parsing, Chunking, Contextual Retrieval

Everything that happens to a document before it's queryable.

| Lines | What |
|---|---|
| 21-58 | `load_documents()` / `load_documents_from_paths()` — PDF (LlamaParse primary, pypdf fallback) / TXT / CSV loaders. Encrypted PDFs raise `ValueError` here (not a raw exception) — caught upstream in `routes/upload.py` as HTTP 422 |
| 109-143 | `chunk_documents()` — `RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)`. See `docs/learning.md` Concept 11 for why paragraph→line→word→char priority matters |
| 144-212 | `contextualize_chunks()` — synchronous version, prepends 2-sentence LLM-generated context per chunk before embedding. Superseded by the async version below in production |
| 213-256 | `_contextualize_one()` — single-chunk contextualization with retry logic. Parses Groq's `"try again in X.Xs"` 429 message to sleep the exact right duration instead of a blind hardcoded wait |
| 257-288 | `contextualize_chunks_async()` — `asyncio.gather` + `Semaphore(max_concurrent)` (default 3). This is Concept 21 in `learning.md` — the 10x-faster parallel version that actually runs in prod |
| 297-341 | `embed_and_store()` — calls Euron API (`text-embedding-3-small`) per chunk, writes into the workspace's ChromaDB collection. Idempotent via md5(source+page+text) chunk IDs |

**Read next:** `server/retriever.py` — how these embedded/contextualized chunks get found again at query time.

---

## 3. `server/retriever.py` (288 lines) — The Hybrid Retrieval Engine

The single most important file in this codebase. Everything in `docs/architecture.md`'s retrieval diagram lives here.

| Lines | What |
|---|---|
| 20-21 | `_vs_cache` / `_retriever_cache` — module-level singleton dicts, one Chroma vectorstore + one `HybridRetriever` per workspace. Without this, every chat request rebuilt the vectorstore from scratch (memory-scoped in `learning.md` under the singleton cache decision) |
| 24-29 | `_get_embeddings()` — Euron API embeddings (`text-embedding-3-small`), not local — see `docs/decisions.md` for why (Groq has no embeddings endpoint) |
| 49-63 | `HybridRetriever(BaseRetriever)` class fields — `dense_weight=0.7`, `sparse_weight=0.3`, `retrieve_k=10`, `rerank_k=5`, `use_hyde`, `use_multi_query`, `filter_docs`. All toggled from `config.yaml` |
| 65-92 | `_multi_query_expand()` — LLM generates 3 rephrasings, returns `[original] + 3 phrasings`. Concept 17 in `learning.md` |
| 94-122 | `_hyde_expand()` — LLM writes a fake 2-sentence answer, embeds *that* instead of the raw question for dense search. Concept 7 in `learning.md` — this is the technique with the biggest single recall lift (+21pp) |
| 124-138 | `_dense_retrieve()` — ChromaDB `similarity_search_with_relevance_scores`, with optional `filter={"source": {"$in": ...}}` for doc-scoped search |
| 140-163 | `_rrf_fuse()` — the actual Reciprocal Rank Fusion math (`k=60` constant, weighted by `dense_weight`/`sparse_weight`). Concept 3a in `learning.md` |
| 165-214 | `_get_relevant_documents()` — the orchestrator. Runs multi-query expansion → for each phrasing, HyDE-expand + dense retrieve + BM25 sparse retrieve → dedup keeping best rank per chunk (lines 185-193) → RRF fuse → cross-encoder rerank (imports `rerank` from `reranker.py`, line 199) → wraps as LangChain `Document` objects with citation metadata |
| 217-233 | `get_retriever()` — cached retriever factory, reads `config.yaml` retrieval params |
| 236-250 | `get_retriever_filtered()` — the metadata-filtering entry point (Concept 22 in `learning.md`). Builds a fresh `HybridRetriever` with `filter_docs` set but reuses the cached vectorstore — never touches `_retriever_cache` |

**Read next:** `server/bm25_index.py` and `server/reranker.py` — the two things `_get_relevant_documents()` calls out to.

---

## 4. `server/bm25_index.py` (96 lines) + `server/reranker.py` (34 lines)

**`bm25_index.py`** — sparse keyword retrieval, module-level singleton (not passed through constructors — see `docs/decisions.md` for why). `BM25Index` class (line 10) wraps `rank_bm25.BM25Okapi`; `get_index()` (line 67) is the global accessor any module can import. `build_from_vectorstore()` (line 73) rebuilds the full-corpus index after every ingest/delete — this is where Concept 22's full-corpus-IDF-with-filtered-candidate-pool logic lives.

**`reranker.py`** — `load_reranker()` (line 11) loads `cross-encoder/ms-marco-TinyBERT-L-2-v2` once at startup (called from `main.py` lifespan). `rerank()` (line 21) scores query+chunk pairs jointly, returns top-k. This is Concept 3b in `learning.md`.

**Read next:** `server/chain.py` — what happens to the 5 chunks `HybridRetriever` hands back.

---

## 5. `server/chain.py` (314 lines) — Retrieval → LLM Answer

Where retrieved chunks + web search results + chat history become a streamed answer.

| Lines | What |
|---|---|
| 11-21 | `SYSTEM_PROMPT` — the instruction set governing citation format `[1]`, hallucination refusal, web-context handling |
| 43-53 | `_create_llm()` — constructs `ChatGroq` directly (model from `config.yaml`). Every LLM call in this codebase goes through this one function — see `docs/learning.md` Concept 24 for why there's no multi-provider abstraction layer here (Cerebras migration attempted, reverted) |
| 56-81 | `build_qa_chain()` — builds `ConversationalRetrievalChain` (used only when web search is unavailable/disabled — the mainline path bypasses this, see next row) |
| 110-147 | `condense_question()` — rewrites a follow-up question into a standalone query using last 3 turns of history, before it's sent to Tavily. Prevents "what about X?" from being searched literally |
| 150-221 | `run_query_with_web()` — **the actual production path.** Retrieves RAG docs directly via `retriever.invoke()` (bypassing the chain's condensation step, which silently strips web context — Concept 5 in `learning.md`), builds a combined context string of doc chunks + Tavily results, calls the LLM once with full chat history as message objects, saves the turn to memory |
| 224-303 | `stream_query_with_web()` — same logic as above but `async for chunk in llm.astream(messages)`, yielding `{"type": "token", ...}` events per chunk and a final `{"type": "done", "sources": ..., "retrieval_method": ...}` — this is what `routes/chat.py` actually calls |

**Read next:** `server/routes/chat.py` and `server/routes/upload.py` — the FastAPI endpoints that call into everything above.

---

## 6. `server/routes/upload.py` (287 lines) — Ingestion Endpoint

| Lines | What |
|---|---|
| 29-34 | `_rebuild_chain()` — the shared "after any corpus change" sequence: invalidate cache → rebuild BM25 → rebuild retriever → new memory → rebuild chain. Called after embed, after contextual refresh, after delete |
| 37-120 | `_embed_and_contextualize_bg()` — the background task. Phase 1 (line 51-60): embed + rebuild chain — user can query within seconds. Then generates briefing (line 62-73, non-critical — wrapped in try/except so a briefing failure never fails the upload). Phase 2 (line 75-109, conditional on `config.yaml` `contextual_retrieval.enabled` **and** chunk count under `max_chunks` gate): contextualize chunks, delete old ChromaDB entries for that source, re-embed, rebuild chain again |
| 123-174 | `POST /upload` — saves files to disk, parses+chunks *synchronously* (so malformed files 422 immediately, line 150-153), registers a job in `app.state.upload_jobs`, hands the rest to `background_tasks.add_task()`, returns 202 |
| 177-188 | `GET /upload/status/{job_id}` — what the frontend polls every 2s |
| 196-241 | `POST /upload/url` — same idea for URL ingestion, fully synchronous (no background task — URLs are smaller/faster than PDF batches) |
| 251-286 | `DELETE /documents/{filename}` — removes chunks from ChromaDB by `where={"source": filename}`, deletes the raw file, rebuilds everything |

**Read next:** `server/routes/chat.py`.

---

## 7. `server/routes/chat.py` (112 lines) — Query Endpoint

| Lines | What |
|---|---|
| 22-38 | `POST /chat` setup — resolves the right retriever: filtered (`get_retriever_filtered`, if `filter_docs` sent) or the cached workspace default |
| 43-104 | `generate()` — the SSE generator. Runs web search first (line 56-61, `condense_question` → `search_web`, wrapped so a Tavily failure doesn't kill the whole response), then streams `stream_query_with_web()` events straight through as `data: {json}\n\n` SSE lines. On the `"done"` event (line 71-90): merges RAG sources + web sources into one list, assigns citation indices, appends the turn to `eval_log` |
| 100-102 | `finally: gc.collect()` + memory logging — every request cleans up explicitly, a holdover from the Render 512MB OOM era (`docs/decisions.md`), harmless now on HF Spaces' 16GB but left in |
| 107-112 | `DELETE /chat/memory` — clears `ConversationBufferWindowMemory` for a fresh conversation |

**Read next:** frontend, starting with `App.jsx`.

---

## 8. Frontend — `frontend/src/`

| File | Lines | What |
|---|---|---|
| `App.jsx` | 108 | Top-level state: current workspace, documents list, messages. `key={workspaceId}` on `ChatArea` (learned the hard way — without it, switching workspaces left stale chat history mounted) |
| `api.js` | 129 | Every backend call in one place. `streamChat()` (line 77) — parses SSE lines from `fetch()`, calls `onToken`/`onDone`/`onError` callbacks. `uploadFiles()` (line 32) + `getUploadStatus()` (line 45) — the 202-then-poll pair matching `routes/upload.py` |
| `components/FileUpload.jsx` | 176 | Drag-and-drop, calls `uploadFiles()`, then polls `getUploadStatus()` every 2s, showing stage label (`"Embedding N chunks..."` etc.) under a spinner until `status === "ready"` |
| `components/ChatArea.jsx` | 171 | Message list + input box, calls `streamChat()`, appends tokens to the in-progress assistant message as they arrive |
| `components/MessageBubble.jsx` | 123 | Renders one message. `CitedText()` (line 32) turns `[1]`, `[2]` markers into clickable `<sup>` elements — `onMouseDown` stopPropagation here is the fix for the popover-toggle bug (documented in memory) |
| `components/CitationPopover.jsx` | 103 | Shows full chunk text, source, page, rerank score for a clicked citation. Positioned from a `DOMRect` snapshot taken at click time, not a live ref |
| `components/Sidebar.jsx` | 251 | Workspace switcher, per-workspace doc list with filter-chip toggles (feeds `filter_docs` into `/api/chat`) |
| `components/EvalPanel.jsx` | 85 | Static RAGAS benchmark display — reads pre-computed JSON, no live endpoint (see `docs/decisions.md` for the `nest_asyncio`/`uvloop` reason) |

---

## Support files (read once the main loop is clear)

| File | Purpose |
|---|---|
| `server/memory.py` (44 lines) | `ConversationBufferWindowMemory` factory, `k=10` sliding window. `output_key="answer"` trap explained in `learning.md` Concept 12 |
| `server/briefing.py` (65 lines) | Ingest-time LLM summary (5 bullets + 3 suggested questions). `ast.literal_eval` JSON-parse fallback for when Groq returns Python dict syntax |
| `server/web_search.py` (63 lines) | Tavily wrapper, `search_depth="advanced"`, results truncated to 800 chars |
| `server/url_loader.py` (65 lines) | URL-to-Document loader with a max-size guard before embedding |
| `server/utils.py` (79 lines) | `load_config()` (reads `config.yaml`), `configure_logging()`, `log_memory_mb()` (psutil RSS logger — OOM-debugging holdover) |
| `server/routes/workspaces.py` (42 lines) | Workspace CRUD, `list_collections()` normalized for chromadb ≥0.5.4 vs older versions |
| `server/eval/*.py` | Offline eval — `precision.py` (deterministic P@K), `faithfulness.py` (unused in prod, RAGAS handles this now), `ragas_eval.py` (lazy-imported RAGAS wrapper, called only by `scripts/run_eval_versioned.py`, never in the request path) |

---

## One-paragraph mental model

A document lands in `ingest.py`, gets chunked and embedded into a per-workspace ChromaDB collection, and its tokens get indexed into a BM25 sparse index (`bm25_index.py`) — both wrapped by a cached `HybridRetriever` (`retriever.py`) that on every query optionally rephrases the question 3 ways and optionally hallucinates a fake answer to search with, retrieves from both dense and sparse indexes for each phrasing, fuses the ranked lists with weighted RRF, and reranks the top candidates with a cross-encoder. Those top-5 chunks, plus a Tavily web search result, plus the chat history, get handed to `chain.py`'s `stream_query_with_web()`, which makes one direct Groq LLM call (bypassing LangChain's chain abstraction entirely, because the abstraction silently strips context) and streams tokens back over SSE through `routes/chat.py` to the React frontend, where citations become clickable popovers.
