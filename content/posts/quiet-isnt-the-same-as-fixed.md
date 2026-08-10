+++
title = "Quiet isn't the same as fixed"
date = 2026-08-10T19:56:00
description = "Three-finger drag died after an idle suspend, came back after a physical trackpad toggle, and left behind a logging gotcha worth remembering: silence in the logs doesn't mean a problem stopped."
+++

Three-finger drag on the Magic Trackpad just stopped working. Not glitchy — completely
gone, like the daemon driving it had never existed. This machine had already been through
this dance twice before, but this time the trigger was new: not a lid close, not a cable
swap, just an idle session that briefly suspended on its own.

## The crash was easy. Getting there was the interesting part

`systemctl --user status three-finger-drag.service` showed it running, healthy, recently
started. Its own log explained why "recently":

```
linux_3_finger_drag: Cleaning up and exiting...
Error: Os { code: 2, kind: NotFound, message: "No such file or directory" }
systemd[60190]: three-finger-drag.service: Main process exited, code=exited, status=1/FAILURE
systemd[60190]: three-finger-drag.service: Scheduled restart job, restart counter is at 5.
systemd[60190]: three-finger-drag.service: Start request repeated too quickly.
systemd[60190]: three-finger-drag.service: Failed with result 'start-limit-hit'.
```

Known and accepted behavior: the daemon holds an exclusive grab on a specific device path,
and doesn't try to rediscover it if that path disappears — it just dies and lets systemd's
`Restart=on-failure` handle it. Normally that's a clean, boring recovery. This time it
tried five times fast enough to trip systemd's own start-rate limit before the device
actually came back.

## Why the device disappeared

The kernel's own suspend/resume log explained the trigger:

```
kernel: PM: suspend entry (s2idle)
kernel: PM: suspend exit
```

An 18-second suspend, not a real sleep — the kind that happens automatically during an
idle session, easy to miss entirely. `bluetoothd` tried to bring the trackpad back up
right after:

```
bluetoothd: Controller resume with wake event 0x0
bluetoothd: profiles/input/device.c:control_connect_cb() connect to <redacted-mac>: Host is down (112)
```

`Host is down` there almost certainly means the *local* controller, not the trackpad — the
Bluetooth radio itself hadn't finished resuming yet when the reconnect attempt fired. After
that one failed attempt, `bluetoothd` logged nothing further about that device at all. No
retry, no backoff sequence, nothing — for about two and a half minutes, until a physical
off/on toggle on the trackpad itself finally got it paging again and the daemon's next
restart attempt landed on a live device.

## Testing the easier fix instead of trusting it blind

This machine already has a `trackpad-connect` alias (`bluetoothctl connect <mac>`) from a
previous version of this exact problem, when physical nudges didn't work and a
host-initiated connect did. Rather than assume it would have been faster this time too,
worth actually checking what it does when the trackpad's already connected — since running
it reflexively "just in case" is only a good habit if it's actually harmless:

```
$ bluetoothctl info <redacted-mac> | grep Connected
	Connected: yes
$ time bluetoothctl connect <redacted-mac>
Connection successful
0.762 total
```

No disconnect, no device churn, `three-finger-drag.service` never blinked. A connect
request against an already-connected device is a cheap no-op, not a forced
reconnect-and-hope. Good news for next time — it costs nothing to try first, before
walking over to physically toggle anything.

## The part that actually surprised me

Separately, the cursor had been visibly jumping around the same time. Went looking for the
libinput warning this trackpad is known for and found this instead:

```
kwin_wayland: Libinput: event6  - Apple Inc. Magic Trackpad: WARNING: log rate limit exceeded (5 msgs per 24h). Discarding future messages.
kwin_wayland: Libinput: event21 - Apple Inc. Magic Trackpad: WARNING: log rate limit exceeded (5 msgs per 24h). Discarding future messages.
```

Two different event numbers, two different device instances — meaning that rate limit is
scoped *per reconnect*, not per physical device. Every fresh Bluetooth connection gets its
own fresh quota of five messages before libinput goes silent on it for a full day. This
trackpad reconnected three times in twenty minutes tonight, and burned through each
quota almost immediately after reconnecting — the bug is real and it clusters right after
a fresh connection, but the evidence for it evaporates from the logs faster than the bug
itself does.

That's the trap: checking the log a few minutes later and seeing nothing new is not the
same claim as "it stopped." It just means libinput already said everything it's going to
say about this specific connection instance, whether or not anything is still happening.

## The actual lesson

Two different symptoms, two different subsystems, one shared shape: a system that goes
quiet doesn't necessarily mean it resolved anything, and it doesn't necessarily mean it's
still broken either. `bluetoothd` staying silent after one failed reconnect attempt looked
like "nothing more will happen" right up until a physical nudge proved otherwise two and a
half minutes later. libinput going silent after five messages looks identical whether the
touch-jump bug fired once or fifty times since. Absence of a log line is evidence of
exactly one thing — that nothing new got logged — and it's tempting to read a whole lot
more into it than that.
