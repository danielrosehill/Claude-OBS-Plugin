---
name: init-workspace
description: Initialize the OBS management workspace — one user-owned folder (ideally a private git repo) shared by all the user's machines, with a machines/<hostname>/ folder per computer for its scenes, hotkeys, device notes, snapshots and health logs. Run once per machine during onboarding (it adds this machine's folder to an existing workspace); safe to re-run.
---

# Initialize OBS Management Workspace

The **management workspace** is a single folder where everything Claude knows or generates about the user's OBS setup lives — config tarballs, exported scene collections, plugin install logs, hardware notes, and any Markdown docs Claude writes about the user's recording rig.

This skill bootstraps that folder and (optionally) turns it into a private GitHub repo so it's backed up and versioned.

## When to run

- During first-time setup, called by the `setup` skill.
- Standalone, if the user wants to relocate or git-ify an existing workspace.

Idempotent: running it twice on an existing workspace just reports the current state.

## Step 1: Resolve the workspace path

Order of resolution:

1. `$OBS_WORKSPACE` env var if set.
2. Existing `workspace_path` in `$CLAUDE_USER_DATA/obs-mgmt/config.json`.
3. Prompt the user. Default suggestion: `~/Documents/OBS-Workspace`.

Confirm the path with the user before creating anything.

## Step 2: Scaffold the directory layout

One workspace serves every machine; each machine gets its own folder named by `hostname`. Don't create one repo per machine. On a second machine, clone the existing workspace repo and add a folder.

```
$OBS_WORKSPACE/
├── README.md                     # machine table: host, folder, OBS version, notes
├── .gitignore
└── machines/<hostname>/
    ├── README.md                 # scenes, hotkeys, setup log, pending items
    ├── machine.json              # hardware, OS, install type, stable device IDs (no secrets)
    ├── scenes/definitions/*.json # apply-scenes specs
    ├── hotkeys.json              # set-hotkeys spec
    ├── snapshots/<date>-<why>/   # raw copies of <config>/basic before risky changes
    ├── backups/                  # backup-config tarballs (gitignored)
    ├── plugins/                  # third-party plugin artifacts and install logs
    └── docs/                     # notes, fixes, health.md
```

Fill `machine.json` from `detect-install` and `health.py` (cameras by `/dev/v4l/by-id/`, mics by pulse source name). Seed `.gitignore` with `*.tar.gz`, `*.zip`, `*.AppImage`, `*.deb`, `*.bak-*`, `.DS_Store`. Never commit the websocket password; the plugin's scripts read it from OBS's own config on each machine.

If the user already has an older single-machine OBS repo, convert it in place: move its content into `machines/<that-host>/` and rename the repo, rather than starting a parallel one.

## Step 3: Offer to make it a private git repo

Ask the user: *"Make this workspace a private GitHub repo so it's versioned and backed up?"*

If yes:

1. Confirm `gh` is authenticated (`gh auth status`).
2. Decide repo name. Default: `OBS-Machine-Configs`. Confirm. If it already exists on GitHub, clone it to `$OBS_WORKSPACE` instead and just add this machine's folder.
3. Initialise and push:

   ```bash
   cd "$OBS_WORKSPACE"
   git init -q
   git add -A
   git commit -q -m "Initial OBS management workspace"
   gh repo create <owner>/<repo-name> --private --source=. --push
   ```

4. Record the repo URL in `$CLAUDE_USER_DATA/obs-mgmt/config.json`:

   ```json
   {
     "workspace_path": "/home/<user>/Documents/OBS-Workspace",
     "workspace_repo": "git@github.com:<owner>/OBS-Machine-Configs.git",
     "machine_dir": "machines/<hostname>",
     "workspace_is_git": true
   }
   ```

If no, just record `"workspace_is_git": false` and note that the user can re-run this skill later to git-ify.

## Step 4: Persist the env var

Suggest the user add to `~/.bashrc` / `~/.zshrc`:

```bash
export OBS_WORKSPACE="$HOME/Documents/OBS-Workspace"
```

Don't write to shell rc files automatically — show the line and let the user paste it.

## Auto-commit hook (optional, suggest only)

Other skills in this plugin (`backup-config`, `install-plugin`) can opt into auto-committing their outputs to the workspace if `workspace_is_git` is true. Mention this to the user but don't enable it without asking — some users prefer to review and commit manually.

## Re-running this skill

If `$OBS_WORKSPACE` already exists and is git-tracked, just print:

```
Workspace OK
  Path:  <path>
  Repo:  <url>
  Last commit: <hash> <date> <subject>
```

If it exists but isn't git-tracked, offer to git-ify it now.
