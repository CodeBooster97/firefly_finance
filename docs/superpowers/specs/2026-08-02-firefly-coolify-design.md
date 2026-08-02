# Design: Firefly III + Data Importer on Coolify

Date: 2026-08-02 · Status: implemented (autonomous session — defaults chosen and documented rather than asked)

## Goal

Self-host Firefly III with the Data Importer on Coolify, pulling transactions
from Monzo, Trade Republic and Trading 212.

## Decisions & trade-offs

- **Bank connectivity** (revised 2026-08-02 after research): GoCardless Bank
  Account Data closed to new registrations July 2025; Salt Edge free tier ended
  Oct 2025. Enable Banking (free restricted mode) is the importer's new free
  provider but covers none of the three target banks (institution list checked
  directly). Chosen paths: Monzo + Trading 212 via **Lunch Flow** (paid
  aggregator, ~$35/yr, native importer support) or free CSV flows; Trade
  Republic via **pytr/tr-api** CSV export (unofficial, fragile since mid-2026
  WAF rate-limiting) or manual export. Trading 212 also has an official public
  API (beta, Invest/ISA only) usable for a DIY sync later. The compose file
  carries env vars for Enable Banking, Lunch Flow and legacy GoCardless so any
  of these work without stack changes.
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
