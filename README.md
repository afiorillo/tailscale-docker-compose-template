# Docker Compose template — service behind Tailscale

Expose a containerised service as its own host in your Tailscale tailnet with
automatic HTTPS and network isolation.

## Architecture

```
[internet / tailnet]
        │
   ┌────▼──────────────────────────────────────────────────┐
   │  tailscale container                                  │
   │  • WireGuard VPN endpoint                             │
   │  • TLS termination (Tailscale-issued cert)            │
   │  • HTTP reverse proxy (serve.json)                    │
   └────┬──────────────────────────────────────────────────┘
        │ internal Docker network (no internet gateway)
   ┌────▼──────────────────────────────────────────────────┐
   │  web container (Caddy / any HTTP service)             │
   │  • no direct internet access                          │
   │  • no published host ports                            │
   │  • outbound via proxy only (HTTP_PROXY / HTTPS_PROXY) │
   └────┬──────────────────────────────────────────────────┘
        │
   ┌────▼──────────────────────────────────────────────────┐
   │  proxy container (Caddy forwardproxy)                 │
   │  • default-deny outbound allowlist (proxy/Caddyfile)  │
   │  • logs denied connection attempts to stdout          │
   └───────────────────────────────────────────────────────┘
```

**Network isolation:** The `app` Docker network is declared `internal: true`,
so the web container has no default internet route. Only the tailscale container
has internet access (via the separate `egress` network), making it the sole
gateway for all inbound and outbound traffic.

## Quick start

1. Use this repo as a template and create a private repository.

2. Copy `.env.example` to `.env` and fill in the values:

   ```sh
   cp .env.example .env
   ```

   | Variable | Description |
   |---|---|
   | `SERVICE_NAME` | Tailscale hostname. Access via `https://<name>.tail<id>.ts.net/` |
   | `TS_AUTHKEY` | Auth key from [Tailscale admin](https://login.tailscale.com/admin/settings/keys). Use an **ephemeral, pre-approved** key with an ACL tag. |
   | `TS_EXTRA_ARGS` | Extra flags for `tailscaled`. `--reset` clears stale serve config on restart. |

3. Replace the `web` service in `docker-compose.yml` with your application.
   Requirements:
   - Service name must stay `web` (referenced in `tsconfig/serve.json`).
   - Container must listen on port `8080` (or update `serve.json` to match).
   - **Do not add a `ports:` mapping** — Tailscale handles all inbound traffic.

4. Start:

   ```sh
   docker compose up -d
   ```

5. Access `https://<SERVICE_NAME>.tail<id>.ts.net/` from any tailnet node.
   The first request may take a few seconds for the certificate to be issued.

## Tailscale Serve configuration

### How it works

`TS_SERVE_CONFIG` points Tailscale to `tsconfig/serve.json`. Tailscale reads
this file on startup and configures its built-in HTTP reverse proxy and TCP
forwarding accordingly.

The config has three top-level keys:

| Key | Purpose |
|---|---|
| `TCP` | Which ports to listen on and how to handle them |
| `Web` | HTTP/HTTPS path routing for ports marked `HTTPS: true` |
| `AllowFunnel` | Whether to expose a port to the public internet via Tailscale Funnel |

`${TS_CERT_DOMAIN}` is an environment variable automatically set by Tailscale
to your node's MagicDNS hostname (e.g. `mynode.tail12345.ts.net`).

### TCP handlers

Each entry under `TCP` is a port number mapped to one of:

```json
"443": { "HTTPS": true }
```
Tailscale terminates TLS and forwards decrypted HTTP to a `Web` handler.

```json
"2222": { "TCPForward": "ssh-host:22" }
```
Raw TCP passthrough — Tailscale does not inspect or terminate the connection.
Use for SSH, databases, game servers, or any non-HTTP protocol. The hostname
is resolved inside the Tailscale container, so Docker service names work.

> **UDP** is not part of ServeConfig. UDP routing is a Tailscale network-level
> concept handled through ACL rules and subnet routing, not this file.

### Web handlers

Keyed by `"${TS_CERT_DOMAIN}:port"` (the port must appear in `TCP` with
`HTTPS: true`). Each handler maps a path prefix to one of:

```jsonc
"/":        { "Proxy": "http://web:8080" }  // reverse-proxy to backend
"/static/": { "Path": "/var/www/html" }     // serve local files
"/health":  { "Text": "ok" }               // static text response
```

Path prefixes are matched most-specific first, so `/api/` takes priority
over `/` for requests under `/api/`.

### Multiple ports / services example

```json
{
  "TCP": {
    "443":  { "HTTPS": true },
    "2222": { "TCPForward": "sshd:22" }
  },
  "Web": {
    "${TS_CERT_DOMAIN}:443": {
      "Handlers": {
        "/":     { "Proxy": "http://web:8080" },
        "/api/": { "Proxy": "http://api:3000" }
      }
    }
  },
  "AllowFunnel": {
    "${TS_CERT_DOMAIN}:443": false
  }
}
```

See [`tsconfig/serve.example.jsonc`](tsconfig/serve.example.jsonc) for the
full annotated reference of every supported field.

## Security

### Network isolation

The `web` container is on a Docker network with `internal: true`. Docker
provides no default internet gateway on this network, so:

- The web container cannot make direct outbound internet connections.
- If an attacker compromises the web container, they cannot trivially download
  payloads or exfiltrate data to arbitrary internet hosts.
- All traffic must go through the Tailscale container, which enforces ACL rules.

#### Allowing whitelisted outbound connections

Outbound connections from the web container are routed through a Caddy forward
proxy sidecar ([`caddyserver/forwardproxy`](https://github.com/caddyserver/forwardproxy)).
The allowlist lives in [`proxy/Caddyfile`](proxy/Caddyfile) — it is the single
source of truth for what the web container can reach. All other destinations
are denied and logged.

**To permit a destination**, add an `allow ips` line to the `acl` block in
`proxy/Caddyfile` and rebuild:

```
acl {
    allow ips 93.184.216.34        # example.com
    allow ips 140.82.112.0/20      # github.com range
    deny  all
}
```

```sh
docker compose build proxy && docker compose up -d proxy
```

To look up IPs for a hostname:
```sh
docker run --rm --network=${SERVICE_NAME}_egress alpine nslookup example.com
```

Denied connection attempts are logged as errors to stdout:
```sh
docker logs forward-proxy
```

The `HTTP_PROXY` and `HTTPS_PROXY` env vars are set on the web service in
`docker-compose.yml`. Most HTTP clients (`curl`, `python-requests`, Go
`net/http`, `node-fetch`) respect them automatically. Add destinations to
`NO_PROXY` to bypass the proxy for specific hosts (e.g. internal services).

### Hardening checklist

- [ ] Auth key is ephemeral and scoped to an ACL tag
- [ ] ACL policy restricts which tailnet peers can reach this node
- [ ] `AllowFunnel` is `false` in `serve.json` (unless you intend public access)
- [ ] Web container image is pinned to a specific digest in production
- [ ] Outbound allowlist in `proxy/Caddyfile` is as narrow as possible

## Tips

- **Ephemeral auth keys** — Tailscale will remove the machine from your tailnet
  shortly after the container stops. Combined with pre-approval and an ACL tag,
  this minimises the window if a key leaks.

- **Changing the web service port** — Update the `Proxy` URL in
  `tsconfig/serve.json` to match the new port. The service name `web` is used
  as the hostname inside the Docker network.

- **Debugging serve config** — Run `docker exec <tailscale-container> tailscale serve status`
  to see the active configuration and any errors.

- **State persistence** — The `ts-lib-var` volume preserves Tailscale's node
  identity across container restarts. Delete it to force re-registration.
