# Deploying Trust next to an existing nginx site

This puts the Trust server on **its own subdomain** (e.g. `trust.zeddm.ir`),
reverse-proxied by your existing nginx to a **localhost-only** port. Your current
landing page keeps serving the main domain unchanged — nothing conflicts because
Trust only claims a new `server_name`, and the app process never binds a public
port.

HTTPS (step 6) is what lets **phones connect** — Android/iOS block plain-HTTP by
default, and TLS also gives you `wss://` for the realtime socket for free.

> Security note: this build has **open registration**, **no rate limiting**, and
> messages are currently dev-plaintext (the MLS encryption isn't wired in yet).
> It's fine for testing with people you invite; don't treat it as private/secure
> until E2E lands.

---

## 0. Prerequisites
- The server (VPS) already runs nginx for your landing page.
- You can `ssh` in with sudo.
- You control DNS for the domain.

## 1. DNS
Add an **A record**: `trust` → your server's public IP (same box as the landing
page). Wait for it to resolve: `dig +short trust.zeddm.ir`.

## 2. Build the server
Internal crates are all part of the `trust-server` repo, so it builds standalone.

```bash
# Toolchain (skip if rustc is already present)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"

# Get the code + build (a small VPS may want swap for the release build)
git clone <your trust-server repo URL> trust-server-src
cd trust-server-src
cargo build --release           # produces target/release/trust-server
```

If a 1 GB box OOMs while linking, add temporary swap:
```bash
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
```

## 3. Install the binary + service
```bash
sudo useradd --system --home /opt/trust-server --shell /usr/sbin/nologin trust
sudo mkdir -p /opt/trust-server /var/lib/trust
sudo cp target/release/trust-server /opt/trust-server/
sudo chown -R trust:trust /opt/trust-server /var/lib/trust

sudo cp trust-deploy/systemd/trust-server.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now trust-server

# Confirm it's up on localhost:
curl -s http://127.0.0.1:8080/health        # -> {"status":"ok",...}
journalctl -u trust-server -f                # live logs
```

## 4. nginx site
```bash
sudo cp trust-deploy/nginx/trust.conf /etc/nginx/sites-available/trust.conf
sudo ln -s /etc/nginx/sites-available/trust.conf /etc/nginx/sites-enabled/
```
The file already targets `trust.zeddm.ir`. Then add the WebSocket `map` once
inside the `http {}` block (see the comment at the bottom of `trust.conf`) —
e.g. create `/etc/nginx/conf.d/ws_upgrade.conf`:
```nginx
map $http_upgrade $connection_upgrade { default upgrade; '' close; }
```
```bash
sudo nginx -t && sudo systemctl reload nginx
```

## 5. Check over plain HTTP
```bash
curl -s http://trust.zeddm.ir/health      # -> {"status":"ok",...}
```

## 6. TLS
```bash
sudo certbot --nginx -d trust.zeddm.ir     # obtains cert, adds 443 + redirect
curl -s https://trust.zeddm.ir/health      # -> {"status":"ok",...}
```

## 7. Point the app at it
On the phone, install the Trust app (build an APK on your dev machine:
`flutter build apk --debug`, then transfer/install `build/app/outputs/flutter-apk/app-debug.apk`).
On the login screen set **Server** to `https://trust.zeddm.ir`, then register.
Two phones (or a phone + a desktop window) can now message each other over the
internet.

## Updating later
```bash
cd trust-server-src && git pull && cargo build --release
sudo systemctl stop trust-server
sudo cp target/release/trust-server /opt/trust-server/
sudo systemctl start trust-server
```

## Data / reset
State lives in `/var/lib/trust` (accounts, profiles, chats, media index, and
`blobs/`). Back it up or delete it to wipe. Snapshots flush every ~3s.
