# agent-plugins

A catalogue of plugins for coding agents, published by [hlgr360](https://github.com/hlgr360).

The catalogue itself is agent-neutral. Each agent reads its own catalogue file from this repository; today that is Claude Code's.

## Claude Code

```
/plugin marketplace add hlgr360/agent-plugins
/plugin install <plugin>@hlgr360
```

The marketplace is named `hlgr360`, so plugins are installed as `<plugin>@hlgr360`. How plugins are built, released and listed is described in [docs/design/001-distribution.md](docs/design/001-distribution.md).

## Plugins

| Plugin | What | Install |
|---|---|---|
| [claude-mnemonic](https://github.com/hlgr360/claude-mnemonic) | Persistent memory for Claude Code: the decisions, findings and fixes from earlier sessions, saved and loaded automatically and searchable, with a local web dashboard. Claude Desktop uses a separate extension (a `.mcpb` file on the plugin's release). A fork of [lukaszraczylo/claude-mnemonic](https://github.com/lukaszraczylo/claude-mnemonic) (MIT). | `/plugin install claude-mnemonic@hlgr360` |
| [local-llm-mcp](https://github.com/hlgr360/local-llm-mcp) | Offloads code-shaped work (generation, review, refactoring, test-writing) to a local LLM (Ollama, llama-server, vLLM or LM Studio), with file-aware tools that read files server-side to save context, and an optional Read-gate hook that nudges Claude toward them. A fork of [Jadael/OllamaClaude](https://github.com/Jadael/OllamaClaude) (AGPL-3.0). | `/plugin install local-llm-mcp@hlgr360` |

Requirements: none for the catalogue itself (the plugins are in this repository, as in-repo sources; an earlier `archive` source needed Claude Code 2.1.224 and was not accepted by Claude Desktop's marketplace sync). `claude-mnemonic` downloads its binaries from its own release on first use and checks them (see the plugin's README); it supports macOS on Apple silicon and Linux on x86-64, and should not be installed next to another install of claude-mnemonic (they register the same hooks). Claude Desktop is not served by this plugin: see the project README for the Desktop extension and the one setting chat needs. `local-llm-mcp` needs Node.js 22 or later (its MCP server is installed with `npx` from the tagged release on first start) and a local LLM server; it is configured through environment variables (see the plugin's README).

`claude plugin validate` on this catalogue reports `claude-mnemonic` as a reserved name (third-party names may not start with `claude-`). That check is made only by the validator; Claude Code installs and loads the plugin all the same.

## Other agents

Not supported yet. The design note describes how GitHub Copilot CLI and pi would fit.

## Layout

| Path | What |
|---|---|
| `.claude-plugin/marketplace.json` | Claude Code's catalogue |
| `plugins/<name>/` | The plugin as released (do not edit by hand): `claude-mnemonic` is unpacked by its own update script; `local-llm-mcp` is copied from its tagged release, and its `.mcp.json` pins the server to that tag |
| `docs/design/` | Design notes |
