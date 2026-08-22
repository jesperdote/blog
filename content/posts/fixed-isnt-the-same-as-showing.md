+++
title = "Fixed isn't the same as showing"
date = 2026-08-23T00:46:05
description = "A Dell DisplayLink dock refused to show a picture on the second monitor. Getting there meant finding and fixing three genuinely real bugs - a kernel crash, a mangled EDID, a stale session lock - and none of them, on their own, were the thing that fixed it."
+++

New dock, new monitor, plug it in. Nothing shows up. That should have been a
five-minute problem. It took most of a night, three confirmed real bugs, one
accidental detour into installing an X11 session that hadn't existed on this
machine in the first place, and the actual fix turned out to be the least
interesting command of the entire night.

## The dock that isn't a cable

First surprise: this dock (a Dell Universal Dock D6000) doesn't pass video
through natively. It's DisplayLink - the monitor's "cable" is actually a USB
device pretending to be a display, compressed and shipped over USB by a
proprietary driver stack. Confirmed via `lsusb`:

```
Bus 002 Device 016: ID 17e9:6006 DisplayLink Dell Universal Dock D6000
```

Installed the driver (`evdi` + `displaylink`, both AUR-only - DisplayLink's
Linux support has never made it into the official repos), enabled the
service. Service running, kernel module loaded, dock detected. Still no
second display, anywhere.

## A crash hiding behind a permission error

First instinct: check `dmesg`. It failed outright:

```
dmesg: read kernel buffer failed: Operation not permitted
```

Not "no output" - permission denied, silently, the whole time. Every earlier
`dmesg | grep evdi` returning nothing hadn't been evidence of anything;
`dmesg_restrict` had just been eating the command before it ever ran.
`journalctl -k` doesn't have that restriction, and it told a completely
different story: a real kernel warning, firing exactly when the dock's USB-C
controller re-enumerated:

```
WARNING: CPU: 6 PID: 242179 at drivers/usb/typec/class.c:311 typec_altmode_update_active+0x63/0x110 [typec]
Call Trace:
 ucsi_altmode_update_active+0x9e/0xf0 [typec_ucsi]
 ucsi_check_altmodes+0x4d/0xa0 [typec_ucsi]
```

DisplayPort Alt Mode - the mechanism that lets a USB-C port carry video at
all - negotiating and hitting a bug in the kernel's own UCSI code. Confirmed
by switching from the LTS kernel to the current mainline build: dock
reconnects went from crash-then-restart to clean, no warning, every time.
Real bug, cleanly reproduced, cleanly fixed. Still no display.

## The monitor that says it isn't one

Kernel side clean now, `evdi` finally created a virtual output -
`card0-DVI-I-1`, connected, enabled. Progress. Except its only reported mode
was `640x480` - the universal "I have no idea what this display actually
supports" fallback. Pulled the raw EDID and decoded it:

```
Display Product Name: 'No Monitor'
Manufacturer: DLM
Serial Number: USB_6006-1802
```

`DLM` is DisplayLink Manager's own vendor code. This wasn't a corrupted read
of the real monitor - it was DisplayLink's software handing over its own
placeholder EDID, literally named "No Monitor," because it never managed to
read the actual one over the video link. Every layer beneath this - USB,
kernel, `evdi` itself - was doing its job correctly and handing the problem
one layer further up, to a piece of software that was quietly giving up and
substituting a lie that happened to be syntactically valid.

## The side quest nobody asked for

Along the way: tried `displaylink-connect`, a companion tool that promised
to fix exactly this class of problem automatically. Read its actual source
before installing it:

```bash
if [ -z "$DISPLAY" ]; then
    echo "User $USER don't use X11" >&2
    exit 11
fi
```

Built entirely on `xrandr`. This machine runs Wayland. The tool would have
run, printed that exact line to a log nobody was watching, and done nothing,
forever. Checking what a script actually does before running it as root
keeps paying for itself.

Chasing an X11 comparison test anyway surfaced something stranger: there
was no X11 session on this system at all. Not disabled - never installed.
`plasma-workspace` ships `startplasma-x11` but not the `.desktop` file that
makes a login manager offer it as an option, and no package in any enabled
repo provides one for Plasma 6. Wrote one by hand to test with. That test
went badly enough to abandon and delete the file again without a clean
answer either way - sometimes a diagnostic just costs you the diagnostic.

Also, entirely separately: installing `dkms`/`base-devel`/`linux-headers`
as a reasonably-sounds-related troubleshooting step pulled in plain Arch's
`linux-headers` instead of this system's actual `linux-cachyos-headers`,
producing a genuinely alarming-looking error -

```
ERROR: Missing 7.1.8-arch1-3 kernel modules tree for module evdi/1.15.0.
```

- for a kernel that was never running and never would be. Scary output,
zero actual effect, one `pacman -R` to clean up later.

## The fix

With the crash fixed and a real EDID finally flowing, the monitor showed
*something* - and KDE fully believed it was a working display: task
switching moved windows onto it, `kscreen-doctor` reported a full mode list,
a real resolution actively set. The panel itself stayed dark. That's a
different kind of bug than everything before it - not "not detected," but
"detected, configured, and still not painted."

The actual cause was sitting in KWin's own log the whole time:

```
atomic commit failed: Permission denied
drmModeListLessees() failed: Permission denied
Applying output configuration failed!
```

KWin had been restarted several times over the course of the night -
reboots, kernel switches, service restarts - and something from an earlier
session was still holding the GPU's exclusive modesetting lock, silently
rejecting every attempt from the *current* session to actually push pixels
to the screen. Nothing in that error mentioned DisplayLink, EDID, or
anything from the previous four hours. The fix was logging out and logging
back in.

## Three real bugs, one non-event

The kernel crash was real, and fixing it mattered - it's the reason the
dock can reconnect at all now without needing a full reboot. The placeholder
EDID was real, and diagnosing it correctly is the reason the monitor now
gets its actual native resolution instead of being stuck at 640x480
forever. Both were genuine, confirmed, fixed. Neither one, on its own, was
why the screen was still black afterward.

The thing that actually mattered - a stale DRM master lock from a KWin
session that no longer existed - left no trace anywhere upstream of it.
Every earlier fix was necessary and none of them were sufficient, and the
one that was sufficient wasn't a fix to anything this investigation had
found. Confirming a bug is real and confirming it's *the* bug are two
different claims, and a long night of debugging can make the first one feel
enough like the second that it's easy to stop checking.
