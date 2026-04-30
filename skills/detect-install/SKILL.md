---
name: detect-install
description: Detect how OBS Studio is installed on a Linux system — native package (apt/dnf/pacman), Flatpak (com.obsproject.Studio), Snap, or AppImage — and return the install type, version, binary path, and config directory. Used by other obs-mgmt skills that need to know where OBS reads config and stores plugins.
---

# Detect OBS Install Type

Returns a structured report so other skills can locate the config dir and plugin dir correctly. Each install type uses a different layout.

## Detection order

Check in this order and stop at the first match. A user can technically have more than one — if so, report all and ask which one to operate on.

### 1. Flatpak

```bash
flatpak info com.obsproject.Studio 2>/dev/null
```

If present:
- **Binary**: `flatpak run com.obsproject.Studio`
- **Config dir**: `~/.var/app/com.obsproject.Studio/config/obs-studio/`
- **Plugin dir (per-user)**: `~/.var/app/com.obsproject.Studio/config/obs-studio/plugins/`
- Caveat: Flatpak sandboxing restricts filesystem access. Recording paths outside the user's home may need `flatpak override --filesystem=...`.

### 2. Snap

```bash
snap list obs-studio 2>/dev/null
```

If present:
- **Binary**: `/snap/bin/obs-studio` (or just `obs`)
- **Config dir**: `~/snap/obs-studio/current/.config/obs-studio/`
- Caveat: Snap confinement is even tighter than Flatpak. Plugin install often requires devmode or classic confinement.

### 3. Native package

```bash
command -v obs && obs --version 2>&1 | head -1
```

Detect package manager backing it:
```bash
# Debian/Ubuntu
dpkg -S "$(command -v obs)" 2>/dev/null
# Fedora/RHEL
rpm -qf "$(command -v obs)" 2>/dev/null
# Arch
pacman -Qo "$(command -v obs)" 2>/dev/null
```

If present:
- **Binary**: `/usr/bin/obs`
- **Config dir**: `~/.config/obs-studio/`
- **Plugin dirs**: `~/.config/obs-studio/plugins/` (per-user) and `/usr/lib/x86_64-linux-gnu/obs-plugins/` or `/usr/lib64/obs-plugins/` (system).

### 4. AppImage

Look for OBS AppImages in common locations:
```bash
find ~/Applications ~/AppImages ~/Downloads -maxdepth 2 -iname '*obs*.AppImage' 2>/dev/null
```

Treat as native for config purposes (`~/.config/obs-studio/`) but flag that plugins must match the AppImage's bundled OBS version.

### 5. Missing

If none of the above match, report `missing` and suggest distro-appropriate install:
- Ubuntu/Debian: `sudo apt install obs-studio`
- Fedora: `sudo dnf install obs-studio`
- Arch: `sudo pacman -S obs-studio`
- Universal: Flatpak from Flathub.

## Output format

Return a summary like:

```
OBS install detected:
  Type:        native (apt)
  Version:     30.2.3
  Binary:      /usr/bin/obs
  Config dir:  /home/<user>/.config/obs-studio/
  Plugin dir:  /home/<user>/.config/obs-studio/plugins/
```

Other skills (`backup-config`, `install-plugin`) consume these paths.
