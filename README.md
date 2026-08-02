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

The Data Importer talks to banks through **GoCardless Bank Account Data** (formerly Nordigen) — free for up to 50 connected accounts.

**One-time setup:**

1. Create an account at <https://bankaccountdata.gocardless.com> (this is the *Bank Account Data* portal, not regular GoCardless payments).
2. Create a **User secret** → set `NORDIGEN_ID` (secret ID) and `NORDIGEN_KEY` (secret key) in Coolify and redeploy.
3. In Firefly III, create your asset accounts (e.g. "Monzo", "Trade Republic", "Trading 212") with the right currencies, so imports have somewhere to land.

### Monzo 🇬🇧

Fully supported via GoCardless. In the importer: *Import from GoCardless* → country **United Kingdom** → **Monzo** → approve in the Monzo app → map the account to your Firefly III "Monzo" asset account. Bank consent lasts ~90 days, then you re-approve.

### Trade Republic 🇩🇪

Trade Republic is a licensed German bank; look for it in the importer under country **Germany**. Note that open-banking access covers the **cash account** — individual securities trades typically show up only as the cash movements they cause.

If it's missing from the GoCardless list (coverage changes over time), the reliable fallback is [pytr](https://github.com/pytr-org/pytr), which exports your Trade Republic history to CSV for the importer's CSV workflow.

### Trading 212

Trading 212 is an investment platform, not a PSD2 payment bank, so it is generally **not** available through open banking. Use its built-in export instead: in Trading 212 go to *History → Export* (CSV, choose your date range) and run it through the importer's **File import**. On the first run, map the columns and accounts, then download the generated **import configuration JSON** and save it in [import-configs/](import-configs/README.md) so subsequent imports are one click (or fully automated, below).

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

   This works well for the GoCardless connections (Monzo, Trade Republic). Trading 212 stays semi-manual since it needs a fresh CSV each time.

## Maintenance

- **Upgrades**: images track `latest`; redeploying in Coolify pulls new versions. Firefly III migrates its database automatically. Check the [release notes](https://github.com/firefly-iii/firefly-iii/releases) before major-version jumps.
- **Backups**: snapshot the `firefly_iii_db` and `firefly_iii_upload` volumes (Coolify's backup feature or `mariadb-dump` on a schedule).
- **Consent renewal**: GoCardless bank consents expire after ~90 days per bank — the importer will tell you when a connection needs re-authorising.
