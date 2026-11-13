# Deploying Tacho on Coolify

`docker-compose.coolify.yml` runs the same stack as `infra/docker-compose.prod.yml` (Postgres, a one-shot migration step, the website, the Discord bot), but Coolify builds the images from the repo and its proxy handles the domain and HTTPS. The `infra/` files, `deploy.yml` workflow and `docs/DEPLOY.md` are for the plain-VM setup and are not used here.

```
internet ──443──> Coolify proxy ──> web:3001 (site, API, /api/live/ws WebSocket)
                                      │
               migrate (every deploy) ──> postgres <── bot
```

## Setup

1. **New resource.** In a Coolify project: **+ New > Public Repository** (or **Private Repository (with GitHub App)**), pick this repo and branch, and name the resource **Tacho**.
   - **Build Pack:** Docker Compose
   - **Base Directory:** `/`
   - **Docker Compose Location:** `/docker-compose.coolify.yml`
2. **Domain.** In the resource's configuration, set the **web** service's domain, e.g. `https://tacho.example.com`. Leave postgres, migrate and bot without a domain. Point the domain's DNS at the Coolify server first so the Let's Encrypt certificate can be issued.
3. **Environment variables** (Environment Variables tab). Coolify generates and keeps these itself; don't change them once data exists:
   - `SERVICE_PASSWORD_POSTGRES`: database password.
   - `SERVICE_PASSWORD_64_BETTERAUTH`: Better Auth secret. Changing it logs everyone out.
   - `SERVICE_URL_WEB`: the web domain; becomes `BETTER_AUTH_URL`.

   Fill in:

   | Variable                            | Required | Value                                                            |
   | ----------------------------------- | -------- | ---------------------------------------------------------------- |
   | `DISCORD_CLIENT_ID`                 | yes      | Discord application > OAuth2                                     |
   | `DISCORD_CLIENT_SECRET`             | yes      | same place                                                       |
   | `DISCORD_BOT_TOKEN`                 | no       | Discord application > Bot. Empty: the bot starts and stays idle. |
   | `STEAM_WEB_API_KEY`                 | no       | shows the linked Steam persona name                              |
   | `TRACKER_DOWNLOAD_URL`              | no       | Windows installer link for `/download`                           |
   | `LIVE_MAP_ETS2_*`, `LIVE_MAP_ATS_*` | no       | map tiles for `/map` (see `apps/web/.env.schema`)                |

4. **Discord OAuth.** In the Discord developer portal, **OAuth2 > Redirects**, add `https://<your domain>/api/auth/callback/discord`.
5. **Deploy.** Coolify builds the `web`, `migrate` and `bot` images, starts Postgres, runs the migrations (`prisma migrate deploy`), and only then starts web and bot. If a migration fails, web and bot are not replaced. Turn on **Auto Deploy** (GitHub App) or add the deploy webhook to deploy on every push.
6. **Tracker.** Release builds must point at the site: build the tracker with `TACHO_API_URL=https://<your domain>`.

## Notes

- **One copy each** of web and bot: the TruckersMP membership check and the live map hub run inside the web process. Don't scale them.
- **Changing the domain later:** update the web service's domain, check that `SERVICE_URL_WEB` changed with it, redeploy, and update the Discord redirect.
- **The migrate container** shows as _exited (0)_ after each deploy. That is expected; `exclude_from_hc` keeps it out of Coolify's health status. It downloads Prisma's schema engine from `binaries.prisma.sh` when it runs, so the server needs outbound HTTPS.
- **Backups:** Postgres data is in the `pgdata` volume. Coolify's scheduled database backups apply to standalone database resources, not to a database inside a compose app, so back it up yourself, e.g. a Coolify **Scheduled Task** on the postgres service: `pg_dump -U tacho -Fc tacho > /var/lib/postgresql/backup-$(date +%F).dump` and copy the dumps off the server. `infra/restore.sh` shows the restore steps.
- Coolify-only keys (`exclude_from_hc`) make plain `docker compose` reject the file. To try it locally, use `docker-compose.yml` (development) or `infra/docker-compose.prod.yml`.
