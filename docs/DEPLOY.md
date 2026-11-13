# Deploying Tacho

One Linux VM runs everything with Docker Compose: Postgres, a one-shot migration step, the website, the Discord bot and Caddy (HTTPS). Files: `infra/` in this repo; on the server they live in `/opt/tacho`.

Hosting on Coolify instead: see [COOLIFY.md](COOLIFY.md) and `docker-compose.coolify.yml`.

```
internet ──80/443──> caddy ──> web:3001 (site, API, /api/live/ws WebSocket)
                                 │
          migrate (once per deploy) ──> postgres <── bot
```

Only Caddy publishes ports. Postgres has no published port.

## What the server needs

- 4 vCPU, 8 GB RAM, 40 GB SSD (docs/PLAN.md). Ubuntu 24.04 LTS or Debian 12.
- Docker Engine with the Compose plugin, v2.20 or newer (`docker compose version`). Install from https://docs.docker.com/engine/install/.
- A user for deploys, here `deploy`, in the `docker` group, owning `/opt/tacho`.
- For off-server backups: `rclone` (S3-compatible bucket) or `rsync` (SSH host).
- A domain whose DNS you control.

Memory limits in `infra/docker-compose.prod.yml`: Postgres 2 GB, web 2 GB, bot 512 MB, Caddy 256 MB, migrate 1 GB (only while it runs). That leaves room for the OS and page cache.

## First-time setup

1. **DNS.** Point an `A` record (and `AAAA` if the server has IPv6) for the domain, e.g. `tacho.example.com`, at the server. Wait until `dig +short tacho.example.com` returns the server's address: Caddy asks Let's Encrypt for the certificate on first start and fails without it.
2. **Firewall.** Allow only SSH, 80 and 443:
   ```bash
   sudo ufw default deny incoming
   sudo ufw allow OpenSSH
   sudo ufw allow 80/tcp && sudo ufw allow 443/tcp && sudo ufw allow 443/udp
   sudo ufw enable
   ```
   Docker-published ports bypass ufw, which is why nothing but Caddy publishes one. Use the provider's firewall too if it has one.
3. **Deploy user and folder.**
   ```bash
   sudo adduser --disabled-password deploy && sudo usermod -aG docker deploy
   sudo mkdir -p /opt/tacho/backups && sudo chown -R deploy:deploy /opt/tacho
   ```
   Make an SSH key pair for GitHub Actions (on your machine: `ssh-keygen -t ed25519 -f tacho-deploy -N ""`) and add `tacho-deploy.pub` to `/home/deploy/.ssh/authorized_keys`.
4. **Env file.** Copy `infra/.env.example` to `/opt/tacho/.env`, `chmod 600 /opt/tacho/.env`, and fill it in: `DOMAIN`, `ACME_EMAIL`, `IMAGE_PREFIX` (`ghcr.io/<owner>/<repo>`, lower case), a hex `POSTGRES_PASSWORD` (`openssl rand -hex 32`), `BETTER_AUTH_SECRET` (`openssl rand -base64 48`), the Discord client id and secret, and `DISCORD_BOT_TOKEN` when the bot should run. This file is the only place secrets live; it is never in the repo or an image.
5. **Discord OAuth.** In the Discord developer portal, the application's **OAuth2 > Redirects**, add `https://<DOMAIN>/api/auth/callback/discord`. Keep the localhost one for development.
6. **GitHub settings** (repository **Settings**):
   - **Environments > New environment** `production`. Add these secrets to it:
     | Secret                                                                                                                   | Value                                                                           |
     | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------- |
     | `DEPLOY_SSH_HOST`                                                                                                        | server IP or hostname                                                           |
     | `DEPLOY_SSH_USER`                                                                                                        | `deploy`                                                                        |
     | `DEPLOY_SSH_KEY`                                                                                                         | the private key `tacho-deploy` (whole file)                                     |
     | `DEPLOY_SSH_KNOWN_HOSTS`                                                                                                 | output of `ssh-keyscan -t ed25519 <host>`, checked against the server's own key |
     | `DEPLOY_SSH_PORT`                                                                                                        | optional, default 22                                                            |
     | Optional variable `DEPLOY_PATH` (default `/opt/tacho`). Add required reviewers there if deploys should wait for a click. |
   - **Actions > General > Workflow permissions**: the workflow asks for `packages: write` itself; nothing to change unless the organisation restricts it.
7. **First start.** Push to `main` (or run **Actions > Deploy > Run workflow** on `main`). The workflow copies `infra/*` to `/opt/tacho` and runs `deploy.sh`, which pulls the images, applies the migrations to the empty database, starts everything and waits for the site to be healthy. Then open `https://<DOMAIN>`, log in with Discord.
8. **Backups.** Set `BACKUP_RCLONE_REMOTE` or `BACKUP_RSYNC_TARGET` in `.env` (see below), then as `deploy`, `crontab -e`:
   ```
   15 3 * * * cd /opt/tacho && bash backup.sh >> /opt/tacho/backups/backup.log 2>&1
   ```
   Run `bash backup.sh` once by hand to see it work.
9. **Tracker.** Release builds must point at the site: build the tracker with `TACHO_API_URL=https://<DOMAIN>` (`apps/tracker/src-tauri/src/api.rs`).

## Deploying

Every push to `main` deploys once CI passes (`.github/workflows/deploy.yml`): it builds `web` and `bot` images tagged `sha-<commit>` (and `latest`), pushes them to GHCR and runs `deploy.sh sha-<commit>` on the server. Pull requests never deploy.

`deploy.sh` writes the tag into `.env`, pulls, applies migrations (`prisma migrate deploy`, the `migrate` service), and only then replaces web and bot. If a migration fails it stops there and the old site keeps running. It keeps the images of the last three deploys.

By hand on the server: `cd /opt/tacho && bash deploy.sh sha-<commit>`. Pulling needs a registry login when the packages are private: `docker login ghcr.io` with a token that has `read:packages` (the workflow logs in for the length of its own run only).

## Rolling back

```bash
cd /opt/tacho
docker image ls ghcr.io/<owner>/<repo>/web     # tags still on the server
bash deploy.sh sha-<earlier commit>
```

Or re-run the Deploy workflow of an earlier run in GitHub. Migrations only go forward: rolling back the image does not undo a migration. Keep migrations backwards compatible (add, then drop in a later release) so the previous image still runs on the newer schema. If a migration itself has to be undone, write a new migration that reverses it, or restore a backup.

## Backups

`backup.sh` (nightly by cron) writes `BACKUP_DIR/tacho-<UTC time>.dump` (`pg_dump` custom format), checks that `pg_restore` can read it, deletes local dumps older than `BACKUP_KEEP_DAYS` (14), and then copies them off the server:

- **S3-compatible bucket** (Backblaze B2, Cloudflare R2, AWS S3, ...): `rclone config` as `deploy` to create a remote, e.g. `offsite`, then `BACKUP_RCLONE_REMOTE=offsite:tacho-backups/db`. Dumps older than 14 days are deleted there too. Give the key write access to that bucket only; bucket versioning or object lock protects against a compromised server deleting them.
- **rsync host**: `BACKUP_RSYNC_TARGET=backup@host:/srv/backups/tacho`, with the `deploy` user's SSH key on that host. The target mirrors the local folder (so also 14 days).

With neither set it keeps local copies only and says so in the log. That is not a backup of the server.

## Restoring

```bash
cd /opt/tacho
bash restore.sh backups/tacho-20261113T031500Z.dump    # asks for confirmation
```

It stops web and bot, drops and recreates the `tacho` database, restores the dump, applies any newer migrations of the running image and starts web and bot again. A dump from elsewhere: copy it into `backups/` first (`rclone copy offsite:tacho-backups/db/<file> backups/`).

To check a dump without touching production, restore it into a scratch container:

```bash
docker run -d --name restore-check -e POSTGRES_PASSWORD=x postgres:18
docker exec -i restore-check sh -c 'until pg_isready -U postgres; do sleep 1; done; createdb -U postgres tacho && pg_restore -U postgres -d tacho --no-owner' < backups/<file>.dump
docker exec restore-check psql -U postgres -d tacho -c 'select count(*) from job'
docker rm -f restore-check
```

## What to watch

- `docker compose -f docker-compose.prod.yml ps`: all `running`, web `healthy`, migrate `exited (0)`.
- Logs: `docker compose -f docker-compose.prod.yml logs -f --tail 100 web` (also `bot`, `caddy`). Each container keeps at most 5 x 10 MB of logs.
- The site from outside: an uptime check on `https://<DOMAIN>/`.
- Certificates: Caddy renews them itself; `caddy` logs show failures (usually DNS or port 80 blocked).
- Backups: `backups/backup.log` ends in `backup done` every night, and the off-server target has a dump from today.
- Disk: `df -h /` and `docker system df`; Postgres data is the `tacho_pgdata` volume.
- The bot: `bot` logs and the `bot_status` table (apps/bot/README.md).
- One copy each of web and bot: the TruckersMP membership check and the live map hub run inside the web process.
