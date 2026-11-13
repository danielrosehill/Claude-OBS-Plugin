---
name: set-hotkeys
description: Set OBS hotkeys on this machine from the workspace's hotkeys.json — scene switching, start/stop/pause recording, screenshots — as OBS in-app hotkeys and/or desktop-global shortcuts (KDE) that drive OBS over obs-websocket. Use on "hotkey to switch scenes", "shortcut to start recording", "change OBS hotkeys".
---

# Set hotkeys

Spec: `$OBS_WORKSPACE/machines/$(hostname)/hotkeys.json`, applied by `python3 ${CLAUDE_PLUGIN_ROOT}/scripts/hotkeys.py <spec> [--dry-run]`.

```json
{
  "frontend": {"OBSBasic.StartRecording": [{"key": "OBS_KEY_F9", "control": true}]},
  "scenes":   {"Webcam": [{"key": "OBS_KEY_1", "control": true, "alt": true}]},
  "global":   [{"id": "obs-scene-webcam", "name": "OBS: scene Webcam", "shortcut": "Meta+Shift+1", "args": ["scene", "Webcam"]}]
}
```

## Pick the layer

- **Wayland (check `$XDG_SESSION_TYPE`): use `global`.** OBS's own hotkeys only fire while the OBS window has focus there, which defeats the point of a hotkey while recording.
- `global` (KDE Plasma): writes `~/.local/share/applications/<id>.desktop` with `X-KDE-Shortcuts=` and an `Exec=` running a stable copy of `obs_ws.py` (`~/.local/share/obs-mgmt/`), then rebuilds sycoca. `args` are any `obs_ws.py` command: `["scene", "<name>"]`, `["trigger", "ToggleRecord"]`, `["trigger", "ToggleRecordPause"]`. Needs OBS running with websocket enabled. Other desktops: write the equivalent in their shortcut system with the same `Exec` line.
- `frontend` -> `[Hotkeys]` in the active profile's `basic.ini`; `scenes` -> `OBSBasic.SelectScene` per scene in the scene collection JSON. Both need **OBS closed** (it rewrites them on exit); the script refuses otherwise and backs up each file to `.bak-<timestamp>`. Common frontend names: `OBSBasic.StartRecording`, `StopRecording`, `PauseRecording`, `UnpauseRecording`, `StartStreaming`, `StopStreaming`, `Screenshot`. Keys use `OBS_KEY_*` names.

## Steps

1. Ask which actions and keys if not given. Check conflicts: `grep -n '<combo>' ~/.config/kglobalshortcutsrc`.
2. `--dry-run`, then apply.
3. **Verify:** for global shortcuts, `qdbus6 --literal org.kde.kglobalaccel /component/<id with - and . as _>_desktop org.kde.kglobalaccel.Component.allShortcutInfos` must list the name and key; then run each `.desktop` `Exec=` line and confirm with `obs_ws.py status`. For in-app ones, restart OBS and check Settings > Hotkeys.
4. Record the table in the machine README and commit + push.
