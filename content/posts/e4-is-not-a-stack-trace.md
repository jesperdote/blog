+++
title = "E4 is not a stack trace"
date = 2026-08-17
description = "A washing machine started throwing E4 mid-cycle, and treating it like a service outage instead of an appliance fault was the only reason the actual cause got found."
+++

Washing machine died mid-cycle with a blinking "E4" on the display and a dead stop — drum
not spinning, water not draining, just the code and a beep every fifteen seconds like a
housekeeping alert nobody was going to read. Old habits kicked in immediately: this is an
outage, not a mystery, treat it like one.

## The first move was the wrong one, and it looked like it worked

Power cycle. Killed it at the breaker, waited ten seconds, brought it back — the appliance
equivalent of `systemctl restart` when you don't yet know what's broken. It came up clean,
no code on the display, accepted a new cycle, ran the fill stage without complaint. Every
instinct said "resolved."

At the 6-minute mark, same cycle, same spot: E4 again. A reboot that fixes the symptom for
six minutes and then reproduces exactly is worse than a reboot that does nothing — it means
the underlying condition is still there, and now there's a false "fixed it" logged.

## No logs, just a code with no line number

Went looking for whatever passed for logs. There aren't any — no history, no timestamp,
nothing but a blink pattern that resets the second you clear it. The manual had one line
for E4: "Water supply error." That's a stack trace with the file name and line number
stripped out, keeping only the exception class.

Fell back to the appliance-forum equivalent of a GitHub issue search — enough reports of E4
to establish the actual failure condition: the control board waits for the drum to reach
target water level within a fixed window, and if it doesn't, it aborts and throws E4. Not
"no water." A timeout on a level check. That distinction mattered for where to look next —
a fully closed valve would announce itself in the first thirty seconds, not six minutes in.

## The bottleneck was upstream, not at the drum

Pulled the inlet hoses off the back — both of them, hot and cold, in case the machine was
using a blend. No kinks, no visible damage, water pressure fine at the wall. Which meant
the restriction was between the hose and the tub, somewhere I couldn't see from the
outside.

The inlet valve has a mesh screen on the intake side — a filter, functionally identical to
a NIC's hardware checksum offload silently corrupting exactly enough traffic to fail a
health check without ever taking the link down. Pulled it: caked in fine grey sediment,
water still passing through it, just at a fraction of rated flow. Enough to fill the drum
eventually. Not enough to fill it inside the timeout window the control board was
enforcing.

## Cleaning the filter fixed the timing, not the symptom

Old toothbrush, a few minutes under the tap, screen went from grey to fully clear.
Reassembled, ran an empty test cycle with a stopwatch this time instead of just watching
for the code: fill stage completed in just under two minutes, comfortably inside whatever
window had been getting missed before. Three more full loads since, no recurrence.

## The habit that actually transferred

The generic first move — power cycle it — wasn't wrong to try, but treating it as
confirmation that things were fine was the actual mistake, the same way a service coming
back green right after a restart doesn't mean the incident's over. E4 was never "no
water." It was a health check with a deadline, failing because something upstream had
gotten slow instead of gone, and the code only ever told me which check failed, not why.
