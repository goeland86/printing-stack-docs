# Run Spoolman with NFC support

Spoolman is the database backend that maps NFC tags → spool IDs and
tracks usage per printer, per tool. We run the
[goeland86 fork](../components/spoolman-nfc.md) until the NFC PR lands
upstream.

## Quickest path — Docker Compose

```yaml
# docker-compose.yml
services:
  spoolman:
    build:
      context: https://github.com/goeland86/Spoolman.git#pr/nfc-support
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
    restart: unless-stopped

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: spoolman
      POSTGRES_PASSWORD: spoolman
      POSTGRES_DB: spoolman
    volumes:
      - spoolman-data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  spoolman-data:
```

```bash
docker compose up -d
```

Wait for `docker compose logs spoolman` to settle, then hit
`http://your-host:7912/` for the UI.

!!! note "Why build from source"
    There is no official published image for the NFC fork. Building
    from the branch tip is the recommended path until the PR merges
    upstream. The build is cached after the first run.

## Quickest path — bare metal

```bash
git clone -b pr/nfc-support https://github.com/goeland86/Spoolman.git
cd Spoolman
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# SQLite for testing (skip postgres if you're not running a fleet)
export SPOOLMAN_DB_TYPE=sqlite
export SPOOLMAN_NFC_ENABLED=TRUE
export SPOOLMAN_TIGERTAG_ENABLED=TRUE

uvicorn spoolman.main:app --host 0.0.0.0 --port 7912
```

For production use, drop into a systemd unit and switch to Postgres.

## Verify NFC is enabled

```bash
curl -sI http://localhost:7912/api/v1/nfc/lookup
# Expect HTTP 405 Method Not Allowed (it's POST-only) — NOT 404
# 404 means SPOOLMAN_NFC_ENABLED isn't set
```

## Register filaments + spools

The standard Spoolman UI flow:

1. **Vendor**: e.g. "PolyTerra"
2. **Filament**: e.g. "PolyTerra PLA — Red"
    - Fill in extruder/bed temperatures
    - For TigerTag spools: set `external_id` to the TigerTag
      `id_product` value (printed on the tag QR or readable via the
      vendor's app)
3. **Spool**: an instance of a filament
    - For OpenPrintTag: the spool's `instance_uuid` is populated
      automatically on first scan

## Register a printer in Spoolman

So usage is tracked per printer:

1. Spoolman UI → Settings → Printers → Add
2. Give it a name matching your printer's hostname or a friendly label
3. Each `klipper-nfc-daemon` instance picks up its printer name from
   `printer_name` in `nfc_spoolman.cfg` (falls back to the host's
   hostname)

## Per-spool extras the daemon writes back

When a tag is scanned, the daemon pushes nozzle-specific extras to the
matched spool:

| Spoolman extra key | Source | Used for |
|--------------------|--------|----------|
| `nozzle_0_4_pressure_advance` (etc.) | Daemon writes Klipper-side PA back to Spoolman after tuning | Round-trip so a freshly tuned PA persists across printer reboots |
| `tigertag_id_product` | Decoded from NTAG213 tag | Used to disambiguate filaments with the same vendor+name |

The nozzle-size encoding uses `_` instead of `.` (Spoolman key
constraint), so `0.4` becomes `0_4`. The daemon does this normalization.

## Backup

```bash
docker exec postgres pg_dump -U spoolman spoolman > spoolman-backup.sql
```

Or for SQLite:

```bash
cp spoolman.db spoolman.db.backup-$(date +%Y%m%d)
```

## Upgrade path

When the NFC PR merges upstream:

1. Switch the Docker `build:` line to `image: ghcr.io/donkie/spoolman:latest`
2. Drop the `SPOOLMAN_TIGERTAG_ENABLED` env var if upstream defaults
   change (check release notes)
3. `docker compose pull && docker compose up -d`

The database schema should migrate cleanly — but back up first.
