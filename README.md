# arxiv-search-rs-mcp

![arxiv-search artwork](assets/social-card.png)

Search arXiv and retrieve prepared paper content through MCP. Retrieve paper metadata and text, prepare content for model ingestion, and perform document-routed retrieval. Run locally or build the Cloudflare Worker target (a reduced subset, see [Cloudflare Worker](#cloudflare-worker)).

Start with a single paper retrieval (`retrieve_paper`). Indexing and document-routed retrieval (`ingest_corpus`, `hdrr`) are optional and only needed when you want to query across several papers.

- **Retrieval for LLM ingestion:** fetches the paper from arXiv (HTML, falling back to PDF), prunes the reference list and boilerplate, and chunks the text. Math appears once as `$TeX$`.
- **Document-routed retrieval:** an optional embedded SQLite index (`embedded-db` feature, on by default) supports two-stage retrieval: route to documents, then search chunks within them. Ranking is TF-IDF cosine similarity, not neural embeddings.
- **Optional hierarchical segmentation:** `segmentation_k` on `retrieve_paper` clusters sentence-like segments (design references: arXiv:2507.09935 for segmentation, arXiv:2603.26815 for HDRR; not re-verified here).
- **Local caching:** fetched HTML, PDF and metadata are cached on disk for 7 days in the OS cache directory (`~/.cache/mcp/` on Linux, `~/Library/Caches/org.arxiv-search.mcp/` on macOS); up to 64 prepared `retrieve_paper` responses are also kept in memory, so repeat requests skip HTTP and re-processing.

Architecture, state, dependencies and the native-vs-Worker differences are described in [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Quick start

Requires a Rust toolchain (stable) and network access to GitHub and crates.io for the build.

```bash
git clone https://github.com/webfold-ai/arxiv-search.git
cd arxiv-search
cargo build --release -p arxiv-search-rs-mcp
# binary: target/release/arxiv-search-mcp
```

Register it with an MCP client. For Claude Code, add this to `.mcp.json` (use the absolute path to your build):

```json
{
  "mcpServers": {
    "arxiv-search": {
      "command": "/absolute/path/to/arxiv-search/target/release/arxiv-search-mcp",
      "args": ["--stdio"]
    }
  }
}
```

Smoke test (no network needed): the server should answer `initialize` and list its tools.

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"smoke","version":"1"}}}' \
  '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  | target/release/arxiv-search-mcp --stdio 2>/dev/null
```

The second response lists `search`, `retrieve_paper`, `hdrr`, `execute` and `ingest_corpus`. Calling `search` or `retrieve_paper` also needs outbound access to arxiv.org.

To have a coding agent do the install for you, see the [agent install prompt](docs/AGENT-INSTALL.md).

## Benchmarks

Measured against [`arxiv-mcp-server`](https://github.com/blazickjp/arxiv-mcp-server) 0.7.2 on the workload in [`benchmarks/`](benchmarks/README.md): 8 pinned papers, end to end over MCP stdio. This is a single run on one machine (4 vCPU cloud container, outbound traffic via an HTTPS proxy, 2026-09-29, commit `e2b6621`), so treat the figures as indicative rather than general. Totals cover the 7 papers where both tools return the paper; the 8th (Adam, `1412.6980`) has a broken arXiv HTML page, which `arxiv-search` serves from the PDF.

| | arxiv-search | arxiv-mcp-server 0.7.2 |
|---|---:|---:|
| First request, paper ID to full text (7 papers, total) | 4.1 s | 12.2 s |
| Repeat request (7 papers, total) | 14 ms | 46 ms |
| Tokens in context (7 papers, total) | 183,625 | 214,798 |
| Peak memory | 23-25 MiB | 81-432 MiB |
| Startup | 88 ms | 664 ms |

Charts, per-paper numbers, methodology and a one-command reproduction are in [`benchmarks/`](benchmarks/README.md). The totals above are sums of per-paper medians from the raw results file in that directory.

## Tools

### `search`
Search arXiv papers from the API. Pass a JSON object:

```json
{"q":"ti:attention AND au:vaswani","n":5,"sort":"relevance"}
```

| Field | Type | Default | Description |
|---|---|---:|---|
| `q` | string | required | arXiv query syntax with field filters and boolean operators |
| `n` | integer | `10` | Max results, capped at 50 |
| `offset` | integer | `0` | OpenSearch pagination offset for fetching subsequent pages |
| `from` | date | - | Start date `YYYY-MM-DD` |
| `to` | date | - | End date `YYYY-MM-DD` |
| `cats` | string[] | - | Category filter, for example `["cs.AI","cs.LG"]` |
| `sort` | string | `relevance` | `relevance` or `date` |

### `retrieve_paper`
Retrieve a paper directly from arXiv content URLs, prune it, and chunk it for model consumption. Pass:

```json
{"paper_id":"1706.03762","prune_references":true,"chunk_chars":4000,"chunk_overlap":200}
```

| Field | Type | Default | Description |
|---|---|---:|---|
| `paper_id` | string | required | arXiv ID, with or without `arxiv:` prefix and version suffix |
| `prune_references` | bool | `true` | Drops trailing references/bibliography noise |
| `chunk_chars` | integer | `4000` | Target chunk size |
| `chunk_overlap` | integer | `200` | Overlap between chunks |
| `segmentation_k` | float | - | Optional. If set, uses hierarchical segmentation with the given sensitivity parameter. |
| `full_payload` | bool | `false` | Return the legacy payload (raw markdown, per-chunk text, embeddings) instead of the compact view |

The response is compact JSON, with the paper text appearing exactly once:

- `paper`: metadata (title, authors, abstract, …)
- `source`: `html` or `pdf`. The PDF is used when there's no HTML rendering, or when arXiv's HTML render is degraded.
- `pruned_markdown`: cleaned paper text. The reference list is dropped, appendices are kept, and math appears once as `$TeX$`.
- `chunks`: `[start_char, end_char)` offsets into `pruned_markdown`, plus any hierarchy `context`

### `hdrr`
Hybrid Document-Routed Retrieval over papers you have already indexed. Stage 1 routes the query to the most relevant documents; stage 2 searches chunks only within those documents. Both stages rank by TF-IDF cosine similarity. Requires the `embedded-db` feature (on by default). Papers are indexed when you call `retrieve_paper` or `ingest_corpus`; with an empty index it returns no documents.

```json
{"q":"transformer scaling laws","limit_docs":5,"limit_chunks":10}
```

| Field | Type | Default | Description |
|---|---|---:|---|
| `q` | string | required | Query text |
| `limit_docs` | integer | `5` | Max documents kept after routing |
| `limit_chunks` | integer | `10` | Max chunks returned from the routed documents |

### `ingest_corpus`
Index several papers for `hdrr` without returning their text. Pass `{"paper_ids":["1706.03762","1810.04805"]}`; `prune_references`, `chunk_chars` and `chunk_overlap` behave as in `retrieve_paper`. The response reports `ingested`, `errors` and `total_requested`.

### `execute`
Single-paper operations selected by `op`: `abstract`, `download` (full text as markdown), `citations`, `recs` (the last two use Semantic Scholar) and `retrieve`. Pass `{"op":"abstract","id":"1706.03762"}`. Errors are returned inside the `result` field rather than as MCP errors.


## Local usage

```bash
cargo run -p arxiv-search-rs-mcp -- --stdio
```

Or run the Streamable HTTP server (MCP 2025-03-26, `POST /mcp`; default `127.0.0.1:3000`):

```bash
cargo run -p arxiv-search-rs-mcp -- --host 127.0.0.1 --port 3000
```

To build without the embedded database (no `hdrr`, no indexing), add `--no-default-features`.

## Cloudflare Worker

`crates/worker` is a separate Cargo workspace (it is not a member of the root workspace) that compiles to `wasm32-unknown-unknown`. It is a reduced subset of the native server:

- Tools: `search_papers` and `retrieve_paper` only (no `hdrr`, `execute`, `ingest_corpus`, no `segmentation_k` or `full_payload`).
- `retrieve_paper` uses arXiv HTML only; there is no PDF fallback, and paper metadata is not fetched (the title is the paper ID).
- Endpoints: `POST /mcp`, `POST`/`GET /sse`, and `GET /` for service info.
- No caching, and a best-effort 3 s pause between outbound arXiv requests within one isolate.

```bash
cd crates/worker
cargo check --target wasm32-unknown-unknown
```

No `wrangler.toml` is checked in, so deployment configuration (for example a `worker-build` setup) has to be supplied by you.

## Environment

- `SEMANTIC_SCHOLAR_API_KEY`: raises Semantic Scholar rate limits for `citations` and `recs`
- `HOST`: HTTP bind host (same as `--host`)
- `PORT`: HTTP bind port (same as `--port`)
- `RUST_LOG`: logging filter (logs go to stderr)
