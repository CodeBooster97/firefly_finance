# Design: Firefly III + Data Importer on Coolify

Date: 2026-08-02 · Status: implemented (autonomous session — defaults chosen and documented rather than asked)

## Goal

Self-host Firefly III with the Data Importer on Coolify, pulling transactions
from Monzo, Trade Republic and Trading 212.

## Decisions & trade-offs

- **Bank connectivity: GoCardless Bank Account Data** (built into the Data
  Importer, free tier) rather than SaltEdge or per-bank custom API scripts.
  Monzo is covered; Trade Republic via its German banking licence where listed;
  Trading 212 is not a PSD2 payment bank, so it uses CSV export + saved import
  config. Alternative considered: custom sync scripts against the Trading 212
  and Trade Republic APIs — rejected for maintenance cost; can be added later
  without changing this stack.
- **Database: MariaDB** (upstream's default compose choice) over Postgres —
  either works; MariaDB keeps parity with official docs.
- **Coolify-first compose**: no published host ports; `SERVICE_FQDN_*` magic
  variables declare routable services, all secrets via `${VAR}` interpolation so
  they surface in Coolify's env UI. A `docker-compose.local.yml` override adds
  ports 8080/8081 for local testing.
- **Cron**: upstream's alpine sidecar pattern hitting
  `/api/v1/cron/<STATIC_CRON_TOKEN>` daily. Automated bank re-imports via the
  importer's `/autoimport` endpoint + Coolify scheduled task (opt-in flags).
- **Auth between importer and Firefly**: Personal Access Token (simplest),
  created after first login and set as `FIREFLY_III_ACCESS_TOKEN`.

## Components

- `app` (fireflyiii/core) ← `db` (mariadb:lts, healthcheck-gated)
- `importer` (fireflyiii/data-importer) → `app` via internal
  `http://app:8080`; public UI on its own domain
- `cron` (alpine) → daily GET to app's cron endpoint
- Volumes: `firefly_iii_db`, `firefly_iii_upload`; repo dir `import-configs/`
  mounted at `/import` for reusable import configurations
