# Joplin + Caddy + ZeroTier (Docker Compose)

Private Joplin Server reachable only over ZeroTier, with public-trusted TLS via Cloudflare ACME DNS-01 using Caddy.

## Prerequisites

- A ZeroTier network and the ability to authorize members.
- DNS hosted on Cloudflare (API token with Zone.DNS:Edit for your zone).
- Split/internal DNS that resolves `joplin.internal.example.com` to the server's ZeroTier IP for clients.

## Files

- `docker-compose.yml`: ZeroTier (host network), Caddy (host network), Joplin app, Postgres, transcribe.
- `caddy/Dockerfile`: Builds Caddy with the Cloudflare DNS plugin.
- `Caddyfile`: Binds to the ZeroTier IP and reverse-proxies to Joplin.
- `caddy/env.example`: Copy to `caddy/env` and fill in values.
- `.env.example`: Copy to `.env` and fill in values.

## Setup

1. Create `.env` from the example and adjust values:

   ```bash
   cp .env.example .env
   ```

2. Create `caddy/env` from the example and set values:

   ```bash
   cp caddy/env.example caddy/env
   ```

   - `CF_API_TOKEN`: Cloudflare API token with Zone.DNS:Edit scope.
   - `ZT_IP`: Set after ZeroTier joins and is authorized.

3. Start the stack:

   ```bash
   docker compose up -d
   ```

4. Join the server to your ZeroTier network:

   ```bash
   docker exec zerotier zerotier-cli info
   docker exec zerotier zerotier-cli join <NETWORK_ID>
   ```

   Authorize the member in your controller, note the ZeroTier IP (e.g., `10.147.17.10`), set `ZT_IP` in `caddy/env`, then:

   ```bash
   docker compose restart caddy
   ```

5. Configure split/internal DNS so clients resolve `joplin.internal.example.com` to `ZT_IP`.

6. In Joplin clients, set the sync URL to:

   ```
   https://joplin.internal.example.com/api
   ```

## Notes

- Postgres and transcribe are kept on internal networks and not exposed to the host.
- Joplin app port is published only on `127.0.0.1` for Caddy to proxy.
- Caddy runs in host network and binds to the ZeroTier IP; ACME DNS-01 issues certs via Cloudflare (no public A/AAAA needed).

## Optional hardening

Use host firewall (e.g., nftables) to allow 443 only from the ZeroTier interface:

```nft
table inet filter {
  chains = {
    input = {
      type filter hook input priority 0; policy drop;
      ct state established,related accept
      iifname "lo" accept
      tcp dport 22 iifname "ztXXXXXX" accept   # SSH optional
      tcp dport 443 iifname "ztXXXXXX" accept  # HTTPS over ZeroTier only
    }
    forward = { type filter hook forward priority 0; policy drop; }
    output  = { type filter hook output  priority 0; policy accept; }
  }
}
```

Replace `ztXXXXXX` with your host's ZeroTier interface name.

## Troubleshooting

- If Caddy can’t issue certificates, verify Cloudflare token scope and that TXT records are created for `_acme-challenge.joplin.internal.example.com`.
- If clients can’t connect, confirm they’re on ZeroTier and resolve the hostname to the ZeroTier IP.
- If Joplin shows "Invalid base URL", ensure `APP_BASE_URL` matches the Caddy hostname and includes `https://`.

## License and Attribution

This repository (docker-compose configuration, Caddy setup, and documentation) is licensed under the [MIT License](LICENSE).

**Joplin Server** is proprietary software licensed under the [Joplin Server Personal Use License](https://github.com/laurent22/joplin/blob/dev/packages/server/LICENSE.md), which restricts use to **non-commercial purposes only**. By deploying Joplin Server using this repository, you agree to comply with that license.

Other components of the Joplin project (desktop/mobile clients) are licensed under [AGPL-3.0-or-later](https://github.com/laurent22/joplin/blob/dev/LICENSE).

This project is not affiliated with or endorsed by Joplin SAS or Laurent Cozic. Joplin® is a registered trademark of JOPLIN SAS.

### Third-party components

- [Joplin Server](https://github.com/laurent22/joplin) – Proprietary (Personal Use License, non-commercial only)
- [Caddy](https://github.com/caddyserver/caddy) – Apache 2.0
- [caddy-dns/cloudflare](https://github.com/caddy-dns/cloudflare) – Apache 2.0
- [ZeroTier](https://github.com/zerotier/ZeroTierOne) – Business Source License 1.1
- [PostgreSQL](https://www.postgresql.org/about/licence/) – PostgreSQL License (BSD-style)
