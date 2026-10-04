# 001: Distributing claude-mnemonic (and later plugins) for coding agents

Status: **proposed**. Written 2026-10-04.

## Goal

Make `claude-mnemonic` (the fork at `hlgr360/claude-mnemonic`) installable as a plugin for **Claude Code and Claude Desktop**, from a public catalogue that can later hold more plugins, and keep the design open for **GitHub Copilot CLI and pi** without building for them yet.

## Decisions taken

| Decision | Choice |
|---|---|
| Catalogue repository | `hlgr360/agent-plugins`, public, agent-neutral name |
| Marketplace name | `hlgr360` (it lives in `.claude-plugin/marketplace.json`; the repository name never appears in an install id) |
| Plugin name | `claude-mnemonic`, installed as `claude-mnemonic@hlgr360` |
| Plugin source | Stays in the `claude-mnemonic` repository, in its `plugin/` directory |
| Binaries | Built and published by a release workflow in the `claude-mnemonic` repository |
| Other agents | Concept only (see below); not built now |

The plugin name stays the same as upstream's so that slash commands (`/claude-mnemonic:dashboard`) and the docs do not change. A marketplace may hold the same plugin name as another marketplace, but the two plugins would share the same commands, hooks, MCP server name, data directory and port, so they are alternatives: install one.

## Architecture

The core is agent-neutral: a local worker (HTTP, SQLite, embeddings), a stdio MCP server and a web dashboard. Only a thin integration layer is per agent.

| Layer | Portable? |
|---|---|
| MCP server | Yes: any MCP client |
| Skills (`skills/<name>/SKILL.md`, the open Agent Skills format) and `AGENTS.md` | Yes, widely adopted |
| Hooks (automatic capture), `plugin.json`, `marketplace.json`, `${CLAUDE_PLUGIN_ROOT}`, slash commands, `.plugin` zip | Claude-specific |
| `.mcpb` extension | Claude Desktop only |

## The catalogue

`.claude-plugin/marketplace.json` in this repository lists plugins as external, pinned sources, for example:

```json
{
  "name": "claude-mnemonic",
  "source": { "source": "git-subdir", "url": "hlgr360/claude-mnemonic", "path": "plugin", "ref": "v1.0.0" }
}
```

The marketplace repository does not need to contain plugin files. Each plugin keeps its own repository and release cycle. Pinning `ref` (and optionally `sha`) to a release tag makes the plugin version and the binaries it fetches the same thing.

Work or company plugins are published in their own marketplaces, never here.

## How the plugin and its binaries fit together

1. A release workflow in the `claude-mnemonic` repository builds the binaries per platform and publishes them as GitHub Release assets, signed (cosign keyless) with a checksums file.
2. The plugin in `plugin/` holds only thin wrappers (hooks, MCP launcher, commands).
3. On first run, and whenever the plugin version changes, the wrapper downloads the matching binaries for the platform from that release, verifies the checksum and signature, and stores them in `~/.claude-mnemonic/bin`, which survives plugin cache replacement.
4. Updating means bumping the `ref` in the catalogue; users update the plugin and the wrapper fetches the new binaries.

An alternative is an `archive` source pointing at a full plugin zip. A catalogue entry takes one URL, so that would need a separate entry per platform. Not chosen.

To decide later: the plugin's wrapper and the application's own self-updater would both manage binaries. One of them should be switched off in plugin mode.

## Release pipeline

- Own GitHub Actions workflow, one job per native runner (macOS arm64, Linux amd64, Windows amd64: the build uses CGO), the same archive layout as today, keyless cosign signing, checksums, then publish.
- Upstream's workflow calls a shared reusable workflow and needs a GoReleaser key the fork does not have; the fork's `Release` workflow is disabled for that reason.
- **Prerequisite:** the in-app updater, `install.sh`, `install.ps1`, `update-marketplace.sh` and `register-plugin.sh` are hard-wired to upstream's repository and signing identity (a fork-signed release fails the updater's check). They must become repository-agnostic first.

## Local build and test

One script (`scripts/build-plugin.sh` in the plugin's repository) is the single entry point, used by CI and by hand: compile, assemble the plugin tree, validate it (`claude plugin validate <tree> --strict`), zip deterministically. Test with `claude --plugin-dir <tree>`; the plugin cache is keyed by version, so a rebuild needs a new version or `--plugin-dir`. The version comes from one source, not several files.

## Claude Desktop

- Code tab: loads the whole plugin. Cowork: skills, commands, hooks and local MCP servers. Chat: skills and commands only.
- Chat therefore needs the **`.mcpb`** (a zip with `manifest.json`, platform binaries and optional `user_config`), built by the same release. The existing config-file route stays.
- An `.mcpb` has no field for persistent instructions. A skill in the plugin could carry "use the memory when I ask about earlier work"; whether skills are honoured in Chat for this purpose needs testing.

## Later targets (concept only)

- **Portable core:** MCP server + `SKILL.md` skills + `AGENTS.md`, with per-agent manifests beside them (`.claude-plugin/` now).
- **GitHub Copilot CLI:** plugins can bundle MCP servers, agents, skills and hooks and install with `/plugin install owner/repo`. Its manifest and catalogue file names are not confirmed.
- **pi:** packages (npm or git) with a `pi` key in `package.json` for extensions, skills, prompts and themes; extensions are TypeScript. Skill-format and MCP support are not confirmed.
- **Build pattern:** one tool-agnostic source and one generated output per agent, as an existing skills repository already does for several agents.
- Automatic capture depends on each agent's hook support, which is not verified.

## Verified and not verified

Confirmed from the Claude Code documentation (as reported by a research pass; URLs: `code.claude.com/docs/en/plugins/marketplace-reference.md`, `.../plugins/publish.md`, `claude.com/docs/plugins/platform-support.md`, `claude.com/docs/connectors/building/mcpb.md`; not individually re-fetched): external plugin sources and pinning, one marketplace per `name`, plugin ids `plugin@marketplace`, where plugins load in Desktop, the `.mcpb` format. `claude plugin validate` exists and validates a marketplace manifest.

Not confirmed: what happens to installed plugins if a marketplace's source moves (docs are silent; likely remove and re-add), Copilot CLI's plugin manifest, pi's skill and MCP support, whether skills carry persistent instructions in Chat, and whether Cowork runs hooks on the user's computer in practice.

## Open decisions

1. Binary delivery: verified download on first run (leaning) or an `archive` source.
2. Which of the plugin wrapper and the self-updater manages binaries in plugin mode.
3. Whether the `marketplace.json` kept in the application repository (used by its install script) is renamed or dropped, since two marketplaces cannot share a name.

## Sequence

1. Make the updater and the scripts repository-agnostic (`claude-mnemonic`).
2. Add the release workflow (`claude-mnemonic`).
3. Add the build script, the plugin manifests and the marketplace rename (`claude-mnemonic`).
4. Add the catalogue entry here once a release exists.
5. Build the `.mcpb` (`claude-mnemonic`, existing ticket on the Desktop extension).
6. Add the instruction skill and test it in Chat.
7. Later: a design note for Copilot CLI and pi.

## Risks

Binary size (about 166 MB of embedded libraries per platform), CGO builds needing native runners, a plugin that diverges from upstream's update channel, and the unverified points above.
