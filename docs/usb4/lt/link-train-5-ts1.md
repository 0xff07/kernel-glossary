---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Link Training Phase 5, TS1

## SPECIFICATIONS

## LINUX KERNEL

- [`drivers/thunderbolt/sb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h): [`USB4_SB_GEN23_TXFFE`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h#L44) (TxFFE register at 0x0D) used during link equalization

Phase 5 link equalization is handled in hardware by the lane adapters and PHY. The negotiated TxFFE parameters are readable through the sideband registers.

## DETAILS

```
┌───────────────────────────────┐       ┌───────────────────────┐       ┌───────────────────────────────┐
│           DFP Router          │       │        Retimer        │       │           UFP Router          │
│        (Host Side)            │       │   (in-cable / board)  │       │     (Device / Peripheral)     │
├───────────────────────────────┤       ├───────────────────────┤       ├───────────────────────────────┤
│  High-Speed Lanes             │       │  High-Speed Lanes     │       │  High-Speed Lanes             │
│     Lane0_TX± ────────────────┼──────▶│     Lane0_TX± ────────┼──────▶│     Lane0_TX±                 │
│     Lane0_RX± ◀───────────────┼───────│───  Lane0_RX± ◀───────┼───────│───  Lane0_RX±                 │
│     Lane1_TX± ────────────────┼──────▶│     Lane1_TX± ────────┼──────▶│     Lane1_TX±                 │
│     Lane1_RX± ◀───────────────┼───────│───  Lane1_RX± ◀───────┼───────│───  Lane1_RX±                 │
│                               │       │                       │       │                               │
│  Sideband Signals             │       │  Sideband Signals     │       │  Sideband Signals             │
│     SBTX  ────────────────────┼──────▶│     SBTX ─────────────┼──────▶│     SBTX                      │
│     SBRX  ◀───────────────────┼───────│───  SBRX ◀────────────┼───────│───  SBRX                      │
│      │                        │       │      │                │       │      │                        │
│      ▼                        │       │      ▼                │       │      ▼                        │
│  ┌─────────────────────────┐  │       │  ┌─────────────────┐  │       │  ┌─────────────────────────┐  │
│  │ Sideband Register Space │  │       │  │ Retimer SB Reg  │  │       │  │ Sideband Register Space │  │
│  │                         │  │       │  │   Space         │  │       │  │                         │  │
│  │                         │  │       │  │ - status/EQ/... │  │       │  │                         │  │
│  └─────────────────────────┘  │       │  └─────────────────┘  │       │  └─────────────────────────┘  │
│                               │       │                       │       │                               │
│  CC / Power                   │       │  CC / Power           │       │  CC / Power                   │
│     CC1/CC2 ──────────────────┼──────▶│     CC1/CC2 ──────────┼──────▶│     CC1/CC2                   │
│     VBUS/VCONN/GND ───────────┼──────▶│     VBUS/VCONN/GND ───┼──────▶│     VBUS/VCONN/GND            │
└───────────────────────────────┘       └───────────────────────┘       └───────────────────────────────┘
```
