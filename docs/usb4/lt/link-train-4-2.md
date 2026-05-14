---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Link Training Phase 4 (Transmit start)

## SPECIFICATIONS

- 4.1.2.4 Phase 4 - Lane Parameters Synchronization and Transmit Start

## LINUX KERNEL

- [`drivers/thunderbolt/retimer.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/retimer.c): Retimer enumeration and management
- [`drivers/thunderbolt/sb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h): [`USB4_SB_OPCODE_ENUMERATE_RETIMERS`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h#L25) opcode

Phase 4 (lane parameter synchronization and transmit start) is handled in hardware. The CM triggers retimer enumeration via the sideband opcode.

## OTHER INFORMATION

- [USB4 Logical Layer, Re-timer and Transport Layer](https://www.usb.org/sites/default/files/D1T1-5%20-%20USB4%20Logical%20Layer%20-%20Retimer%20-%20Transport.pdf)

## DETAILS

This stage is the stage where the Lanes first send the traffic!

### Sending SLOS1 through trained Lanes

The first traffic sent over the Lanes is the `SLOS1` ordered set:


```
           ┌───────────────────────────────┐
           │         USB4 Router           │
           │ (internal functional blocks)  │
           ├───────────────────────────────┤
Host IF ──▶│Protocol Adapters (PCIe/DP/USB)│
           │                               │
           │   ┌───────────────────────┐   │
           │   │   USB4 Port (x1)      │   │
           │   │                       │   │
           │   │  High-Speed Lanes     │   │
           │   │     Lane0_TX± ────────┼───┼──────▶ "SLOS1"
           │   │     Lane0_RX± ◀───────┼───┼─────── "SLOS1"
           │   │     Lane1_TX± ────────┼───┼──────▶ "SLOS1"
           │   │     Lane1_RX± ◀───────┼───┼─────── "SLOS1"
           │   │                       │   │
           │   │  Sideband Channel     │   │
           │   │   ┌────────┐          │   │
           │   │   │ SB Regs│ SBTX ====┼===┼======▶ "RT_Resume"
           │   │   │        │ SBRX ◀===┼===┼======= "RT_Resume"
           │   │   └────────┘          │   │
           │   ├───────────────────────┤   │
           │   │  CC / Power Ctrl      │   │
           │   │     CC (detect, mux)──┼───┼──────▶
           │   │     VBUS/VCONN/GND ───┼───┼──────▶
           │   └───────────────────────┘   │
           │                               │
           └───────────────────────────────┘
```

### Emit the LT_Resume packet

On sending of the first valid `SLOS1` signal, the Routers must also send a `RT_Resume` packet via the Sideband channels, and set the Tx Active bit in its `TxFFE` Sideband Register to `1`.

```
Broadcast RT Transaction (2 DW = 8 Bytes)

┌───────────────┐  ┌───────────────────────────────────┐  ┌───────────────────────────────────┐
│ 1111 1110b    │  │ StartRT │ Bc │Idx │E3Tx│  SelGen  │  │          Link Parameters          │
│ 0xFE (DLE)    │  ├─────────┼────┼────┼────┼──────────┤  ├────────────┬───────────┬──────────┤
│ Start of RT   │  │   01    │ 1  │0000│  0 │   Gen    │  │   Lane ID  │   Rate    │  Other   │
└───────────────┘  └───────────────────────────────────┘  └────────────┴───────────┴──────────┘
                       │        │    │    │       │
                       │        │    │    │       └── Bits[7:4] = Selected Generation
                       │        │    │    └────────── Bit3      = Enable 3Tx
                       │        │    └─────────────── Bits[4:1] = Index (0000b)
                       │        └──────────────────── Bit5      = Broadcast (must =1)
                       └───────────────────────────── Bits[7:6] = StartRT (01b)
Notes:
- Lane ID   = Indicates which lane(s) the broadcast applies to
- Rate      = Defines signaling rate (Gen1 / Gen2 / Gen3 / Gen4)
- Other     = Vendor/reserved parameters depending on RT type
```

