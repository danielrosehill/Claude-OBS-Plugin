---
name: setup
description: First-run setup for the obs-mgmt plugin. Verifies OBS Studio is installed, enables and configures obs-websocket (bundled in OBS 28+), stores connection details in $CLAUDE_USER_DATA, and registers royshil/obs-mcp as an MCP server in Claude Code so later skills can drive OBS programmatically. Run this once before any other skill in this plugin.
---

# OBS Plugin Setup

This skill is the prerequisite for every other skill in `obs-mgmt`. After it succeeds, later skills assume:

- A reachable OBS instance with obs-websocket enabled.
- An `obs-mcp` MCP server registered in Claude Code.
- Connection details persisted under `$CLAUDE_USER_DATA/obs-mgmt/config.json`.

## Data location

Resolve the plugin data root:

```bash
DATA_ROOT="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/obs-mgmt"
mkdir -p "$DATA_ROOT"
```

`config.json` will hold:

```json
{
  "install_type": "native|flatpak|snap",
  "websocket": { "host": "localhost", "port": 4455, "password": "<generated>" },
  "mcp_registered": true
}
```

Never store user-owned data (recordings, scene exports) under `$CLAUDE_USER_DATA` — that's for plugin config only.

## Step 1: Detect / verify OBS install

Delegate to the `detect-install` skill in this plugin. It returns one of `native`, `flatpak`, `snap`, or `missing`. If `missing`, stop and tell the user how to install OBS for their distro; do not proceed.

## Step 2: Enable obs-websocket

OBS 28+ bundles obs-websocket v5. The user must enable it once via the GUI (it cannot be enabled headlessly without editing config files OBS may overwrite):

1. Open OBS → **Tools → WebSocket Server Settings**.
2. Tick **Enable WebSocket server**.
3. Leave port at `4455` unless it conflicts.
4. Tick **Enable Authentication**, click **Show Connect Info**, and copy the password.
5. Click **Apply** then **OK**.

Prompt the user for the password and store it in `config.json` (chmod 600 the file). Do not echo the password back.

For older OBS (<28), instruct the user to install the standalone `obs-websocket` plugin from https://github.com/obsproject/obs-websocket/releases and restart OBS before continuing.

## Step 3: obs-mcp is bundled with this plugin

This plugin ships [royshil/obs-mcp](https://github.com/royshil/obs-mcp) as a bundled MCP server via `.mcp.json` at the plugin root — no manual `claude mcp add` needed. When the plugin is enabled, Claude Code spawns the server automatically via `npx -y obs-mcp@latest`.

The bundled server reads two environment variables:

- `OBS_WEBSOCKET_URL` (default `ws://localhost:4455`)
- `OBS_WEBSOCKET_PASSWORD` (required)

Set them in the user's shell profile so they're inherited by Claude Code:

```bash
# ~/.bashrc or ~/.zshrc
export OBS_WEBSOCKET_PASSWORD="<password from step 2>"
# Optional, only if non-default:
# export OBS_WEBSOCKET_URL="ws://<host>:<port>"
```

Reload the shell, then restart Claude Code so the MCP server picks up the new env. Verify with:

```bash
claude mcp list
```

The `obs` server should appear and report `connected`.

Prerequisite: Node.js 18+ on PATH so `npx` can fetch `obs-mcp`. If the user prefers a pinned local install, they can `npm i -g obs-mcp` and Claude Code's npx call still resolves it.

## Step 4: Validate

In the same Claude Code session (or after a restart so the MCP server loads), call an `obs-mcp` tool that lists scenes — e.g. `GetSceneList` or equivalent. A successful response confirms:

- obs-websocket is reachable.
- The password is correct.
- obs-mcp is registered and running.

If the call fails:

- **Connection refused** → OBS isn't running, or websocket isn't enabled. Re-run step 2.
- **Auth failed** → password mismatch. Re-prompt and rewrite `config.json`.
- **Tool not found** → obs-mcp didn't register. Check `claude mcp list` and re-run step 3.

## Step 5: Mark setup complete

Write `config.json` with `"setup_complete": true` and a timestamp. Other skills in this plugin should read this file and refuse to run if setup is incomplete, pointing the user back here.
