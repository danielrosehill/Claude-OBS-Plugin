---
name: health-check
description: Check the health of OBS on this machine — install and update status, websocket, recording path and free space, cameras and mics, hardware encoder availability, dropped frames and errors in the latest OBS log, CPU/RAM/temperature. Read-only. Use on "is OBS OK", "OBS health", "why did my recording drop frames", or before an important recording.
---

# OBS health check

1. `python3 ${CLAUDE_PLUGIN_ROOT}/scripts/health.py [--record-path PATH] [--min-free-gb N]` prints JSON with `overall` and per-check `status` (`ok|info|warn|fail`).
2. If OBS is running with websocket on, add `uv run --with obsws-python python3 ${CLAUDE_PLUGIN_ROOT}/scripts/obs_ws.py status` for live state (canvas, fps, current scene, recording, record directory). The live record directory beats the one guessed from config.
3. Report only warn/fail items, each with the fix. Known meanings:
   - `install.apt_source` = `not in apt lists`: the local apt lists don't list the installed package. This is usually stale lists, not a dead source: run `sudo apt update` and recheck before concluding anything (`apt_lists_age_days` shows how stale they are).
   - `logs.worst_drop_pct` > 1: encoding lag means a slower x264 preset or a hardware encoder; rendering lag means a GPU bottleneck or too many sources.
   - `encoders.vaapi_h264_encode` unknown: suggest `sudo apt install vainfo` to confirm.
   - `devices.cameras` empty: camera unplugged, privacy shutter or kill switch, or held by another app (`fuser /dev/video*`).
   - DeckLink, translation, and DroidCam usbmuxd lines are filtered as known startup noise.
4. Append a dated summary to `$OBS_WORKSPACE/machines/$(hostname)/docs/health.md` and commit, so drift over time is visible.
