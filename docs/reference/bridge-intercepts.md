# Bridge intercepts

The [snapmaker_moonraker bridge](../components/snapmaker-bridge.md)
intercepts a specific set of GCode commands natively rather than
forwarding them to the J1S over SACP — because they're Klipper-isms
the J1S doesn't understand but Mainsail and the NFC daemon depend on.

Everything *not* in this table is forwarded as-is.

## Intercepted commands

| Command | Handler | Behavior |
|---------|---------|----------|
| `RESPOND TYPE=command MSG="…"` | `handleRespond` | Broadcast as `notify_gcode_response` with `// ` prefix. Mainsail's prompt-dialog renderer keys off these. |
| `RESPOND MSG="…"` | `handleRespond` | Same broadcast, used for plain console messages. |
| `SET_GCODE_VARIABLE MACRO=_NFC_STATE VARIABLE=… VALUE=…` | `handleSetGCodeVariable` | Store the field in the bridge's `NFCState` (in-memory) for use by `NFC_ASSIGN_TOOL`. VALUE is unquoted (handles `'"…"'` wrapping). |
| `SET_GCODE_VARIABLE MACRO=<other> …` | (drop) | Silently ignored — the bridge only knows `_NFC_STATE`. |
| `NFC_ASSIGN_TOOL T=<n>` *or* `NFC_ASSIGN_TOOL TOOL=<n>` | `handleNFCAssignTool` | Read pending data from `NFCState`, call Spoolman `SetSpoolID(spool, tool)`, persist `nfc_t{n}_*` to the `save_variables` database namespace, emit `action:prompt_end`, clear pending. |
| `NFC_CANCEL` | `handleNFCCancel` | Clear pending NFC state, emit `action:prompt_end`, log cancellation. |
| `SAVE_VARIABLE VARIABLE=… VALUE=…` | (database) | Stored in the `save_variables` namespace of the bridge's persistent DB. Exposed via the `printer.save_variables.variables` object. |

## Multi-line scripts

The Klipper `printer.gcode.script` JSON-RPC method accepts multi-line
scripts. The bridge handles this with an `isKlipperCommand` gate +
`interceptSingleGCode` dispatcher: each line is checked against the
intercept table, intercepted lines are handled natively, the rest are
forwarded as a (possibly empty) script to SACP.

This matters because the NFC daemon pushes the entire prompt as a
single multi-line script:

```
SET_GCODE_VARIABLE MACRO=_NFC_STATE VARIABLE=pending_spool_id VALUE=42
SET_GCODE_VARIABLE MACRO=_NFC_STATE VARIABLE=pending_material VALUE='"PLA"'
…
RESPOND TYPE=command MSG="action:prompt_begin Assign spool"
RESPOND TYPE=command MSG="action:prompt_text PolyTerra PLA Red"
RESPOND TYPE=command MSG="action:prompt_button T0|NFC_ASSIGN_TOOL T=0"
RESPOND TYPE=command MSG="action:prompt_button T1|NFC_ASSIGN_TOOL T=1"
RESPOND TYPE=command MSG="action:prompt_button Cancel|NFC_CANCEL"
RESPOND TYPE=command MSG="action:prompt_show"
```

Every line above is intercepted; nothing is sent to the J1S.

## `printer.save_variables` object

The bridge exposes a `save_variables` object in its
`printer.objects.query` response:

```json
{
    "save_variables": {
        "variables": {
            "nfc_t0_spool_id": 42,
            "nfc_t0_material": "PLA",
            "nfc_t1_spool_id": 87,
            "nfc_t1_material": "ASA",
            "..."
        }
    }
}
```

Backed by the bridge's persistent database namespace `save_variables`.
Reads via Mainsail's printer-object query, writes via the
`SAVE_VARIABLE` intercept above. Read by macros via the standard
Klipper `printer.save_variables.variables.<name>` pattern — so macros
written for real Klipper Just Work.

## Source

The handlers live in
[`moonraker/nfc.go`](https://github.com/goeland86/snapmaker_moonraker/blob/main/moonraker/nfc.go)
and
[`moonraker/handler_printer.go`](https://github.com/goeland86/snapmaker_moonraker/blob/main/moonraker/handler_printer.go).
The `PrinterObjects.SaveVariables()` method in
[`moonraker/objects.go`](https://github.com/goeland86/snapmaker_moonraker/blob/main/moonraker/objects.go)
exposes the persistent map.
