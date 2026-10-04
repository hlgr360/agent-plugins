# agent-plugins

A catalogue of plugins for coding agents, published by [hlgr360](https://github.com/hlgr360).

The catalogue itself is agent-neutral. Each agent reads its own catalogue file from this repository; today that is Claude Code's.

## Claude Code and Claude Desktop

```
/plugin marketplace add hlgr360/agent-plugins
/plugin install <plugin>@hlgr360
```

The marketplace is named `hlgr360`, so plugins are installed as `<plugin>@hlgr360`. How plugins are built, released and listed is described in [docs/design/001-distribution.md](docs/design/001-distribution.md).

## Plugins

| Plugin | What | Install |
|---|---|---|
| [claude-mnemonic](https://github.com/hlgr360/claude-mnemonic) | Persistent memory for Claude Code and Claude Desktop: the decisions, findings and fixes from earlier sessions, searchable and shared by both, with a local web dashboard. A fork of [lukaszraczylo/claude-mnemonic](https://github.com/lukaszraczylo/claude-mnemonic) (MIT). | `/plugin install claude-mnemonic@hlgr360` |

Requirements: Claude Code 2.1.224 or later (the entries are `archive` sources). `claude-mnemonic` downloads its binaries from its own release on first use and checks them (see the plugin's README); it supports macOS on Apple silicon and Linux on x86-64, and should not be installed next to another install of claude-mnemonic (they register the same hooks). Claude Desktop chat needs one extra setting, described in the plugin's README.

`claude plugin validate` on this catalogue reports `claude-mnemonic` as a reserved name (third-party names may not start with `claude-`). That check is made only by the validator; Claude Code installs and loads the plugin all the same.

## Other agents

Not supported yet. The design note describes how GitHub Copilot CLI and pi would fit.

## Layout

| Path | What |
|---|---|
| `.claude-plugin/marketplace.json` | Claude Code's catalogue |
| `docs/design/` | Design notes |
