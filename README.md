# agent-plugins

A catalogue of plugins for coding agents, published by [hlgr360](https://github.com/hlgr360).

The catalogue itself is agent-neutral. Each agent reads its own catalogue file from this repository; today that is Claude Code's.

## Claude Code and Claude Desktop

```
/plugin marketplace add hlgr360/agent-plugins
/plugin install <plugin>@hlgr360
```

The marketplace is named `hlgr360`, so plugins are installed as `<plugin>@hlgr360`. Which plugins are listed, and how they are built and released, is described in [docs/design/001-distribution.md](docs/design/001-distribution.md). The catalogue is empty until the first plugin has a release to point at.

## Other agents

Not supported yet. The design note describes how GitHub Copilot CLI and pi would fit.

## Layout

| Path | What |
|---|---|
| `.claude-plugin/marketplace.json` | Claude Code's catalogue |
| `docs/design/` | Design notes |
