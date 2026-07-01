# proxy

Shared [Caddy](https://caddyserver.com) reverse proxy that fronts every
application running on `alpha-numerical.com`. Owns ports **80/443** on the host
and terminates TLS with automatic Let's Encrypt certificates.

See `Caddyfile` for the currently served domains.

## Prerequisites

- Docker + Docker Compose v2.
- A shared external Docker network named `web`. Every app container fronted by
  this proxy attaches to it:

  ```bash
  docker network create web   # one-time, on the host
  ```

- DNS `A` records pointing every served hostname at this server's public IP.

## Usage

```bash
cd ~/proxy
docker compose up -d
docker compose logs -f caddy
```

Stop:

```bash
docker compose down
```

## Adding a new app

1. In the app's `docker-compose.yml` (or a prod overlay):
   - Give the public-facing container a stable `container_name` (e.g.
     `myapp-frontend`).
   - Attach that container to the external `web` network.
   - Do **not** publish ports 80/443 on the host — this proxy owns them.

2. Add a site block to `Caddyfile` in this repo, e.g.:

   ```caddyfile
   myapp.alpha-numerical.com {
       reverse_proxy myapp-frontend:3000
       encode gzip
   }
   ```

3. Pull the change on the server, then hot-reload Caddy (no downtime):

   ```bash
   cd ~/proxy
   git pull
   docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
   ```

4. Point DNS `A` record for the new hostname at this server. Caddy will obtain
   the TLS certificate automatically on first request.

## Currently proxied apps

| Host                                | Target container    | Repo                                          |
|-------------------------------------|---------------------|-----------------------------------------------|
| `smartvoter.alpha-numerical.com`    | `smartvoter-frontend` + `smartvoter-backend` | [kertser/SmartVoter](https://github.com/kertser/SmartVoter) |
| `tdg.alpha-numerical.com`           | `tdg-nginx`         | [kertser/TDG](https://github.com/kertser/TDG) |

## Certificates

Issued Let's Encrypt certificates live in the `proxy_caddy_data` Docker volume
(mounted at `/data` inside the container). **Do not delete this volume** — you
will hit ACME rate limits when re-issuing.

Backup:

```bash
docker run --rm -v proxy_caddy_data:/from -v $(pwd)/backup:/to alpine \
    sh -c "cp -a /from/. /to/"
```

## Migration from an app-embedded Caddy

If you're moving from a Caddy that used to live inside an app stack (e.g.
SmartVoter previously had its own `caddy` service in `docker-compose.prod.yml`),
copy the old certificate volume into this stack's volume before starting, so
you don't re-request every certificate from Let's Encrypt:

```bash
# 1. Snapshot the old volume (adjust the source volume name)
docker run --rm -v smartvoter_caddy_data:/from -v /tmp/caddy_backup:/to alpine \
    sh -c "cp -a /from/. /to/"

# 2. Create this stack's volume and load the snapshot
docker volume create proxy_caddy_data
docker run --rm -v /tmp/caddy_backup:/from -v proxy_caddy_data:/to alpine \
    sh -c "cp -a /from/. /to/"

# 3. Now start this stack — Caddy will reuse the existing certificates.
docker compose up -d
```
