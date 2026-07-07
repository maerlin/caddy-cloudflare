# Caddy Cloudflare

Caddy Docker image with Cloudflare DNS-01 ACME validation, built automatically by GitHub Actions and published to the GitHub Container Registry (GHCR).

A scheduled workflow checks daily for new Caddy releases and updates to the bundled modules, and rebuilds the image when needed. No secrets or configuration required — everything is derived from the repository name.

## Included modules

- [caddy-dns/cloudflare](https://github.com/caddy-dns/cloudflare) — Cloudflare DNS-01 ACME challenge provider
- [WeidiDeng/caddy-cloudflare-ip](https://github.com/WeidiDeng/caddy-cloudflare-ip) — trust Cloudflare proxy IP ranges (real client IPs in logs)
- [fvbommel/caddy-combine-ip-ranges](https://github.com/fvbommel/caddy-combine-ip-ranges) — combine multiple IP range sources

## Setup

1. Push this repository to GitHub.
2. Enable GitHub Actions in the repository's `Actions` tab if prompted.
3. Trigger the first build: run the `Check Caddy Releases and Trigger Build` workflow manually (Actions tab → select workflow → `Run workflow`), or wait for the daily schedule.
4. The image appears under the repository's Packages as `ghcr.io/OWNER/REPO`.

To build a specific Caddy version, run the `Build and Push Docker Image` workflow manually and pass the version (e.g. `2.11.4`).

> After changing the modules in the `Dockerfile`, update `MODULES` in `scripts/check_caddy_status/config.py` to match, and trigger a manual build.

## Usage

```yaml
services:
  caddy:
    image: ghcr.io/OWNER/REPO:latest
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - $PWD/Caddyfile:/etc/caddy/Caddyfile
      - $PWD/site:/srv
      - caddy_data:/data
      - caddy_config:/config
    environment:
      - CLOUDFLARE_API_TOKEN=your_cloudflare_api_token

volumes:
  caddy_data:
    external: true
  caddy_config:
```

Defining the data volume as [external](https://docs.docker.com/compose/compose-file/compose-file-v3/#external) makes sure `docker-compose down` does not delete the volume. You may need to create it manually using `docker volume create caddy_data`.

### Caddyfile

Set the ACME DNS challenge provider globally ([`Caddyfile_global`](Caddyfile_global)):

```
{
  acme_dns cloudflare {env.CLOUDFLARE_API_TOKEN}
}

example.com {
    root * /usr/share/caddy
    file_server
    encode gzip
}
```

Or per site ([`Caddyfile_per_site`](Caddyfile_per_site)):

```
example.com {
    root * /usr/share/caddy
    file_server
    encode gzip

    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }
}
```

To trust Cloudflare's proxy IPs so logs show real client IPs, add to the global block:

```
{
  acme_dns cloudflare {env.CLOUDFLARE_API_TOKEN}

  servers {
    trusted_proxies cloudflare
    client_ip_headers Cf-Connecting-Ip
  }
}
```

### Cloudflare API token

Create a token at [dash.cloudflare.com](https://dash.cloudflare.com) → My Profile → API Tokens → Create Token → Custom Token, with:

- **Zone → Zone → Read**
- **Zone → DNS → Edit**

Scope it to the zones you need, then pass it to the container as `CLOUDFLARE_API_TOKEN`.

#### Troubleshooting

You may encounter `solving challenges: presenting for challenge: adding temporary record for zone xyz.: got error status: HTTP 403.`
In such cases, try setting custom DNS resolvers to bypass resolver issues:

```
tls {
  dns cloudflare {env.CLOUDFLARE_API_TOKEN}
  resolvers 1.1.1.1
}
```

[Official troubleshooting guide](https://github.com/caddy-dns/cloudflare?tab=readme-ov-file#troubleshooting)

## Tags

- **`latest`** — most recent stable Caddy release
- **`<version>`** — e.g. `2.11.4` (exact), `2.11` (latest patch), `2` (latest minor)

## Platforms

Images are built for `linux/amd64` and `linux/arm64`. To add or remove platforms, change the `platforms:` list in [`.github/workflows/build-docker-image.yml`](.github/workflows/build-docker-image.yml) and `REQUIRED_PLATFORMS` in [`scripts/check_caddy_status/config.py`](scripts/check_caddy_status/config.py) — they must match.

## License

MIT — see [LICENSE](LICENSE). Based on [caddybuilds/caddy-cloudflare](https://github.com/caddybuilds/caddy-cloudflare).
