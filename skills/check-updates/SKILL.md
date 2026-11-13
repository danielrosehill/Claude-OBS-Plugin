---
name: check-updates
description: Check whether OBS Studio (and its third-party plugins) on this machine are up to date, and upgrade through the right channel for the install type (official PPA, Flatpak, distro archive). Use on "is OBS up to date", "update OBS", "upgrade OBS".
---

# Check and apply OBS updates

1. Run `python3 ${CLAUDE_PLUGIN_ROOT}/scripts/health.py` and read `checks.install`: `installed`, `apt_candidate`, `apt_source`, `upstream_latest` (GitHub `obsproject/obs-studio` latest release).
2. Decide the route:
   | Situation | Route |
   |---|---|
   | native, source `ppa:obsproject` | `sudo apt update && sudo apt install --only-upgrade obs-studio` |
   | native, `distro archive`, behind upstream | Distro versions lag. Offer the official PPA: `sudo add-apt-repository ppa:obsproject/obs-studio && sudo apt update && sudo apt install obs-studio`. Check first that the PPA publishes for this release (`lsb_release -cs`) |
   | native, `orphaned` (no repo ships it, e.g. after a release upgrade) | It will never update. Same PPA route; if the PPA lacks this release, offer Flatpak (`flathub com.obsproject.Studio`). Flatpak config lives in `~/.var/app/com.obsproject.Studio/config/obs-studio`, so migrate the config first |
   | flatpak | `flatpak update com.obsproject.Studio` |
3. **Ask before changing anything:** adding a PPA, sudo installs and channel switches all need the user's yes.
4. Before upgrading, run `backup-config` and close OBS (`pgrep -x obs`). Major versions can migrate the config format one way.
5. After upgrading: start OBS, rerun `health.py` and `obs_ws.py status`. Confirm third-party plugins in `<config>/plugins/` still load by grepping the new log for their names. Plugins built against the old major version often fail and need `install-plugin` with a matching release.
6. Record the old version, the new version and the route in the machine README and commit.
