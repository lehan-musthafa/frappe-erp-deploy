# Frappe CRM deployment

Generated 2026-11-14 by running the compose-generation step of
`frappe/bench` `easy-install.py deploy` against `frappe_docker` main, with the
flags from the Frappe CRM docs (`--image=ghcr.io/frappe/crm --version=stable --app=crm`).

- `easy-install-exact.yml`: exactly what the script produces, including its own Traefik on ports 80/443.
  The only change is that the secrets and site name are placeholders: `${DB_PASSWORD}`, `${SITE_NAME}`, `${LETSENCRYPT_EMAIL}`.
- `coolify-compose.yml`: the same stack, minus Traefik and the published ports, for use behind Coolify's proxy.
  The only `$` in it is `${SERVICE_PASSWORD_MARIADB}`, which Coolify generates automatically. Config values are written out literally, and `FRAPPE_SITE_NAME_HEADER` is left to the nginx entrypoint's default (`$host`).
  It also uses newer images than the script: MariaDB 12.3 LTS instead of 11.8, Redis 8.10 instead of 8.6. Frappe does not officially test these versions.

## Image (read first)

The official `ghcr.io/frappe/crm` images have **not contained the crm app since v1.67.1**
([frappe/crm#2538](https://github.com/frappe/crm/issues/2538)). `stable`, `v16.0.3` and `v1.92.0` all contain only Frappe.
`v1.67.0` is the last good one.

Build your own image instead:

1. Push this folder to a GitHub repo.
2. Run **Actions → Build Frappe CRM image** (defaults: CRM `version-16` on Frappe `version-16`).
   The workflow fails if `crm` is missing from the image.
3. Make the `frappe-crm` package public, or add GHCR credentials in Coolify (Servers → Registries).
4. In `coolify-compose.yml`, replace `YOUR_GITHUB_USER` with your lowercase GitHub user or org.
   For reproducible deploys, pin the `version-16-<run number>` tag.

## Coolify

1. Create a new **Docker Compose Empty** resource and paste in `coolify-compose.yml`.
2. Nothing to set. Coolify generates `SERVICE_PASSWORD_MARIADB`, which you can see in the Environment Variables tab.
3. Set the domain on the `frontend` service to `https://crm.yourdomain.com:8080`.
4. Deploy, then create the site once from the `backend` container terminal.
   This is the same command the script runs after `up`:

   ```
   bench new-site --mariadb-user-host-login-scope='%' --db-root-password=<SERVICE_PASSWORD_MARIADB value> \
     --admin-password=<choose one> --install-app crm crm.yourdomain.com
   ```

   The site name **must equal the domain**. nginx selects the site from the Host header
   (`FRAPPE_SITE_NAME_HEADER: $host`).
5. Open `https://crm.yourdomain.com/crm`.

## Upgrade (what `easy-install.py upgrade` does)

Rerun the build workflow, change the image tag, redeploy, then run `bench --site all migrate` in `backend`.

## Backups

The `cron` (ofelia) container runs `bench --site all backup` every 6 hours.
Backups land in the `sites` volume at `sites/<site>/private/backups`. Copy them off the server.
