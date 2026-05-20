# Fleet overview

A mix of Klipper printers and one Snapmaker J1S pretending to be one
from the operator's perspective. The shared layer is everything above
the printer's TCP socket: Mainsail UI, NFC spool selection, Spoolman
tracking, and consistent `save_variables` for print-start macros.

## Block diagram

```mermaid
flowchart LR
    subgraph Operator["👤 Operator"]
        Browser["Browser<br/>(Mainsail / Fluidd)"]
        Phone["Phone<br/>(Mainsail PWA)"]
    end

    subgraph Klipper["🖨️ Klipper printers"]
        V24["Voron 2.4 — StealthChanger<br/>Recore A7 · 5 toolheads (RP2040)"]
        Trident["Voron Trident 300<br/>Recore A8"]
        V0["Voron V0<br/>Recore A6"]
        CR30["CR-30 (Klipper mod)"]
        Alcheman["Elyarchi Alcheman"]
    end

    subgraph J1S["🖨️ Snapmaker J1S"]
        Pi["Raspberry Pi<br/>(custom image)"]
        Bridge["snapmaker_moonraker<br/>(Go bridge)"]
        J1SFw["J1S stock firmware<br/>(SACP)"]
        Pi --> Bridge --> J1SFw
    end

    subgraph Readers["📇 PN532 readers (one per printer, USB-UART)"]
        R1[reader]
    end

    subgraph Shared["🌐 Shared services"]
        Spoolman["Spoolman<br/>(goeland86/Spoolman<br/>pr/nfc-support)"]
        Mainsail["Mainsail / Fluidd<br/>(static, served per host)"]
    end

    Browser --> Mainsail
    Phone --> Mainsail
    Mainsail -->|"Moonraker<br/>JSON-RPC"| V24
    Mainsail -->|"Moonraker<br/>JSON-RPC"| Trident
    Mainsail -->|"Moonraker<br/>JSON-RPC"| V0
    Mainsail -->|"Moonraker<br/>JSON-RPC"| CR30
    Mainsail -->|"Moonraker<br/>JSON-RPC"| Alcheman
    Mainsail -->|"Moonraker<br/>JSON-RPC"| Bridge

    Readers -.->|USB-UART| V24
    Readers -.->|USB-UART| Trident
    Readers -.->|USB-UART| V0
    Readers -.->|USB-UART| CR30
    Readers -.->|USB-UART| Alcheman
    Readers -.->|USB-UART| Pi

    V24 -->|REST| Spoolman
    Trident -->|REST| Spoolman
    V0 -->|REST| Spoolman
    CR30 -->|REST| Spoolman
    Alcheman -->|REST| Spoolman
    Bridge -->|REST| Spoolman

    classDef fork fill:#ffeaa7,stroke:#fdcb6e,color:#000
    classDef custom fill:#a29bfe,stroke:#6c5ce7,color:#fff
    classDef vanilla fill:#dfe6e9,stroke:#b2bec3,color:#000

    class Bridge,Spoolman fork
    class Bridge custom
    class V24,Trident,V0,CR30,Alcheman,J1SFw,Pi,Mainsail vanilla
```

Legend:

- :material-square: **Vanilla** — upstream, unmodified
- :material-square: **Fork** — patched, see [Repo map](repo-map.md)
- :material-square: **Custom** — written from scratch in this stack

## The fleet, side by side

| Aspect | Klipper printers (Voron 2.4 / Trident / V0 / CR-30 / Alcheman) | Snapmaker J1S |
|--------|----------------------------------------------------------------|---------------|
| **Firmware** | Klipper (upstream) | Snapmaker stock — proprietary, SACP over TCP |
| **Host** | Recore A6/A7/A8 (Vorons) or operator's host of choice (CR-30, Alcheman) | Raspberry Pi 3 with the [custom image](../how-to/build-j1s-image.md) |
| **Klipper-side API** | Real Moonraker | [`snapmaker_moonraker`](../components/snapmaker-bridge.md) bridge (Go) on :7125 |
| **Toolheads** | 5 (Voron 2.4 / StealthChanger) or 1 (all others) | 2 × built-in extruders |
| **Macros / `[respond]` / `[save_variables]`** | Native (real Klipper) | [Intercepted by bridge](../reference/bridge-intercepts.md) |
| **Mainsail prompt dialogs** | Yes (real `notify_gcode_response`) | Yes (broadcast by bridge from intercepted RESPOND) |
| **NFC reader** | PN532 on USB-UART (one per printer) | PN532 on USB-UART (built into image) |

The point of the bridge is that **from Mainsail's perspective there is no
difference between the J1S and any Klipper printer**. The same macros,
the same `save_variables` queries, the same prompt dialogs work. That's
what makes a single fleet-wide NFC daemon possible.

## Where the NFC stack plugs in

Every printer in the fleet runs an identical `klipper-nfc-daemon`
instance. The daemon doesn't know or care whether it's talking to real
Klipper or the bridge — it speaks Moonraker JSON-RPC + emits Klipper
macros, and the underlying host (real Klipper or bridge) does the right
thing.

```mermaid
flowchart LR
    Tag[("NFC tag<br/>NTAG213 / NFC-V")] -.->|RF| Reader[PN532 reader]
    Reader -->|USB-UART| Daemon["klipper-nfc-daemon<br/>(Python)"]
    Daemon -->|"POST /api/v1/nfc/lookup"| Spoolman
    Daemon -->|"WS notify_gcode_response<br/>SAVE_VARIABLE<br/>RESPOND TYPE=command"| Host["Moonraker<br/>(real or bridge)"]
    Spoolman -->|"spool_id"| Daemon
    Host -->|"notify_gcode_response"| UI[Mainsail prompt dialog]
    UI -->|"NFC_ASSIGN_TOOL T=N"| Host
```

The single daemon handles both single-tool ("J1S left extruder")
and multi-tool ("Voron T2") spool assignments — see [NFC spool
selection workflow](../workflows/nfc-spool.md).

## Read next

- [Data flow diagrams](data-flow.md) — every cross-process message labelled
- [Repo map](repo-map.md) — what's upstream, what's a fork, what's
  bespoke, and why
