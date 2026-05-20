# NFC spool selection

The end-to-end story of "tap a spool, the printer knows what it is."

## Single-tool mode

```mermaid
sequenceDiagram
    autonumber
    actor U as Operator
    participant R as NFC reader
    participant D as klipper-nfc-daemon
    participant SM as Spoolman
    participant H as Moonraker / Bridge
    participant K as Klipper / Bridge internal
    participant UI as Mainsail

    U->>R: physically tap spool on reader
    R-->>D: tag UID + raw bytes
    D->>D: debounce check (skip if same tag within window)
    D->>SM: POST /api/v1/nfc/lookup<br/>{uid, protocol, data}
    SM-->>D: spool_id = 42
    D->>SM: GET /api/v1/spool/42
    SM-->>D: PolyTerra PLA Red, 210/60, PA 0.04
    D->>H: POST /server/spoolman/spool_id {id: 42}
    H->>K: persist active spool
    D->>H: SAVE_VARIABLE nfc_material '"PLA"'
    D->>H: SAVE_VARIABLE nfc_extruder_temp 210
    D->>H: SAVE_VARIABLE nfc_bed_temp 60
    Note over D,H: …rest of metadata
    D->>H: update Mainsail preheat preset (optional)
    H-->>UI: notify_status_update (active spool changed)
    UI-->>U: spool name shown in Mainsail spool panel
```

Operator sees nothing other than the spool name updating in the
Mainsail spool panel and (optionally) a new preheat preset.

## Multi-tool mode

The daemon doesn't know which physical tool the just-scanned spool
belongs to. It asks the operator via a Mainsail prompt dialog:

```mermaid
sequenceDiagram
    autonumber
    actor U as Operator
    participant R as NFC reader
    participant D as klipper-nfc-daemon
    participant SM as Spoolman
    participant H as Moonraker / Bridge
    participant UI as Mainsail

    U->>R: tap spool
    R-->>D: tag UID + raw bytes
    D->>SM: POST /api/v1/nfc/lookup
    SM-->>D: spool_id = 42
    D->>SM: GET /api/v1/spool/42
    SM-->>D: spool details

    Note over D,H: Stash pending data in _NFC_STATE macro vars
    D->>H: SET_GCODE_VARIABLE MACRO=_NFC_STATE VARIABLE=pending_spool_id VALUE=42
    D->>H: SET_GCODE_VARIABLE … pending_material '"PLA"'
    D->>H: SET_GCODE_VARIABLE … pending_vendor '"PolyTerra"'
    D->>H: SET_GCODE_VARIABLE … pending_extruder_temp 210
    Note over D,H: …rest of metadata

    Note over D,H: Build prompt
    D->>H: RESPOND TYPE=command MSG="action:prompt_begin Assign spool"
    D->>H: RESPOND TYPE=command MSG="action:prompt_text PolyTerra PLA Red"
    D->>H: RESPOND TYPE=command MSG="action:prompt_button T0|NFC_ASSIGN_TOOL T=0"
    D->>H: RESPOND TYPE=command MSG="action:prompt_button T1|NFC_ASSIGN_TOOL T=1"
    D->>H: RESPOND TYPE=command MSG="action:prompt_button Cancel|NFC_CANCEL"
    D->>H: RESPOND TYPE=command MSG="action:prompt_show"

    H-->>UI: notify_gcode_response "// action:prompt_*"
    UI->>U: modal: "Assign spool — PolyTerra PLA Red" [T0] [T1] [Cancel]

    U->>UI: tap T1
    UI->>H: printer.gcode.script {script: "NFC_ASSIGN_TOOL T=1"}

    Note over H: Real Klipper: macro fires.<br/>Bridge: intercepted natively.
    H->>SM: PATCH /api/v1/spool/42 (tool=1)
    H->>H: SAVE_VARIABLE nfc_t1_material '"PLA"'
    H->>H: SAVE_VARIABLE nfc_t1_extruder_temp 210
    Note over H: …per-tool metadata
    H-->>UI: notify_gcode_response "// action:prompt_end"
    H-->>UI: notify_gcode_response "// Spool #42 assigned to T1"
    UI->>U: dialog closes, console shows assignment
```

## Why `_NFC_STATE` exists

The prompt dialog and the assignment command are decoupled — the
daemon broadcasts the prompt and then returns to its poll loop. When
the operator eventually clicks a tool button, the resulting
`NFC_ASSIGN_TOOL` macro/intercept needs to know which spool to assign.

The daemon stashes the pending data in Klipper variables under a fake
macro named `_NFC_STATE`:

```
SET_GCODE_VARIABLE MACRO=_NFC_STATE VARIABLE=pending_spool_id VALUE=42
SET_GCODE_VARIABLE MACRO=_NFC_STATE VARIABLE=pending_material VALUE='"PLA"'
…
```

On real Klipper this needs a stub macro:

```ini
[gcode_macro _NFC_STATE]
variable_pending_spool_id: 0
variable_pending_material: ""
# …etc, in nfc_macros.cfg
gcode:
    # never executed; storage only
```

On the bridge there is no macro — the bridge intercepts every
`SET_GCODE_VARIABLE MACRO=_NFC_STATE …` and stores the field in its
own in-memory `NFCState` (see
[Bridge intercepts](../reference/bridge-intercepts.md)).

## Cancel flow

Clicking **Cancel** sends `NFC_CANCEL` which clears the pending state
and emits `action:prompt_end`:

```
// action:prompt_end
// NFC spool assignment cancelled
```

Same flow on real Klipper (macro) and bridge (intercept).

## Debounce

The daemon's `debounce_time` (default 5 s) suppresses re-trigger from
the same tag UID. Useful because PN532 readers will happily report the
same tag once per poll while a spool sits on the reader.

## What happens if Spoolman is unreachable

The daemon logs an error, skips the rest of the flow, and goes back to
polling. No partial state written to the printer. The operator sees
nothing in Mainsail — they should check `journalctl -u nfc-spoolman -f`.

## What happens if the tag isn't in Spoolman

- **TigerTag (NTAG213)**: no match → logged, ignored. Use Spoolman's
  UI to create a filament profile with the right `external_id`.
- **OpenPrintTag (NFC-V)**: if `auto_create = true`, the daemon
  auto-creates a placeholder spool linked to the tag's UUID and writes
  the lookup back to Spoolman; otherwise same as TigerTag.

## Per-printer reader

Each printer in the fleet has its own NFC reader, each running its own
`nfc-spoolman` daemon, each pointed at the same shared Spoolman
instance. That way assignments are always scoped to the printer the
operator is standing in front of.
