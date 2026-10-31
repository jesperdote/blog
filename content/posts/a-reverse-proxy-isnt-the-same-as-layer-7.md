+++
title = "A reverse proxy isn't the same as layer 7"
date = 2026-10-31T12:00:00
description = "I wanted my phone to reach my tailnet dashboards at home without switching Tailscale on. The fix was nginx, which I've used for years, doing something I didn't know it could: forwarding bytes it never reads."
+++

My homelab dashboards live on the tailnet. A VPS runs Caddy as a small front door for them:
the sports dashboard I've been building, Jenkins, Grafana. They have real HTTPS names and a
real certificate, but the names only resolve to a Tailscale address, so nothing outside the
tailnet can reach them. That's the point.

It also means that on my phone, every time I want to glance at something, I open the
Tailscale app, switch it on, wait, look, and switch it off again. Away from home that's fair.
At home it's silly: the phone is on the same Wi-Fi as a BananaPi that has been sitting in a
corner for five years, powered from the router's USB port, and is itself on the tailnet.

So: could the BananaPi be the phone's way in, at home only, with nothing exposed publicly?

## Two small jobs

It turned out to need two things on the BananaPi.

**Lie about one name.** A dnsmasq container answers DNS on the BananaPi's LAN address. For my
homelab names it answers with *its own* address. Everything else goes upstream as normal:

```
listen-address=192.168.1.53
bind-interfaces
no-resolv
server=1.1.1.1
server=192.168.1.1

# the tailnet-only names: send them here instead
address=/homelab.example.org/192.168.1.53
```

On the phone, the home Wi-Fi's DNS points at the BananaPi. Nothing else in the house changes,
because the router doesn't hand that DNS out to anyone.

**Pass the connection along without opening it.** This was the part I didn't know about. I've
used nginx as a reverse proxy for years, always the same shape: a `server` block inside
`http {}`, `location /something/`, `proxy_pass http://...`, certificates on nginx. Here I
didn't want any of that. I didn't want the BananaPi to hold a certificate, and I didn't want it
to see the traffic at all. I just wanted it to take whatever arrived on port 443 and hand it to
Caddy over Tailscale.

nginx has a block for exactly that:

```nginx
stream {
    server {
        listen 192.168.1.53:443;
        proxy_pass 100.64.0.10:443;   # Caddy, over Tailscale
    }
}
```

No `http {}` at all. That's the whole config.

## What actually happens

```
 phone ──① DNS "dashboards.homelab.example.org?"──► BananaPi dnsmasq
       ◄──── "192.168.1.53" (itself) ──────────────
 phone ──② TCP :443 ──► BananaPi nginx (stream)
                          └──③ new TCP :443 ── Tailscale ──► VPS Caddy
 phone ◄════④ TLS handshake, real certificate, end to end ════► Caddy
 phone ◄════⑤ GET /dashboard/ … encrypted the whole way ══════► Caddy
```

The phone thinks it's talking to the dashboard host, and in every way that matters it is. The
TLS handshake runs between the phone and Caddy, *through* nginx, so the browser sees Caddy's
real certificate and a valid lock. Caddy decrypts, reads the path and routes it like any other
request. The BananaPi copies encrypted bytes in both directions and nothing more. It can tell
a connection happened and how big it was; it can't see a path, a page or a password.

## Layer 4, it turns out

That's the thing I learned. I'd been treating "reverse proxy" and "layer 7" as the same idea,
because every reverse proxy I'd ever written spoke HTTP. They aren't:

- Inside `http {}`, nginx is an HTTP proxy. It parses requests, so it can route by path, add
  headers and terminate TLS. That's layer 7.
- Inside `stream {}`, nginx sees TCP or UDP connections and nothing else. It routes by port,
  or by the name in the TLS handshake if you turn on `ssl_preread`, and forwards bytes. That's
  layer 4.

Both are reverse proxies: the client connects to nginx and nginx connects onward on its
behalf. The difference is how much nginx understands about what it's carrying. And `stream`
isn't new. It has been in open-source nginx since 1.9.0, back in 2015. I just never had a
reason to scroll past the `http` half of the docs.

## Things that went sideways

The first version had a firewall: only my phone's Wi-Fi address could use the gateway. Then I
asked myself who exactly I was keeping out. It's my own LAN, the router doesn't advertise this
DNS, and the only things behind it are dashboards I'm happy for my household to see (Jenkins
still has its own login). I took it out.

Then it didn't work, and for a while it looked like DNS was broken. It wasn't. I had typed the
BananaPi's address into the phone's *router* field instead of its DNS field. The BananaPi saw
plenty of traffic from the phone and not a single DNS query. Logging which ports the phone
actually hit is what gave it away.

## Where it stands

At home, the dashboards open on the phone with Tailscale off, using the same bookmarks and the
same certificate as before. Away from home, nothing changed: no Tailscale, no dashboards.

The trade-off is that the phone's DNS at home now depends on the BananaPi. If it hangs, every
name lookup on that phone fails until I set DNS back to automatic. Five years of uptime says I
can live with that. The more likely failure is the SD card, so the next small job is backing
it up.

Two containers, about twenty lines of config. The surprise wasn't how little it took. It was
finding out that a tool I use every week had a whole second mode I'd never opened.
