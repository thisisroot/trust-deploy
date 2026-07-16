# trust-deploy

One command to self-host the entire Trust stack on any server/IP/domain. The official instance runs the identical stack.

Self-hosters **configure and run pre-built container images** — they never build from source.

## What it deploys

A single deployable unit bundling:

- **trust-server** — the [reference server](../trust-server) (pre-built image).
- **Database** — durable state.
- **Cache / pub-sub** — presence + real-time fan-out.
- **Object storage** — encrypted media blobs.
- **SFU** — WebRTC call relay.
- **Reverse proxy** — automatic HTTPS (e.g. Caddy / Traefik) for the configured domain.

## Usage (intended)

```bash
cp .env.example .env      # set DOMAIN, admin creds, storage keys, etc.
./install.sh              # pulls images, provisions, starts the stack
```

After it's up, point a [`trust-client`](../trust-client) at your domain (or set it as the default in a custom build).

## Layout (planned)

```
compose/
  docker-compose.yml      # full stack
  docker-compose.dev.yml  # local dev overrides
.env.example              # all configurable settings, documented inline
install.sh                # one-command bring-up (pull, configure, start)
config/
  proxy/                  # reverse-proxy / HTTPS config
  server/                 # trust-server config templates
```

## Principles

- **Pre-built images only** — versions pinned to a protocol version.
- **One command** — bring the whole stack up on a fresh host.
- **Same stack everywhere** — official instance == what you run.
