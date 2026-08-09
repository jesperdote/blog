+++
title = "The trackpad's triple life"
date = 2026-08-07T16:59:44
description = "Three-finger drag on a Magic Trackpad worked perfectly over Bluetooth, then vanished the instant it switched to USB - same physical device, two completely different identities."
+++

Three-finger drag on my Magic Trackpad worked fine — right up until I plugged the cable
in to charge it. The instant it went from Bluetooth to USB, the gesture just stopped.
Not glitchy, not laggy — gone, like the daemon driving it had never existed.

## One device, three identities

The trackpad's firmware treats Bluetooth and USB as genuinely different devices, not two
transports for the same one. Over Bluetooth it shows up with vendor `004c` (Apple's
Bluetooth SIG company ID) and its own MAC-derived serial. Plug the cable in and Bluetooth
pairing itself refuses to complete — the trackpad falls back to acting as a wired USB HID
device instead, this time under vendor `05ac` (Apple's actual USB vendor ID) with a
completely different serial.

That would already be enough to break anything hardcoded to one identity. But USB adds a
second twist: the trackpad enumerates as *two* separate `/dev/input/eventN` nodes, both
claiming the exact same vendor, product, and serial:

```
I: Bus=0003 Vendor=05ac Product=0265 Version=0110
N: Name="Apple Inc. Magic Trackpad"
U: Uniq=<redacted-serial>
H: Handlers=event20 mouse7

I: Bus=0003 Vendor=05ac Product=0265 Version=0110
N: Name="Apple Inc. Magic Trackpad"
U: Uniq=<redacted-serial>
H: Handlers=event21 mouse8
```

Identical on paper. `libinput list-devices` settled which one actually matters — only
`event21` shows up there, meaning only one of the two HID interfaces is the real
multitouch device; the other is present but unused. Nothing in the sysfs attributes I'd
normally reach for (`uniq`, `id/vendor`, `id/product`) tells them apart. The only
difference is buried in the USB interface number.

## A udev rule that looked right and could never match

The existing setup already had a udev rule keying a stable symlink off the trackpad's
Bluetooth identity, since Bluetooth `eventN` numbers reshuffle on every reconnect. Adding
USB support looked like the same trick twice — match the USB identity, add a
`KERNELS=="*:1.1"` clause to pick out the second interface, done:

```
SUBSYSTEM=="input", KERNEL=="event*", KERNELS=="*:1.1", \
  ATTRS{uniq}=="<redacted-serial>", ATTRS{id/vendor}=="05ac", ATTRS{id/product}=="0265", \
  SYMLINK+="input/magic-trackpad-event"
```

It deployed cleanly, `udevadm control --reload-rules` ran without complaint, and the
symlink still never appeared. No error anywhere — the rule simply behaved as if it wasn't
there.

The bug was in how udev walks the device tree. `ATTRS{}` and `KERNELS{}` are both
"ancestor-search" keys — for a single rule, udev doesn't match each key against whatever
ancestor happens to have it; it walks up the chain one device at a time and requires *all*
such keys to match against that *same* device before moving to the next ancestor. `uniq`,
`id/vendor`, and `id/product` live on the input device itself. The `:1.1` interface
number lives several levels further up, on the USB interface node. There is no single
device in the chain that has both — so the rule was unsatisfiable from the moment I wrote
it, and udev had no way to say so.

The actual fix was to stop trying to match the ancestor at all and use a property that's
already been resolved onto the device itself:

```
SUBSYSTEM=="input", KERNEL=="event*", ENV{ID_USB_INTERFACE_NUM}=="01", \
  ATTRS{uniq}=="<redacted-serial>", ATTRS{id/vendor}=="05ac", ATTRS{id/product}=="0265", \
  SYMLINK+="input/magic-trackpad-event"
```

`ID_USB_INTERFACE_NUM` is set earlier in udev's own rule chain (it's the same property
behind the `-if01-` suffix on the default `/dev/input/by-id/` symlink), so by the time my
rule runs it's just a flat property on the current device — no ancestor walk, no
matching-level mismatch. `ENV{}` and `ATTRS{}` can coexist in one rule precisely because
one of them doesn't care about the device tree at all.

## The second bug was in the deploy script, not the rule

Fixing the rule and re-running the install script didn't actually help the first time,
which was its own small lesson. The install script's idempotency check for these udev
rules was `if [[ ! -f "$dest" ]]` — install once, then leave alone forever. That's a fine
default for a file that's static after creation. It's the wrong default for a *template*
that gets edited later: the file already existed from the original Bluetooth-only setup,
so the script quietly skipped reinstalling it, and my supposedly-fixed rule never reached
the machine at all. Confusing failure mode — a code fix that produces zero observable
change looks a lot like a code fix that didn't work.

Swapped it for a content comparison instead of an existence check — render the template
to a temp file, `cmp` against what's installed, only touch it if they actually differ.
Existence-based idempotency and content-based idempotency look interchangeable right up
until the source file changes after the first install.

## The lesson

Two failures here had the same shape: something that looked correctly scoped
(`KERNELS=="*:1.1"`) or correctly idempotent (`! -f "$dest"`) turned out to be checking
the wrong thing entirely, and both failed silently rather than loudly. No error, no
warning — just a symlink that never showed up and a file that never got rewritten. When
a fix produces no visible change, the instinct is to assume the fix was wrong. Sometimes
it's that the fix never actually ran.
