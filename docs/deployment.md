# Deployment

Docker Compose is the primary way to run Daymark. The image is published for `amd64` and `arm64`
(Raspberry Pi 4/5) at `ghcr.io/bhadid/daymark`. To build the image yourself from a checkout
instead, use `docker compose -f compose.yaml -f compose.build.yaml up -d --build`.

## Minimal: private network, no login

```yaml
services:
  daymark:
    image: ghcr.io/bhadid/daymark:latest
    restart: unless-stopped
    environment:
      AUTH_MODE: none
    volumes:
      - daymark-data:/data
    ports:
      - "8080:8080"

volumes:
  daymark-data:
```

Anyone who can reach port 8080 can use it. Fine on a home LAN or behind a VPN like WireGuard or
Tailscale; don't expose it to the internet this way.

## Behind Caddy

Put Daymark and Caddy on the same Docker network and **don't publish Daymark's port** — Caddy is the
only way in.

```yaml
services:
  daymark:
    image: ghcr.io/bhadid/daymark:latest
    restart: unless-stopped
    environment:
      AUTH_MODE: none          # or proxy, see below
    volumes:
      - daymark-data:/data
    networks: [web]

networks:
  web:
    external: true            # the network your Caddy container is on

volumes:
  daymark-data:
```

```caddy
daymark.home.example {
    reverse_proxy daymark:8080
}
```

## Multiple users: auth at the proxy (`AUTH_MODE=proxy`)

Daymark has no passwords of its own ([ADR 0003](adr/0003-auth-modes.md)). Instead, an auth service
behind Caddy's `forward_auth` decides who you are, and Caddy passes the username to Daymark in a
header. The first time a username shows up, Daymark creates that user with their own activities and
calendar.

Good fits for a homelab:

- **[Pocket ID](https://pocket-id.org)** — a tiny OIDC provider that logs in with passkeys only.
  Pair it with a forward-auth service such as Tinyauth or oauth2-proxy.
- **[Tinyauth](https://tinyauth.app)** — the simplest forward-auth service; can use Pocket ID (or
  others) as the login.
- **[Authelia](https://www.authelia.com)** / **[Authentik](https://goauthentik.io)** — full-featured.

Example with Authelia (adjust the verify URL for other services; see their Caddy docs):

```caddy
daymark.home.example {
    # The future device API authenticates with its own tokens, not SSO.
    @device path /api/v1/device/*
    handle @device {
        reverse_proxy daymark:8080
    }

    handle {
        forward_auth authelia:9091 {
            uri /api/authz/forward-auth
            copy_headers Remote-User Remote-Name Remote-Email Remote-Groups
        }
        reverse_proxy daymark:8080
    }
}
```

And in Daymark's environment:

```yaml
AUTH_MODE: proxy
# Defaults shown; change only if your proxy uses other header names.
AUTH_PROXY_USER_HEADER: Remote-User
AUTH_PROXY_NAME_HEADER: Remote-Name
# Narrow this to your Docker network's subnet (docker network inspect web).
AUTH_TRUSTED_PROXIES: 172.16.0.0/12
```

Why both rules matter: Daymark believes the `Remote-User` header. If anything other than Caddy can
reach Daymark, it could claim to be anyone. Not publishing the port and setting
`AUTH_TRUSTED_PROXIES` close it off twice.

Switching an existing install from `none` to `proxy`: the `none` user is stored with the username
`local`. Export your data (Settings → Data) before switching and import it after logging in through
the proxy.

## Backups

Everything lives in `/data/daymark.db` (plus `-wal`/`-shm` files while running).

- Simplest: Settings → Data → Export JSON, regularly.
- File level, consistent while running:
  `docker compose exec daymark bun -e 'new (require("bun:sqlite").Database)("/data/daymark.db").exec("VACUUM INTO \"/data/backup.db\"")'`
  then copy `backup.db` from the volume.
- Or stop the container and copy the volume.

## Upgrades

`docker compose pull && docker compose up -d`. Database migrations run automatically at startup.
Pin a version tag (e.g. `ghcr.io/bhadid/daymark:0.1`) if you prefer to upgrade deliberately.
