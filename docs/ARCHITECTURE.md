# Architecture

This document describes how the code in this repository is organised and how a request flows through it. It was written against the source at the time of the commit that introduced it; when in doubt, the code wins.

## Overview

The server turns arXiv into something an LLM agent can consume cheaply: search returns structured metadata, retrieval returns a single pruned, chunk-annotated copy of the paper text, and an optional embedded index supports document-routed retrieval across several papers.

```
 MCP client ──(stdio | Streamable HTTP)──> crates/native ──> crates/core (pure logic)
                                               │
                                               ├── HTTP ──> export.arxiv.org (API), arxiv.org (HTML, PDF, search page)
                                               │            api.semanticscholar.org (citations, recommendations)
                                               ├── disk cache (OS cache dir)
                                               └── SQLite index (optional, `embedded-db`)

 MCP client ──(HTTP JSON-RPC)──> crates/worker ──> crates/core ──> export.arxiv.org, arxiv.org (HTML only)
```

## Inputs and outputs

All native tools take a JSON object as MCP tool arguments and return one JSON text block. Field-level documentation is in the [README](../README.md#tools); the same schemas are served at runtime as the MCP resource `arxiv://openapi`.

| Tool | Input | Output |
|---|---|---|
| `search` | `q`, `n`, `offset`, `from`, `to`, `cats`, `sort` | Papers parsed from the arXiv Atom feed (metadata only) |
| `retrieve_paper` | `paper_id`, `prune_references`, `chunk_chars`, `chunk_overlap`, `segmentation_k`, `full_payload` | `paper`, `source` (`html` or `pdf`), `pruned_markdown`, `chunks` (offsets into `pruned_markdown`), optional `hierarchical_chunks`. With `full_payload` the legacy shape (raw markdown, per-chunk text) |
| `ingest_corpus` | `paper_ids`, chunking options | Counts of ingested papers plus per-ID errors |
| `hdrr` | `q`, `limit_docs`, `limit_chunks` | Routed document IDs and the top chunks within them |
| `execute` | `op` (`abstract`, `download`, `citations`, `recs`, `retrieve`), `id`, options | `{id, op, result}`; failures appear as `result.error` |

Inputs are lenient about names: `query` is accepted for `q`, `id`/`arxiv_id` for `paper_id`, and so on (see the `serde(alias = ...)` attributes in `crates/native/src/tool.rs`).

## Request path

### 1. Search

1. `build_query_params` (core) sanitises Lucene special characters (keeping `:` and `"` so field filters and phrases work), adds category (`cat:`) and `submittedDate` range filters, maps `sort` to `relevance`/`submittedDate`, and clamps `n` to 1-50.
2. `FetchClient::fetch_arxiv_query` (native) calls `https://export.arxiv.org/api/query`. If the call errors or returns a non-success status, it falls back to scraping `https://arxiv.org/search/` using the original plain-text query (field filters and boolean operators are not honoured on that path).
3. `parse_response` (core, `quick-xml`) converts the Atom XML to `Paper` records.

### 2. Retrieve

1. `normalize_paper_id` strips an `arxiv:` prefix and version suffix and rejects empty or digit-free IDs.
2. Full text: fetch `https://arxiv.org/html/<id>`. If it exists and converts to at least 2,000 characters (`MIN_HTML_TEXT_CHARS`), use it (`source: "html"`). Otherwise (no HTML, 404, or a degraded render) fetch `https://arxiv.org/pdf/<id>` and extract text (`source: "pdf"`). If the PDF fails but a degraded HTML render exists, the degraded HTML is returned.
3. Metadata: the arXiv API lookup runs concurrently in a spawned task. If the HTML page carries its own title/authors/abstract and the API call has not finished, the HTML metadata is used and the API lookup keeps running to warm the metadata cache. If neither is available, a stub `Paper` with the ID as title is returned.
4. Conversion (core): `html::to_markdown` extracts the article, unwraps in-page links, de-duplicates math to `$TeX$` and converts with `htmd`. PDFs go through `pdf-inspector`.

### 3. Prepare and chunk (`prepare_paper`, core)

1. `normalize_markdown` collapses blank lines and trailing whitespace.
2. `prune_markdown` drops noise lines and, when `prune_references` is true, the reference/bibliography section. Skipping ends at the next heading of the same or higher level, so appendices after the bibliography are kept.
3. `chunk_text` produces overlapping windows over `pruned_markdown`. `chunk_chars` is floored at 1,000 and the overlap is capped below the chunk size. Chunks are reported as character offsets, so the text appears once in the response.
4. If `segmentation_k` is set, sentence-like segments are vectorised with TF-IDF and clustered by `HierarchicalSegmenter`, producing `hierarchical_chunks`.

### 4. Index and scoped retrieval (`embedded-db`, native only)

1. After preparation, `retrieve_paper` and `ingest_corpus` call `ArxivServer::ingest`: chunk texts are TF-IDF vectorised and stored with the paper's title and abstract in one SQLite transaction. Ingestion is skipped if the same number of chunks is already stored for that paper.
2. `hdrr` stage 1 (`route_documents`) ranks all stored papers by TF-IDF cosine similarity between the query and `title + abstract`, keeping up to `limit_docs` with non-zero similarity.
3. `hdrr` stage 2 (`retrieve_chunks_scoped`) loads only the chunks of the routed papers, ranks them by TF-IDF cosine similarity and keeps up to `limit_chunks`.

Both stages recompute TF-IDF over the data they load on every query, so cost grows with the size of the index. The stored `embedding_blob` vectors are written at ingest time; the query path as implemented re-vectorises text instead of reading them.

## Key crates and modules

| Path | Role |
|---|---|
| `crates/core` (`arxiv-search-rs-mcp-core`) | Target-independent logic with no network or filesystem access: `arxiv` (query building, ID normalisation, Atom parsing), `html` (HTML to markdown, page metadata), `pdf` (text extraction; a stub that returns an error on `wasm32`), `content` (normalise, prune, chunk, `agent_view`), `segmentation` and `tfidf` (clustering and vectors), `semantic_scholar` (response parsing), `rate_limit` (`RateLimiter` trait and no-op), `paper`, `error` |
| `crates/native` (`arxiv-search-rs-mcp`, binary `arxiv-search-mcp`) | `main` (CLI, stdio and Streamable HTTP transports via `rmcp` and `axum`), `tool` (tool handlers, in-memory response cache, OpenAPI resource), `fetch` (`reqwest` client, arXiv and Semantic Scholar calls), `persistence` (disk cache), `db` (SQLite index, feature `embedded-db`), `rate_limit` (file-lock based limiter) |
| `crates/worker` (`arxiv-search-rs-mcp-worker`) | Cloudflare Worker (`cdylib`) with a hand-written JSON-RPC dispatcher over `search_papers` and `retrieve_paper` |
| `benchmarks/` | Python harness, pinned corpus, results and a separate chart-rendering crate (own workspace) |

## State and caching

| What | Where | Lifetime |
|---|---|---|
| Fetched HTML, PDF and arXiv API metadata | Files in the OS cache directory: `ProjectDirs("org", "arxiv-search", "mcp")`, for example `~/.cache/mcp/` on Linux; falls back to the system temp directory if no home directory is known | 7 days (`DEFAULT_CACHE_TTL`); expired files are removed when the server starts |
| Serialized `retrieve_paper` responses | In-memory map in `ArxivServer`, keyed by normalised ID and options | Process lifetime; cleared entirely when it reaches 64 entries |
| HDRR index | `arxiv_rag.db` (SQLite, WAL mode) in the same cache directory | Persistent while in use. The startup cleanup (`ArxivCache::cleanup_expired`) treats every regular file in the cache directory alike, so an index file not modified for 7 days is deleted at the next start, before the database is opened, and the index starts empty |

Citations and recommendations from Semantic Scholar are not cached. The Worker has no cache.

## Error and resource boundaries

- **Input validation.** Invalid IDs and malformed dates return MCP `invalid_params`. Cache file names reject `/`, `\` and `..`; the database path rejects `..` components.
- **Network.** One shared `reqwest` client with a 15 s request timeout and 5 s connect timeout, rustls TLS. Failures surface as MCP `internal_error` with the underlying message. Search degrades to the HTML search page (above). There is no retry or backoff logic.
- **Per-tool error shape.** `search`, `retrieve_paper` and `hdrr` return MCP errors. `execute` catches the error and returns it inside `result`. `ingest_corpus` continues past failing IDs and lists them in `errors`.
- **Rate limiting.** `FileRateLimiter` (a cross-process lock file, `fd-lock`) exists in `crates/native/src/rate_limit.rs` and is covered by tests, but `FetchClient` does not currently call it, so the native server does not pace its own arXiv requests. The Worker paces requests to 3 s within one isolate. Clients should follow arXiv's guidance on request rates.
- **Memory.** Downloaded PDFs and HTML are held in memory while converting; there is no explicit size cap. The in-memory response cache is bounded by entry count, not bytes.
- **Index failures.** If the database cannot be opened at startup, the server logs the error and continues without it; `hdrr` then returns "Database not initialized". If the crate is built without `embedded-db`, `hdrr` returns "embedded-db feature not enabled" and nothing is indexed.
- **Concurrency.** The database connection and response cache sit behind `std::sync::Mutex` locks; a poisoned lock surfaces as an error (database) or a cache miss (response cache).

## Dependencies and optional features

### Git and path dependencies

Only dependencies resolved from a git repository or a local path are listed. All other dependencies come from crates.io (see the `Cargo.toml` files and `Cargo.lock`).

| Dependency | Source | Pin | Required or feature-gated | Purpose |
|---|---|---|---|---|
| `pdf-inspector` | git: `https://github.com/firecrawl/pdf-inspector` | `rev = "704ca588"` (`crates/core/Cargo.toml`) | Required by `core`; always declared. Its code path is compiled only on non-`wasm32` targets, and the Worker uses a stub that returns an error | PDF to markdown extraction for the PDF fallback |
| `lopdf` | git: `https://github.com/J-F-Liu/lopdf` | `rev = 7a05512d831415b1f2b1ce522391d6beab8a1284` (resolved in `Cargo.lock`, pinned by `pdf-inspector`; transitive) | Required, via `pdf-inspector` | PDF parsing |
| `arxiv-search-rs-mcp-core` | path: `crates/core` (`../core`) | Workspace-local; version `0.1.0` | Required by `native` and `worker` | Shared logic (crate is not published) |

Both git sources are allow-listed in `deny.toml` (`unknown-git = "deny"`).

### Optional features

| Feature | Crate | Default | Effect |
|---|---|---|---|
| `embedded-db` | `native` | on | Enables `rusqlite` (bundled SQLite), the `db` module, indexing on retrieval, `ingest_corpus` storage and `hdrr`. Build without it using `--no-default-features` |

`cargo check -p arxiv-search-rs-mcp --no-default-features` compiles (with dead-code warnings). With the feature off, `retrieve_paper` and `search` work and `hdrr` returns an error.

## Native vs Worker targets

| | Native (`crates/native`) | Worker (`crates/worker`) |
|---|---|---|
| Build | Root workspace member | Separate workspace, `wasm32-unknown-unknown` (CI builds it from `crates/worker`) |
| Transport | stdio, or Streamable HTTP on `POST /mcp` (`rmcp`) | `POST /mcp`, `POST`/`GET /sse` (own JSON-RPC handling) |
| Tools | `search`, `retrieve_paper`, `hdrr`, `execute`, `ingest_corpus` | `search_papers`, `retrieve_paper` |
| Full text | HTML, then PDF fallback | HTML only |
| Paper metadata on retrieve | arXiv API or HTML page metadata | None (ID used as title) |
| `segmentation_k`, `full_payload` | Supported | Not supported |
| Cache and index | Disk cache, in-memory responses, SQLite | None |
| Deployment config | None needed | No `wrangler.toml` in the repository |

## Known gaps in this document

- Nothing here was exercised against live arXiv or Semantic Scholar from the environment in which this document was written (network access to arXiv was unavailable); the offline MCP handshake and tool listing of the native binary were run.
