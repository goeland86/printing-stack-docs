# Manage NFC tags

Everything around the physical sticker: which kind to use, how to
write one, how to bind it to a spool, what to do when a spool runs
out, and how to safely re-use the tag for the next spool.

For the *daemon* side of tags (decoders, matching logic, byte layouts)
see [reference/nfc-tag-formats.md](../reference/nfc-tag-formats.md).
This page is operator-facing.

## Picking a tag type

| Tag type | Chip | Reader requirement | Writable by you? | Best for |
|----------|------|--------------------|--------------------|----------|
| **TigerTag** | NTAG213 (ISO 14443A) | Any PN532 / PN5180 / ACR1552U | No — vendor-written | Spools that ship with a TigerTag already |
| **OpenPrintTag** | NFC-V / ICODE SLIX2 (ISO 15693) | PN5180 / ACR1552U (NOT PN532) | Yes | Any spool you want to tag yourself |

For this fleet, the reference reader is the **PN532**, which means
**OpenPrintTag is unusable on a PN532-only printer**. If you want to
self-tag spools on a PN532 fleet, write a **NTAG213 sticker** with a
TigerTag-compatible payload — see below.

## Where to source blank stickers

- **NTAG213 stickers** — generic NFC supplier; sold as round
  cardboard-friendly stickers. Pick a size that fits on the spool
  side label without overhanging the rim.
- **NFC-V (ICODE SLIX2) stickers** — same suppliers, slightly more
  expensive; only useful if any printer in your fleet has a PN5180 or
  ACR1552U reader.

## Binding a tag to a spool

### Tag-first vs. spool-first

Two valid orderings:

1. **Spool-first** (recommended): create the spool in Spoolman
   ([inventory how-to](manage-spool-inventory.md)), then write a tag
   carrying its identifier, then stick the tag.
2. **Tag-first** (auto-create): tap a blank NFC-V on the reader with
   `auto_create = true` in the daemon config; Spoolman creates a
   placeholder spool, you fill in details afterwards.

Spool-first is cleaner if you're processing a batch of new spools
methodically. Tag-first is faster when you've already got the spool in
your hand and don't want to context-switch to the Spoolman UI first.

### Writing a TigerTag-compatible NTAG213

1. Get the spool's filament `external_id` from Spoolman (or pick one
   when you create the filament).
2. Use a tag writer that supports raw byte writes — recommended:
    - **NFC Tools** on Android (free / paid pro version)
    - **NXP TagWriter** on Android (free, NXP-official)
    - **libnfc / nfc-mfclassic** from the command line
3. Write the TigerTag-formatted payload — the byte layout the
   [Spoolman fork](../components/spoolman-nfc.md) expects. The exact
   layout lives in the fork's decoder source; if you don't have a
   pre-built writer, the simplest path is to use the daemon's
   `auto_create` path on an NFC-V tag instead.

!!! tip "Daemon-side writing is a known gap"
    The `klipper-nfc-daemon` reads tags but doesn't write them today.
    Writing happens out-of-band with the tools above. If batch tag
    writing becomes routine for this fleet, a `WRITE_TAG` command
    inside the daemon is the obvious follow-up project.

### Writing an OpenPrintTag

1. Create the spool in Spoolman.
2. Use a PN5180 / ACR1552U-backed writer (NFC-V) — same options as
   TigerTag plus dedicated OpenPrintTag tooling if you have it.
3. Write the OpenPrintTag-formatted payload with `instance_uuid` set
   to a value you also paste into the spool's `extra.instance_uuid`
   field in Spoolman.

Or skip steps 2 and 3 with `auto_create = true`:

1. Tap a blank NFC-V on a printer's reader.
2. The daemon POSTs to Spoolman; Spoolman creates a spool with a
   fresh `instance_uuid` and writes it back into the tag's payload
   bookkeeping.
3. Open the new spool in Spoolman and fill in vendor / filament /
   colour / temps.

## Sticking the tag

Practical tips that prevent future re-scan failures:

- **Flat surface**: the cardboard / plastic side label is best. Avoid
  the curved metal core (RF detuning).
- **Away from the spool's own metal**: NFC range collapses next to
  conductors.
- **Operator-side**: place where it's easy to bring the spool to the
  reader without rotating awkwardly.
- **Mark spent stickers**: when you archive a spool (see below), put
  a sharpie X through the *old label* the sticker was on if you're
  re-binding the same physical sticker to a new spool. Avoids
  confusion mid-shelf.

## Re-using a tag from an archived spool

Empty spools get archived in Spoolman (not deleted — usage history is
worth keeping). The physical NFC sticker, however, has no reason to
be thrown away. Two paths:

### Path 1 — peel and re-stick on a new spool

Best when the sticker is in good condition and the new spool is the
**same SKU** as the old one.

1. Verify the original spool is **archived** in Spoolman.
2. Peel the sticker off the empty spool (or just bring the whole core
   over).
3. On the new spool, you have two sub-options:
    - **If the same filament profile applies and a new spool entry
      already exists**: re-bind by updating the new spool's
      `extra.instance_uuid` (OpenPrintTag) to match the tag's UUID,
      so a tap resolves to the new spool.
    - **Same SKU, no new spool entry yet**: create a new spool entry
      first, then re-bind.

### Path 2 — wipe and re-write

Best when the sticker is going onto a **different SKU**, or you want
clean tooling state.

1. **For NTAG213 / TigerTag-formatted**: re-write the tag with the
   new spool's `external_id` payload using the same writer tooling
   from earlier in this page.
2. **For NFC-V / OpenPrintTag**: simpler — change the spool's
   `extra.instance_uuid` in Spoolman to the existing tag UUID
   (no rewrite needed) **or** re-write with a fresh UUID and update
   Spoolman. Either works.

### Path 3 — keep the same UUID, different filament

NFC-V tags' UUID is derived from hardware UID, so the tag's identity
never changes. If you peel a tag off Spool A (archived) and stick it
on Spool B, just update Spoolman:

1. Old spool (A) — already archived; its `instance_uuid` is harmless.
2. New spool (B) — set its `extra.instance_uuid` to the same value
   that's currently on A.
3. Either clear A's `instance_uuid` or rely on the fact that A is
   archived so the daemon's lookup prefers the non-archived B.

!!! warning "Don't leave two active spools with the same instance_uuid"
    If both A and B are non-archived with the same `instance_uuid`,
    the daemon's behaviour depends on Spoolman's match ordering and
    will likely surprise you. **Always archive the old spool first,
    then re-bind the new one.**

## When a tag stops working

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Reader sees nothing | Sticker damaged / detuned by spool's metal core | Re-place on a flatter / non-metal area, or replace the sticker entirely |
| Reader sees the tag but Spoolman returns 404 | Spool archived, or `external_id` / `instance_uuid` mismatch | Verify in Spoolman the spool is non-archived and the matching field is set |
| Two spools resolve to the same UID | Duplicate `instance_uuid` after re-use | Archive the old, leave the new |
| Range is intermittent | Reader → tag distance too far, or RF noise | Tap closer / slower; verify with `nfc-list` |

## See also

- [Manage spool inventory in Spoolman](manage-spool-inventory.md)
- [Install klipper-nfc-daemon](install-nfc-daemon.md)
- [NFC tag formats reference](../reference/nfc-tag-formats.md)
- [Operator workflow](../workflows/operator-workflow.md) — where this all lands in practice
