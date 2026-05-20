# Data flow

Every cross-process message in the stack, labelled with the actual
protocol and endpoint.

## Mainsail ⇄ host

```mermaid
sequenceDiagram
    participant UI as Mainsail
    participant H as Moonraker / Bridge

    UI->>H: WebSocket connect /websocket
    H-->>UI: notify_status_update (server.info)

    loop polling + push
        UI->>H: printer.objects.query (toolhead, extruder, …)
        H-->>UI: status snapshot
        H-->>UI: notify_status_update (deltas)
        H-->>UI: notify_gcode_response (// …)
    end

    UI->>H: printer.gcode.script {script: "G28"}
    H-->>UI: ok / error
```

Identical for real Moonraker and the bridge. The bridge implements the
same JSON-RPC namespace (`printer.objects.query`,
`printer.gcode.script`, `server.files.upload`, etc.) and the same
websocket notification stream.

See [Bridge intercepts](../reference/bridge-intercepts.md) for the
specific gcode commands the bridge handles natively versus passing
through.

## klipper-nfc-daemon ⇄ Spoolman

```mermaid
sequenceDiagram
    participant D as klipper-nfc-daemon
    participant SM as Spoolman (NFC fork)

    Note over D: NFC tag scanned, raw bytes in hand

    D->>SM: POST /api/v1/nfc/lookup<br/>{uid, protocol, data (base64)}
    SM-->>D: 200 {spool_id, format: "tigertag" | "openprinttag"}

    D->>SM: GET /api/v1/spool/{spool_id}
    SM-->>D: {filament, vendor, color_hex, extruder_temp, bed_temp, …}
```

The lookup endpoint and the env vars (`SPOOLMAN_TIGERTAG_ENABLED`,
`SPOOLMAN_NFC_ENABLED`) live only on the
[`pr/nfc-support`](https://github.com/goeland86/Spoolman/tree/pr/nfc-support)
branch of the fork — see [Spoolman fork](../components/spoolman-nfc.md).

## klipper-nfc-daemon ⇄ host (single-tool mode)

```mermaid
sequenceDiagram
    participant D as klipper-nfc-daemon
    participant H as Moonraker / Bridge
    participant K as Klipper / Bridge internal

    D->>H: POST /server/spoolman/spool_id {spool_id: 42}
    H->>K: persist active spool
    H-->>D: 200

    D->>H: POST /printer/gcode/script<br/>{script: "SAVE_VARIABLE VARIABLE=nfc_material VALUE='\"PLA\"'"}
    H->>K: SAVE_VARIABLE …
    H-->>D: 200

    D->>H: POST /printer/gcode/script {script: "SAVE_VARIABLE VARIABLE=nfc_extruder_temp VALUE=210"}
    H-->>D: 200

    Note over D,H: …one POST per variable (see Save Variables reference)
```

## klipper-nfc-daemon ⇄ host ⇄ UI (multi-tool mode)

The interesting case — the daemon doesn't know which tool the spool
goes on; the operator does. Solved with a Mainsail prompt dialog:

```mermaid
sequenceDiagram
    participant D as klipper-nfc-daemon
    participant H as Moonraker / Bridge
    participant UI as Mainsail

    D->>H: SET_GCODE_VARIABLE MACRO=_NFC_STATE VARIABLE=pending_spool_id VALUE=42
    D->>H: SET_GCODE_VARIABLE … pending_material '"PLA"'
    D->>H: SET_GCODE_VARIABLE … pending_vendor '"PolyTerra"'
    Note over D,H: …one per metadata field

    D->>H: RESPOND TYPE=command MSG="action:prompt_begin Assign spool"
    D->>H: RESPOND TYPE=command MSG="action:prompt_text PolyTerra PLA Red"
    D->>H: RESPOND TYPE=command MSG="action:prompt_button T0|NFC_ASSIGN_TOOL T=0"
    D->>H: RESPOND TYPE=command MSG="action:prompt_button T1|NFC_ASSIGN_TOOL T=1"
    D->>H: RESPOND TYPE=command MSG="action:prompt_button Cancel|NFC_CANCEL"
    D->>H: RESPOND TYPE=command MSG="action:prompt_show"

    H-->>UI: notify_gcode_response "// action:prompt_*"
    UI->>UI: render modal with buttons

    UI->>H: printer.gcode.script {script: "NFC_ASSIGN_TOOL T=1"}
    H->>H: NFC_ASSIGN_TOOL handler<br/>(real macro on Klipper,<br/>intercepted on bridge)
    H->>H: SetSpoolID(42, tool=1)<br/>SAVE_VARIABLE nfc_t1_*
    H-->>UI: notify_gcode_response "// action:prompt_end"
    H-->>UI: notify_gcode_response "// Spool #42 assigned to T1"
```

On real Klipper the `NFC_ASSIGN_TOOL` macro lives in `nfc_macros.cfg`
(shipped with the daemon). On the bridge it's
[intercepted natively](../reference/bridge-intercepts.md) — no real
Klipper required.

## J1S upload pipeline (bridge-only)

```mermaid
flowchart LR
    Slicer[PrusaSlicer<br/>OrcaSlicer<br/>Mainsail] -->|"HTTP multipart<br/>POST /server/files/upload"| MR["MultipartReader<br/>(stream)"]
    MR -->|"io.Copy"| Disk1[("gcodes/foo.gcode<br/>on disk")]
    Disk1 -->|"ProcessFile<br/>(2-pass scan)"| GCP[gcode<br/>post-processor]
    GCP -->|"streamed"| Disk2[("gcodes/foo.gcode.tmp<br/>V1 header + body")]
    Disk2 -->|"ReadAt 60 KB chunks<br/>+ md5 streaming"| SACP[SACP upload<br/>over TCP :8888]
    SACP -->|"chunked"| J1S[Snapmaker J1S]
```

Pre-v1.6.0 each stage held the entire file in RAM, which OOM-killed the
Pi on 250 MB tile prints. The current pipeline never holds more than a
few KB. See the
[v1.6.0 release notes](https://github.com/goeland86/snapmaker_moonraker/blob/main/Release_Notes.md)
for the gory details.

## Toolchange (StealthChanger, Voron)

```mermaid
sequenceDiagram
    participant G as GCode (T1)
    participant K as Klipper
    participant TC as klipper-toolchanger
    participant T0 as Active tool T0
    participant T1 as Next tool T1

    G->>K: T1
    K->>TC: tool change to 1
    TC->>T0: park sequence (Z hop, dock approach, release)
    TC->>K: switch active extruder to T1
    TC->>T1: pickup sequence (approach, latch, leave dock)
    TC->>K: apply T1 offsets (X, Y, Z, pressure_advance)
    TC->>K: restore Z, resume motion
    K-->>G: ok
```

The toolchange itself is entirely upstream — see
[viesturz/klipper-toolchanger](https://github.com/viesturz/klipper-toolchanger).
What's stack-specific is that the NFC-loaded per-tool metadata
(`nfc_t{N}_extruder_temp`, `nfc_t{N}_pressure_advance`, etc.) feeds the
toolchange macros — see [Toolchange workflow](../workflows/toolchange.md).
