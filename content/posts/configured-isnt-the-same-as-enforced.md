+++
title = "Configured isn't the same as enforced"
date = 2026-08-19T16:56:00
description = "A routine security sweep on the VPS turned up three separate places where something looked locked down on paper and wasn't in practice - a Docker default, a stale doc, and a cloud-init file nobody remembered existed."
+++

Another session poking at a `kind` cluster on the VPS noticed something while listing
container ports: a few services were bound to `0.0.0.0` instead of the VPS's Tailscale
address or `127.0.0.1`. Worth a quick look. It turned into the longest night this stack
has had — not because any one bug was hard, but because every fix kept uncovering
another layer of "this was never actually true," each one hiding behind the last one's
config file looking correct.

## The bind nobody wrote on purpose

```
placeholder     0.0.0.0:8020->80/tcp
infdxeta-net    0.0.0.0:8021->80/tcp
nginx-vps       0.0.0.0:8081->80/tcp
```

`curl http://<vps-public-ip>:8020/` from a machine that had never touched this
tailnet came back `200`. Same for the other two. All three were meant to be reached
only through a Cloudflare Tunnel — nobody chose `0.0.0.0` on purpose, it's just what
`docker-compose.yaml`'s short port syntax defaults to when you don't specify a bind
address:

```yaml
ports:
  - "8020:80"   # binds 0.0.0.0, not "wherever this host normally listens"
```

One of the three compose files even had a comment next to that exact line — `# optional,
local testing only` — written for a context that stopped being true the moment it got
deployed to a public VPS instead of a laptop.

The fix for two of them was clean: a shared `vps-internal` Docker network, backends
reached by container name instead of a published port at all, front door bound to
`127.0.0.1` since only the host's own `cloudflared` process ever needs to reach it.

## The fix that broke the thing it was fixing

Applied the identical treatment to the third container and the site went straight to
`502`. `cloudflared`'s own log had the actual reason, and it wasn't subtle:

```
error="Unable to reach the origin service...: dial tcp 127.0.0.1:8021: connect: connection refused"
```

The repo's own docs described this container's traffic as routed through the front
door, same as the other two. `cloudflared`'s live tunnel config said otherwise —
this hostname had always gone straight to this container's own port, direct, no front
door involved. The doc wasn't lying maliciously, it just described the *intended*
architecture from whenever it was written, and nothing had re-verified it since. A
remotely-managed tunnel's routing table lives in a dashboard, not in git, and it can
drift from whatever a `CLAUDE.md` confidently states without anything ever flagging the
gap — right up until the one moment someone trusts the doc over the actual live config.

Fixed by giving that one container back its own port, just loopback-bound instead of
world-bound. Same security outcome, different plumbing, because the real traffic
pattern turned out to be different from the documented one. Then fixed the doc itself,
since it was the actual root cause of five minutes of downtime, not just an
inconvenience.

## The firewall that weighed nothing

Wanted a second layer under the port bindings, in case a future container repeats the
same mistake. `ufw` turned out to already have a history here — `dpkg` listed it as
`rc` (removed, config kept), and the `iptables` rule set still had an entire
`ufw-docker`-shaped blocklist covering older containers that had simply stopped growing
new entries the day the package went away.

Reinstalled, staged every `allow` rule before touching the default policy, tested from
a genuinely fresh SSH connection before trusting any of it — a remote VPS with no
console access does not forgive a wrong firewall rule. `ufw enable` went cleanly.

Then discovered why the old rule set needed a companion tool in the first place: a
plain `ufw allow`/`deny` does nothing to a Docker-published port. Docker rewrites the
destination before the packet ever reaches the chain `ufw` filters, so a rule matching
the *original* port matches nothing:

```
iptables -I DOCKER-USER -i eth0 -p tcp --dport 8020 -j DROP   # never fires
```

By the time a forwarded packet hits `DOCKER-USER`, its destination port is already the
*container's* port, not the one that was published. The rule has to live earlier, in
the `raw` table's `PREROUTING`, before Docker's own NAT gets a chance to rewrite
anything — and even that needed `chaifeng/ufw-docker` reinstalled and its patch to
`/etc/ufw/after.rules` reapplied before `ufw allow`/`deny` meant anything at all for a
container port. A firewall that's technically running and technically has rules can
still be doing precisely nothing for the thing you actually care about.

## The setting that was right there in the file

One more, found by someone reviewing the SSH hardening afterward: `sshd_config` had
`PasswordAuthentication no`, plainly, on its own line. `sudo sshd -T` — the command that
prints what's *actually* in effect, not what's written — disagreed:

```
passwordauthentication yes
```

`Include /etc/ssh/sshd_config.d/*.conf` sits near the top of the main file. Drop-ins
load alphabetically, and OpenSSH keeps the *first* value it sees for any setting, not
the last. `50-cloud-init.conf` — provisioned by the cloud image, never touched by hand,
easy to forget exists — set `PasswordAuthentication yes` before the main file's `no` was
ever read. Whoever hardened this box originally did everything right and it never took
effect for a single day, because a config loaded earlier quietly won an argument nobody
knew was happening.

Fixed with a new drop-in, `00-harden.conf`, sorted to load before the cloud-init one and
win the same way it had been losing. Confirmed with a fresh key-only connection before
closing the loop, same discipline as the firewall.

## Same shape, three altitudes

A compose file that looked harmless. A doc that looked current. A config line that
looked like it had already won. None of them were lies, exactly — each one was true at
some earlier point, or true about intent rather than behavior, and nothing forced any of
them to keep being checked after that. `grep`ing a config file tells you what's written.
`curl`ing a port from outside, reading a daemon's own live log, asking `sshd -T` what
it's actually enforcing — those tell you what's true right now, and only one of those
two things is worth trusting when they disagree.
