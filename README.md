# Claude OBS Plugin

Claude Code plugin for managing OBS Studio on Linux. Detects how OBS is installed (native, Flatpak, Snap, AppImage), enables and validates obs-websocket, ships a bundled MCP server ([royshil/obs-mcp](https://github.com/royshil/obs-mcp)) so Claude can drive OBS programmatically, backs up configs, and installs third-party OBS plugins.

## What's bundled

This plugin ships an MCP server (`obs-mcp`) via `.mcp.json` at the plugin root. When the plugin is enabled, Claude Code launches the server automatically — no manual `claude mcp add` step. You just need to export `OBS_WEBSOCKET_PASSWORD` in your shell.

## Skills

| Skill | What it does |
|---|---|
| `setup` | First-run: verify OBS, walk through enabling obs-websocket, set env vars, validate via the bundled `obs-mcp` server. Run this once before anything else. |
| `detect-install` | Identify how OBS is installed (native package / Flatpak / Snap / AppImage / missing) and resolve the correct config and plugin directories. |
| `backup-config` | Tarball the OBS config directory to a user-chosen folder, with retention. Pointer-only persistence — backups stay in user-owned space. |
| `install-plugin` | Install third-party OBS plugins from a release URL, GitHub repo, or local archive into the correct per-install plugin directory. |

## Requirements

- Linux desktop with OBS Studio 28+ (older OBS works but obs-websocket must be installed manually).
- Node.js 18+ on PATH (for the bundled `obs-mcp`).

## Environment variables

Set these in your shell profile (`~/.bashrc` / `~/.zshrc`) before launching Claude Code:

| Var | Required | Purpose |
|---|---|---|
| `OBS_WEBSOCKET_PASSWORD` | yes | Password from OBS → Tools → WebSocket Server Settings. Consumed by the bundled `obs-mcp` server. |
| `OBS_WEBSOCKET_URL` | no | Defaults to `ws://localhost:4455`. Override for remote OBS or non-default port. |
| `OBS_WORKSPACE` | recommended | Folder where backups, scene exports, and Claude-generated notes about your OBS setup are stored. Suggested: `~/Documents/OBS-Workspace/`. The `setup` skill will prompt and scaffold this if unset. |
| `OBS_CONFIG_PATH` | no | Override the auto-detected OBS config dir. Useful for non-standard installs or running multiple OBS profiles side by side. |

## Installation

Add the marketplace if you haven't already, then install:

```bash
claude plugins marketplace add danielrosehill/Claude-Code-Plugins
claude plugins install obs-mgmt@danielrosehill
```

Or install at project scope from inside a specific repo:

```bash
claude plugins install obs-mgmt@danielrosehill --scope project
```

## License

MIT
