# Import configurations

Store Firefly III Data Importer JSON configuration files here. After you run an
import once through the importer UI, it offers the configuration as a JSON
download — save it in this directory (e.g. `monzo.json`, `trading212-csv.json`).

This directory is mounted into the importer container at `/import`
(`IMPORT_DIR_ALLOWLIST=/import`), so saved configurations can be reused for
scheduled imports via the `/autoimport` endpoint (see the main README).

Note: these files can contain GoCardless requisition/account identifiers and
your Firefly III account IDs. No passwords or API keys, but treat them as
private — keep this repository private.
