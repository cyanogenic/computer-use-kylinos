---
name: computer-use-kylinos
description: Use wlcctrl on KylinOS V11 Desktop / UKUI / wlcom Wayland desktops for fast GUI automation tests: find windows/outputs, launch or focus apps, screenshot, click/type/drag, handle scale-aware coordinates, and verify GUI results.
---

# Computer Use KylinOS

Use for GUI automation on KylinOS V11 Desktop with wlcom `wlcctrl`.

## Fast path

Resolve script paths relative to this skill directory.

```bash
# Inspect once per task
bash scripts/wlcctrl-info.sh

# Launch/focus app, then find target window
# For Kaiming apps, prefer: kaiming list | grep -i '<app>'; kaiming run <app-id>
wlcctrl --search '<app_id_or_title_regex>'
wlcctrl --getwindowgeometry <w_uuid>

# Capture window for coordinate picking; keep the user's window position by default
wlcctrl --windowcapture <w_uuid> --path /tmp/window.png

# Keyboard is fastest when available
wlcctrl --windowactivate <w_uuid>
wlcctrl --keystring 'text'
wlcctrl --key ctrl+s

# Click a point from window-screenshot coordinates when keyboard is insufficient
bash scripts/wlcctrl-window-click.sh <w_uuid> <relative_x> <relative_y> [button]

# Drag from one window-relative point to another (e.g. slider, scrollbar)
bash scripts/wlcctrl-drag.sh <w_uuid> <start_x> <start_y> <end_x> <end_y> [button]
```

Do **not** move a window to `(0,0)` as a default setup step. Use `bash scripts/wlcctrl-move-window.sh` only when the test specifically requires a known position, a window is partly off-screen, or the user asks for it.

## Rules to remember

- ⚠️ **man page is WRONG about coordinates.** The `wlcctrl` man page (dated 2026-05-21) documents `--mousemove <dx,dy>` and `--windowmove -x/-y` as *relative displacements*. In reality **both are ABSOLUTE coordinates** in the output's logical space. Do NOT add the current cursor/window position — just pass the absolute target.
- `--mousemove` is absolute, not relative (see warning above).
- On tested V11 machines, `--mousemove` input = logical coordinate × output scale; `--getmouselocation` returns logical coordinates.
- `--windowmove -x/-y` is absolute in logical coordinates (pass the target position directly — no scale, no offset). See the warning above. Avoid moving windows unless needed.
- **Multi-monitor:** if a window's geometry is output-relative (not global), add that output's `position` to `WX,WY` before computing the target. Many actions take `-o <o_uuid>` to target a specific output; `--mousemove` has no `-o` (global). Details/rationale → details.md.
- Use `wlcctrl --outputs` + `--getdisplaygeometry <o_uuid>` to check output UUID, position, mode, and scale.
- For wheel/spinner controls, prefer mouse drag with `--mousepress`/`--mousemove`/`--mouserelease`; `--scroll` may jump multiple values.
- Current `wlcctrl v1.0.0` help/man shows screenshots but no recording subcommand.
- Avoid `--windowclose` / `--windowkill` unless explicitly requested.

## Core commands

```bash
wlcctrl --outputs
wlcctrl --getdisplaygeometry <o_uuid>
wlcctrl --list
wlcctrl --search '<regex>'
wlcctrl --getactivewindow
wlcctrl --getwindowname <w_uuid>
wlcctrl --getwindowgeometry <w_uuid>
wlcctrl --windowactivate <w_uuid>
wlcctrl --windowmove <w_uuid> -x <x> -y <y>
wlcctrl --windowsize <w_uuid> -w <w> -h <h>
wlcctrl --windowminimize <w_uuid>
wlcctrl --windowmaximize <w_uuid> [-o <o_uuid>]
wlcctrl --windowfullscreen <w_uuid> [-o <o_uuid>]
wlcctrl --windowmap <w_uuid>            # restore a minimized window
wlcctrl --movewindowtooutput <w_uuid>:<o_uuid>
wlcctrl --windowcapture <w_uuid> --path /tmp/window.png
wlcctrl --fullscreencapture --path /tmp/full.png [-o <o_uuid>]
wlcctrl --setarea <x,y,width,height> --path /tmp/area.png [-o <o_uuid>]
wlcctrl --getpixelcolor <x>,<y> [-o <o_uuid>]   # fast RGBA probe, alt to full screenshot + PIL
wlcctrl --mousemove <x>,<y>
wlcctrl --getmouselocation
wlcctrl --mousebutton 1
wlcctrl --mousepress 1 --mouserelease 1
wlcctrl --scroll -y <delta>
wlcctrl --keystring 'text'
wlcctrl --key ctrl+s
wlcctrl --keydown <key>      # hold a key (pair with --keyup)
wlcctrl --keyup <key>        # release a held key
wlcctrl --exec '<shell>'     # run a shell command
wlcctrl --sleep <ms>         # in-call delay, e.g. --keydown a --sleep 500 --keyup a
```

## Keyboard

`--key` takes `+`-separated combos (e.g. `--key ctrl+alt+t`). Friendly aliases (`ctrl`→`control_l`, etc.), lock-key caveats, and hold-key patterns → see details.md.

## Coordinate formula

**Asymmetry (critical):** `mousemove` input = logical × output scale `S`; `windowmove -x/-y` takes logical coordinates directly (NO × S). Don't mix them up.

### mousemove (clicks, drags) — multiply by scale

Given window geometry `(WX,WY)`, output scale `S`, and target `(RX,RY)` inside a window screenshot:

```text
logical_x = WX + RX
logical_y = WY + RY
mousemove_x = round(logical_x * S)
mousemove_y = round(logical_y * S)
```

This applies to both clicks and drags: every `--mousemove` argument (start point and end point) is converted the same way. The `wlcctrl-window-click.sh` and `wlcctrl-drag.sh` scripts do this conversion for you. Then verify with `wlcctrl --getmouselocation` before clicking.

### windowmove (move a window) — NO scale

Given the desired on-screen position `(TX,TY)` in logical coordinates:

```text
windowmove -x = TX      # logical, do NOT multiply by S
windowmove -y = TY      # logical, do NOT multiply by S
```

`wlcctrl-move-window.sh` passes `TX,TY` straight through. Do not add `S` here.

## Speed tips

- Cache output scale and window geometry for a task; repeated `wlcctrl` calls are relatively expensive.
- Batch simple command sequences in one shell script, but keep screenshot/vision checkpoints only where they change the next decision.
- For Kaiming-packaged apps, launch by app ID with `kaiming run <app-id>` instead of executing wrapper symlinks directly.

## When more detail is needed

Read `references/details.md` for examples, multi-step test pattern, spinner controls, screenshots, and caveats.
