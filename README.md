# mcp-notes-rag

An MCP (Model Context Protocol) server that gives any AI agent semantic + full-text search over your personal notes vault, plus daily note and task management.

- **Transport:** stdio (runs locally, consumed by Pi, Claude Code, OpenCode, etc.)
- **Search:** Hybrid vector + FTS5 with reciprocal rank fusion (RRF)
- **Embeddings:** Ollama (default: `snowflake-arctic-embed2`)
- **Index:** SQLite native (no C extensions, no ChromaDB)
- **Vaults:** Supports English (`~/Notes`) and Spanish (`~/Notas`) via separate entry points

---

## Prerequisites

- Python **3.12+**
- [Ollama](https://ollama.com) running locally (or reachable via `OLLAMA_HOST`)
- The embedding model pulled: `ollama pull snowflake-arctic-embed2`
- Your notes vault at `~/Notes` (EN) or `~/Notas` (ES)

---

## Installation

### With `uv` (recommended)

```bash
uv pip install -e .
```

### With `pip`

```bash
pip install -e .
```

This registers two CLI entry points:
- `mcp-server-notes-en`
- `mcp-server-notes-es`

---

## Configuration

All settings are environment variables. Nothing is hardcoded.

| Variable | Default (EN) | Default (ES) | Description |
|----------|--------------|--------------|-------------|
| `NOTES_VAULT_PATH` | `~/Notes` | `~/Notas` | Path to your notes vault |
| `NOTES_INDEX_DB` | `~/.local/share/notes-rag/index.db` | `~/.local/share/notas-rag/index.db` | SQLite index location |
| `OLLAMA_HOST` | `http://localhost:11434` | same | Ollama API endpoint |
| `NOTES_EMBED_MODEL` | `snowflake-arctic-embed2` | same | Embedding model name |

Example before running:

```bash
export NOTES_VAULT_PATH="$HOME/Notes"
export OLLAMA_HOST="http://localhost:11434"
export NOTES_EMBED_MODEL="snowflake-arctic-embed2"
```

---

## Available Tools

All tool **descriptions** are localized to the server language, so the LLM invokes them correctly.

| Tool | Purpose |
|------|---------|
| `query_notes` | Hybrid semantic + keyword search with filters (type, tag, date range). |
| `get_daily_note` | Read a daily note by date expression. Creates it from template if missing. |
| `add_task` | Append a task `- [ ] ...` under a category in the daily note. |
| `complete_task` | Mark a pending task `[x]` by matching its text. |
| `get_note` | Read the full content of any note by relative vault path. |
| `list_notes` | List notes in a directory or of a specific type. |
| `index_vault` | (Re)build the search index. Incremental by default; full rebuild runs in background. |
| `index_status` | Show index health: notes indexed, chunks, pending changes, model used. |
| `create_zettel` | Create a new atomic Zettelkasten note with mandatory YAML frontmatter. |

---

## Pi Setup

Copy `.pi/mcp.json` into your project or user Pi config, or add the server directly:

```bash
pi mcp add notes-rag-es --env OLLAMA_HOST='http://localhost:11434' -- uv run --project ~/Projects/mcp-rag-notes mcp-server-notes-es
```

Or with `pip` (if installed in the active environment):

```bash
pi mcp add notes-rag-es -- mcp-server-notes-es
```

Check the connection:

```bash
pi mcp list
```

Then start Pi and the tools are available immediately (exposure is `direct` in the example config).

---

## Quick Test

After installing, test the server directly:

```bash
# Spanish server
mcp-server-notes-es

# English server
mcp-server-notes-en
```

The server expects MCP stdio messages. To test end-to-end, connect via Pi or the MCP inspector:

```bash
npx -y @anthropics/mcp-inspector mcp-server-notes-es
```

---

## Architecture Notes

- **One package, two entry points.** `mcp-server-notes-en` and `mcp-server-notes-es` share 100% of the code; only the locale and default paths change.
- **Async-safe.** The MCP SDK is async, but all blocking work (SQLite I/O, Ollama HTTP, file reads) runs in threads via `asyncio.to_thread`. The event loop is never blocked.
- **Crash-proof tools.** Every tool is wrapped in `try/except`. Uncaught exceptions never crash the stdio process; they return an `isError=True` result with a readable message.
- **Incremental indexing.** `index_vault` scans only changed files by SHA-256 hash by default. A full rebuild is triggered with `full_rebuild=true` and runs in a background thread so the MCP response is instant.
- **Hybrid search.** Vector similarity (cosine) is fused with FTS5 keyword ranking via reciprocal rank fusion (RRF). Results include snippet highlighting.
- **Chunking by headings.** Notes are split at H1/H2/H3 boundaries; each chunk is enriched with title, aliases, tags, and description before embedding.

---

## Vault Structure Expected

Your vault should follow a standard Obsidian-like layout:

```
~/Notas or ~/Notes/
  ├── journal/           # Daily notes: YYYY-MM-DD.md
  ├── zettelkasten/      # Atomic notes
  ├── projects/          # Project folders with README.md
  ├── finanzas/          # Year-based finance notes (ES) / finance/ (EN)
  ├── templates/         # Templates
  └── inbox/             # Inbox
```

Top-level folders named `archived`, `assets`, and any folder starting with `.` (`.obsidian`, `.git`, `.trash`) are ignored by the indexer.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `Ollama connection refused` | Start Ollama: `ollama serve` (or check `OLLAMA_HOST`). |
| `Model changed` error | Run `index_vault` with `full_rebuild=true` after changing `NOTES_EMBED_MODEL`. |
| `No results` | Run `index_vault` first to build the index. |
| Slow first query | The first `query_notes` auto-syncs changed files. Run `index_vault` periodically to keep it fast. |
| Daily note not created | Ensure `journal/` exists in the vault or that the server has write permissions. |

---

## License

MIT
