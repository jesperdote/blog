+++
title = "The freeze that lied about its alibi"
date = 2026-07-25
description = "A ThinkPad froze mid-session with zero log trail - traced through disabled kernel lockup watchdogs to a driver crash that had nowhere to leave a note."
+++

My ThinkPad had been freezing during active use. Not on suspend, not overnight — mid-session,
screen goes black, completely unresponsive, forced power-off required. The kind of bug that's
maddening precisely because the system leaves no note explaining itself.

## The logs said nothing happened

First move: `journalctl --list-boots`, then walk every prior boot looking for the usual
suspects — MCE (hardware errors), OOM-kills, GPU resets, thermal shutdowns. Nothing. Every
single boot in the history ended with a clean `systemd-logind: System is powering down` —
a normal, graceful shutdown sequence. Not one of them looked like a boot that had actually
frozen mid-session.

That's the part that should have been suspicious sooner. A machine that's genuinely freezing
doesn't usually leave behind a log that looks this tidy.

## Two separate things were gagging the kernel

It turned out the kernel's own lockup detectors — the mechanisms that notice when a CPU
or the whole system hangs and *say something about it* — were both disabled, for two
completely unrelated reasons stacked on top of each other:

1. **`nowatchdog`** was sitting in the kernel command line
   (`/etc/default/limine`). It wasn't part of any deliberate tuning — best guess, it got
   copied in incidentally during an unrelated touchpad-debugging session weeks earlier,
   during a run of `limine-update` calls that had nothing to do with lockup detection.
2. **CachyOS's own `/usr/lib/sysctl.d/70-cachyos-settings.conf`** ships
   `kernel.nmi_watchdog = 0` as a package default — a small performance/power tradeoff
   that re-disables the NMI hardlockup watchdog moments after boot, independently of
   whatever the kernel command line says.

Either one alone is enough to mean: if something hangs, nothing gets written down. Together,
they made a real, reproducible freeze look, after the fact, exactly like a clean shutdown.

Fix was mechanical once found: strip `nowatchdog` from the cmdline, and drop a
`99-nmi-watchdog-enable.conf` into `/etc/sysctl.d/` (sorts after CachyOS's `70-`, so it
wins) with `kernel.nmi_watchdog = 1`. Confirmed both `kernel.watchdog` and
`kernel.nmi_watchdog` reporting `1`, and moved on figuring I'd need to wait a while for
the next freeze to actually test whether this did anything.

It happened again that same day.

## The very next freeze told the whole story

This time, the kernel actually said something:

```
BUG: kernel NULL pointer dereference, address: 000000000000010e
...
RIP: 0010:raydium_i2c_irq+0x138/0x370 [raydium_i2c_ts]
...
kernel BUG at arch/x86/kernel/cet.c:133!
...
Fixing recursive fault but reboot is needed!
```

Reconstructing the timeline from the surrounding log lines: the machine went to sleep
(`s2idle`) after the screen locked. On resume, the Raydium touchscreen driver's interrupt
handler fired before its internal state had finished re-initializing post-resume, and
dereferenced a near-null pointer. That crash cascaded into a second, more serious fault —
a CET (control-flow-integrity) violation — while the kernel tried to unwind the dead
thread. And then the kernel told me, in as many words, that it had given up: *fixing
recursive fault but reboot is needed*.

Everything after that point was the system limping. Bluetooth and Wi-Fi actually
reconnected fine post-resume — which lined up with something I'd noticed earlier and
dismissed: a Bluetooth headset's audio had briefly cut out, then the headset reconnected,
and I'd assumed it was just Spotify being flaky. It wasn't Spotify. The kernel was already
mid-collapse; the network stack just happened to still be functional while the display
never came back.

## Not a unique snowflake

A quick search turned up other people hitting the identical crash signature —
`raydium_i2c_irq` NULL-pointer-dereferencing on resume from sleep. Known bug class, not
something specific to this exact laptop. There's a related upstream patch for a Raydium
resume issue floating around, but confirming it's actually present in this kernel build
felt like more effort than the alternative:

```
blacklist raydium_i2c_ts
```

One line in `/etc/modprobe.d/`, plus unloading the already-running module. Touchscreen
input on a laptop I mostly drive with a keyboard and trackpad was an easy trade for a
crash class that required a hard power-off every time it happened.

## The actual lesson

The freeze was never invisible because it was mysterious. It was invisible because two
completely unrelated decisions — one accidental, one a distro's own performance default —
happened to both point the same direction: *don't let the kernel report lockups*. Once
that stopped being true, the real bug surfaced on the very first occurrence afterward,
fully explained, in about four log lines.

If a "freeze" ever looks suspiciously clean in the logs afterward, that cleanliness is
itself the finding.
