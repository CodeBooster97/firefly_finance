# firefly_finance

Self-hosted [Firefly III](https://www.firefly-iii.org/) + [Data Importer](https://docs.firefly-iii.org/how-to/data-importer/), deployed on [Coolify](https://coolify.io/), pulling in transactions from **Monzo**, **Trade Republic** and **Trading 212**.

## What's in the stack

| Service    | Image                       | Purpose                                            |
|------------|-----------------------------|----------------------------------------------------|
| `app`      | `fireflyiii/core`           | Firefly III itself (listens on 8080)               |
| `importer` | `fireflyiii/data-importer`  | Web UI + API for bank/CSV imports (listens on 8080)|
| `db`       | `mariadb:lts`               | Database                                           |
| `cron`     | `alpine`                    | Fires Firefly III's daily cron (recurring transactions, bills, budgets) |

Persistent data lives in two named volumes: `firefly_iii_db` (database) and `firefly_iii_upload` (attachments). Back those up.

## Deploying on Coolify

1. **Create the resource**: in Coolify, *+ New → Docker Compose* (or *Public/Private Repository* pointing at this repo, build pack "Docker Compose"). Coolify will parse `docker-compose.yml`.
2. **Set environment variables** in the resource's *Environment Variables* tab — everything listed in [.env.example](.env.example). The required ones:
   - `APP_KEY` — exactly 32 chars: `openssl rand -hex 16`
   - `STATIC_CRON_TOKEN` — exactly 32 chars: `openssl rand -hex 16`
   - `DB_PASSWORD` — `openssl rand -hex 24`
   - `APP_URL` — the public URL you'll give Firefly III, e.g. `https://finance.yourdomain.com`
3. **Assign domains**: the compose file declares `SERVICE_FQDN_APP_8080` and `SERVICE_FQDN_IMPORTER_8080`, so Coolify knows to route a domain to each of the two web services. In the resource settings, set the `app` service's domain to match `APP_URL`, and give `importer` its own domain (e.g. `https://import.yourdomain.com`). No host ports are published — traffic goes through Coolify's proxy with TLS.
4. **Deploy**, then open Firefly III and register your account (the first registered user becomes admin).
5. **Create a Personal Access Token** in Firefly III: *Profile → OAuth → Personal Access Tokens → Create new token*. Put it in the `FIREFLY_III_ACCESS_TOKEN` env var in Coolify and redeploy (or paste it into the importer UI each time).

## Running locally (for testing)

```bash
cp .env.example .env   # then fill in the secrets
docker compose -f docker-compose.yml -f docker-compose.local.yml up -d
```

Firefly III: <http://localhost:8080> · Importer: <http://localhost:8081>

## Linking the banks

> **State of play (verified Aug 2026):** the two providers that used to be the free defaults are gone — **GoCardless Bank Account Data stopped accepting new registrations in July 2025**, and **Salt Edge ended free-tier access in Oct 2025** (support is being removed from the importer). The importer's current providers are Enable Banking, Lunch Flow, GoCardless (existing accounts only), FinTS, SimpleFIN, teller.io, basiq and Sophtron — but of these, **none of the free ones cover Monzo, Trade Republic or Trading 212** (Enable Banking's institution list was checked directly: none of the three are on it). The realistic options per bank are below.

Before importing anything, create the asset accounts in Firefly III ("Monzo", "Trade Republic", "Trading 212") with the right currencies.

### Monzo 🇬🇧

- **Recommended — Lunch Flow** (`LUNCH_FLOW_API_KEY`): paid aggregator (~$35/year) with native Data Importer support; it connects Monzo through its own commercial GoCardless agreement, so the signup closure doesn't affect it. Sign up at [lunchflow.app](https://lunchflow.app), add Monzo as a connection, create an *API destination*, put the key in Coolify. Fully automatable.
- **Free — CSV**: Monzo app → *Export & statements* → CSV monthly, imported via the importer's file workflow with a saved config from [import-configs/](import-configs/README.md).
- **DIY — Monzo Developer API**: [developers.monzo.com](https://developers.monzo.com) gives a personal access token to your own account; note the SCA rule — beyond 5 minutes after auth you can only fetch the last 90 days, so a sync script must run regularly.

### Trade Republic 🇩🇪

**No aggregator covers it** (checked: not on Enable Banking, not on Lunch Flow; the GoCardless connection is closed to new users). Options:

- **[pytr](https://github.com/pytr-org/pytr)** (unofficial API client): `pytr export_transactions` produces CSV the importer handles. Caveat: it's reverse-engineered — in mid-2026 Trade Republic's AWS-WAF bot protection started rate-limiting pytr's login; [tr-api](https://github.com/cdamken/tr-api) is a newer client that works around this via browser-cookie reuse or push-approval login. Expect occasional breakage; run it on your own machine, not unattended on the server.
- **Manual**: monthly statement/CSV from the app → file import.

### Trading 212

Trading 212 has an **official public API** (beta) — the only one of the three with first-party API access. Key facts (from [docs.trading212.com/api](https://docs.trading212.com/api)): Invest and Stocks ISA accounts only (no CFD), API key generated in the app (*Settings → API*), HTTP Basic auth (`key:secret` base64), live env `https://live.trading212.com/api/v0`, plus a demo env. Relevant endpoints: order/dividend/transaction history and **CSV report exports** (`POST /history/exports` → poll for a `downloadLink`).

- **Simplest**: *History → Export* CSV from the app → importer file workflow with a saved config.
- **Automated**: Lunch Flow also covers Trading 212 (via SnapTrade), same subscription as Monzo — making one provider cover two of your three banks.
- **DIY**: a small cron script calling the exports endpoint and POSTing the CSV to the importer's `/autoupload` — possible later without changing this stack.

### If you have a pre-2025 GoCardless account

Set `NORDIGEN_ID`/`NORDIGEN_KEY` and the GoCardless flow still works (Monzo is on it; Trade Republic was too). Existing accounts keep working for now, but treat it as borrowed time.

> Firefly III tracks money, not portfolio positions — for all three, imports represent deposits, withdrawals, fees and trade cash flows, not live stock valuations.

## Automating imports

Two levels of automation:

1. **Firefly III housekeeping** — already handled by the `cron` container (daily at 03:00).
2. **Recurring bank imports** — the importer can re-run a saved configuration via its `/autoimport` endpoint:
   - Set `AUTO_IMPORT_SECRET` (e.g. `openssl rand -hex 16`), `CAN_POST_AUTOIMPORT=true` in Coolify and redeploy.
   - Save the import configuration JSON (offered after each import) into `import-configs/` — the folder is mounted into the importer at `/import`.
   - Add a Coolify *Scheduled Task* on the `importer` service, e.g. daily:

     ```
     wget -qO- "http://localhost:8080/autoimport?directory=/import&secret=YOUR_AUTO_IMPORT_SECRET"
     ```

   This works well for API-backed connections (e.g. Lunch Flow). CSV-based flows (Trade Republic via pytr, manual Trading 212 exports) stay semi-manual since they need a fresh file each time.

## Maintenance

- **Upgrades**: images track `latest`; redeploying in Coolify pulls new versions. Firefly III migrates its database automatically. Check the [release notes](https://github.com/firefly-iii/firefly-iii/releases) before major-version jumps.
- **Backups**: snapshot the `firefly_iii_db` and `firefly_iii_upload` volumes (Coolify's backup feature or `mariadb-dump` on a schedule).
- **Consent renewal**: open-banking consents (Lunch Flow/GoCardless/Enable Banking) expire after ~90 days per bank — re-authorise when the importer or provider dashboard flags it.
