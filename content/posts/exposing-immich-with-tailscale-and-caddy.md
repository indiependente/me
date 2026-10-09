---
title: "📸 Exposing Immich from a homelab with Tailscale and a small VPS"
date: 2026-10-09T09:20:45Z
toc: true
draft: false
description: "How I connected a public VPS to my home Immich instance using Tailscale and Caddy, with HTTPS terminating at TierHive."
tags:
  - self-hosting
  - homelab
  - immich
  - tailscale
  - caddy
  - vps
---

Getting a photo library online is one thing. Uploading a large video from outside your home network is another.

I was exposing my home `Immich` instance through `Cloudflare Tunnel`, but a large-video upload ran into Cloudflare's proxy upload-size restriction. I wanted to keep the photos at home while changing the path used to reach them.

![Ron Swanson from Parks and Recreation throws a computer into a dumpster](https://media1.tenor.com/m/uxmr2FhQHSsAAAAd/ron-swanson-throw-out-computer.gif)

The solution: **a small VPS as the public entry point, `Tailscale` to reach the homelab, and `Caddy` to forward requests to `Immich`**. I used `TierHive`, which provides a managed `HAProxy` frontend and HTTPS certificates alongside its NAT VPS instances.

This is the setup that worked for me, including the certificate issue I ran into and what I still haven't established about upload speed.

## What does the traffic path look like?

```text
Browser or Immich app
  → TierHive HAProxy (public HTTPS)
  → Caddy on the VPS (private HTTP)
  → Tailscale subnet router at home
  → Immich (LAN HTTP, port 2283)
```

`Cloudflare` still hosts the DNS record, but **DNS only**: it no longer proxies the HTTP traffic for this hostname.

There are three separate connections here:

1. The client connects to `TierHive` over HTTPS. The certificate lives at the managed frontend, not on the VPS.
2. The frontend forwards requests to `Caddy` on the VPS's private address over HTTP.
3. `Caddy` reaches the home network through an encrypted `Tailscale` connection, then the subnet router forwards to `Immich` over LAN HTTP.

This is not end-to-end TLS to `Immich`: the provider terminates HTTPS and can see the traffic. The VPS also becomes part of the trust boundary.

The same relay pattern can work with another provider, but a VPS with its own public IP would need a different listener and certificate configuration.

## What do we need?

My relay runs `Debian 13`, with **1 GB RAM and 5 GB NVMe storage**. `Caddy` and `Tailscale` are installed as native packages; there is no Docker stack on this VPS. The photo library stays at home.

Before starting, we need:

- A working `Immich` instance on the home network.
- An existing `Tailscale` subnet router advertising the home LAN, with its route approved in the admin console.
- A VPS with `Caddy` and `Tailscale` installed (package instructions are linked at the end).
- A hostname we can configure in DNS and in the provider's managed frontend.

`Immich` itself does not need `Tailscale` installed in this setup: the subnet router provides the route to its LAN address.

All addresses below are **examples**, not my live infrastructure. `example.com` is reserved for documentation, and `203.0.113.10` belongs to the documentation-only `TEST-NET-3` range. The private addresses are illustrative too:

| Component | Example value |
| --- | --- |
| Public hostname | `immich.example.com` |
| Managed HAProxy public IP | `203.0.113.10` |
| VPS private IP | `10.0.0.2` |
| Immich LAN address | `192.168.1.50:2283` |

Use the actual values from your provider and home network. In particular, the public frontend IP, private VPS IP and forwarded SSH endpoint are different things.

## 1. Give the relay access to Immich, not the whole LAN

A public-facing machine should not have unrestricted access to the homelab. The relay needs one destination: **the `Immich` host on TCP port `2283`**.

First, define a tag and its access rule in the `Tailscale` policy. This is an excerpt to merge into the existing policy, not a replacement for it:

```json
{
  "tagOwners": {
    "tag:immich-relay": ["autogroup:admin"]
  },
  "grants": [
    {
      "src": ["tag:immich-relay"],
      "dst": ["192.168.1.50/32"],
      "ip": ["tcp:2283"]
    }
  ],
  "tests": [
    {
      "src": "tag:immich-relay",
      "proto": "tcp",
      "accept": ["192.168.1.50:2283"],
      "deny": [
        "192.168.1.50:22",
        "192.168.1.50:443",
        "192.168.1.10:8006"
      ]
    }
  ]
}
```

**Rules are additive**: adding this narrow grant does not cancel an existing allow-all rule. Remove or narrow any broader grant or ACL that also applies to the relay, preserving the access other devices need. Run the policy tests in the editor before saving; the `deny` entries are assertions, not firewall rules.

Then join the new VPS to the tailnet:

```bash
sudo tailscale up \
  --hostname=immich-relay \
  --advertise-tags=tag:immich-relay \
  --accept-routes=true \
  --accept-dns=false \
  --ssh=false
```

`--accept-routes=true` lets the VPS use the approved home subnet route. `--ssh=false` disables Tailscale's built-in SSH feature, not the ordinary OpenSSH server.

This command is for a new relay; don't rerun it blindly on an already configured machine.

Before adding a reverse proxy, check the backend from the VPS:

```bash
ip route get 192.168.1.50
curl -sS --connect-timeout 10 --max-time 20 \
  http://192.168.1.50:2283/api/server/ping
```

In my setup the route used `tailscale0`, and the API returned `{"res":"pong"}`. If this fails, check route approval and the access policy before changing the public DNS record.

## 2. Configure Caddy on the private VPS address

Because public TLS terminates upstream, `Caddy` listens on HTTP. The `http://` prefix is deliberate: we are not asking this listener to obtain another certificate.

Back up `/etc/caddy/Caddyfile` before replacing its contents:

```bash
sudo cp /etc/caddy/Caddyfile "/etc/caddy/Caddyfile.backup.$(date +%Y%m%d%H%M%S)"
```

The configuration I used, with example addresses substituted:

```caddyfile
http://immich.example.com {
    bind 10.0.0.2

    reverse_proxy 192.168.1.50:2283 {
        header_up X-Forwarded-Proto https
        header_up X-Real-IP {remote_host}
    }
}
```

The upstream frontend must preserve the original `Host` header. `X-Forwarded-Proto` is explicitly `https`, because that is the protocol the public client uses, even though the frontend-to-VPS connection is HTTP.

One limitation of this configuration: `X-Real-IP` contains **Caddy's immediate peer**, not necessarily the original client. I haven't configured trusted-proxy handling for the real client IP. Before relying on client IPs for logging or rate limiting, verify the provider's proxy source addresses and forwarded headers; don't simply trust headers supplied by anyone.

Keep the HTTP listener private and restrict access to the intended frontend using the provider's networking controls. This route is meant to be reached through HTTPS, not exposed directly as public HTTP.

Validate the file before reloading:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile && sudo systemctl reload caddy
```

Then test `Caddy` locally on the VPS:

```bash
curl -sS --connect-timeout 10 --max-time 20 \
  -H 'Host: immich.example.com' \
  http://10.0.0.2/api/server/ping
```

Again, we want `{"res":"pong"}`. This checks the proxy and the route home without involving public DNS or certificates.

`Immich` needs the root of its own hostname, not a sub-path such as `/immich`. `Caddy` streams the uploads, and I didn't add a request-body limit here. That doesn't establish what limits or timeouts the managed frontend applies.

## 3. Connect the managed frontend and DNS

In the `TierHive` panel, I configured:

1. The public hostname, using **Single Server / Regional Access**.
2. Domain validation, with the TXT record supplied by the panel.
3. The backend in my chosen region, pointing to the VPS's private IP on port `80`.
4. A Let's Encrypt certificate, with automatic renewal enabled.

For `immich.example.com`, the validation record was named `_tierhive-validation.immich` in the DNS zone. Copy the value from the panel, not from somebody else's tutorial.

Before changing an existing route, save its actual DNS record and tunnel configuration so you can roll back.

The public DNS record then points to the **managed HAProxy endpoint**, not the VPS's private address:

```text
Type: A
Name: immich
Value: 203.0.113.10
Proxy: DNS only (grey cloud)
```

Leaving the orange cloud enabled puts Cloudflare's HTTP proxy back in the path, including its upload restrictions. Check for a stale `AAAA` record too: the managed frontend was IPv4-only when I configured it.

### Certificate issued does not always mean certificate deployed

The first surprise: the panel showed **SSL Active**, but the regional certificate distribution still showed **No SSL**.

![Ben Wyatt from Parks and Recreation looks mildly confused](https://media.giphy.com/media/fElrlOhBVcBLW/giphy.gif)

The client failed with:

```text
curl: (35) LibreSSL/3.3.6: error:1404B458:SSL routines:ST_CONNECT:tlsv1 unrecognized name
```

Once the regional distribution went green, HTTPS worked. The problem was the frontend certificate deployment, not the backend `Caddy` configuration.

If you see this, check certificate distribution for the region serving your hostname. Rewriting the `Caddyfile` won't fix a certificate that hasn't reached the public frontend. Don't use `curl -k` to treat a TLS failure as fixed.

## 4. Check the public path, then try a real upload

We can test the frontend before relying on local DNS caches, while still verifying its certificate:

```bash
curl -sS --connect-timeout 10 --max-time 20 \
  --resolve immich.example.com:443:203.0.113.10 \
  https://immich.example.com/api/server/ping
```

Then test normal DNS resolution and the HTTP redirect:

```bash
curl -sS --max-time 20 \
  https://immich.example.com/api/server/ping

curl -I http://immich.example.com/
```

In my setup, the API returned `{"res":"pong"}` and HTTP returned `301 Moved Permanently`, pointing to HTTPS.

The useful test was the next one: **a friend successfully uploaded the large video that previously hit the Cloudflare limit**. That confirms this upload worked through the new path; it doesn't prove unlimited uploads, every mobile-app feature or every proxy timeout. Test those with the clients and file sizes you actually use.

Once the replacement is working, disable the old `Immich` tunnel route if it is no longer needed. Don't stop `cloudflared` if it also serves other applications, and keep the saved configuration available for rollback. I haven't confirmed that cleanup in my setup yet.

## What about upload speed?

The upload worked, but plateaued at around **3.5 MiB/s (roughly 29.4 Mbps)**. I haven't established the bottleneck.

The direction matters:

```text
Friend's upload
  → VPS
  → Tailscale
  → home download
  → Immich storage
```

My home's **100 Mbps upload** is relevant when someone downloads from `Immich`; it is not the headline access-link limit for a file being uploaded into my house.

I later observed a direct `Tailscale` connection to the home subnet router, but that doesn't prove the earlier transfer used a direct path throughout. Likewise, a nearest `DERP` region in `tailscale netcheck` is fallback information, not proof that the transfer used a relay.

Before blaming a provider cap or paying for an upgrade, I would:

1. Measure the sender's upload speed and the home's download speed.
2. Compare another large file that hasn't already been uploaded.
3. Check `tailscale status` during the transfer for direct versus relayed connectivity.
4. Check VPS CPU contention and home-side CPU/storage activity during the same transfer.

The provider's advertised network rate is only one part of that path, not a promise of application throughput.

## Conclusions

**The VPS is a relay, not a second photo server**: `Immich` and the library stay at home, while the public HTTPS entry point moves outside the homelab.

This solved the large-upload problem I encountered. The trade-offs are another machine to maintain, trusting the HTTPS frontend, and a network path whose performance still needs measuring.

`Tailscale` protects and restricts the route home; it doesn't authenticate public `Immich` users. Keep `Immich` updated, use strong credentials, and harden VPS access with SSH keys. If changing SSH authentication, keep an existing session open and verify a fresh login before closing it.

For HTTPS inside the homelab, my earlier [DuckDNS and Caddy guide]({{< relref "sslcerthomelab.md" >}}) covers the simpler case where `Caddy` manages the certificate itself.

## Useful reads

- [Immich reverse proxy documentation](https://docs.immich.app/administration/reverse-proxy/): hostname, headers and upload requirements.
- [Caddy package installation](https://caddyserver.com/docs/install#debian-ubuntu-raspbian): native packages for the VPS.
- [Caddy reverse proxy](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy): proxy behaviour and trusted headers.
- [Tailscale installation](https://tailscale.com/download/linux): installing the relay and subnet router software.
- [Tailscale subnet routers](https://tailscale.com/docs/features/subnet-routers): advertising and approving the route home.
- [Tailscale grants](https://tailscale.com/docs/reference/syntax/grants): restricting the relay to the backend it needs.
- [Tailscale performance troubleshooting](https://tailscale.com/docs/reference/troubleshooting/poor-performance-tailnet): investigating direct and relayed connections.
- [TierHive HAProxy overview](https://tierhive.com/blog/general/haproxy): the provider-specific public frontend.

If you choose `TierHive` for a similar setup, here is <a href="https://tierhive.com/r/18CC80A95F1D" rel="sponsored">my referral link</a>. I may earn TierHive tokens if you use it.

Thanks for reading.

![Thanks](https://media.giphy.com/media/a3IWyhkEC0p32/giphy.gif)
