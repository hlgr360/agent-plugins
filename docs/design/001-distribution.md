# 001: Distributing claude-mnemonic (and later plugins) for coding agents

Status: **implemented for Claude Code and Claude Desktop** (2026-10-04); other agents are concept only. Written 2026-10-04, updated the same day after the first release (`v0.21.95.1`).

## Goal

Make `claude-mnemonic` (the fork at `hlgr360/claude-mnemonic`) installable as a plugin for **Claude Code and Claude Desktop**, from a public catalogue that can later hold more plugins, and keep the design open for **GitHub Copilot CLI and pi** without building for them yet.

## Decisions taken

| Decision | Choice |
|---|---|
| Catalogue repository | `hlgr360/agent-plugins`, public, agent-neutral name |
| Marketplace name | `hlgr360` (it lives in `.claude-plugin/marketplace.json`; the repository name never appears in an install id) |
| Plugin name | `claude-mnemonic`, installed as `claude-mnemonic@hlgr360` |
| Plugin source | A **thin zip with no binaries**, built and signed by the `claude-mnemonic` release workflow and listed here as an `archive` source (URL and sha256). One zip serves every platform. |
| Binaries | Built on native runners and published as GitHub Release assets by the `claude-mnemonic` repository; the plugin downloads and verifies them on first use |
| Versions | The upstream version the release contains plus a fork number: `0.21.95.1`, `.2`, ... (see below) |
| Other agents | Concept only (see below); not built now |

The plugin name stays the same as upstream's so that nothing existing changes. A marketplace may hold the same plugin name as another marketplace, but the two plugins would share the same hooks, MCP server name, data directory and port, so they are alternatives: install one. `claude plugin validate` reports the name as reserved (third-party names may not start with `claude-`); Claude Code installs and loads such a plugin all the same, and the install through this catalogue was tested.

## Architecture

The core is agent-neutral: a local worker (HTTP, SQLite, embeddings), a stdio MCP server and a web dashboard. Only a thin integration layer is per agent.

| Layer | Portable? |
|---|---|
| MCP server | Yes: any MCP client |
| Skills (`skills/<name>/SKILL.md`, the open Agent Skills format) and `AGENTS.md` | Yes, widely adopted |
| Hooks (automatic capture), `plugin.json`, `marketplace.json`, `${CLAUDE_PLUGIN_ROOT}`, slash commands, `.plugin` zip | Claude-specific |
| `.mcpb` extension | Claude Desktop only |

## The catalogue

`.claude-plugin/marketplace.json` in this repository lists each plugin as an external, pinned source. For claude-mnemonic:

```json
{
  "name": "claude-mnemonic",
  "source": {
    "source": "archive",
    "url": "https://github.com/hlgr360/claude-mnemonic/releases/download/v0.21.95.1/claude-mnemonic-plugin_0.21.95.1.zip",
    "sha256": "e5806ebeeefccde2ded4333d8bda56dc14a93bdb75ce5219c737efef28b6917c"
  }
}
```

An `archive` source is a zip over HTTPS with a sha256 pin; Claude Code refuses a download that does not match (tested: a wrong value fails with "Plugin archive integrity check failed" and nothing is installed). It needs Claude Code 2.1.224 or later. The plugin root is at the top of the zip. The marketplace repository does not need to contain plugin files; each plugin keeps its own repository and release cycle. Work or company plugins are published in their own marketplaces, never here.

**A new release is a catalogue change:** point `url` and `sha256` at the new release's plugin zip (its sha256 is in the release's `checksums.txt`). The plugin version comes from the zip's `plugin.json`, so the entry carries no `version` of its own.

## How the plugin and its binaries fit together

1. The release workflow in the `claude-mnemonic` repository builds the binaries per platform and publishes them as release assets, with a `checksums.txt` signed by cosign (keyless), together with the plugin zip. The signature covers the zip too.
2. The plugin holds only thin wrappers (hooks, MCP launcher, skills) and `lib/ensure-binaries.sh`.
3. On first use the wrapper downloads the archive of the plugin's own version, checks it against `checksums.txt` and, when cosign is installed, against the signature (the same check the in-app updater makes), and installs the binaries into `~/.claude-mnemonic/bin`, where the hooks and the worker already look first. Supported: macOS arm64 and Linux amd64; Windows is not supported by the plugin yet.
4. **Who manages the binaries: the plugin installs, the in-app updater keeps working.** It installs when the worker or MCP server is missing, or when its version marker is older than the plugin's; it **never replaces binaries that have no marker** (`make install`, `install.sh`) and never downgrades. The plugin version is the floor.
5. Hooks never wait for the download: the session-start hook starts it in the background, so the first session may run without memory; the MCP server waits for it.

The user's data (`~/.claude-mnemonic`: the database, settings and embeddings) is separate from the binaries and is never touched by the plugin. The repository's unregister and uninstall scripts keep it unless `--purge` is given.

## Release pipeline

- Own GitHub Actions workflow (`release-native.yaml` in the `claude-mnemonic` repository), one job per native runner (macOS arm64, Linux amd64, Windows amd64: the build uses CGO), a plugin job, keyless cosign signing of `checksums.txt` with a verification step that uses the updater's own arguments, then publish. Only a version tag publishes.
- Upstream's workflow calls a shared reusable workflow and needs a GoReleaser key the fork does not have; it is guarded to run only in upstream's own repository.
- The in-app updater, the installers and the release scripts take the release repository and the signing identity from one place each, so a fork needs no code change.
- **Versions:** a fork release is named for the upstream version it contains plus a fork number, `v0.21.95.1`, `.2`, ...; the first release is `v0.21.95.1`. A letter suffix (`0.21.95a`) was rejected: the updater compares dotted numbers only and would mis-read it. Neither form is valid semver, which matters only for the `.mcpb` bundle.

## Local build and test

`scripts/build-plugin.sh <version>` in the `claude-mnemonic` repository is the single entry point, used by CI and by hand: assemble the tree, validate it (`claude plugin validate --strict`, accepting exactly the reserved-name error), write a deterministic zip. Try it with `claude --plugin-dir dist/plugin` (its hooks run for real). The catalogue entry itself can be tested without touching a real setup: run `claude plugin marketplace add` and `claude plugin install claude-mnemonic@hlgr360` with `HOME` and `CLAUDE_CONFIG_DIR` pointing at an empty directory.

## Claude Desktop

- Code tab: loads the whole plugin. Cowork: skills, commands, hooks and local MCP servers. Chat: skills and commands only.
- A plugin uploaded to a Claude account syncs into Claude Code as `<name>@synced`; it is not loaded while a local plugin of the same name exists.
- An `.mcpb` has no field for persistent instructions. The plugin carries the instruction as a skill, but in Chat the skill did **not** make the model call the connector (one prompt measured), so the pasted instruction is still required. The `.mcpb` itself was evaluated and not built.

## Later targets (concept only)

- **Portable core:** MCP server + `SKILL.md` skills + `AGENTS.md`, with per-agent manifests beside them (`.claude-plugin/` now).
- **GitHub Copilot CLI:** plugins can bundle MCP servers, agents, skills and hooks and install with `/plugin install owner/repo`. Its manifest and catalogue file names are not confirmed.
- **pi:** packages (npm or git) with a `pi` key in `package.json` for extensions, skills, prompts and themes; extensions are TypeScript. Skill-format and MCP support are not confirmed.
- **Build pattern:** one tool-agnostic source and one generated output per agent, as an existing skills repository already does for several agents.
- Automatic capture depends on each agent's hook support, which is not verified.

## Verified and not verified

Verified (2026-10-04):
- The release `v0.21.95.1`: six assets, checksums match, `cosign verify-blob` passes with the updater's arguments, the certificate identity is the release workflow at the tag.
- The plugin's first-run download against that real release with real cosign, in an isolated home directory.
- The catalogue entry: added to an empty Claude config, installed (version `0.21.95.1`, 3 skills, 6 hooks, 1 MCP server), and a wrong sha256 is refused.
- `claude plugin validate` reports only the reserved-name error for the plugin name.

Not verified: how Claude Desktop's own plugin pages show this plugin after the catalogue install, Windows, what happens to installed plugins if a marketplace's source moves (docs are silent; likely remove and re-add), Copilot CLI's plugin manifest, pi's skill and MCP support, and whether Cowork runs the hooks in practice.

## Sequence

1. Make the updater and the scripts repository-agnostic (`claude-mnemonic`). Done.
2. Add the release workflow (`claude-mnemonic`). Done.
3. Add the build script and the thin plugin (`claude-mnemonic`). Done; the marketplace rename was not needed (the names differ).
4. Add the catalogue entry here once a release exists. Done with `v0.21.95.1`.
5. Build the `.mcpb` (`claude-mnemonic`, ticket #24). Evaluated, not built.
6. The instruction skill and its test in Chat. Done; the paste step stays.
7. Later: a design note for Copilot CLI and pi (issue #2).

## Risks

Binary size (about 31 MB per platform archive), CGO builds needing native runners, a plugin that diverges from upstream's update channel, a catalogue entry that must be updated by hand for each release, and the unverified points above.
