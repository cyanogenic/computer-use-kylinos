# wlcom GUI Automation Details

## Context

KylinOS V11 Desktop uses UKUI on Wayland with `wayland-compositor` / `wlcom`. Prefer `wlcctrl` over X11 assumptions for GUI automation.

## Output and scale

```bash
wlcctrl --outputs
wlcctrl --getdisplaygeometry <output_uuid>
gsettings get org.ukui.SettingsDaemon.plugins.xsettings scaling-factor 2>/dev/null || true
```

Example output:

```text
physical size: 300x190 mm
scale: 1.750000
mode: 2880 x 1800 @ 90
position: 0, 0
```

`xdpyinfo` may show Xwayland logical DPI such as `96x96`; do not use it alone for Wayland coordinate conversion.

**Multi-monitor:** each output has a `position` (e.g. a screen right of primary shows `1920,0`). On tested setups `--getwindowgeometry` returns global coordinates that already include this offset, so the click/drag scripts work directly. If a machine reports window geometry relative to its output, add that output's `position` to `WX,WY` before computing `logical`, or the cursor lands on the wrong display. Actions that accept `-o <o_uuid>` to target a specific output: `--fullscreencapture`, `--setarea`, `--getpixelcolor`, `--windowmaximize`, `--windowfullscreen`. `--mousemove` has no `-o` and uses global coordinates.

## Windows

```bash
wlcctrl --list
wlcctrl --search '<regex>'
wlcctrl --getactivewindow
wlcctrl --getwindowname <w_uuid>
wlcctrl --getwindowgeometry <w_uuid>
wlcctrl --windowactivate <w_uuid>
wlcctrl --windowmove <w_uuid> -x 0 -y 0   # only when a known position is required
wlcctrl --windowsize <w_uuid> -w 800 -h 600
wlcctrl --movewindowtooutput <w_uuid>:<o_uuid>
```

## Screenshots

```bash
wlcctrl --windowcapture <w_uuid> --path /tmp/window.png
wlcctrl --fullscreencapture --path /tmp/full.png
wlcctrl --setarea <x,y,width,height> --path /tmp/area.png
wlcctrl --workspacecapture <workspace_uuid:output_uuid> --path /tmp/workspace.png
```

## Keyboard and pointer

```bash
wlcctrl --key ctrl+alt+t
wlcctrl --keystring 'text to type'
wlcctrl --mousebutton 1
wlcctrl --mousepress 1 --mouserelease 1
wlcctrl --scroll -y 10
wlcctrl --getmouselocation
```

For drag actions, move to start, press, move to end, release. `--mousemove` and `--windowmove` are absolute despite the man page describing displacement (see the ⚠️ warning in SKILL.md).

### Keyboard key names, aliases, and combos

`--key` / `--keydown` / `--keyup` accept single keys or `+`-separated combos (e.g. `--key ctrl+alt+t`). The following friendly names are aliases `wlcctrl` resolves automatically — you usually do **not** need to write the raw `control_l` form:

| Type this | Resolves to |
|-----------|-------------|
| `ctrl`    | `control_l` |
| `shift`   | `shift_l`   |
| `alt`     | `alt_l`     |
| `winleft` | `super_l`   |
| `esc`     | `escape`    |
| `enter`   | `return`    |
| `PrintScreen` | `sys_req` |

Other valid keys include `escape, return, shift_l/r, control_l/r, alt_l/r, meta_l/r, page_up, page_down, menu, caps_lock, pause, break, print, sys_req, scroll_lock, num_lock`, etc.

- **Lock-key warning:** `CapsLock` (code 58), `NumLock` (69), and `ScrollLock` (70) may **not** toggle reliably via `wlcctrl`. For reliable lock-key state changes, use `ydotool` instead.
- **Hold a key:** `--keydown <key>` ... `--keyup <key>`, optionally with `--sleep <ms>` between (e.g. `--keydown a --sleep 500 --keyup a`).
- **Type text:** `--keystring 'text'` (no commands can chain after `--keystring`).

For wheel/spinner controls such as time pickers, prefer drag gestures over `--scroll` when exact values matter. In testing, `--scroll` on the UKUI Clock minute wheel jumped several values, while repeated one-row drags changed minutes predictably.

## Coordinate recipe

Given:

- output scale = `S`
- window geometry = `(WX, WY) WIDTH x HEIGHT`
- target point in captured window image = `(RX, RY)`

Then:

```text
logical_x = WX + RX
logical_y = WY + RY
mousemove_x = round(logical_x * S)
mousemove_y = round(logical_y * S)
```

Run:

```bash
wlcctrl --mousemove "${mousemove_x},${mousemove_y}"
wlcctrl --getmouselocation
wlcctrl --mousebutton 1
```

Known tested example:

- output `eDP-1`, scale `1.75`
- calculator window `geometry: (607, 176) 432 x 628`
- `=` center in window screenshot `(376,593)`
- logical target `(983,769)`
- `--mousemove` input about `(1720,1346)`
- `--getmouselocation` returns about `(983,769)`

## Reliable test pattern

1. Launch or focus the app and identify the target window.
2. Capture the window in its current position; do not move it unless the test requires fixed placement, visibility recovery, or multi-monitor placement.
3. Identify target controls from screenshot coordinates.
4. Prefer keyboard shortcuts/input when possible.
5. For pointer tests, use scaled absolute `--mousemove`, verify `--getmouselocation`, then click.
6. For spinners/wheels, use controlled drags and verify after a few steps rather than relying on large scroll deltas.
7. Capture again and verify visually or with OCR/image analysis.
8. Log commands and outputs under `artifacts/` when reproducibility matters.

## Performance notes

- A single `wlcctrl` invocation can cost ~0.3s due to startup/graphics initialization.
- Avoid repeated `--outputs`, `--getdisplaygeometry`, and `--getwindowgeometry` inside loops when values are stable.
- Avoid screenshot + vision analysis unless the UI changed or verification is required.
- For known apps, cache window geometry and known control coordinates.
- Use one shell block for short deterministic sequences to reduce round trips, but keep a screenshot/vision checkpoint before fragile branches.
- For Kaiming apps, executable symlinks under `/opt/kaiming/bin/` may be wrappers; use `kaiming list` to find the app ID and launch with `kaiming run <app-id>`.
