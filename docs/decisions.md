# Technical Decisions — Prism

## Decision log

| Date | Decision | Rationale | Status |
|------|----------|-----------|--------|
| 2026-05 | Euron API for embeddings (not local sentence-transformers) | Groq has no embeddings endpoint; Euron API keeps RAM footprint near zero vs local model | Active |
| 2026-05-24 | Groq (llama-3.3-70b-versatile) for LLM; Euron retained for embeddings | Groq: faster inference, open-weight model. Euron kept for embeddings — Groq has no embeddings endpoint. | Active |
| 2026-05 | Render (Docker) over Railway for backend | Free tier RAM fit confirmed: embeddings ~150MB + reranker ~17MB = ~230MB total | **Superseded 2026-06-22** |
| 2026-06-22 | HF Spaces Docker over Render for backend | Render 512MB free tier caused repeated OOM under contextual refresh + concurrent chat. HF Spaces CPU Basic = 16GB RAM free. No memory guards needed. Port 7860, user UID 1000, HF_HOME=/app/.cache/huggingface baked at build time. | Active |
| 2026-05 | ParentDocumentRetriever (child 200 / parent 800) | Better faithfulness: small chunks retrieved precisely, large chunks give LLM full context | Active |
| 2026-05 | BM25 weight 0.3 in RRF fusion | Regulatory text has exact keyword matches; sparse recall is complementary, not dominant | Active |
| 2026-05-30 | RAGAS pre-computed locally, dashboard JSON-driven | `nest_asyncio` cannot patch `uvloop` on Render — live eval endpoint always 500s. Run `scripts/run_ragas_local.py` (8B judge model) → commit `ragas_benchmark.json` → Vercel builds dashboard. | Active |
| 2026-05-30 | RAGAS judge uses `llama-3.1-8b-instant` not `llama-3.3-70b` | 70B model exhausts Groq free-tier 100k TPD in one eval run. 8B has 500k TPD and is sufficient for statement-level faithfulness checks. Answer generation still uses 70B. | Active |
| 2026-05 | LangSmith tracing via env var (no code changes) | LangChain reads LANGCHAIN_TRACING_V2 automatically; zero instrumentation cost | Active |
| 2026-05-30 | Reranker switched to TinyBERT-L-2-v2 (~17MB) from MiniLM-L-6-v2 (~85MB) | Lower memory footprint with acceptable ranking quality at demo corpus scale | Active |
| 2026-05 | Cross-encoder reranker pre-downloaded at Docker build time | Avoids cold-start latency on first request in production | Active |
| 2026-05 | Idempotent ingestion via md5(source+page+text) chunk IDs | Re-running ingest does not duplicate chunks in ChromaDB | Active |
| 2026-06-14 | Multi-workspace: one ChromaDB collection per workspace | Isolated document sets per workspace; `workspace_id` passed on every request; `list_collections()` normalised for chromadb ≥0.5.4 (returns `list[str]`) and <0.5 (returns `list[Collection]`) | Active |
| 2026-06-14 | Multi-workspace frontend: workspace switcher + per-workspace doc list and chat | Sidebar shows all workspaces; switching remounts ChatArea via React key prop to clear stale messages and state | Active |
| 2026-06-14 | Singleton vectorstore/retriever cache keyed by workspace_id | Every chat request was creating a new Chroma instance (full embedding reload) on top of the existing one → OOM on repeated queries. Cache dict in `retriever.py` reuses instances; invalidated after ingest. | Active |
| 2026-06-14 | URL size guard in url_loader.py before embedding | Large external pages (news, filings) could exhaust 512MB RAM during URL ingest. Guard truncates/rejects oversized content before embed call. | Active |
| 2026-06-16 | HyDE for dense retrieval; toggled via config.yaml `hyde_enabled` | Hypothetical answer embedding lands closer to real answer chunks in vector space than raw query. BM25 + reranker still use original query. Off by default — adds one Groq call (~200ms); enable to measure RAGAS lift before committing. | Active |
| 2026-06-17 | Web search mandatory (not toggle) | Opt-in toggle caused users to get hallucinated answers grounded in wrong corpus docs when web search was off. Always-on Tavily + RAG gives grounded answers for both corpus and open-domain queries. | Active |
| 2026-06-17 | Separate eval-dashboard as own Vercel project | Eval tooling is not user-facing; separating avoids bloating the main frontend and lets eval dashboard evolve independently | Active |
| 2026-06-17 | answer_correctness replaces faithfulness as primary metric | faithfulness (judge vs retrieved chunks) is circular — inflates when eval pairs were designed alongside corpus. answer_correctness (judge vs ground_truth reference) is independent signal | Active |
| 2026-06-17 | Per-message faithfulness badge removed from user UI | Badge added noise without value to end users; saves one Groq call per query (~200ms latency reduction); eval moved to dedicated dashboard | Active |
| 2026-06-17 | eval_pairs.json expanded 20 → 50 pairs | 20 samples not statistically meaningful; added multi-hop, comparative, negative, numeric, edge-case question types | Active |
| 2026-06-17 | Versioned eval JSON runs + index.json registry | Single flat JSON had no history; versioned runs let dashboard show metric evolution across architecture changes | Active |

| 2026-06-20 | Contextual retrieval shipped to production via BackgroundTask | Eval proved +18% recall. Two-phase: sync non-contextual embed first (user queryable <3s), background replaces with contextual. Avoids blocking upload on ~52s Groq calls. | Active |
| 2026-06-20 | `contextualize_chunks_async()` with Semaphore(3) | Parallel Groq calls 10× faster than sequential. `max_concurrent=3` keeps burst at ~3000 TPM — safe under Groq free tier 6000 TPM limit. `max_concurrent=20` caused OOM + 429 storm. | Active |
| 2026-06-20 | Retry parses wait time from 429 error message | Groq 429 includes "Please try again in X.Xs". Parsing gives accurate sleep duration. Hardcoded 2s was too short for 10s rate limit windows. | Active |
| 2026-06-20 | 422 for encrypted PDF upload instead of 500 | `pypdf.FileNotDecryptedError` propagated as unhandled 500. Now caught in `load_documents_from_paths()`, re-raised as `ValueError`, caught in upload route → HTTP 422 with clear message. | Active |
| 2026-06-20 | Config-driven maintenance banner in `frontend/src/config.js` | Single file to toggle `MAINTENANCE_MODE` + `MAINTENANCE_MESSAGE`. Edit + push → Vercel redeploys in ~30s. No hardcoded HTML. | Active |
| 2026-06-24 | `ast.literal_eval` fallback + control char strip in `generate_briefing()` | Groq LLM sometimes returns Python dict syntax (single quotes) or embeds ASCII control chars (0x00–0x1f) that break `json.loads`. Strip control chars first; fall back to `ast.literal_eval` on `JSONDecodeError`. Both are safe — input is already extracted from LLM regex match. | Active |
| 2026-06-24 | HyDE enabled by default in config.yaml | Re-eval (v1.1.0_20260624, 18 samples) shows +21pp recall (0.51→0.72) with HyDE. Latency cost 2× (p50 4018ms vs 2029ms) accepted — recall gain outweighs latency. | Active |
| 2026-06-26 | Citation highlighting via popover, not PDF viewer pane | Sources are PDF/URL/TXT/CSV — a PDF-only viewer fails for most. Popover with full chunk text works for all types. PDF sources get bonus "Open page N →" link via browser's built-in viewer. Zero new npm deps. | Active |
| 2026-06-26 | `DOMRect` snapshot at click time (not live ref) for popover positioning | Simpler than passing anchorRef into CitationPopover — captures position once at click, no ref forwarding complexity. Stale after scroll (acceptable for demo). | Active |
| 2026-06-26 | `onMouseDown` stopPropagation on citation `<sup>` | Document-level mousedown in CitationPopover fires before `onClick` on the marker. Without stopPropagation, clicking an open citation closes (mousedown) then immediately reopens (click). stopPropagation on mousedown lets toggle logic in onClick run correctly. | Active |
| 2026-06-26 | `Path.is_relative_to()` over `startswith()` for file serving guard | `startswith()` on raw strings has prefix-confusion bug: `/data/rawevil` passes `/data/raw` check. `is_relative_to()` (Python 3.9+) is separator-aware and correct. | Active |
| 2026-06-26 | `UPLOAD_DIR` as absolute `__file__`-relative path | Relative `Path("data/raw")` resolves against process CWD — breaks if uvicorn started from non-project-root directory. `Path(__file__).resolve().parent.../ "data" / "raw"` is stable regardless of CWD. | Active |

| 2026-06-27 | Metadata filtering: full-corpus BM25 scoring with candidate pool filter | Spec suggested rebuilding BM25 on filtered subset. Full-corpus IDF is correct IR — rare terms don't get inflated weight in a 2-doc subset. Filter which indices enter candidate pool; score with global index. | Active |
| 2026-06-27 | Metadata filtering: one-off retriever per filtered request | `get_retriever_filtered()` creates new `HybridRetriever` with `filter_docs` set; reuses cached vectorstore. Singleton cache (`_retriever_cache`) untouched. Thread-safe: vectorstore (heavy) stays shared, retriever (cheap) is ephemeral. | Active |
| 2026-06-27 | Empty `filter_docs` treated as no filter | Backend guard: `body.filter_docs if body.filter_docs else None`. Frontend sends `null` when array is empty. Prevents zero-result queries from empty selection. | Active |
| 2026-06-27 | `source_type` metadata added to all chunks at ingest | PDF/TXT/CSV tagged at load time in `ingest.py`; URL-ingested docs tagged in `url_loader.py`. Enables citation badge display and potential future source-type filtering. | Active |
| 2026-06-27 | Filter state resets on workspace switch | `useEffect(() => setFilterDocs([]), [currentWorkspace])` in `App.jsx`. Stale filter from workspace A cannot pollute queries in workspace B. | Active |
| 2026-07-05 | HyDE + Multi-Query disabled in production (free-tier stability) | HyDE + MQ = 2 extra Groq calls/query on top of condense + answer = 4 total. Free tier 6000 TPM → 429 storms. Disabled in `config.yaml`; contextual stays ON (Euron model, no Groq impact). Re-enable on paid tier. | Active |
| 2026-07-05 | Eval dashboard marks live version and blocked versions explicitly | `index.json` gets `is_live: true` + `live_note` on closest proxy run; `blocked_by` on runs not deployable. Dashboard renders LIVE badge + constraint warnings. Honest representation of prod vs best-measured gap. | Active |

## Rejected alternatives

| Alternative | Why rejected |
|-------------|-------------|
| sentence-transformers local embeddings | ~400MB RAM → OOM on Render free tier |
| Pinecone / Weaviate vector store | Adds external dependency and cost; ChromaDB sufficient for demo scale |
| LlamaIndex instead of LangChain | LangChain has better ParentDocumentRetriever and ConversationalRetrievalChain support |
| Streaming LLM responses | Adds frontend complexity; acceptable latency at demo scale |
| BM25 persisted to disk | Not required for demo; rebuild on startup is fast enough (~1s for sample corpus) |
| Cerebras migration (2026-06-29) | Free tier only has `gpt-oss-120b` + `zai-glm-4.7` at 5 req/min — Llama 3.3 70B not available. Worse rate limit than Groq. All 8 commits hard-reset. |
| HyDE + Multi-Query in production on free Groq tier | 4 Groq calls/query → 429 storms. Disabled in config.yaml, not removed — zero code change to re-enable on paid tier. |
