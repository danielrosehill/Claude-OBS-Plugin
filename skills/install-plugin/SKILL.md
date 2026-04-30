---
name: install-plugin
description: Install a third-party OBS Studio plugin from a release URL, GitHub repo, or local archive. Resolves the correct plugin install location based on install type (native vs Flatpak vs Snap), unpacks the artifact, sets file permissions, and prompts the user to restart OBS to load the new plugin. Supports .deb, .tar.gz/.tar.xz, .zip, and prebuilt directories.
---

# Install OBS Plugin

Installs a third-party OBS plugin into the active OBS install's plugin directory.

## Prerequisites

- `setup` and `detect-install` have run.
- A backup via `backup-config` is **strongly recommended** before installing plugins, especially on Flatpak/Snap. Offer to run it first.

## Install type determines target path

| Install type | Per-user plugin dir |
|---|---|
| native (apt/dnf/pacman) | `~/.config/obs-studio/plugins/<plugin-name>/bin/64bit/` (binary) and `~/.config/obs-studio/plugins/<plugin-name>/data/` (assets) |
| Flatpak | `~/.var/app/com.obsproject.Studio/config/obs-studio/plugins/<plugin-name>/...` |
| Snap | Strict confinement blocks most third-party plugins. Warn the user; they may need to switch to native or Flatpak. |
| AppImage | Per-user dir as native, but plugin must match the AppImage's bundled OBS ABI. |

System-wide install (`/usr/lib/.../obs-plugins/`) requires sudo and is not the default — only do it on explicit request.

## Inputs

Accept any of:

- A direct URL to a release artifact (`.deb`, `.tar.gz`, `.zip`).
- A GitHub repo URL (`https://github.com/<owner>/<repo>`) — fetch the latest release via `gh release view --repo <owner>/<repo>` and pick the Linux asset matching the install type.
- A local file path.

## Procedure

1. Confirm OBS is not running.
2. Download / locate the artifact. Verify checksum if the release page publishes one.
3. Inspect the artifact:

   ```bash
   file <artifact>
   tar -tzf <artifact> | head -30  # for tarballs
   unzip -l <artifact> | head -30  # for zips
   dpkg -c <artifact> | head -30   # for debs
   ```

   Flag anything suspicious (writes outside the plugin dir, setuid bits, unexpected absolute paths) and stop for user confirmation.

4. Install per artifact type:

   - **`.deb` (native only, Debian/Ubuntu)**: `sudo apt install ./<file>.deb` — apt resolves any system deps.
   - **Tarball**: extract into the per-user plugin dir. Most OBS plugins ship with a top-level layout that drops directly into `~/.config/obs-studio/plugins/`.
   - **Zip**: unzip into the same location.
   - **Pre-built directory**: copy with `cp -r`.

5. Verify the binary lands at `<plugin-dir>/<plugin-name>/bin/64bit/<plugin>.so`.
6. Set permissions: `chmod 644` on `.so` and data files, `755` on directories.
7. Tell the user to restart OBS, and to check **Help → Log Files → View Current Log** for plugin-load errors after restart.

## Common plugins to suggest

Useful plugins worth mentioning when the user asks "what should I install":

- **StreamFX** — extra filters/transitions/encoders.
- **obs-backgroundremoval** — AI portrait segmentation, no green screen needed.
- **obs-vkcapture** — Vulkan/OpenGL game capture on Linux (essential for Linux gaming streams).
- **obs-pipewire-audio-capture** — per-application audio capture on Pipewire systems.
- **advanced-scene-switcher** — automation/conditional scene logic.

Always link to the plugin's official GitHub release page rather than third-party mirrors.

## Uninstall

To remove a plugin: stop OBS, delete the `<plugin-dir>/<plugin-name>/` folder, restart OBS. For `.deb` installs, use `sudo apt remove <pkg>` instead.
