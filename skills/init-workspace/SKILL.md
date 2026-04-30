---
name: init-workspace
description: Initialize the OBS management workspace — a user-owned folder (default ~/Documents/OBS-Workspace) that stores config backups, scene exports, and Claude-generated notes about the user's OBS setup. Optionally turns it into a private GitHub repo so the workspace is version-controlled and survives machine wipes. Run once during onboarding; safe to re-run (idempotent).
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

Create this structure if missing:

```
$OBS_WORKSPACE/
├── README.md
├── backups/           # config tarballs from backup-config skill
├── scenes/            # exported scene collection JSON
├── plugins/           # downloaded OBS plugin artifacts + install logs
├── docs/              # Claude-generated notes (hardware, encoder choices, recording profiles)
└── .gitignore
```

Seed `README.md` with a one-paragraph description of the workspace and a pointer to this plugin. Seed `.gitignore` with:

```
# Large/regenerable artifacts
*.tar.gz
*.zip
*.AppImage
*.deb

# OS noise
.DS_Store
Thumbs.db
```

By default, ignore the actual backup tarballs — the user can override this if they want backups committed (rarely a good idea given size).

## Step 3: Offer to make it a private git repo

Ask the user: *"Make this workspace a private GitHub repo so it's versioned and backed up?"*

If yes:

1. Confirm `gh` is authenticated (`gh auth status`).
2. Decide repo name. Default: `OBS-Workspace`. Confirm.
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
     "workspace_repo": "git@github.com:<owner>/OBS-Workspace.git",
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
