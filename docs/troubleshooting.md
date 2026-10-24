[< Back to docs index](index.md)

# Troubleshooting

Symptom-keyed fixes for the real Wispr Flow for Linux failure modes. The
headings are the symptoms — search for yours.

## First step: `wispr-flow --doctor`

Hey! Before you go spelunking, run the built-in diagnostics. I wired `--doctor`
up precisely so I'd stop guessing at which of a dozen Linux quirks was biting me
on any given machine — it checks them all in one shot:

```bash
wispr-flow --doctor
# AppImage:
./wispr-flow-*.AppImage --doctor
```

It prints `[PASS]` / `[WARN]` / `[FAIL]` lines with inline fix commands, grouped:

| Section | Checks |
|---|---|
| Display | Wayland/X11 detection, desktop family (KDE / GNOME / wlroots) |
| Text Injection (uinput) | `/dev/uinput` writability, `input` group membership |
| Clipboard | `wl-clipboard` (Wayland) / `xclip`/`xsel` (X11) |
| Accessibility (AT-SPI) | `toolkit-accessibility` / a11y bus reachability |
| GNOME Window Bridge | (GNOME only) extension installed + active |
| Helper / Singleton / Crashes | helper binary present, stale lock, recent crash count |

The exit status is non-zero if any check FAILs. Do me a favor and attach the
full output to bug reports — it's the single most useful thing you can hand me.

## Push-to-talk doesn't fire / shortcut recorder captures no keystrokes (including during onboarding)

The in-app shortcut recorder (including the one on the onboarding setup screen)
and push-to-talk are both fed entirely by `KeypressEvent` frames the helper
streams from the OS key layer. If those frames never arrive, the recorder shows
nothing and captures nothing — the app has no other path for global hotkey input.

The helper has two capture backends:

- **evdev** (`/dev/input/event*`) — works on Wayland **and** X11. Needs read
  access to the input devices.
- **XInput2** — true X11 sessions only (not XWayland). Needs no device access.

On **Wayland**, only evdev is available, and it needs the udev access grant.

### Fix

Run the built-in diagnostics first — the `Push-to-Talk (input monitor)` section
will tell you exactly what's missing:

```bash
wispr-flow --doctor
```

If it prints `[FAIL] /dev/input: none of N event device(s) readable`, install
the udev rule that grants your session access to input devices (this is a
one-time step; it survives reboots):

```bash
wispr-flow --install-udev-rules
```

The command escalates via `pkexec` (graphical sudo prompt) or falls back to
`sudo`. After it completes you may need to **log out and back in** (or replug
your keyboard) for the new ACL to take effect on already-open devices. Then
re-run `--doctor` to confirm the check passes and try the shortcut recorder
again.

Alternatively, if you're already a member of the `input` group (check with
`id -nG | grep input`), just re-login — logind should grant uaccess on session
start.

> [!NOTE]
> On **X11** the helper uses XInput2, which needs no device access at all. If
> the shortcut recorder is still dead on X11 after confirming the helper is
> running (`--doctor` shows the helper launch as OK), file a bug with the
> full `--doctor` output.

## Push-to-talk is blank in Settings, or bound to a key that does nothing

Before the `linux-main-shortcut-defaults.sh` patch, a fresh Linux profile was
seeded with the macOS shortcut map, and its push-to-talk key has no Linux
keycode. The stored binding is the `-1` sentinel:

```json
"shortcuts": { "-1": "ptt", "-1+32": "popo", "-1+162": "lens" },
"modifierShortcut": "9"
```

Settings renders that binding blank and no key can ever match it. Current
builds seed `162+91` (Ctrl+Meta) instead, but the patch only changes what a
**new** profile gets. A profile created by an older build keeps its `-1`
entries across upgrades.

### Fix

Open Settings → Shortcuts and record a new push-to-talk key (Ctrl+Meta is the
default the patch seeds). Do the same for any other shortcut that shows
blank. If the recorder captures nothing, that is the helper key monitor, not
this bug: see the section above.

To start over instead, quit the app and delete the `shortcuts` and
`modifierShortcut` keys from `~/.config/Wispr Flow/config.json`; the next
launch reseeds them with the Linux map.

## Paste does nothing / transcription doesn't get typed into my app

This is the whole reason the app exists, so when it goes silent it hurts. In my
experience it's almost always one of two things: `/dev/uinput` or the clipboard.
Run `--doctor` and work the failures top-down:

### Fix

1. **`/dev/uinput` not writable** — injection has nowhere to type. The virtual
   keyboard can't open the device. Grant access (then re-run `--doctor`):

   ```bash
   # Immediate (this session):
   sudo setfacl -m u:$USER:rw /dev/uinput

   # Persistent: add yourself to the input group, then log out / back in:
   sudo usermod -aG input "$USER"
   ```

   If `/dev/uinput` is missing entirely, the kernel module isn't loaded yet —
   load it: `sudo modprobe uinput`.

   The module can also be unloaded while `/dev/uinput` still exists: a
   `crw------- root root` static placeholder that udev has never processed.
   Then `--install-udev-rules` and `udevadm trigger` change nothing, because
   udev has no device to apply the rule to. `udevadm info /dev/uinput`
   answers `Unknown device` in that state. Load the module, persist it, and
   re-trigger:

   ```bash
   echo uinput | sudo tee /etc/modules-load.d/uinput.conf
   sudo modprobe uinput
   sudo udevadm trigger --subsystem-match=misc --sysname-match=uinput
   getfacl -p /dev/uinput   # expect a user:<you>:rw- line
   ```

   > [!NOTE]
   > Seen on Omarchy 4.0.6 (Arch, Hyprland 0.56.2) with the
   > `wispr-flow-appimage` AUR package: `--doctor` kept failing
   > `/dev/uinput` after `--install-udev-rules` until the module was loaded.

2. **`wl-clipboard` missing (Wayland)** — clipboard-based paste has no tool to
   set the selection with. Install it:

   ```bash
   sudo dnf install wl-clipboard    # Fedora/RHEL
   sudo apt install wl-clipboard    # Debian/Ubuntu
   ```

   On X11, the equivalent dep is `xclip` or `xsel`.

3. **Not in the `input` group and no uaccess ACL** — some distros don't ACL
   uinput through logind (Arch is the one that caught me), so group membership
   is your grant path instead. Go back to step 1's `usermod` and relogin.

4. **`wl-copy` hangs instead of returning (Wayland)** — `wl-clipboard` is
   installed and `--doctor` is all-green, but the log shows `gRPC transcription
   successful` followed by `PasteText: Request timed out`, and nothing reaches
   the focused app. Check whether the clipboard tool returns at all:

   ```bash
   printf 'test' | timeout 4 wl-copy; echo "$?"   # 124 means it hung
   pgrep -x wl-copy                               # stray copies piling up
   ```

   Why it hangs is not established. On a compositor with no data-control
   protocol, `wl-clipboard` falls back to mapping a surface that has to take
   keyboard focus before it can own the selection, and a daemon's `wl-copy`
   never gets that focus. Mutter has advertised `ext-data-control` since
   GNOME 48, though, and the helper's own in-process clipboard source relies
   on it (see [wayland-injection.md](learnings/wayland-injection.md)), so on
   a current GNOME that fallback should not be in play. Whatever the cause,
   routing `wl-copy` through `xclip`, which reaches the clipboard over
   Xwayland, gets around it. Use a shim earlier in `PATH` than `/usr/bin`
   (needs `xclip` and a running Xwayland):

   ```bash
   #!/bin/bash
   # ~/.local/bin/wl-copy
   sel=clipboard
   type=
   while [[ $# -gt 0 ]]; do
   	case "$1" in
   		-p | --primary) sel=primary ;;
   		-t | --type) shift; type="$1" ;;
   		--type=*) type="${1#--type=}" ;;
   		-*) ;;
   		*) break ;;
   	esac
   	shift
   done
   [[ -z $type || $type == text/plain* ]] &&
   	exec xclip -selection "$sel" -i
   exec xclip -selection "$sel" -t "$type" -i
   ```

   > [!NOTE]
   > Seen on GNOME Shell 50.2 / mutter 50.2 / wl-clipboard 2.3.0 (CachyOS,
   > Wayland). GNOME paste is validated working on Ubuntu 24.04 / GNOME —
   > see [compatibility.md](compatibility.md) — so this is not GNOME-wide.
   > Run the check above before assuming it's your problem.

Want the why behind all this? See
[configuration.md](configuration.md#text-injection-devuinput-access) for how the
udev rule grants access, and
[learnings/wayland-injection.md](learnings/wayland-injection.md) for the
mechanism.

## Window / app detection wrong on GNOME

`--doctor` shows the GNOME Window Bridge as not active, or the active app /
running-apps list is empty or stale on a GNOME session.

### Fix

Here's the gotcha that ate an afternoon of mine: the bridge is a GNOME Shell
extension, and **GNOME scans extensions only at session start**. Enabling it
mid-session isn't enough on its own:

```bash
gnome-extensions info wispr-flow-window-bridge@wispr.flow
gnome-extensions enable wispr-flow-window-bridge@wispr.flow
```

Then **log out and back in.** That's the part people skip. The first run after
install quietly falls back to AT-SPI and logs a "log out and back in" notice;
once you've cycled the session the bridge sticks around for good. The whole story
is in [learnings/gnome-shell-extension.md](learnings/gnome-shell-extension.md).

> [!NOTE]
> This is GNOME-specific. KDE uses an in-process KWin script and wlroots
> compositors (Sway, Hyprland) use AT-SPI — neither needs the extension, and
> `--doctor` hides the GNOME Window Bridge section off GNOME.

## "no such table" / database errors

The app launches but every DB-backed feature errors with **"no such table"**,
and the log shows **"Executed 0 migrations"**.

### Fix

This one is sneaky, and it took me a while to trust the cause. Your Electron
launcher is named `electron`. Electron forces `app.isPackaged=false` whenever the
launcher is literally named `electron`, so the app goes hunting for migrations in
the (nonexistent) *dev* path, runs zero of them, and then every query slams into
a table that was never created. The launcher has to be renamed to `wispr-flow`:

- **Installed packages already do this** — the makers rename the binary for you
  and the launcher exports `ELECTRON_FORCE_IS_PACKAGED=true`, so you shouldn't
  see this from a `.deb` / `.rpm` / AppImage. If you do, that's a bug — file it
  with your `--doctor` output.
- **Run-in-place / manual builds** — you're on the hook for the rename here:
  move the Electron binary off `electron` (e.g. to `wispr-flow`) before you
  launch, or set `ELECTRON_FORCE_IS_PACKAGED=true`.

I wrote the whole thing up in
[learnings/ispackaged-rename.md](learnings/ispackaged-rename.md).

## Dictation history is empty after an update

The app starts signed in, but the Hub shows no past dictations, notes or
dictionary words.

### Fix

Earlier builds kept the database in
`~/Library/Application Support/Wispr Flow`. The launcher moves that directory
into `~/.config/Wispr Flow` on the first start after the update. It skips the
move while Wispr Flow is still running, and when any file in it would
overwrite one already in `~/.config/Wispr Flow`. `--doctor` says which case
you are in:

```bash
wispr-flow --doctor | grep -A3 'Legacy data dir'
grep 'Legacy data dir' ~/.cache/wispr-flow/launcher.log
```

If Wispr Flow was running during the update, quit it from the tray and start
it again. If a file clashes, the new database in `~/.config/Wispr Flow` only
holds what you did since the update. To keep the old history instead, quit
Wispr Flow and move the new copies aside:

```bash
cd ~/.config/Wispr\ Flow
mkdir -p ../wispr-flow-after-update
mv flow.sqlite flow.sqlite-wal flow.sqlite-shm ../wispr-flow-after-update/ 2>/dev/null
```

The next start moves the old directory in.

## Wispr Flow starts at login but no window opens

After logging in, the status pill is there, but the Hub window doesn't open.

### Fix

That is the intended login start. "Open at login" launches with `--hidden`,
the same as on Windows and macOS. Open the Hub from the tray icon, or start
Wispr Flow again from your app launcher. A second launch brings up the Hub
of the running instance. Stock GNOME has no tray without an extension, so
there the app launcher is the way in.

To stop starting at login, turn off "Open at login" in Settings, or disable
the entry in your desktop's startup-apps settings. `wispr-flow --doctor`
shows which applies. It also warns when an AppImage entry points at a file
that has moved; starting the AppImage once from its new location repairs it.

## "Open at login" is on but Wispr Flow does not start at login

The toggle in Settings shows the saved preference, which is on by default,
and an older build never wrote an autostart entry for it.

### Fix

Turn "Open at login" off and on again. That writes
`~/.config/autostart/wispr-flow.desktop`, and `wispr-flow --doctor` then
reports `Open at login: on`. Disabling the entry in your desktop's
startup-apps settings does not change the toggle, which keeps showing the
preference.

## App won't start from a terminal

Launching from an SSH session or bare TTY does nothing, or the launcher log says
no display is available.

### Fix

No surprise here once you know it: Wispr Flow needs a **graphical session** — a
live Wayland compositor or X server. It can't run headless from a TTY, full
stop. Quick check:

```bash
echo "$WAYLAND_DISPLAY $DISPLAY"   # at least one must be non-empty
```

If both come back empty, you're on a bare TTY or an SSH session with no display —
launch the app from inside your desktop session instead. (If you genuinely need
headless, that's what the artifact tests do: they wrap the launch in `xvfb-run` +
`dbus-run-session` — see [tests/README.md](../tests/README.md).)

## Blank window / GPU crash on launch

The window renders blank, or the GPU process crashes — common on broken drivers,
VMs, or remote (XRDP) sessions.

### Fix

Disable hardware acceleration:

```bash
# One-off:
WISPR_DISABLE_GPU=1 wispr-flow

# Persistent:
echo 'export WISPR_DISABLE_GPU=1' >> ~/.profile
```

Under the hood that passes `--disable-gpu --disable-software-rasterizer`. Good
news if you're on XRDP: the launcher already sets those flags for you there, no
env var needed (it detects XRDP specifically — other RDP backends still need the
variable). More knobs in
[configuration.md](configuration.md#environment-variables).

## App won't launch a second time (stale singleton lock)

After a crash or unclean shutdown, the app silently quits on every launch and
nothing appears.

### Fix

A stale `SingletonLock` left behind by a dead process is what's blocking new
instances. The launcher is supposed to clear it once the owning PID is gone, but
if it somehow lingers you can nuke it by hand:

```bash
rm -f ~/.config/Wispr\ Flow/SingletonLock
```

And yes, `--doctor` flags a stale lock under its singleton-lock check, so you
don't have to go looking for it yourself.

## Clicks near the status pill are swallowed, or the pill's buttons don't respond, on Wayland

The status pill sits in a transparent always-on-top window that is larger
than the pill you see. On X11 the launcher can shape that window so only the
painted pill takes input; on native Wayland there is no input-shaping
protocol, so a compositor either treats the whole transparent box as
clickable (clicks near the pill never reach the app underneath) or, once the
window is set click-through, keeps the pill's own hover buttons out of reach.
GNOME can get shaping back through the helper's shell extension; KDE,
Hyprland and the other wlroots compositors have nothing yet.

### Fix

Run the app under XWayland, which brings the X11 behaviour back on every
compositor. The trade-off is blurry HiDPI scaling on a fractional-scale
display.

```bash
# One-off:
WISPR_USE_X11=1 wispr-flow

# Persistent:
echo 'export WISPR_USE_X11=1' >> ~/.profile
```

Only the toolkit backend changes. Text injection still goes through the
Wayland path (`/dev/uinput` and `wl-clipboard`), so nothing else in
`--doctor` should move; its display line reports `Mode: XWayland forced`.
If the dead zone is still there under XWayland, that is a different bug:
open an issue with `--doctor` output and say which compositor.

## Status pill sits in a bordered box mid-screen, or keyboard input goes to Wispr Flow on another workspace (Hyprland)

On Hyprland the status pill's transparent window (class `wispr-flow`) is
managed like any other window: it gets a border and shadow, opens centered
on one workspace, and takes keyboard focus. With `misc:focus_on_activate`
on (Omarchy's default), the app can also pull focus to a Wispr Flow window
on another workspace, so typing in the window in front of you appears dead
until you Alt+Tab back to it.

### Fix

Add window rules. This is the Hyprland 0.55+ Lua syntax; put it in any
user Lua file your `hyprland.lua` loads (on Omarchy,
`~/.config/hypr/looknfeel.lua`), then run `hyprctl reload` and
`hyprctl configerrors`:

```lua
-- Never let Wispr Flow windows pull focus from the app you're typing in.
hl.window_rule({ match = { class = "^wispr-flow$" }, focus_on_activate = false })

-- The status pill: no decorations, never focused.
hl.window_rule({
  match = { class = "^wispr-flow$", title = "^(Status|Flow Status Indicator)$" },
  float = true,
  no_focus = true,
  no_initial_focus = true,
  no_dim = true,
  no_shadow = true,
  no_blur = true,
  border_size = 0,
  opacity = "1 1",
})
```

The pill maps with an empty title and is named `Flow Status Indicator` and
then `Status` afterwards, so the rules Hyprland only evaluates at map time
(`pin`, `move`) don't match it, while the ones re-evaluated on title change
(above) do. Pin it and move it to bottom-center by hand, once per app start:

```bash
addr=$(hyprctl clients -j |
	jq -r '.[] | select(.class=="wispr-flow" and .title=="Status") | .address')
hyprctl dispatch "hl.dsp.window.pin({ window = 'address:$addr' })"
# x = (monitor width / scale - window width) / 2, y = monitor height / scale - window height
hyprctl dispatch "hl.dsp.window.move({ x = 512, y = 250, window = 'address:$addr' })"
```

`hl.dsp.window.pin` toggles, so run it once. The coordinates above are for
a 1920x1080 panel at scale 1.25 with the pill's 512x614 window.

> [!NOTE]
> Seen on Omarchy 4.0.6 / Hyprland 0.56.2. An `hl.on("window.title", ...)`
> handler that runs the two dispatches automatically should work, but
> hasn't been tested yet.

To stop Wispr Flow from a terminal, use `pkill -x wispr-flow`. It runs
cleanup tasks first, so it takes a few seconds to exit. `pkill -f` with the
install path also matches the shell running it.

## Wispr Flow and Chromium look zoomed in (Hyprland)

Hyprland's `scale = "auto"` picks 1.5 on many 1080p laptop panels, and
Electron and Chromium follow the compositor scale on native Wayland. Set an
explicit scale in your monitor config (on Omarchy, `~/.config/hypr/monitors.lua`):

```lua
hl.monitor({ output = "", mode = "preferred", position = "auto", scale = 1.25 })
```

Hyprland applies it on save; check with `hyprctl monitors | grep scale`.

## Dictation pastes into GUI apps but not into a terminal or SSH session

The helper always pastes with a uinput Ctrl+V chord. Terminals don't treat
Ctrl+V as paste: they pass a raw `^V` to whatever runs inside, so over SSH a
remote Claude Code reports "no image on clipboard" and a shell inserts a
literal. The text is on the clipboard; only the keystroke is wrong. Same
class of bug as #35 (Ghostty, Warp).

### Fix

Remap only the helper's virtual keyboard with [keyd](https://github.com/rvaiya/keyd),
so its Ctrl+V becomes Shift+Insert, which pastes in terminals and in
GTK/Qt/Chromium/Electron apps alike. Your physical Ctrl+V is untouched. The
helper's device reports `1234:5678`
(`grep -A1 'Wispr Flow Linux Helper' /proc/bus/input/devices`).

Use the full config from the next section, not a bare `v = S-insert`
mapping: routing the helper through keyd without dropping its modifiers
turns a helper bug into stuck Ctrl/Alt/Super.

```bash
sudo pacman -S keyd            # or your distro's package
sudo install -Dm644 wispr-flow.conf /etc/keyd/wispr-flow.conf
sudo systemctl enable --now keyd
```

> [!IMPORTANT]
> Once keyd grabs a keyboard, the helper only hears it through
> `keyd virtual keyboard`, and the helper only scans for keyboards at start.
> Restart Wispr Flow after keyd first starts and after every keyd restart
> (a reboot is fine: keyd starts first). Otherwise push-to-talk goes dead.
> `pkill -x wispr-flow`, then launch it again.

## Ctrl, Alt or Super stays stuck after a dictation (Super+1 opens a menu, Backspace deletes a line)

After a paste, apps behave as if Ctrl, Alt and/or Super are held: Super+N
opens compositor menus instead of switching workspaces, Backspace deletes a
whole word or line, push-to-talk lands in Command Mode (`ctrl + win + alt` in
the launcher log). Pressing the modifiers yourself clears it for a while.

The helper's `chord()` (`src/backend/uinput.rs` in the helper repo)
snapshots the modifiers held on every readable `/dev/input/event*` device
with `EVIOCGKEY`, releases them, sends Ctrl+V, then presses them again on
its own virtual keyboard. You later release the physical key, but nothing
releases the copy on the helper's device, so it stays down. The helper's
own device is in the next snapshot too, so the stuck set re-presses itself
on every paste. Check for it while touching no keys:

```python
# held-mods.py: run with python3 while touching no keys
import fcntl, glob, os
MODS = {29: "LCtrl", 97: "RCtrl", 42: "LShift", 54: "RShift",
        56: "LAlt", 100: "RAlt", 125: "LSuper", 126: "RSuper"}
L = 0x2ff // 8 + 1
req = (2 << 30) | (L << 16) | (0x45 << 8) | 0x18
for p in sorted(glob.glob("/dev/input/event*")):
    try:
        fd = os.open(p, os.O_RDONLY | os.O_NONBLOCK)
    except OSError:
        continue
    b = bytearray(L)
    fcntl.ioctl(fd, req, b)
    os.close(fd)
    held = [n for c, n in MODS.items() if b[c // 8] >> (c % 8) & 1]
    name = open(f"/sys/class/input/{os.path.basename(p)}/device/name").read()
    if held:
        print(p, name.strip(), held)
```

A `Wispr Flow Linux Helper  ['LCtrl', 'LAlt', 'LSuper']` line with your
hands off the keyboard is this bug.

### Fix

Restarting Wispr Flow destroys the stuck device. To stop it recurring,
filter the helper's device with keyd: drop every modifier it sends, and make
its Ctrl a silent layer that only adds Ctrl to the shortcuts the helper
actually sends. A stuck restore then emits nothing.

```ini
# /etc/keyd/wispr-flow.conf
[ids]
1234:5678

[main]
leftcontrol = layer(wispr_ctrl)
rightcontrol = layer(wispr_ctrl)
leftshift = noop
rightshift = noop
leftalt = noop
rightalt = noop
leftmeta = noop
rightmeta = noop

[wispr_ctrl]
# Shift+Insert so paste also works in terminals (previous section).
v = S-insert
c = C-c
x = C-x
a = C-a
z = C-z
```

`sudo keyd reload`, then restart Wispr Flow. The real fix belongs in the
helper: release the restored modifiers, or leave its own device out of the
snapshot.

> [!NOTE]
> Seen on Omarchy 4.0.6 / Hyprland 0.56.2, helper 0.1.2, Wispr Flow
> 1.6.957 and 1.6.1209.

## Shortcut recorder refuses a single key ("Shortcut must include a modifier key")

The app only accepts shortcuts that contain a modifier (Ctrl, Alt, Shift,
Win, Fn) or a mouse button, so a lone F13 is rejected. Fn is not an option
on Linux: `fn` maps to `-1` in
[`keycodes.json`](reference/keycodes.json), and most laptops handle Fn in
firmware without ever reporting it (Fn+F1 arrives as `KEY_MUTE`, Fn alone
as nothing).

### Fix

To hold one key for push-to-talk, have keyd make it send Wispr Flow's
default chord. For Caps Lock on a built-in keyboard (find its
`vendor:product` with `keyd monitor` or `/proc/bus/input/devices`):

```ini
# /etc/keyd/capslock-ptt.conf
[ids]
1532:0259

[main]
# Held Caps Lock = held Ctrl+Super; release releases both. Not oneshot.
capslock = layer(wispr_ptt)

[wispr_ptt:C-M]

[shift]
# Shift+Caps Lock still toggles caps lock.
capslock = capslock
```

Restart Wispr Flow afterwards (see the keyd note above). Holding Caps Lock
and pressing another key acts as Ctrl+Super plus that key.

## Hiding the Hub window on a tiling compositor

Hyprland and the other tiling compositors have no minimize. Close the Hub
(Super+W on Omarchy): the app keeps running, the status pill and
push-to-talk keep working, and the tray icon or launching Wispr Flow again
brings the Hub back.

## Selection reads come back empty

`GetSelectedTextViaCopy` returns nothing in some apps.

### Fix

Don't panic — this is usually working as intended. Selection reads ride on the
AT-SPI accessibility bus, and apps without an a11y bridge (bare terminals, a
handful of Electron apps) just don't expose a selection at all. It's expected,
and it's scoped to those specific apps. First confirm AT-SPI is actually on:

```bash
wispr-flow --doctor   # check the "Accessibility (AT-SPI)" section
```

The helper turns on session accessibility itself, so if `--doctor` still reports
the a11y bus as unreachable, odds are the AT-SPI registry just isn't running on
your compositor. [configuration.md](configuration.md#at-spi-accessibility) has
the details.

## "ERROR: Linux helper not staged at resources/Release/wispr-flow-linux-helper"

A `./build.sh` run fails in the packaging step with this error (often after a
`helper not present in staged tree` warning during the resources sync).

### Fix

The packaging makers refuse to ship a tree without the helper, and staging
didn't get one. Staging auto-fetches the prebuilt helper pinned in
`helper-version.txt` when `HELPER_BIN` is unset, so this means the fetch failed
or an explicit `HELPER_BIN` pointed at a missing/non-executable file — scroll up
to the `[WARN]` lines from Step 0 for which one.

1. **Auto-fetch failed** (no network, or `gh`/`curl` unavailable) — restore
   network access and re-run, or fetch by hand:

   ```bash
   scripts/setup/fetch-helper-bin.sh x86_64   # or aarch64
   ./build.sh --build deb
   ```

2. **`HELPER_BIN` override is wrong** — the build respects an explicit override
   and never fetches over it. Point it at a *built binary* (not a source
   checkout), or unset it to auto-fetch:

   ```bash
   HELPER_BIN=/path/to/helper/target/release/wispr-flow-linux-helper \
     ./build.sh --build deb
   ```

See the helper section of [building.md](building.md#the-clean-room-helper-prebuilt-with-a-helper_bin-override).

## More

Curious which setups are actually validated, and which are only wired through?
That's all spelled out in [compatibility.md](compatibility.md), and the design
rationale behind these calls is in [decisions.md](decisions.md).
