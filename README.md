# User Information Collector MCP Server

An MCP (Model Context Protocol) server that collects contact information via tools and stores it in a local SQLite database. Includes a browser dashboard for viewing and deleting records.

## Run locally

```powershell
uv sync
uv run .\main.py
```

The server listens on `http://localhost:8008`.

| Page | Purpose |
|---|---|
| `http://localhost:8008/` | Contacts dashboard (view, delete) |
| `http://localhost:8008/docs` | How-to-use reference |
| `http://localhost:8008/mcp` | MCP streamable-HTTP endpoint for clients |

## Run with Docker

```bash
docker compose up -d --build
```

This runs the app behind [Caddy](https://caddyserver.com/) as a reverse proxy. Copy [.env.example](.env.example) to `.env` and set `ALLOWED_HOSTS` — compose refuses to start without it. Contact data persists in a named Docker volume (`contacts-data`) across rebuilds.

Caddy publishes on host ports **8080/8443**, not 80/443 — a common setup where the VPS already runs another reverse proxy (e.g. nginx) on the standard ports for other services. Access the app at `http://<vps-ip>:8080/`, `http://<vps-ip>:8080/docs`, and point MCP clients at `http://<vps-ip>:8080/mcp`. If ports 80/443 are actually free on your VPS, change them back to `80:80`/`443:443` in `docker-compose.yml`.

[Caddyfile](Caddyfile) serves both a domain and the bare IP. **Note**: since Caddy isn't on host port 80, its automatic Let's Encrypt HTTPS for the domain block won't complete (the ACME HTTP-01 challenge needs port 80 reachable from the internet) — that only works if you switch Caddy back to 80/443, or instead reverse-proxy through your existing nginx to `mcp-server` on an internal port. `ALLOWED_HOSTS` should list whichever host(s) you're actually using, comma-separated.

## Tools

### `insert_contact`

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Trimmed; rejected if empty |
| `phone_number` | string | no | Defaults to empty |
| `email` | string | no | Lowercased; must be unique if provided |

### `delete_contact`

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `contact_id` | integer | yes | The contact's `id`, shown on the dashboard |

Full parameter tables and example calls are on the `/docs` page once the server is running.

## Configuration

Set these as environment variables:

| Variable | Default | Purpose |
|---|---|---|
| `DATABASE_PATH` | `./contacts.db` | Where the SQLite file lives |
| `MCP_DEBUG` | `false` | Enables debug mode and verbose logging |
| `ALLOWED_HOSTS` | *(none — required in Docker)* | Comma-separated Host header(s) to accept, e.g. `203.0.113.5` or `your-domain.example.com`. Enables DNS-rebinding protection; must match the address in `Caddyfile`. |
| `RATE_LIMIT_MAX_REQUESTS` | `120` | Max requests per client IP per window, across all routes (dashboard, delete, `/mcp`). Responds `429` once exceeded. |
| `RATE_LIMIT_WINDOW_SECONDS` | `60` | Window length in seconds for the rate limit above. |

## Security status

There is currently **no authentication** on the dashboard, the delete action, or the `/mcp` endpoint — anyone who can reach the server can read, insert, or delete contacts. This is a deliberate decision to keep the server open to any client; revisit before storing real PII long-term. Two mitigations are in place given that: `ALLOWED_HOSTS` (DNS-rebinding protection) and a per-IP rate limit (`RATE_LIMIT_MAX_REQUESTS` / `RATE_LIMIT_WINDOW_SECONDS`, see [ratelimit.py](ratelimit.py)) — the rate limit is in-memory per instance, so it resets on restart and won't coordinate across multiple replicas. Traffic over the bare IP is plain HTTP (unencrypted) — only the domain path gets HTTPS. Don't rely on this for sensitive data until you're using the domain and, ideally, add auth. Reasonable options when ready: HTTP basic auth in `Caddyfile`, a bearer-token check in `main.py`, or restricting `Caddyfile` to an IP allowlist/VPN.

## Project layout

| File | Responsibility |
|---|---|
| `main.py` | FastMCP setup, tools, and HTTP routes |
| `config.py` | Environment-driven configuration |
| `db.py` | SQLite schema and queries |
| `ratelimit.py` | Per-IP rate-limiting middleware |
| `templates.py` | Renders `templates/*.html` |
| `templates/dashboard.html` | Contacts page |
| `templates/docs.html` | How-to-use page |
| `static/style.css` | Shared styling for both pages |
