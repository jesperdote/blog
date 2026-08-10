+++
title = "The shutdown that never finished"
date = 2026-08-10T21:00:00
description = "A reboot from the command line just kept spinning. The machine hadn't crashed - it had spent three days quietly filling up swap, and shutdown finally choked on the bill."
+++

Tried to restart from the command line and it just sat there. No error, no progress, the
kind of nothing that eventually forces you to hold the power button down. Rebooted clean
on the second try and went looking for what the first attempt had actually been doing
while it hung.

## The log doesn't lie, it just stops

`journalctl -b -2` (the boot that hung) reads completely normally almost all the way to
the end — services stopping, mounts unwinding, one by one, in order:

```
Stopped target Local File Systems.
Unmounting /boot...
Unmounting /home...
Unmounting /root...
Unmounting /run/docker/netns/default...
Unmounting /srv...
Unmounting /tmp...
Unmounting /var/cache...
Unmounting /var/lib/docker/rootfs/overlayfs/04423c875d74a9679a409f086ccec6fe95baece525401bc439ffa0052d27e2da...
Unmounting /var/tmp...
...
Unmounted /root.
Failed unmounting /var/cache.
Unmounted /home.
```

And then nothing. No more lines, ever, in that boot's log. Not a crash with a stack trace,
not a kernel panic — just a shutdown sequence that got partway through unmounting things
and stopped producing output. Whatever it was stuck on, it was stuck on it silently, which
is a worse kind of stuck to debug than one that at least complains.

## The number that explained it

`user.slice`'s own resource summary was sitting right there in the same log, a few lines
earlier, already answering the question before I'd finished asking it:

```
user.slice: Consumed 15h 13min 58.489s CPU time over 18h 58.092s wall clock time,
            12.4G memory peak, 9G memory swap peak
```

Eighteen hours of wall clock for fifteen hours of actual CPU work is already a machine
spending a lot of its life waiting on something. Nine gigabytes peaked in swap is the
something. This machine runs on `zram` — compressed RAM used as swap, not disk — and it
had been up for three straight days on suspend-only uptime, lid closed and opened
repeatedly, never once actually rebooted. Swap doesn't get reclaimed on its own just
because nothing's actively using those pages anymore; it sits there until something asks
for the memory back. Three days is enough time for a lot of idle pages to pile up with
nothing prompting a cleanup.

By the time shutdown tried to tear down a Docker overlayfs mount under that much swap
pressure, it's a plausible enough villain: unmounting can need to fault pages back in, and
faulting pages back in under 9G of swap backlog is exactly the kind of thing that goes
from "a bit slow" to "never finishes" without any single step being the one obvious
culprit.

## Chasing the number, not the symptom

The actual fix wasn't about Docker or unmounts at all — those were just where the bill
came due. The real lever is why swap usage climbs unbounded over multi-day uptime in the
first place, and that traces to `vm.swappiness`. CachyOS ships a sane general default:

```
# /usr/lib/sysctl.d/70-cachyos-settings.conf
vm.swappiness = 100
```

But `cat /proc/sys/vm/swappiness` at runtime read `150`, not `100`. Something was
overriding the distro's own default, and it wasn't another sysctl file — it was a udev
rule:

```
# /usr/lib/udev/rules.d/30-zram.rules
ACTION=="change", KERNEL=="zram0", ATTR{initstate}=="1", SYSCTL{vm.swappiness}="150"
```

This fires the moment the zram device actually initializes, which happens *after*
early-boot sysctl processing has already applied the `70-` file's `100`. Documented
reasoning in the rule's own comment: zram compression is cheap, so for a zram-backed swap
device it's genuinely better to prefer swapping anonymous pages over evicting page cache.
Reasonable advice for normal uptimes. Considerably less reasonable for a machine that
treats "reboot" as something that happens to other people.

## An override has to win the same race

Dropping a plain `sysctl.d` file with `vm.swappiness=100` in it would just get overwritten
again the next time zram initializes — same problem as trying to fix `nowatchdog` with a
sysctl file back when *that* got silently reset by a different CachyOS default. The fix
has to intercept the same event the original rule reacts to, not just run once at boot and
hope nothing touches it again:

```
# 31-zram-swappiness-override.rules
ACTION=="change", KERNEL=="zram0", ATTR{initstate}=="1", SYSCTL{vm.swappiness}="100"
```

Identical trigger, sorted to run after CachyOS's own rule by filename (`31-` after `30-`),
so its `SYSCTL{}` assignment is simply the one still standing when the dust settles.
Triggered it live without waiting for the next boot:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger --action=change --name-match=/dev/zram0
```

`cat /proc/sys/vm/swappiness` came back `100`. Not a guess this time — confirmed.

## The lesson

Nothing about this incident was mysterious in isolation. A hung shutdown, a high swap
peak, and an overridden sysctl are each individually boring facts. What made it worth
tracing all the way through is that the visible symptom — a reboot that wouldn't finish —
was three layers removed from the actual cause, and every layer in between looked
completely reasonable on its own. The zram rule is good advice. The swap number was just a
number until it was read next to the wall-clock-versus-CPU gap. And the shutdown hang
would have been very easy to blame on Docker and leave it there, since that's exactly
where the trail visibly went cold. Three days of quietly-accumulating debt doesn't send a
warning before the bill comes due — it just shows up disguised as whatever process happens
to be doing the unlucky work at the time.
