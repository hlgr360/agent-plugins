# Claude Mnemonic

Persistent memory for Claude Code and Claude Desktop. It keeps the decisions, findings and fixes from your earlier
sessions, searchable and shared by both, and has a web dashboard (`/memory-dashboard`). This is a fork of
[lukaszraczylo/claude-mnemonic](https://github.com/lukaszraczylo/claude-mnemonic) (MIT); its releases are at
https://github.com/hlgr360/claude-mnemonic.

## What this plugin does

- **In Claude Code**, hooks save what happens in a session and load the project's memory at the start, and the MCP
  server gives Claude the search and related tools.
- **In Claude Desktop** the plugin adds its skills and commands only: Desktop does not give a plugin's tools to chat,
  and Cowork cannot reach the worker from where it starts them. The memory **tools** for chat and Cowork come from the
  Desktop extension, `claude-mnemonic-desktop_<version>.mcpb` on the same release (Settings > Extensions > Install
  Extension). Both share one local worker.
- **Skills:** `/memory-dashboard` opens the dashboard, `/memory-restart` restarts the local worker,
  and `project-memory` carries the instruction below.
- **The memory is stored on your computer** (`~/.claude-mnemonic`), by a local worker that the hooks and the MCP server
  start when needed.
- **The plugin carries no binaries.** On first use it downloads the binaries of its own version from the release,
  checks them against the release's checksums (and against the cosign signature when cosign is installed), and
  installs them in `~/.claude-mnemonic/bin`. Binaries from `make install` or the install script are never replaced.
  Supported: macOS on Apple silicon and Linux on x86-64. The first session may run without memory until the download
  has finished.
- Do not use it next to another install of claude-mnemonic (`install.sh`, `make install`, upstream's plugin): they
  register the same hooks.

## Claude Desktop chat needs one setting

Chat has a built-in memory of its own and answers "search my memory" from that, without calling this plugin. With only
the `project-memory` skill, chat still did not call the connector (one prompt measured; see DESKTOP.md in the
repository). Paste the text below **once** into Claude Desktop's personal preferences (Settings, in the field for
custom instructions). It lives in your claude.ai account, so a plugin cannot set it for you.

```text
I keep a persistent memory of my project work in the claude-mnemonic connector. It is shared with Claude Code and holds decisions, findings and fixes from earlier sessions.

Use it, in addition to any built-in memory, whenever I ask about my past work, earlier decisions or project history, or when I say "memory" or "remember" about my projects. Tell me which source an answer came from.

How to use it:
1. Call the project_suggest tool of the claude-mnemonic connector with my message and show me the projects it returns by name. Ask me which one I mean, or whether to continue without one. Never pick a project for me.
2. If I choose one, load its context before answering. If I am continuing earlier work, also call catch_up for it. While we work, keep one short checkpoint per thread of work with the checkpoint tool (goal, progress, decisions, next steps) after meaningful progress. Save anything else with remember only when I ask you to.
3. If I decline, search and catch up read-only, and do not save anything and do not checkpoint.
4. If two projects share a name, ask me which one. If the connector says they are probably the same project, tell me and offer to merge them; do not make me choose between copies of one project.
5. If this conversation has been compacted or summarised and you lose track of the project or of what we were doing, call catch_up for the project (ask me which one, as in 1, if you do not know) before carrying on, instead of asking me to repeat it.
6. When I ask how something came about, what led to a decision or whether a problem was ever fixed, find the note with search, then call related with its id and follow the connections it lists.
7. When I ask to see, open or manage my memory in a browser, call dashboard and give me the link.

Do not use it for general questions that do not refer to my own earlier work.
```

More about the tools, projects and how this was measured: DESKTOP.md in the repository.
