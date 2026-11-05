# Agent install prompt

Paste the block below into any coding-agent context (opencode, Claude Code,
Codex, Cursor, etc.) to have the agent build, install, and wire the server as
an MCP tool. Nothing else is needed — the prompt is self-contained.

For a manual install, see [Quick start](../README.md#quick-start) in the README.

> Notes (outside the prompt):
> - The prompt text below is the original, moved from the README. The only edit
>   is the repository slug, now the canonical `webfold-ai/arxiv-search`.
> - The native server currently exposes five tools: `search`, `retrieve_paper`,
>   `hdrr`, `execute` and `ingest_corpus`. The prompt's closing paragraph names
>   only the first four.

```text
Install webfold-ai/arxiv-search as a local MCP server and register it in this
session's agent config so I can search arXiv from future turns.

Steps:
1. Clone into a persistent location (skip if you already have a checkout):
   `gh repo clone webfold-ai/arxiv-search ~/projects/arxiv-search`
   Below, replace `<repo>` with that path.
2. Build the native binary:
   `cargo build --release --manifest-path <repo>/Cargo.toml --bin arxiv-search-mcp`
3. Install it onto PATH:
   `mkdir -p ~/.local/bin && cp <repo>/target/release/arxiv-search-mcp ~/.local/bin/`
4. Register as an MCP server in this project's agent config:
   - opencode: add to `~/.config/opencode/opencode.jsonc` (or project `.opencode/opencode.json`):
       "mcp": {
         "arxiv-search": {
           "type": "local",
           "command": ["/home/USER/.local/bin/arxiv-search-mcp", "--stdio"],
           "enabled": true
         }
       }
     Replace USER with `whoami`. Keep any pre-existing top-level keys; only add `mcp`.
   - Claude Code: append to `.mcp.json`:
       { "mcpServers": { "arxiv-search": { "command": "/home/USER/.local/bin/arxiv-search-mcp", "args": ["--stdio"] } } }
5. Smoke test before telling me to restart. Run the binary with `--host 127.0.0.1 --port 8765`
   in the background, then POST an `initialize` and a `tools/call` to `search`:
     curl -sSi -X POST http://127.0.0.1:8765/mcp \
       -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
       -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"smoke","version":"1"}}}'
   Capture the `mcp-session-id` header, send `notifications/initialized`, then call:
     {"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"search","arguments":{"q":"ti:test","n":1}}}
   Expect a `papers` array in the response. Kill the background server.
6. Tell me to restart the agent so the MCP server loads natively. After restart the
   `search` / `retrieve_paper` / `hdrr` / `execute` tools will be first-class.

When invoked natively the tool exposes four MCP tools — `search`, `retrieve_paper`,
`hdrr`, `execute`. Query syntax is arXiv Lucene (`ti:`, `au:`, `abs:`, boolean
`AND`/`OR`/`NOT`, quoted phrases). Cache lives at `~/.cache/mcp/` (Linux).
```
