+++
title = "A hook that couldn't wake itself up"
date = 2026-08-09T00:06:26
description = "A systemd-sleep hook fixed the touchpad going dead on resume - and introduced an 87-second full-session freeze on every wake after that, the fix hiding inside the fix."
+++

Yesterday I fixed the touchpad going dead after resume — a `systemd-sleep` hook that
rebinds the driver and restarts the input remapper on every wake. Today I opened the lid
and the touchpad worked fine. The rest of the machine didn't. A few seconds of nothing —
no cursor, no response to a keypress, screen just sitting there — before it came back to
life.

## Slow is a story. Frozen is a different one

First instinct was "resume is just slow now," but `journalctl` said something more
specific:

```
23:49:59 kernel: PM: suspend exit
23:51:29 systemd[1]: user.slice: Unit now thawed.
```

Ninety seconds between the kernel finishing resume and `user.slice` — the cgroup holding
the entire desktop session, compositor included — actually thawing. That's not a slow
wake-up, that's the session sitting in a cgroup freezer the whole time, unable to run at
all. And it wasn't a one-off:

```
01:04:30 → 01:06:00   (90s)
01:06:14 → 01:07:44   (90s)
12:14:02 → 12:15:32   (90s)
23:49:59 → 23:51:29   (90s)
```

Every single resume that day, same gap, suspiciously close to a round number.

## It started exactly when the fix was installed

Before reaching for anything exotic, I checked resumes from earlier the same boot —
before yesterday's touchpad hook went in:

```
19:34:11.004 kernel: PM: suspend exit
19:34:11.009 systemd-sleep: Successfully thawed unit 'user.slice'.
```

Five milliseconds. That's what a normal thaw looks like. The 90-second version only shows
up after the hook was installed, on every resume since. Whatever this was, it wasn't
pre-existing — the fix for one bug had shipped with a second one riding along in it.

## Asking a frozen process to answer the phone

The hook's last step restarts the input remapper's service to clear a stale device grab:

```bash
sudo -u "$TARGET_USER" systemctl --user restart toshy-config.service
```

`systemd-sleep` hooks run in two phases: freeze `user.slice` *before* suspend, then run
every `post` hook, then — only once they've all returned — thaw it. But `user.slice`
doesn't just hold your terminal and browser. It holds your own user's systemd instance,
the thing that `systemctl --user` actually talks to. So this line is a hook that hasn't
been allowed to finish yet, asking a process that's still frozen to please restart a
service for it. Neither side can move: the call can't get a response until thaw, and thaw
won't happen until the call returns.

It's not a true deadlock — something eventually gives, which is why the machine comes
back after ~90 seconds instead of never — but for that entire window the fix for
yesterday's bug was the reason today's machine wouldn't respond to a keypress.

## Don't wait for the answer

The rebind itself talks directly to `/sys`, no user session involved, so it was never the
problem. Only the restart call needed to stop blocking:

```bash
setsid sudo -u "$TARGET_USER" systemctl --user restart toshy-config.service &
disown
```

Let the hook return immediately, let `user.slice` thaw on schedule, and let the restart
land a couple seconds later once there's actually something on the other end to answer
it. Checked the next resume: thawed in three seconds, and the service restart still shows
up in the log a moment after — just no longer gating anything.

## The lesson

The touchpad fix was correct and I'd tested it — the touchpad came back every time.
What I hadn't tested was *what state the machine was in while the fix ran*, and that's
exactly the detail a `systemd-sleep` hook can't afford to get wrong: it executes at a
specific, narrow point in the resume sequence, before the thing it's about to talk to is
guaranteed to be listening. A fix that works can still be broken, just along an axis you
didn't check.
