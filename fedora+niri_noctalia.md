# Fedora 44 + Niri + Noctalia — Desktop Environment Setup

Reproduces the niri Wayland compositor + Noctalia shell (bar, launcher,
notifications, control center) desktop used on this machine. Target hardware
noted where it matters: Intel HD 520 (Broadwell, GLES 3.2), 4-core Skylake-U.
Blur and shadows are disabled on purpose to save frames on that iGPU.

Versions confirmed on this system:
- Fedora 44
- niri 26.04
- Noctalia 5.3.0
- foot 1.27.0 (terminal)
- pipewire + wireplumber (audio, via `wpctl`)
- brightnessctl, playerctl

## 1. Enable package sources and install

niri and Noctalia are not in Fedora base. Enable rpmfusion-free (niri) and the
Noctalia source, then install:

```bash
# Enable RPM Fusion free (needed for niri on Fedora)
sudo dnf install -y \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm

# Install the desktop stack
sudo dnf install -y \
  niri \
  noctalia \
  foot \
  brightnessctl \
  playerctl \
  pipewire \
  wireplumber
```

PipeWire/WirePlumber are usually pulled in by the desktop install already;
re-installing is harmless.

## 2. Create the niri config directory

```bash
mkdir -p ~/.config/niri
```

### ~/.config/niri/config.kdl

Full niri config (niri 26.04 KDL format). Validate anytime with `niri validate`.

```kdl
// niri 26.04 config -- Fedora 44, Intel HD 520 (Broadwell, GLES 3.2)
//
// Notes for this machine:
//   * 4-core Skylake-U, 2015 iGPU. Compositor blur is a multi-pass
//     fullscreen effect and WILL cost frames here, so it is disabled.
//     See the BLUR section below if you want to try it anyway.
//   * Noctalia v5.2.1 is the bar/launcher/notifications shell.
//
// Validate this file any time with:  niri validate

// ── Input ────────────────────────────────────────────────────────────────────

input {
    keyboard {
        xkb {
            // Empty = inherit your system layout via locale1.
            // Uncomment and edit if you need a specific one:
            // layout "us"
            // options "grp:alt_shift_toggle"
        }
        numlock
    }

    touchpad {
        tap
        natural-scroll
        // accel-speed 0.3
    }

    mouse {
        accel-speed 0.2
    }
}

// ── Layout ───────────────────────────────────────────────────────────────────

layout {
    // Let a column take a full screen width when it is the only one.
    center-focused-column "never"

    gaps 8

    // Spaces between windows, including around the edges of the screen.
    // This is what makes the focus ring look right on a single window.
    border {
        off
    }

    focus-ring {
        width 2
        active-color "#88c0d0"
        inactive-color "#4c566a"
    }

    shadow {
        // Left off: shadows are another per-frame effect. Turn on only
        // if you have headroom, it is visible when dragging windows.
        off
    }
}

// ── Screenshots ──────────────────────────────────────────────────────────────

screenshot-path "~/Pictures/Screenshots/Screenshot from %Y-%m-%d %H-%M-%S.png"

// ── Startup ──────────────────────────────────────────────────────────────────

// Noctalia provides the bar, launcher, notifications, OSD and control center.
spawn-at-startup "noctalia"

// ── Animation ────────────────────────────────────────────────────────────────

animations {
    // Set to 2.0 if you want it snappier on this hardware.
    // slowdown LOWER = animations play faster = fewer frames rendered.
    // On 2 physical cores that is the cheapest CPU saving available here.
    slowdown 1.0
}

// ── Window rules ─────────────────────────────────────────────────────────────

window-rule {
    geometry-corner-radius 10
    clip-to-geometry true
}

// Let apps omit their own decorations so niri can draw consistent corners.
prefer-no-csd

// Firefox picture-in-picture floats.
window-rule {
    match app-id=r#"firefox$"# title="^Picture-in-Picture$"
    open-floating true
}

// Never let password managers end up in screenshots or screen shares.
window-rule {
    match app-id=r#"^org\.keepassxc\.KeePassXC$"#
    match app-id=r#"^org\.gnome\.World\.Secrets$"#
    block-out-from "screen-capture"
}

// ── Noctalia layer rules ─────────────────────────────────────────────────────
// Noctalia publishes its own blur regions automatically, so nothing is required
// here for it to work. These rules only control how that blur samples content.

// BLUR (disabled on purpose for the HD 520).
// Uncomment to enable. If you do, also set `compositor_blur = true` under
// [bar.main] in ~/.config/noctalia/config.toml.
// layer-rule {
//     match namespace="^noctalia-(bar-[^ " ]+|notification|dock|panel|attached-panel|osd)$"
//     background-effect { xray false }
// }
// window-rule { background-effect { blur true xray false } }
// blur {
//     passes 2
//     offset 3.0
//     noise 0.03
//     saturation 1.0
// }

// Do NOT set place-within-backdrop on the Noctalia wallpaper.
// That option moves the surface *into* the overview backdrop, which niri only
// shows during the Overview and workspace-switch animations -- the wallpaper
// then disappears during normal use. A background-layer wallpaper shows up in
// the overview on its own, so no layer rule is needed here.

// ── Keybindings ──────────────────────────────────────────────────────────────

binds {
    Mod+Shift+Slash { show-hotkey-overlay; }

    // --- Noctalia panels / shell ---
    Mod+D       hotkey-overlay-title="Noctalia: launcher"       { spawn-sh "noctalia msg panel-toggle launcher"; }
    Mod+Shift+D hotkey-overlay-title="Noctalia: clipboard"      { spawn-sh "noctalia msg panel-toggle clipboard"; }
    Mod+Shift+N hotkey-overlay-title="Noctalia: control center" { spawn-sh "noctalia msg panel-toggle control-center"; }
    Mod+N       hotkey-overlay-title="Noctalia: toggle Do Not Disturb" { spawn-sh "noctalia msg notification-dnd-toggle"; }
    Mod+E       hotkey-overlay-title="Noctalia: toggle dock"    { spawn-sh "noctalia msg dock-toggle"; }
    Mod+Shift+W hotkey-overlay-title="Noctalia: wallpaper panel" { spawn-sh "noctalia msg panel-toggle wallpaper"; }
    Mod+Shift+S hotkey-overlay-title="Noctalia: settings"       { spawn-sh "noctalia msg settings-toggle"; }
    Mod+Shift+E hotkey-overlay-title="Noctalia: session menu (log out / reboot)" { spawn-sh "noctalia msg panel-toggle session"; }
    Mod+Shift+M hotkey-overlay-title="Noctalia: toggle light/dark theme" { spawn-sh "noctalia msg theme-mode-toggle"; }

    // Alt+Tab window switcher. "hold" makes a quick tap return to the
    // previously focused window, like every other desktop.
    // repeat=false: once the overlay grabs keyboard focus it handles Tab
    // itself, so letting niri repeat would spawn a process per key event.
    Alt+Tab repeat=false { spawn-sh "noctalia msg window-switcher hold"; }
    // Same, but without the hold behaviour (overlay stays until Enter/Esc).
    Alt+Shift+Tab repeat=false { spawn-sh "noctalia msg window-switcher"; }

    Mod+T hotkey-overlay-title="Open a Terminal: foot" { spawn "foot"; }
    Super+Alt+L hotkey-overlay-title="Lock the Screen" { spawn-sh "noctalia msg session lock"; }
    Super+Alt+Shift+L hotkey-overlay-title="Lock and suspend" { spawn-sh "noctalia msg session lock-and-suspend"; }

    // --- Audio ---
    XF86AudioRaiseVolume allow-when-locked=true { spawn-sh "wpctl set-volume @DEFAULT_AUDIO_SINK@ 0.1+ -l 1.0"; }
    XF86AudioLowerVolume allow-when-locked=true { spawn-sh "wpctl set-volume @DEFAULT_AUDIO_SINK@ 0.1-"; }
    XF86AudioMute        allow-when-locked=true { spawn-sh "wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle"; }
    XF86AudioMicMute     allow-when-locked=true { spawn-sh "wpctl set-mute @DEFAULT_AUDIO_SOURCE@ toggle"; }

    // --- Media / brightness ---
    XF86AudioPlay  allow-when-locked=true { spawn-sh "playerctl play-pause"; }
    XF86AudioStop  allow-when-locked=true { spawn-sh "playerctl stop"; }
    XF86AudioPrev  allow-when-locked=true { spawn-sh "playerctl previous"; }
    XF86AudioNext  allow-when-locked=true { spawn-sh "playerctl next"; }

    XF86MonBrightnessUp   allow-when-locked=true { spawn "brightnessctl" "--class=backlight" "set" "+10%"; }
    XF86MonBrightnessDown allow-when-locked=true { spawn "brightnessctl" "--class=backlight" "set" "10%-"; }

    // --- Overview ---
    Mod+O repeat=false { toggle-overview; }

    // --- Focus ---
    Mod+Q repeat=false { close-window; }

    Mod+Left  { focus-column-left; }
    Mod+Down  { focus-window-down; }
    Mod+Up    { focus-window-up; }
    Mod+Right { focus-column-right; }
    Mod+H     { focus-column-left; }
    Mod+J     { focus-window-down; }
    Mod+K     { focus-window-up; }
    Mod+L     { focus-column-right; }

    Mod+Ctrl+Left  { move-column-left; }
    Mod+Ctrl+Down  { move-window-down; }
    Mod+Ctrl+Up    { move-window-up; }
    Mod+Ctrl+Right { move-column-right; }
    Mod+Ctrl+H     { move-column-left; }
    Mod+Ctrl+J     { move-window-down; }
    Mod+Ctrl+K     { move-window-up; }
    Mod+Ctrl+L     { move-column-right; }

    Mod+Home { focus-column-first; }
    Mod+End  { focus-column-last; }
    Mod+Ctrl+Home { move-column-to-first; }
    Mod+Ctrl+End  { move-column-to-last; }

    Mod+Shift+Left  { focus-monitor-left; }
    Mod+Shift+Down  { focus-monitor-down; }
    Mod+Shift+Up    { focus-monitor-up; }
    Mod+Shift+Right { focus-monitor-right; }

    // --- Workspaces ---
    Mod+Page_Down      { focus-workspace-down; }
    Mod+Page_Up        { focus-workspace-up; }
    Mod+U              { focus-workspace-down; }
    Mod+I              { focus-workspace-up; }
    Mod+Ctrl+Page_Down { move-column-to-workspace-down; }
    Mod+Ctrl+Page_Up   { move-column-to-workspace-up; }
    Mod+Ctrl+U         { move-column-to-workspace-down; }
    Mod+Ctrl+I         { move-column-to-workspace-up; }
    Mod+Shift+Page_Down { move-workspace-down; }
    Mod+Shift+Page_Up   { move-workspace-up; }
    Mod+Shift+U         { move-workspace-down; }
    Mod+Shift+I         { move-workspace-up; }

    Mod+1 { focus-workspace 1; }
    Mod+2 { focus-workspace 2; }
    Mod+3 { focus-workspace 3; }
    Mod+4 { focus-workspace 4; }
    Mod+5 { focus-workspace 5; }
    Mod+6 { focus-workspace 6; }
    Mod+7 { focus-workspace 7; }
    Mod+8 { focus-workspace 8; }
    Mod+9 { focus-workspace 9; }
    Mod+Ctrl+1 { move-column-to-workspace 1; }
    Mod+Ctrl+2 { move-column-to-workspace 2; }
    Mod+Ctrl+3 { move-column-to-workspace 3; }
    Mod+Ctrl+4 { move-column-to-workspace 4; }
    Mod+Ctrl+5 { move-column-to-workspace 5; }
    Mod+Ctrl+6 { move-column-to-workspace 6; }
    Mod+Ctrl+7 { move-column-to-workspace 7; }
    Mod+Ctrl+8 { move-column-to-workspace 8; }
    Mod+Ctrl+9 { move-column-to-workspace 9; }

    Mod+WheelScrollDown      cooldown-ms=150 { focus-workspace-down; }
    Mod+WheelScrollUp        cooldown-ms=150 { focus-workspace-up; }
    Mod+Ctrl+WheelScrollDown cooldown-ms=150 { move-column-to-workspace-down; }
    Mod+Ctrl+WheelScrollUp   cooldown-ms=150 { move-column-to-workspace-up; }

    Mod+WheelScrollRight { focus-column-right; }
    Mod+WheelScrollLeft  { focus-column-left; }

    // --- Columns ---
    Mod+BracketLeft  { consume-or-expel-window-left; }
    Mod+BracketRight { consume-or-expel-window-right; }
    Mod+Comma  { consume-window-into-column; }
    Mod+Period { expel-window-from-column; }

    Mod+R { switch-preset-column-width; }
    Mod+Shift+R { switch-preset-column-width-back; }
    Mod+Ctrl+R { reset-window-height; }
    Mod+Ctrl+Shift+R { switch-preset-window-height; }

    Mod+F { maximize-column; }
    Mod+Shift+F { fullscreen-window; }
    Mod+M { maximize-window-to-edges; }
    Mod+Ctrl+F { expand-column-to-available-width; }
    Mod+C { center-column; }
    Mod+Ctrl+C { center-visible-columns; }

    Mod+Minus { set-column-width "-10%"; }
    Mod+Equal { set-column-width "+10%"; }
    Mod+Shift+Minus { set-window-height "-10%"; }
    Mod+Shift+Equal { set-window-height "+10%"; }

    Mod+V       { toggle-window-floating; }
    Mod+Shift+V { switch-focus-between-floating-and-tiling; }
    Mod+W { toggle-column-tabbed-display; }

    // --- Screenshots ---
    Print { screenshot; }
    Ctrl+Print { screenshot-screen; }
    Alt+Print { screenshot-window; }

    // --- Session ---
    Mod+Escape allow-inhibiting=false { toggle-keyboard-shortcuts-inhibit; }
    // Note: Mod+Shift+E is the Noctalia session menu, so quit lives on Mod+Shift+Q.
    Mod+Shift+Q { quit; }
    Ctrl+Alt+Delete { quit; }
    Mod+Shift+P { power-off-monitors; }
}
include "noctalia.kdl"
```

### ~/.config/niri/noctalia.kdl

Included by `config.kdl` above — sets the focus-ring/border/tab colors to match
the Noctalia theme.

```kdl
layout {

    focus-ring {
        active-color   "#b8bb26"
        inactive-color "#000000"
        urgent-color   "#fb4934"
    }

    border {
        active-color   "#b8bb26"
        inactive-color "#000000"
        urgent-color   "#fb4934"
    }


    tab-indicator {
        active-color   "#b8bb26"
        inactive-color "#40420b"
        urgent-color   "#fb4934"
    }

    insert-hint {
        color "#b8bb2680"
    }
}

recent-windows {
    highlight {
        active-color "#b8bb26"
        urgent-color "#fb4934"
    }
}
```

## 3. Create the Noctalia config

```bash
mkdir -p ~/.config/noctalia
```

### ~/.config/noctalia/config.toml

```toml
# Noctalia v5.2.1 configuration
# Location: ~/.config/noctalia/config.toml
#
# This file is the hand-written/base layer. Anything you toggle in the Noctalia
# Settings GUI is written to ~/.local/state/noctalia/settings.toml and WINS over
# what is written here. If a value below seems ignored, that is why.
#
# Both files are hot-reloaded on change.
#
# Tune for: Intel HD 520 (Broadwell, 2015), 4-core Skylake-U, 19GiB RAM.
# The guiding rule below is "no fullscreen effects": blur is the one thing that
# reliably costs frames on this GPU, so it is off. If it turns out smooth,
# switch blur on -- see ~/.config/niri/config.kdl for the matching compositor
# rules, and set compositor_blur = true under [bar.main] in this file.

[shell]
telemetry_enabled = false

# Snappier than default. The 4 cores here are better spent on the app you are
# actually using than on easing curves.
[shell.animation]
speed = 1.6

# bar.main.compositor_blur must stay false while blur is off, otherwise the bar
# is transparent with nothing behind it to blur.
[bar.main]
position           = "top"
thickness          = 36
compositor_blur    = false
background_opacity = 1.0
shadow             = false
# Noctalia insets the bar 100 logical px from each end by default (margin_ends),
# which makes it float instead of spanning the screen. 0 = full width.
margin_ends         = 0
# With a full-width bar the default concave_edge_corners would carve notches
# into the two inner corners; false keeps all four square.
concave_edge_corners = false
# scale / font_scale left at 1.0 so the text keeps its normal size.
start  = ["launcher", "wallpaper", "workspaces"]
center = ["clock"]
end    = ["tray", "notifications", "clipboard", "network", "volume",
          "battery", "control-center", "session"]

# Sampling CPU/GPU stats in the bar costs cycles on a 4-core machine. Polling
# less often is nearly invisible to the eye and gives the CPU back.
# This is a 2-core/4-thread i5-6300U, so the intervals are stretched further.
[system.monitor]
enabled            = true
cpu_poll_seconds   = 10.0
gpu_poll_seconds   = 30.0
memory_poll_seconds = 15.0
network_poll_seconds = 20.0
disk_poll_seconds  = 45.0

[dock]
enabled     = true
position    = "bottom"
auto_hide   = true      # gets out of the way on a small laptop panel
reserve_space = false   # ...and does not permanently steal screen height
show_running  = true
icon_size     = 44
pinned        = ["foot", "firefox"]

[wallpaper]
# Kept to cheap transitions only; zoom/honeycomb/stripes animate many layers.
transition          = ["fade", "wipe"]
transition_duration = 700
fill_mode           = "crop"
fill_color          = "#1e1e2e"

[theme]
mode   = "dark"
source = "builtin"
builtin = "Noctalia"

[backdrop]
enabled = false   # an extra blurred wallpaper copy is expensive here

# Lockscreen, OSD, notifications and the control center are left at their
# built-in defaults.
```

## 4. Set up the Noctalia autostart wrapper

niri already launches Noctalia via `spawn-at-startup "noctalia"`. An XDG
autostart entry would launch a *second* copy on niri (Noctalia is a singleton
and refuses, printing an error). The wrapper below checks the per-display lock
and exits quietly when Noctalia is already up. It also skips Noctalia on
MangoWM sessions.

Create `~/.local/bin/noctalia-autostart` (make it executable):

```bash
mkdir -p ~/.local/bin
cat > ~/.local/bin/noctalia-autostart <<'EOF'
#!/bin/sh
# Autostart wrapper for Noctalia.
#
# Why this exists instead of a plain `Exec=noctalia`:
#
#   niri starts Noctalia itself via `spawn-at-startup "noctalia"` in
#   ~/.config/niri/config.kdl. An XDG autostart entry runs in EVERY session,
#   so on niri it would try to launch a second copy. Noctalia is a singleton --
#   it takes a per-display lock at $XDG_RUNTIME_DIR/noctalia-$WAYLAND_DISPLAY.lock
#   and refuses to start twice with "error: noctalia is already running" --
#   but that refusal prints an error to the session log on every login.
#
#   So check the lock first and exit 0 quietly when Noctalia is already up.
#
#   The MangoWM session does not use Noctalia at all: it runs the classic
#   stack (waybar + rofi + swaybg + dunst) started by mango's exec_once.
#   Starting Noctalia there would put a second bar, launcher and notification
#   daemon on top of it. mango sets XDG_CURRENT_DESKTOP=mango and exports
#   MANGO_INSTANCE_SIGNATURE to its children, so either marker means "this
#   is a MangoWM session" - check both, the display manager may have set
#   XDG_CURRENT_DESKTOP=GNOME before handing over to the session.

case "${XDG_CURRENT_DESKTOP:-}" in
    *mango*)
        exit 0
        ;;
esac

if [ -n "${MANGO_INSTANCE_SIGNATURE:-}" ]; then
    exit 0
fi

lock="$XDG_RUNTIME_DIR/noctalia-${WAYLAND_DISPLAY}.lock"

if [ -e "$lock" ]; then
    exit 0
fi

exec noctalia
EOF
chmod +x ~/.local/bin/noctalia-autostart
```

Create the XDG autostart desktop entry `~/.config/autostart/noctalia.desktop`:

```ini
[Desktop Entry]
Type=Application
Version=1.0
Name=Noctalia Shell
Comment=Bar, launcher, notifications and control centre (niri / GNOME sessions)

Exec=/home/affan/.local/bin/noctalia-autostart
Terminal=false
StartupNotify=false
X-GNOME-Autostart-enabled=true
```

> Replace `/home/affan` with your own `$HOME` path, or use a relative-friendly
> `Exec=$HOME/.local/bin/noctalia-autostart` if your autostart handler expands it.

## 5. Validate and launch

Validate the niri config before logging in:

```bash
niri validate
```

Expected: no errors.

Log out and select the **niri** Wayland session at the display manager (SDDM),
or start niri directly from a TTY:

```bash
niri
```

Noctalia starts automatically via `spawn-at-startup "noctalia"` in
`config.kdl`; the autostart wrapper guards against a duplicate launch.

## 6. Hardware-specific tuning notes

- **Intel HD 520 / Broadwell**: blur and compositor shadow are OFF in both
  `config.kdl` and `config.toml` (`compositor_blur = false`). Fullscreen blur
  is a multi-pass effect and costs frames on this iGPU. To enable blur anyway,
  uncomment the BLUR `layer-rule`/`window-rule`/`blur` block in
  `~/.config/niri/config.kdl` AND set `compositor_blur = true` under
  `[bar.main]` in `~/.config/noctalia/config.toml`.
- **Animations**: `slowdown 1.0` in niri and `speed = 1.6` in Noctalia keep
  motion snappy on 4 cores.
- **System monitor polls** are stretched (CPU 10s, GPU 30s, disk 45s) to spare
  the 2-core/4-thread CPU.

## 7. Keybinding cheat sheet (Mod = Alt / Super as bound)

| Binding | Action |
|---------|--------|
| Mod+D | Noctalia launcher |
| Mod+Shift+D | Noctalia clipboard |
| Mod+Shift+N | Noctalia control center |
| Mod+N | Toggle Do Not Disturb |
| Mod+E | Toggle dock |
| Mod+Shift+W | Wallpaper panel |
| Mod+Shift+S | Noctalia settings |
| Mod+Shift+E | Session menu (log out / reboot) |
| Mod+Shift+M | Toggle light/dark theme |
| Alt+Tab | Window switcher (hold) |
| Alt+Shift+Tab | Window switcher (sticky) |
| Mod+T | Open foot terminal |
| Super+Alt+L | Lock screen |
| Super+Alt+Shift+L | Lock and suspend |
| Mod+O | Toggle overview |
| Mod+Q | Close window |
| Mod+Shift+Q | Quit niri |
| Mod+1…9 / Mod+Ctrl+1…9 | Focus / move to workspace |
| Mod+U / Mod+I | Focus workspace down / up |
| Print / Ctrl+Print / Alt+Print | Screenshot / screen / window |
| XF86Audio* / XF86MonBrightness* | Volume / media / brightness |

## 8. Files to copy to replicate this DE

On the source machine these are the canonical files; copy them to the same
paths on the target machine:

- `~/.config/niri/config.kdl`
- `~/.config/niri/noctalia.kdl`
- `~/.config/noctalia/config.toml`
- `~/.config/autostart/noctalia.desktop`
- `~/.local/bin/noctalia-autostart` (chmod +x)

Then run `niri validate`, install the package list from section 1, and log into
the niri session.
