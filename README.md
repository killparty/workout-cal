# Daymark

**Press a button, mark the day.**

Daymark is a small, self-hosted activity calendar. Each activity you care about — yoga, gym, a run — is
a big colored button. Press it on a day you did it, and that day gets a dot in its color. Over weeks
and months the calendar fills up, and you can see at a glance how active you've been.

That's the whole idea, kept deliberately simple:

- **One press, done.** A press always means *today*. Pressing again changes nothing.
- **No backdating.** Forgot yesterday? Then you'll remember today.
- **No streaks, no goals, no pressure.** Just your days, marked.
- **Nothing is lost.** Every change lands in a Changes log and can be reverted.
- **Yours.** Self-hosted, a single SQLite file, export to JSON or CSV at any time.

<p>
  <img src="docs/screenshots/desktop-dark.png" alt="Daymark in dark mode on a laptop: Yoga, Gym, Meditation and Study buttons in a sidebar, six weeks of colored dots beside them">
</p>
<p>
  <img src="docs/screenshots/home-light.png" alt="Daymark on a phone: Yoga and Meditation marked today, six weeks of colored dots below" width="300">
</p>

A physical companion — a small e-paper device with real colored buttons that mirror the web buttons
1:1 — is planned. See [docs/device.md](docs/device.md).

## Quick start (Docker Compose)

Only Docker is needed.

```sh
mkdir daymark && cd daymark
curl -fsSLO https://raw.githubusercontent.com/bhadid/daymark/main/compose.yaml
docker compose up -d
```

Open <http://localhost:8080>. Data is stored in the `daymark-data` volume.

> **Default is no login (`AUTH_MODE=none`)**: anyone who can reach the port can use it. Run it on a
> private network or behind a VPN, or put it behind an auth proxy — see
> [docs/deployment.md](docs/deployment.md).

Without Compose:

```sh
docker run -d --name daymark -p 8080:8080 -v daymark-data:/data ghcr.io/bhadid/daymark:latest
```

### Build from source

To run unreleased changes, or if you'd rather not pull the published image:

```sh
git clone https://github.com/bhadid/daymark && cd daymark
docker compose -f compose.yaml -f compose.build.yaml up -d --build
```

## Features

- Buttons with a label, color and icon (icons keep them recognisable without color, too)
- Calendar: the last six weeks by default, or month by month, with dots per day
- Filter the calendar by activity, with days-per-month counts
- Tap (or right-click) a day to see what was marked and when, and to remove an entry
- Changes log with revert (reverts can be reverted, too)
- Haptic tick and a soft click sound on press, both optional per device
- Light, dark and system theme; follows your locale's first day of the week, or pick one
- Timezone aware: "today" is wherever you are right now; a marked day never moves afterwards
- Multiple users through a reverse-proxy login (e.g. Caddy + an SSO)
- Import / export (JSON), export (CSV)
- Installable as a web app on your phone's home screen

## Configuration

All settings are environment variables; see [.env.example](.env.example).

| Variable | Default | |
|---|---|---|
| `PORT` | `8080` | |
| `DATA_DIR` | `./data` (`/data` in Docker) | Where `daymark.db` lives |
| `AUTH_MODE` | `none` | `none` or `proxy` |
| `AUTH_PROXY_USER_HEADER` | `Remote-User` | Username header set by the proxy |
| `AUTH_PROXY_NAME_HEADER` | `Remote-Name` | Optional display-name header |
| `AUTH_TRUSTED_PROXIES` | private networks | Only these may set the headers |

## Development

Requires [Bun](https://bun.sh) 1.4+.

```sh
bun install
bun run dev        # API on :8080 and the web app with hot reload on http://localhost:5173
bun run check      # lint, typecheck, tests, build — what CI runs
```

| | |
|---|---|
| `src/shared` | Types, validation and date logic shared by server and web |
| `src/server` | Bun + Hono API, Drizzle + SQLite |
| `src/web` | Svelte 5 single-page app |
| `drizzle/` | SQL migrations (`bun run db:generate` after changing the schema) |
| `docs/` | Architecture, decisions, deployment, API |

Read [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) first; the reasoning behind each larger choice is
in [docs/adr](docs/adr).

## License

[MIT](LICENSE)
