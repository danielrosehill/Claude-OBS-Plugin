---
name: apply-scenes
description: Create or update OBS scenes on this machine from declarative JSON specs in the workspace (machines/<hostname>/scenes/definitions/). Idempotent; reuses inputs across scenes by name and lays out full-frame and picture-in-picture items. Use on "set up scenes", "create a webcam / screen / screen + webcam scene", "add a scene", or after moving to a new machine.
---

# Apply scenes

Scenes are data in the workspace, applied over obs-websocket by `scripts/obs_ws.py`. Never hand-build scenes in the GUI and then forget them: write the spec, apply it, commit it.

## Prerequisites

- `setup` done: OBS running, obs-websocket enabled. The script reads the password from OBS's own `plugin_config/obs-websocket/config.json` on this machine (or `OBS_WEBSOCKET_PASSWORD`), so nothing needs storing.
- Workspace resolved (`init-workspace`): `$OBS_WORKSPACE/machines/$(hostname)/`.

## Spec format

One file per scene, `scenes/definitions/NN-<slug>.json`; items listed **top to bottom**:

```json
{
  "scene": "Screen + Webcam",
  "purpose": "Screen capture with webcam PiP bottom right.",
  "items": [
    {"input": "Webcam", "kind": "v4l2_input", "fit": "pip-br", "pip_width": 0.25, "pip_margin": 32},
    {"input": "Screen", "kind": "pipewire-screen-capture-source", "settings": {"ShowCursor": true}, "fit": "full"}
  ]
}
```

- `fit`: `full` (scale to canvas, keep aspect) or `pip-br|pip-bl|pip-tr|pip-tl` (`pip_width` = fraction of canvas width, 16:9). Omit to leave the transform alone.
- `settings` only matter where the input is first created; later scenes reuse the input by name.
- Common Linux kinds: `v4l2_input` (webcam: set `device_id` to the stable `/dev/v4l/by-id/...-video-index0` path, never `/dev/videoN`), `pipewire-screen-capture-source` (Wayland), `xshm_input` (X11), `pulse_input_capture` (mic), `image_source`, `text_ft2_source_v2`.
- v4l2 packed values: `pixelformat` MJPG = `1196444237`; `resolution` = `(width << 32) | height` (1280x720 = `5497558139600`).

## Steps

1. Gather devices: `python3 scripts/health.py` (cameras by-id, mics, session type). Ask which scenes are wanted if not stated.
2. Write the specs. Check `obs_ws.py status` for existing scenes and the canvas size.
3. Apply: `uv run --with obsws-python python3 ${CLAUDE_PLUGIN_ROOT}/scripts/obs_ws.py apply-scenes <defs dir>`. Run it twice; the second run must print `nothing to do`.
4. **Verify the thing:** switch to each scene (`obs_ws.py scene "<name>"`) and read `GetSceneItemList` transforms. `sourceWidth/Height` of 0 means the source delivers no frames. Then check the newest file in `<config>/logs/`.
5. Record the result in `machines/<host>/README.md` (scene table, anything that failed and why) and commit + push the workspace.

## Traps

- **One camera, one input.** A second `v4l2_input` on the same device fails with `Device or resource busy`. Remove or reuse the old input. After removing it, OBS can keep the device open until restarted.
- **Mic doubling.** OBS's global Mic/Aux (Settings > Audio) already records the default mic. Don't add a per-scene mic input on top of it unless you disable the global one.
- **Wayland screen capture** opens the desktop portal "share screen" dialog when created. A human must pick the screen once; OBS then stores a `RestoreToken`. Until then the source is 0x0, which is expected.
- Scenes are added to the top of OBS's scene list; there is no websocket request to reorder them.
