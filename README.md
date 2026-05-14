# N8N Automation Local Setup (Docker + Cloudflare Tunnel)

Docker Compose stack for **n8n** with **PostgreSQL**, optional **Cloudflare Tunnel** for HTTPS webhooks and remote access, and a **local-files** bind mount for workflow file I/O.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/) v2 (`docker compose`)
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)

## Quick start (local only)

1. **Clone the repo** and enter the project directory.

2. **Create your environment file** from the example and edit secrets:

   ```sh
   cp .env.example .env
   ```

   - Set **`N8N_ENCRYPTION_KEY`**: n8n uses this to encrypt credentials in the database. Generate once and **never change it** after you have data you care about, or existing credentials become unreadable.

     ```sh
     openssl rand -hex 32
     ```

   - Set **`POSTGRES_PASSWORD`** and matching **`DB_POSTGRESDB_PASSWORD`** to the same strong value.

   For pure local use, defaults like `N8N_HOST=localhost`, `N8N_PROTOCOL=http`, `WEBHOOK_URL`, and `N8N_EDITOR_BASE_URL` pointing at `http://localhost:5678/` are fine. Leave **`N8N_PROXY_HOPS`** unset unless you use a reverse proxy in front of n8n.

3. **Start the stack**:

   ```sh
   docker compose up -d
   ```

4. **Open n8n**: [http://localhost:5678](http://localhost:5678) and complete the first-owner signup in the browser.

5. **Optional: workflow files on disk**  
   The compose file mounts **`./local-files`** into the container at **`/files`**. Use that path in n8n (for example Read/Write Binary File nodes) when you need durable files next to the repo.

## Useful commands

| Task | Command |
|------|---------|
| Start (detached) | `docker compose up -d` |
| Stop | `docker compose down` |
| Follow n8n logs | `docker compose logs -f n8n` |
| Follow all services | `docker compose logs -f` |

Data lives in named volumes **`postgres_data`** and **`n8n_data`**; `docker compose down -v` removes them (destructive).

## Cloudflare Tunnel (recommended)

The **`cloudflared-tunnel`** service exposes n8n through [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/) so you get HTTPS, stable DNS, and webhooks that work from the internet.

1. In Cloudflare Zero Trust, create a tunnel and download the **credentials JSON** for that tunnel.

2. Place the credentials file under **`cloudflared/`** (for example `cloudflared/<tunnel-uuid>.json`). This path is gitignored.

3. Copy the example config and fill in your tunnel id, credentials filename, and hostname:

   ```sh
   cp cloudflared/config.example.yml cloudflared/config.yml
   ```

   Edit **`cloudflared/config.yml`**: set `tunnel`, `credentials-file` (path inside the container is `/etc/cloudflared/...`, same filename as on disk), and the `ingress` hostname. The example already routes to **`http://n8n:5678`**.

4. Update **`.env`** for your public URL (replace with your real host):

   - `N8N_HOST` — hostname only, no scheme or path  
   - `N8N_PROTOCOL=https`  
   - `N8N_PROXY_HOPS=1`  
   - `WEBHOOK_URL` and `N8N_EDITOR_BASE_URL` — full base URLs with trailing slash, e.g. `https://n8n.example.com/`

5. Start (or restart) the stack: `docker compose up -d`.

If you do **not** need a tunnel, you can stop only that service or remove the `cloudflared-tunnel` block from `docker-compose.yml` for a smaller footprint.
