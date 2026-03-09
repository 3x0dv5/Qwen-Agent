# Python→Rust Performance Research for Qwen-Agent

## Scope and method
I reviewed the core runtime paths involved in RAG parsing/retrieval and tokenization, with emphasis on repeated Python loops, regex-heavy text processing, and per-request rebuilding of ranking data structures.

## High-value slowdown candidates and Rust opportunities

### 1) Document chunking path repeatedly tokenizes and splits text in Python
**Where:** `qwen_agent/tools/doc_parser.py`
- `split_doc_to_chunk()` repeatedly performs:
  - sentence splitting via regex (`re.split(r'\. |。', txt)`),
  - per-sentence token counting (`count_tokens(s)`),
  - token slicing/string reconstruction for long sentences,
  - overlap reconstruction in `_get_last_part()` with additional splitting and reverse scans.

**Why this can be slow:**
- It is a deeply nested loop over pages → paragraphs → sentences.
- It repeatedly crosses Python/C boundaries for tokenization in small units.
- The overlap logic is string-heavy and branchy in pure Python.

**Rust layer idea:**
- Build a `rag_chunker` Rust extension (PyO3/maturin) that:
  - consumes parsed page/paragraph structure,
  - performs sentence segmentation + chunk packing + overlap generation in Rust,
  - returns chunk boundaries/metadata to Python.
- Keep Python orchestration/cache behavior unchanged.

**Expected impact:** Medium-to-high for large PDFs/office docs and long paragraphs.

---

### 2) Per-paragraph token counting during parsing is O(total paragraphs) with Python loop overhead
**Where:** `qwen_agent/tools/simple_doc_parser.py`
- After parsing, each paragraph/table is assigned token count in a nested Python loop:
  - `for page in parsed_file: for para in page['content']: para['token'] = count_tokens(...)`

**Why this can be slow:**
- Large parsed docs can contain many short segments.
- Each short `count_tokens` call has fixed overhead even if backend tokenization is optimized.

**Rust layer idea:**
- Expose a batch token count API (`count_tokens_batch(list[str]) -> list[int]`) in Rust.
- Replace one-by-one token counting with batched counting in parser and chunker flows.

**Expected impact:** Medium (mainly latency reduction on many small segments).

---

### 3) Keyword search rebuilds BM25 corpus and does regex/token filtering in Python every query
**Where:** `qwen_agent/tools/search_tools/keyword_search.py`
- Per query it:
  - flattens all chunks,
  - tokenizes each chunk with Python regex + stemming,
  - rebuilds `BM25Okapi([...])`,
  - scores with `get_scores`.

**Why this can be slow:**
- Re-tokenizing and re-indexing every query is expensive when querying the same docs repeatedly.
- Regex/token filtering + stopword checks are Python-loop-heavy.

**Rust layer idea:**
- Create `keyword_index` Rust module with:
  - fast tokenizer/normalizer/stem pipeline,
  - persistent BM25 index object keyed by document hash,
  - query-time scoring only.
- Fallback to Python path if extension unavailable.

**Expected impact:** High for interactive multi-turn RAG over the same files.

---

### 4) Hybrid rank fusion is currently Python nested loops with an explicit perf TODO
**Where:** `qwen_agent/tools/search_tools/hybrid_search.py`
- Merges multiple rank lists using nested loops and Python dict/list updates.
- Contains explicit note: `# TODO: This needs to be adjusted for performance`.

**Why this can be slow:**
- Cost scales with number of chunks × number of searchers.
- Pure Python rank aggregation adds overhead in hot retrieval paths.

**Rust layer idea:**
- Implement a Rust `fuse_ranks()` function that:
  - ingests scorer outputs,
  - applies reciprocal-rank style accumulation,
  - outputs sorted `(doc_id, chunk_id, score)`.

**Expected impact:** Low-to-medium alone; medium when combined with fast keyword/vector retrieval.

---

### 5) PDF post-processing involves geometry checks with O(text_blocks × tables) Python loops
**Where:** `qwen_agent/tools/simple_doc_parser.py` (`postprocess_page_content`)
- Compares each text block against detected table bounding boxes.
- Merges paragraph lines based on font/geometry heuristics.

**Why this can be slow:**
- Heavy branching and bbox comparisons in Python for dense pages.

**Rust layer idea:**
- Move bbox filtering + line merge heuristic to Rust data structs.
- Return cleaned content back to existing Python parse pipeline.

**Expected impact:** Medium for table-heavy technical PDFs.

## Lower-priority / likely not worth Rust first
- `qwen_agent/utils/tokenization_qwen.py` mostly delegates to `tiktoken` (already native-backed), so only surrounding Python glue is likely not the top bottleneck.
- `vector_search.py` relies on FAISS/langchain stack where core similarity search is already native.

## Recommended implementation order
1. **Keyword indexing + caching (Rust)**
2. **Chunking/overlap engine + batch token counting (Rust)**
3. **Hybrid rank fusion (Rust)**
4. **PDF postprocess geometry pass (Rust)**

This order should maximize query-latency wins first, then ingestion/chunking throughput.

## Integration blueprint
- Packaging: `maturin` + `PyO3` as optional extra (`qwen-agent[rust-accel]`).
- API pattern: thin Python wrapper with graceful fallback.
- Cache compatibility: keep existing storage format and hash keys unchanged.
- Observability: add timing logs for Python vs Rust path selected.

## Validation plan
- Create repeatable benchmark fixtures:
  - 1 short doc, 1 medium mixed-language doc, 1 long PDF with tables.
- Measure p50/p95:
  - parse time,
  - chunking time,
  - keyword retrieval latency,
  - end-to-end `retrieval.call()` latency.
- Gate rollout by proving no retrieval-quality regressions (same/similar top-k chunks).
