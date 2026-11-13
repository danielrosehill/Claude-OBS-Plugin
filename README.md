# Claude OBS Plugin

Claude Code plugin for managing OBS Studio on Linux. Detects how OBS is installed (native, Flatpak, Snap, AppImage), enables and validates obs-websocket, ships a bundled MCP server ([royshil/obs-mcp](https://github.com/royshil/obs-mcp)) so Claude can drive OBS programmatically, backs up configs, and installs third-party OBS plugins.

## What's bundled

This plugin ships an MCP server (`obs-mcp`) via `.mcp.json` at the plugin root. When the plugin is enabled, Claude Code launches the server automatically — no manual `claude mcp add` step. You just need to export `OBS_WEBSOCKET_PASSWORD` in your shell.

## Skills

| Skill | What it does |
|---|---|
| `setup` | First-run: verify OBS, walk through enabling obs-websocket, bootstrap the management workspace, validate via the bundled `obs-mcp` server. Run this once before anything else. |
| `init-workspace` | Create the **management workspace**: one private repo for all your machines, with a `machines/<hostname>/` folder per computer (scenes, hotkeys, device notes, snapshots, health log). Run on each new machine to add its folder. |
| `detect-install` | Identify how OBS is installed (native package / Flatpak / Snap / AppImage / missing) and resolve the correct config and plugin directories. |
| `backup-config` | Tarball the OBS config directory to the management workspace (or a user override path), with retention. |
| `apply-scenes` | Create scenes from declarative JSON specs in the workspace (full-frame and picture-in-picture layouts, inputs reused by name). Idempotent. |
| `set-hotkeys` | Scene-switch and recording hotkeys from `hotkeys.json`: OBS in-app, or desktop-global (KDE) shortcuts that drive OBS over websocket, which is what works on Wayland. |
| `health-check` | Read-only machine check: install/update status, websocket, disk, cameras and mics, VAAPI, dropped frames and errors in the latest log, CPU/RAM/temperature. |
| `check-updates` | Compare the installed OBS with upstream and upgrade through the right channel (PPA, Flatpak, distro); catches installs no repo ships any more. |
| `install-plugin` | Install third-party OBS plugins from a release URL, GitHub repo, or local archive into the correct per-install plugin directory. |

## The management workspace

This plugin keeps user-owned data in one folder, the **management workspace** (`$OBS_WORKSPACE`), ideally a private git repo shared by all your machines. Each machine has its own folder, named by hostname:

```
machines/<hostname>/
├── machine.json              # hardware, OS, install type, stable device IDs
├── scenes/definitions/*.json # declarative scenes  -> apply-scenes
├── hotkeys.json              # declarative hotkeys -> set-hotkeys
├── snapshots/  docs/  plugins/  backups/
└── README.md                 # what is set up and what is pending
```

Scenes and hotkeys are data, so the same plugin serves every machine while each machine's devices and choices stay in its own folder. The websocket password is never stored: the scripts read it from OBS's own config on the machine. Plugin config (small pointers only) lives separately under `$CLAUDE_USER_DATA/obs-mgmt/`.

## Requirements

- Linux desktop with OBS Studio 28+ (older OBS works but obs-websocket must be installed manually).
- Node.js 18+ on PATH (for the bundled `obs-mcp`).
- `uv` (the scripts in `scripts/` run with `uv run --with obsws-python`). Global hotkeys: KDE Plasma (other desktops: same `Exec` line in their shortcut system).

## Environment variables

Set these in your shell profile (`~/.bashrc` / `~/.zshrc`) before launching Claude Code:

| Var | Required | Purpose |
|---|---|---|
| `OBS_WEBSOCKET_PASSWORD` | for `obs-mcp` | Password from OBS → Tools → WebSocket Server Settings. Needed by the bundled `obs-mcp` server; `scripts/obs_ws.py` falls back to reading it from OBS's local config. |
| `OBS_WEBSOCKET_URL` | no | Defaults to `ws://localhost:4455`. Override for remote OBS or non-default port. |
| `OBS_WORKSPACE` | recommended | Folder where backups, scene exports, and Claude-generated notes about your OBS setup are stored. Suggested: a clone of a private `OBS-Machine-Configs` repo. The `setup` skill will prompt and scaffold this if unset. |
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
