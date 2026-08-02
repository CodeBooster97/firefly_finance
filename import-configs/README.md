# Import configurations

## monzo.json

Matches the 16-column Monzo app export (Transaction ID ... Category split),
as produced by Monzo's CSV export and its Google Sheets auto-export.
Duplicate detection keys on the Transaction ID column, so re-importing a
growing sheet only adds new transactions. On the first UI import, select your
Monzo asset account (the config's `default_account` is unset); afterwards you
can download the completed config from the importer and overwrite this file to
make it fully non-interactive.

Automated pull from the Google Sheet (Coolify Scheduled Task on the
`importer` service, requires `AUTO_IMPORT_SECRET`, `CAN_POST_FILES=true`,
`CAN_POST_AUTOIMPORT=true` and `FIREFLY_III_ACCESS_TOKEN` to be set):

```
curl -s "https://docs.google.com/spreadsheets/d/<SHEET_ID>/export?format=csv&gid=<GID>" -o /tmp/monzo.csv && curl -s -X POST "http://localhost:8080/autoupload?secret=$AUTO_IMPORT_SECRET" -F "importable=@/tmp/monzo.csv" -F "json=@/import/monzo.json"
```

Store Firefly III Data Importer JSON configuration files here. After you run an
import once through the importer UI, it offers the configuration as a JSON
download — save it in this directory (e.g. `monzo.json`, `trading212-csv.json`).

This directory is mounted into the importer container at `/import`
(`IMPORT_DIR_ALLOWLIST=/import`), so saved configurations can be reused for
scheduled imports via the `/autoimport` endpoint (see the main README).

Note: these files can contain GoCardless requisition/account identifiers and
your Firefly III account IDs. No passwords or API keys, but treat them as
private — keep this repository private.
