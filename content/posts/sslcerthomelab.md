---
title: "🔐 Free SSL certificates for your homelab with DuckDNS and DNS-01"
date: 2024-09-11T17:57:11Z
toc: true
draft: false
images:
tags:
  - homelab
  - ssl
  - free
  - duckdns
  - letsencrypt
  - DNS-01
  - caddy
---

![Immich login page](https://res.cloudinary.com/indiependente/image/upload/indiependente.dev/homelab/immich-login.png)

If you like me have been wondering on how to have nice links in your homelab without having to pay for a domain and a certificate, this post is for you.

Especially if you want to use a subdomain of a free domain like DuckDNS, you can get a free SSL certificate from Let's Encrypt using the DNS-01 challenge.

## DNS-01 challenge

The DNS-01 challenge is a way to prove that you own a domain by adding a specific DNS record. This is the most secure way to get a certificate as it doesn't require any server to be reachable from the internet.

## Setting up Caddy with DuckDNS

Caddy is a modern web server that makes it incredibly easy to set up HTTPS with automatic certificate management. When combined with DuckDNS, it becomes a powerful solution for your homelab.

### Prerequisites

1. A DuckDNS account and domain (e.g., `yourdomain.duckdns.org`)
2. Your DuckDNS token
3. Caddy installed on your server

### Configuration

Here's how to set up Caddy with DuckDNS DNS-01 challenge:

1. First, create a Caddyfile with the following structure:

```caddyfile
(tls) {
    tls {
        dns duckdns {env.DUCKDNS_TOKEN}
    }
}

*.yourdomain.duckdns.org, yourdomain.duckdns.org {
    import tls
    
    # Your reverse proxy configurations here
    handle @service {
        reverse_proxy localhost:port
    }
}
```

2. Set your DuckDNS token as an environment variable:

```bash
export DUCKDNS_TOKEN=your-token-here
```

3. For each service you want to expose, add a handle block with the appropriate subdomain:

```caddyfile
@service host service.yourdomain.duckdns.org
handle @service {
    reverse_proxy localhost:port
}
```

### Benefits of this setup

- Automatic HTTPS certificate management
- No need to manually renew certificates
- Support for wildcard certificates (`*.yourdomain.duckdns.org`)
- Clean and maintainable configuration
- Built-in reverse proxy capabilities

## Useful reads

- [Let's Encrypt: DNS-01 challenge](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge)
- [Caddy Documentation](https://caddyserver.com/docs/)
- [DuckDNS Documentation](https://www.duckdns.org/domains)
