# 001: Distributing claude-mnemonic (and later plugins) for coding agents

Status: **implemented**: the plugin for Claude Code (this catalogue) and a Desktop extension for Claude Desktop (a file on the `claude-mnemonic` release); other agents are concept only. Written 2026-10-04; revised 2026-10-05 after it turned out that Desktop does not give a plugin's tools to chat or Cowork (see "Claude Desktop").

## Goal

Make `claude-mnemonic` (the fork at `hlgr360/claude-mnemonic`) installable for **Claude Code (a plugin) and Claude Desktop (an extension)**, from a public catalogue that can later hold more plugins, and keep the design open for **GitHub Copilot CLI and pi** without building for them yet.

## Decisions taken

| Decision | Choice |
|---|---|
| Catalogue repository | `hlgr360/agent-plugins`, public, agent-neutral name |
| Marketplace name | `hlgr360` (it lives in `.claude-plugin/marketplace.json`; the repository name never appears in an install id) |
| Plugin name | `claude-mnemonic`, installed as `claude-mnemonic@hlgr360` |
| Plugin source | A **thin zip with no binaries**, built and signed by the `claude-mnemonic` release workflow and committed here, unpacked, as an in-repo source (it was an `archive` source with URL and sha256 until `0.21.95.3`). One zip serves every platform. |
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

`.claude-plugin/marketplace.json` in this repository lists each plugin with an **in-repo source**, and the plugin itself is committed under `plugins/<name>/`. For claude-mnemonic:

```json
{
  "name": "claude-mnemonic",
  "source": "./plugins/claude-mnemonic"
}
```

The tree under `plugins/claude-mnemonic/` is the release's plugin zip, unpacked by `scripts/update-catalogue.sh` in the `claude-mnemonic` repository after it has checked the download (the sha256 equals the one in the release's signed `checksums.txt`, the signature verifies, `plugin.json` carries the tag's version and is within the upload form's limits) and installed the edited catalogue in an isolated Claude config. The pull request records the zip's sha256. Because the plugin is a thin zip with no binaries, it is small (a few hundred lines); each plugin keeps its own repository and release cycle, and this repository only carries the released tree. Work or company plugins are published in their own marketplaces, never here.

**A new release is a catalogue change:** run the script for the new tag; it replaces `plugins/claude-mnemonic/` and opens the pull request. The plugin version comes from the tree's `plugin.json`, so the entry carries no `version` of its own.

**Why not an `archive` source.** The first releases used one (the zip's URL and a sha256 pin; Claude Code refuses a download that does not match, tested). It needs Claude Code 2.1.224 or later, and Claude Desktop's marketplace sync, which goes through claude.ai's servers, failed with "Marketplace sync failed" for it while Claude Code installed fine. After the switch to the in-repo source (`0.21.95.3`) the maintainer confirmed that adding the marketplace and installing the plugin works (2026-10-04). The control (an in-repo marketplace failing or working before the change) was not run, so the `archive` source is the probable cause, not an isolated one.

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

**Decision (2026-10-05): two routes, no overlap.** Claude Code installs the plugin from this catalogue; Claude Desktop (chat and Cowork) installs the **`.mcpb` extension** from the release (`claude-mnemonic-desktop_<version>.mcpb`, Settings > Extensions > Install Extension, or an organization's custom extension upload). The plugin does not go to Desktop.

Why (documented, and checked in Desktop's logs on 2026-10-05):
- Anthropic's docs: "A plugin's local MCP server runs in Cowork and Claude Code, not in chat." Desktop's logs agree: the plugin's server runs on the Mac and is announced to Desktop's local bridge, but the chat's web layer never attaches it (no `plugin:` server in its logs, ever), while config servers and extensions are attached.
- In Cowork, `claude` runs in a Linux VM and the plugin is mounted there; its server cannot start (no binaries there, a blocked download, no build for the VM's architecture) and could not reach the worker anyway. The VM-side log is not on the Mac, so which of these fails is not seen; the docs say plugin MCP servers run on the device in local Cowork, which this build does not do.
- Docs: "Desktop extensions run locally and are only available in Claude Desktop and Claude Code." An installed extension of the same shape (a `binary` extension, manifest 0.3) is attached to both chat and Cowork in the logs; ours was seen to reach Cowork.
- An earlier statement here, that chat can use the plugin's own MCP server (observed 2026-10-04), was wrong: what was observed was the connector from `claude_desktop_config.json`, whose tool calls carry no `plugin:` prefix.

What each gives:
| Surface | Gets | From |
|---|---|---|
| Claude Code | hooks (save and load automatically), the MCP server, `/memory-dashboard`, `/memory-restart` | the plugin |
| Claude Desktop chat and Cowork | the memory tools (including `dashboard` and `restart`, which replace the commands there: the sandbox cannot run them) | the extension |
They share one local worker and one database. Chat also needs the instruction pasted into the personal preferences (an extension has no field for persistent instructions; a skill in the plugin was tried and did not make chat call the tools, and was removed with the plugin's Desktop role). The `make install-desktop` route (a connector in `claude_desktop_config.json`) stays for developers building from source.

Other facts: a plugin uploaded to a Claude account syncs into Claude Code as `<name>@synced` and is not loaded while a local plugin of the same name exists. Never install the plugin twice (the org inventory and the marketplace): the hooks would be registered twice.

The extension's manifest needs semver, so release `0.21.95.3` is `0.21.95-fork.3`; an organization uploads a new version by raising it ("Upload new version", documented).

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
- `v0.21.95.2` (the current entry) the same way: checksums, `cosign verify-blob`, certificate identity. `v0.21.95.1` shipped a 514-character plugin description, which Claude's org upload form rejects (limit 500; `claude plugin validate` does not check it); `.2` has 371, and the claude-mnemonic build now checks the limit. The release build is reproducible: the plugin zip built in CI has the same sha256 as one built locally.

- `v0.21.95.3` and the in-repo catalogue (2026-10-04): the release checks as before, and the catalogue edited by `scripts/update-catalogue.sh` installs `claude-mnemonic@hlgr360 0.21.95.3` in an isolated Claude config. Claude Desktop's marketplace sync accepts it and the plugin installs (confirmed by the maintainer the same day; see above).

Not verified: the released extension in Chat (the spike reached Cowork), the org upload of the extension, how Desktop's own plugin pages show this plugin, Windows, what happens to installed plugins if a marketplace's source moves (docs are silent; likely remove and re-add), Copilot CLI's plugin manifest, pi's skill and MCP support, and whether Cowork runs the hooks in practice.

## Sequence

1. Make the updater and the scripts repository-agnostic (`claude-mnemonic`). Done.
2. Add the release workflow (`claude-mnemonic`). Done.
3. Add the build script and the thin plugin (`claude-mnemonic`). Done; the marketplace rename was not needed (the names differ).
4. Add the catalogue entry here once a release exists. Done with `v0.21.95.1`.
5. Build the `.mcpb` (`claude-mnemonic`, tickets #24 and #118). Done in `v0.21.95.4`; it is the route for Claude Desktop.
6. The instruction skill and its test in Chat. Done; it did not help and was removed; the paste step stays.
7. Later: a design note for Copilot CLI and pi (issue #2).

## Risks

Binary size (about 31 MB per platform archive), CGO builds needing native runners, a plugin that diverges from upstream's update channel, a catalogue entry that must be updated by hand for each release, and the unverified points above.
