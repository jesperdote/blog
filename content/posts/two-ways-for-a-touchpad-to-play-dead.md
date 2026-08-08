+++
title = "Two ways for a touchpad to play dead"
date = 2026-08-07
+++

Closed the lid, came back a few minutes later, and the built-in touchpad was just gone.
Not laggy, not misbehaving — clicking did nothing, tapping did nothing, even the
TrackPoint nub was dead. Everything else about the machine had resumed fine. This one
took two separate bugs, stacked on top of each other, to actually explain.

## A device that's there but not there

First instinct: check if the kernel still sees it.

```
N: Name="Synaptics TM3471-030"
H: Handlers=event16 mouse3
```

Still listed. `evtest` opened `/dev/input/event16` without complaint, printed the full
capability report, sat there waiting for input. I tapped, clicked, dragged — fifteen
seconds of it — and got back exactly zero `Event:` lines. Not "wrong" events. None. The
device object was structurally present and behaviorally empty, which is a more annoying
failure mode than either "gone" or "broken" on their own, because every tool you'd reach
for to investigate a missing device reports that it's right there.

## The transport never came back

`dmesg` had the answer for *why* nothing was arriving, once I stopped grepping for the
device name and started grepping for the driver's own setup sequence:

```
psmouse serio1: synaptics: Trying to set up SMBus access
rmi4_smbus 16-002c: registering SMbus-connected sensor
```

Both lines appear exactly once in the whole boot log — at boot. Never again after a
resume. The touchpad runs over SMBus (a deliberate earlier fix,
`psmouse.synaptics_intertouch=1`, to get proper palm rejection instead of legacy PS/2
emulation), and whatever handshake sets that transport up apparently isn't part of the
normal suspend/resume dance. Confirmed it at the interrupt level too — `/proc/interrupts`
has a line for the touchpad's own 2D-sensor function, and its count didn't move by a
single tick across a solid six seconds of active tapping. The hardware interrupt simply
wasn't firing.

The fix for *that* layer is a driver rebind — kick `psmouse` off the PS/2 AUX port and
let it re-probe from scratch:

```bash
echo -n "serio1" | sudo tee /sys/bus/serio/drivers/psmouse/unbind
echo -n "serio1" | sudo tee /sys/bus/serio/drivers/psmouse/bind
```

`dmesg` immediately showed the SMBus handshake running again, a fresh input device
enumerating. Structurally, this worked.

## The fix that fixed nothing you could see

Tapped the touchpad again. Still nothing. Checked the interrupt counter again,
before-and-after across a clean six-second window this time. Flat. Same as before the
rebind.

That's a genuinely confusing result — the kernel log said the transport reinitialized
successfully, a brand new input device showed up with a brand new sysfs path, and yet
the symptom hadn't budged at all. Two possibilities: either the rebind hadn't actually
fixed the thing I thought it fixed, or something *else* entirely was eating the input
further downstream, after the kernel driver had already done its job correctly.

## The other culprit was hiding in a remapper

This machine runs Toshy for macOS-style keybindings, which works by having `xwaykeyz`
exclusively grab input devices and re-synthesize remapped events via `uinput`. "Exclusive
grab" here means `EVIOCGRAB` — an ioctl that tells the kernel *only deliver this device's
events to me*, full stop. Every other reader of that same `/dev/input/eventN`, including
`evtest`, including the compositor, gets nothing while the grab holds. It doesn't log
anything when it does this. It doesn't have to.

Checked what `xwaykeyz` actually had open:

```
lrwx------ 1 <user> <user> 64 ... 17 -> /dev/input/event16
lrwx------ 1 <user> <user> 64 ... 21 -> /dev/input/event3
```

There it was — the touchpad, held open by the remapper. Its own log around the resume
window told the rest of the story:

```
(EE) Retrying to initialize '/dev/input/event8' due to BrokenPipeError. Attempt 9 of 9.
(EE) Device may be in transition (KVM switch?), will retry on next event
```

`xwaykeyz` watches for hotplug events and tries to grab devices as they appear —
including, apparently, pointer devices, not just keyboards. If it tries mid-resume, while
the SMBus transport above is still mid-flap, it gets a broken pipe, retries nine times,
gives up, and leaves things in some half-grabbed state that never resolves on its own.
Two independent systems were racing on resume, and the loser was whichever one happened
to touch the device while it was still coming back to life.

## Logging out did what a driver rebind couldn't

Confirmed the theory the annoying way: a full logout/login cycle, which kills and
respawns Toshy's services fresh, brought the touchpad back immediately — even though the
driver-level rebind by itself, done earlier in the same session, had not. That's the
signature of a stuck userspace grab rather than a hardware problem: nothing about logging
out touches the kernel or the SMBus controller, but it does guarantee `xwaykeyz` gets a
clean slate.

## Making it permanent

Two bugs, two fixes, both needed, neither sufficient alone. Turned both into a single
`systemd-sleep` hook so resume fixes itself instead of requiring a diagnosis session each
time:

```bash
# $1=post $2=suspend, from systemd-logind
sleep 2  # let the SMBus controller actually finish resuming first
echo -n "serio1" > /sys/bus/serio/drivers/psmouse/unbind
echo -n "serio1" > /sys/bus/serio/drivers/psmouse/bind
systemctl --user restart toshy-config.service
```

(The real version resolves the logged-in user dynamically via `loginctl` rather than
assuming one, since `systemd-sleep` hooks always run as root.)

## The lesson

"The device is listed" and "the device works" are different claims, and every layer of
tooling I reached for first — `/proc/bus/input/devices`, `evtest` opening the node
successfully — only ever answers the first one. The second requires actually watching
data flow: interrupt counters ticking, `Event:` lines printing. And a stuck exclusive
grab is uniquely good at hiding, because the standard tool for checking "is this device
alive" is itself just another reader that a grab can silently starve. Two unrelated
systems each assuming the other had finished initializing, both wrong at once, and the
only visible symptom was a trackpad that looked exactly as dead either way.
