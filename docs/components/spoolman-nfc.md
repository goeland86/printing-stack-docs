# Spoolman (NFC fork)

[Spoolman](https://github.com/Donkie/Spoolman) is an upstream filament
spool tracker. The [goeland86 fork](https://github.com/goeland86/Spoolman)
adds NFC tag lookup so an arbitrary daemon can hand it raw tag bytes and
get back a `spool_id`.

| | |
|---|---|
| **Repo** | [`goeland86/Spoolman`](https://github.com/goeland86/Spoolman) |
| **Branch** | [`pr/nfc-support`](https://github.com/goeland86/Spoolman/tree/pr/nfc-support) |
| **Upstream** | [`Donkie/Spoolman`](https://github.com/Donkie/Spoolman) |
| **License** | MIT (same as upstream) |
| **Status** | PR pending merge upstream |

## What the fork adds

| Surface | Purpose |
|---------|---------|
| `POST /api/v1/nfc/lookup` | Accept raw tag bytes (UID, protocol, base64 data). Return the matched `spool_id`, or `404` if no match. |
| `SPOOLMAN_TIGERTAG_ENABLED=TRUE` env var | Enables NTAG213 (ISO 14443A) "TigerTag" decoder. |
| `SPOOLMAN_NFC_ENABLED=TRUE` env var | Enables the `/nfc/*` endpoints overall. |
| `auto_create` flow on `/nfc/lookup` | If an unrecognized OpenPrintTag (ISO 15693 / NFC-V) is scanned, optionally create a spool with sensible defaults so the operator can fill in the rest. |
| `external_id` matching for TigerTag | TigerTags encode an `id_product` field; the fork matches it against `filament.external_id`. |
| `instance_uuid` matching for OpenPrintTag | OpenPrintTags carry a UUID derived from the tag's hardware UID; the fork matches it against a per-spool `instance_uuid`. |

See [NFC tag formats](../reference/nfc-tag-formats.md) for the two tag
ecosystems.

## Running it

The simplest deployment is the upstream docker-compose with one extra
env block:

```yaml
services:
  spoolman:
    image: ghcr.io/goeland86/spoolman:pr-nfc-support   # or build from source
    ports:
      - "7912:8000"
    environment:
      SPOOLMAN_NFC_ENABLED: "TRUE"
      SPOOLMAN_TIGERTAG_ENABLED: "TRUE"
      SPOOLMAN_DB_TYPE: postgres
      SPOOLMAN_DB_HOST: postgres
      SPOOLMAN_DB_NAME: spoolman
      SPOOLMAN_DB_USERNAME: spoolman
      SPOOLMAN_DB_PASSWORD: spoolman
    depends_on:
      - postgres
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: spoolman
      POSTGRES_PASSWORD: spoolman
      POSTGRES_DB: spoolman
    volumes:
      - spoolman-data:/var/lib/postgresql/data
volumes:
  spoolman-data:
```

See [How-to: run Spoolman with NFC](../how-to/run-spoolman-nfc.md) for a
full walkthrough including building the image from the fork if no
published image is available.

## Why the patches don't live upstream yet

The NFC PR depends on:

- The tag-format decoders (TigerTag, OpenPrintTag) being baked into the
  server. Upstream's policy is to keep server logic minimal and let
  external services do tag decoding.
- A new dependency surface (NTAG/NFC-V tag parsing).

The fork is the working solution until that conversation lands upstream.
When it does, both `klipper-nfc-daemon` and `snapmaker_moonraker` will
flip their docs to point at `Donkie/Spoolman` and the fork goes away.

## Field-key conventions

The fork looks up filament profile matches using `external_id` and the
spool-specific NFC daemon writes back metadata extras using a
`nozzle_<size>` prefix (with `.` normalized to `_`, e.g.
`nozzle_0_4_pressure_advance`). Keep nozzle size consistent in the
daemon's config or the per-tool PA push won't find a match.

## Where the daemon reads vs. writes

| Endpoint | Direction | Used by |
|----------|-----------|---------|
| `POST /api/v1/nfc/lookup` | daemon → Spoolman | tag scan resolution |
| `GET /api/v1/spool/{id}` | daemon → Spoolman | fetch full spool details (material, vendor, temps, PA, color) |
| `POST /api/v1/spool` | daemon → Spoolman | auto-create on unrecognized OpenPrintTag (if `auto_create = true`) |
| `PATCH /api/v1/spool/{id}` | daemon → Spoolman | update spool's `instance_uuid` after auto-create |
| `POST /server/spoolman/spool_id` | daemon → Moonraker (which proxies to Spoolman) | set the printer's active spool |
