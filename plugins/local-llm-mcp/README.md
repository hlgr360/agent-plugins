# local-llm-mcp

Claude Code plugin for [local-llm-mcp](https://github.com/hlgr360/local-llm-mcp): an MCP server
that hands code-shaped work (generation, review, refactoring, explanation, test-writing) to a
local LLM, so it doesn't use your Claude context. The file-aware tools read files inside the
server, so Claude never has to read them itself.

```
/plugin marketplace add hlgr360/agent-plugins
/plugin install local-llm-mcp@hlgr360
```

## What it installs

- **The `local_llm` MCP server** (15 `local_llm_*` tools), run with
  `npx -y github:hlgr360/local-llm-mcp#v3.3.0`. A plugin has no build step, so the server is
  installed by `npx` on first start; this needs Node.js 22 or later and network access to GitHub
  the first time.
- **A Read-gate hook** (`PreToolUse` on `Read`, needs `jq` and `curl`). It blocks the first
  `Read` of a file over 300 bytes per session and points Claude at `local_llm_review_file`,
  `local_llm_explain_file` or `local_llm_analyze_files` instead. Reading the same file again
  goes through, so editing still works. It only does this while a local LLM server answers
  (the same one the MCP server finds, see below); otherwise it lets every `Read` through.

## Finding the server

Nothing to configure for the common setups: the server looks for a running **Ollama** at
`localhost:11434`, then **llama-server** at `localhost:8080`, and uses the first that
answers. If neither does, the error says which one is installed but not running, and how to
start it.

## Configuration

The server reads its settings from the environment Claude Code is started in, e.g. your shell
profile. All are optional:

| Variable | Default | What |
|---|---|---|
| `LOCAL_LLM_BASE_URL` | found automatically | Set it for any other server or port, e.g. vLLM or LM Studio |
| `LOCAL_LLM_CHAT_MODEL` | auto-detected | Model for the chat tools, e.g. `gemma4:e4b-mlx` |
| `LOCAL_LLM_EMBEDDING_MODEL` | auto-detected | Model for `local_llm_semantic_similarity`, e.g. `nomic-embed-text` |
| `LOCAL_LLM_MIN_CTX` | `32768` | Ollama only: minimum context window; a larger configured one is kept |

With Ollama, a configured model that isn't installed is pulled on first use
(`LOCAL_LLM_AUTO_PULL=0` turns that off). All settings, backends and tools are described in the
[project README](https://github.com/hlgr360/local-llm-mcp#readme).

The project repository also has its own marketplace (`local-llm-mcp@local-llm-mcp-marketplace`).
Install the plugin from one marketplace only, or the hook and server are registered twice.

## Licence

AGPL-3.0-only, see `LICENSE` and `NOTICE`. A fork of
[Jadael/OllamaClaude](https://github.com/Jadael/OllamaClaude).
