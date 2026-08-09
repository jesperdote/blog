+++
title = "The boot logo with no actual instructions"
date = 2026-08-07
description = "Modding a ThinkPad's boot logo turned out to be well-documented everywhere except the one step that actually mattered."
+++

Back in May, I decided my ThinkPad T14 Gen 2i should boot into something other than the
stock Lenovo logo. Not a huge ask, in theory — ThinkPad boot logo mods have been a forum
topic for over a decade. In practice, every source I found either didn't apply to my
exact board or stopped short of actually explaining the part that mattered.

## A tool that didn't know my ThinkPad existed

First stop was [lenovo-logo-changer](https://github.com/chnzzh/lenovo-logo-changer), a
Rust tool that reads a UEFI variable, drops your image into the ESP partition, and flips
a flag so the DXE firmware displays it at boot — no BIOS file editing required, if your
board supports it. Its device list includes "ThinkPad T14P Gen 1." Mine is a T14 Gen 2i.
Different board, different UEFI implementation, no dice.

## A forum thread that was actually just a cry for help

Next was a [badcaps.net thread](https://www.badcaps.net/forum/troubleshooting-hardware-devices-and-electronics-theory/troubleshooting-laptops-tablets-and-mobile-devices/bios-requests-only/3724979-request-thinkpad-t14-gen-1-20s1-custom-boot-logo-modification)
titled like it'd have the answer: "Request: ThinkPad T14 Gen 1 (20S1) custom boot logo
modification." Read the whole thing hoping for steps. It's three posts: someone asking
how to do it, a moderator replying "post backup bios," and the original poster still
stuck — *"idk how to put the new image in the bios with no brick frv my laptop."* Same
question I had. No answer in sight.

## The thread that actually said something useful

The one that broke it open wasn't even about adding a logo — it was
[someone trying to remove one](https://www.badcaps.net/forum/troubleshooting-hardware-devices-and-electronics-theory/troubleshooting-laptops-tablets-and-mobile-devices/bios-requests-only/103571-how-do-i-remove-custom-logo-from-t590)
from a T590. Buried in their own reply to themselves: download the BIOS update package,
extract it without running the update, launch `WINUPTP.exe` from the extracted folder,
and read the prompts carefully — it asks outright whether you want to keep, replace, or
remove a custom logo.

That's the actual mechanism. Lenovo's own official flashing tool has custom-logo
handling built in. Nobody has to patch a BIOS image by hand or touch a hex editor. You
just need to know the utility is even looking for one.

## Getting the exact file

Boot logo support lives in the platform firmware, so a mismatched BIOS version is a real
risk, not just an inconvenience. Lenovo keys everything to your exact board — mine
reports as:

```
$ cat /sys/class/dmi/id/bios_version /sys/class/dmi/id/product_name
N34ET69W (1.69)
ThinkPad T14 Gen 2i
```

`N34E` is the BIOS ID prefix Lenovo uses to file downloads under, and it's what actually
matters — not the marketing name. Lenovo Support's page for this board bundles the
utility as both a Windows `.exe` (`n35uj35w.exe`) and a bootable ISO
(`n35ur35w.iso`) for booting from USB instead of running it inside Windows. I went with
the ISO.

## No spec sheet for the image itself

The extracted package ships with no default logo file sitting there waiting to be
swapped — the custom-logo prompt only shows up at all once *something* has been placed
where the utility looks for it. And there's no documented size, resolution, or format
requirement anywhere I could find. Not in the utility, not in either forum thread. I
found the right dimensions by trial and error: try a size, boot the flash utility, see if
it accepts it or complains, adjust, repeat.

## The logo itself

I'm a cat person — two of them, actually — so the mod idea was never going to be
anything else. The actual artwork is a Gemini generation: Lenovo's "ThinkPad" wordmark
styling, with "Pad" swapped for "Cat," one cat draped over the top of the lettering and
another curled underneath it. One for each of mine.

![The ThinkCat logo](https://infdxeta.info/blog/thinkcat-logo.jpg)

[Download the full-resolution image](https://raw.githubusercontent.com/jesperdote/blog/main/static/thinkcat-logo.jpg)
if you want it for your own ThinkPad — no promises it'll fit whatever your particular
model's flash utility expects, given there's no spec sheet for that either.

Flashed it, rebooted, and there it was — full Lenovo splash screen, logo centered on
black, "To interrupt normal startup, press Enter" underneath, just with a cat where the
logo used to be. Small thing, but there's something satisfying about a laptop that
nobody wrote the instructions for anyway.

<video src="https://infdxeta.info/blog/thinkcat-boot.mp4" controls width="100%"></video>
