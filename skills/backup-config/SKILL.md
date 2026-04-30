---
name: backup-config
description: Back up the OBS Studio config directory (scenes, profiles, global settings, plugin configs) to a user-defined folder. Resolves the correct config path based on install type (native, Flatpak, Snap), creates a timestamped tarball, and prunes old backups based on a retention policy. Stores the user's chosen backup folder pointer in plugin config so subsequent runs don't re-prompt.
---

# Backup OBS Config

Creates a timestamped tarball of the OBS config directory at a user-chosen location.

## Prerequisites

- `setup` skill has run.
- `detect-install` resolves the active OBS config dir.

## Backup folder resolution

Resolve the backup folder in this order:

1. **`$OBS_WORKSPACE/backups/`** — default. The `setup` skill prompts for `OBS_WORKSPACE` and scaffolds the `backups/` subfolder.
2. **`backup_folder` override in `$CLAUDE_USER_DATA/obs-mgmt/config.json`** — if the user explicitly set a non-default location (e.g. an external drive mount).
3. **First-run prompt** — if neither is set, ask for an absolute path. Suggest:
   - `$OBS_WORKSPACE/backups/` (preferred — keeps everything in one workspace).
   - An external drive mount, e.g. `/run/media/<user>/<drive>/obs-backups/`.

Persist any override in `config.json`:

```json
{
  "backup_folder": "/run/media/<user>/<drive>/obs-backups",
  "retention_count": 10
}
```

The backup folder is **user-owned data**. Store only the pointer under `$CLAUDE_USER_DATA`; the tarballs themselves live in `$OBS_WORKSPACE` (or the user's chosen override path).

## Backup procedure

1. Resolve config dir via `detect-install` (Flatpak, Snap, and native paths all differ — see that skill).
2. Confirm OBS is **not running** (`pgrep -x obs` returns nothing). If it is, warn that hot-backups can capture mid-write state and ask the user whether to continue or close OBS first.
3. Build the archive name:

   ```
   obs-config-<install_type>-YYYY-MM-DD-HHMMSS.tar.gz
   ```

4. Create the tarball:

   ```bash
   TS=$(date +%Y-%m-%d-%H%M%S)
   ARCHIVE="$BACKUP_FOLDER/obs-config-$INSTALL_TYPE-$TS.tar.gz"
   tar -czf "$ARCHIVE" -C "$(dirname "$CONFIG_DIR")" "$(basename "$CONFIG_DIR")"
   ```

5. Verify the archive: `tar -tzf "$ARCHIVE" >/dev/null && echo OK`.
6. Print the archive path and size.

## Retention

After a successful backup, prune older archives in the backup folder beyond `retention_count` (default 10), oldest-first. Match only files following the `obs-config-*-*.tar.gz` pattern so user files in the same folder aren't touched.

```bash
ls -1t "$BACKUP_FOLDER"/obs-config-*.tar.gz 2>/dev/null \
  | tail -n +$((RETENTION + 1)) \
  | xargs -r rm -v
```

## Restore (informational)

This skill only backs up. To restore:

1. Stop OBS.
2. Move the existing config dir aside: `mv ~/.config/obs-studio ~/.config/obs-studio.bak`.
3. Extract the tarball into the parent of the original config dir.
4. Start OBS and verify scenes/profiles load.

If the user asks to restore, do it explicitly under their supervision — never auto-restore.
