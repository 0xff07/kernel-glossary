---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Link Training Phase 4 (Re-timer enumeration)

## SPECIFICATIONS

- 4.1.2.4 Phase 4 - Lane Parameters Synchronization and Transmit Start

## LINUX KERNEL

- [`drivers/thunderbolt/retimer.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/retimer.c): Retimer enumeration and management
- [`drivers/thunderbolt/sb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h): [`USB4_SB_OPCODE_ENUMERATE_RETIMERS`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h#L25) opcode

Phase 4 (lane parameter synchronization and transmit start) is handled in hardware. The CM triggers retimer enumeration via the sideband opcode.

## OTHER INFORMATION

- [USB4 Logical Layer, Re-timer and Transport Layer](https://www.usb.org/sites/default/files/D1T1-5%20-%20USB4%20Logical%20Layer%20-%20Retimer%20-%20Transport.pdf)

## DETAILS

### Re-timer Enumeration

The Router broad cast a Broadcast RT packet through the Sideband Signal:

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
           │   │     Lane0_TX± ────────┼───┼──────▶
           │   │     Lane0_RX± ◀───────┼───┼───────
           │   │     Lane1_TX± ────────┼───┼──────▶
           │   │     Lane1_RX± ◀───────┼───┼───────
           │   │                       │   │
           │   │  Sideband Channel     │   │
           │   │   ┌────────┐          │   │
           │   │   │ SB Regs│ SBTX ====┼===┼======▶ "Broadcast RT"
           │   │   │        │ SBRX ◀===┼===┼======= "Broadcast RT"
           │   │   └────────┘          │   │
           │   ├───────────────────────┤   │
           │   │  CC / Power Ctrl      │   │
           │   │     CC (detect, mux)──┼───┼──────▶
           │   │     VBUS/VCONN/GND ───┼───┼──────▶
           │   └───────────────────────┘   │
           │                               │
           └───────────────────────────────┘
```

Whenever a Re-timer receive this packet, it intrements the index field in `STX` by onem and pass down to the next Re-timer. This essentially lets the re-timers do count-off. The count-off number then becomes its Re-timer index:

```
┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
│ Router │─0─▶│ Retimer│─1─▶│ Retimer│─2─▶│ Retimer│─3─▶│ Retimer│─4─▶│ Retimer│─5─▶│ Retimer│─6─▶│ Router │
│        │◀─6─│        │◀─5─│        │◀─4─│        │◀─3─│        │◀─2─│        │◀─1─│        │◀─0─│        │
└────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘

Forward Index →    1────────────2──────────────3─────────────4─────────────5────────────6

Reverse Index ←    6────────────5──────────────4─────────────3─────────────2────────────1
```

Note that each direction of traffic sees different Re-timer index.

### The Broadast RT Packet

Here are the bytes of a Broadcast RT packet:

```
     8          1          2          3          4          5          6          7
+----------+----------+----------+----------+----------+----------+----------+----------+
|   DLE    |   STX    | Link#1   | Link#2   |  LCRC    |  HCRC    |   DLE    |   ETX    |
+----------+----------+----------+----------+----------+----------+----------+----------+
    0xFE                                                              0xFE       0x40
```

In which, the `STX` field is the field that got incremented:

```
Byte1 (STX Symbol)

  7   6   5   4   3   2   1   0
+---+---+---+---+---+---+---+---+
| Type  |Brd|     Index     |CNR|
+---+---+---+---+---+---+---+---+

[7:6] Type = StartRT (01b)
[5]   Broadcast = 1b
[4:1] Index = 0000b
[0]   CmdNotResp = 1b
```

### Broadcast the Link Parameters

The Link Parameter 1 and Link Parameter 2 bytes are used by the Router to broadcast the configuration of the Link to the Re-timers.

#### Link Parameter 1

```
Byte2 (Link Params #1)

  7   6   5   4   3   2   1   0
+---+---+---+---+---+---+---+---+
|    Rsv    |TB3|SSC|RSF|Rsv|USB|
+---+---+---+---+---+---+---+---+

[7:5] Reserved
[4]   TBT3-Compatible Speed
[3]   SSCAlwaysOn
[2]   RS_FEC
[1]   Reserved
[0]   USB4 = 1b
```

#### Link Parameter 2

```
Byte3 (Link Params #2)

  7   6   5   4   3   2   1   0
+---+---+---+---+---+---+---+---+
| Selected Gen  |E3T|E3R|L1E|L0E|
+---+---+---+---+---+---+---+---+

[7:4] Selected Gen (0001=Gen2, 0010=Gen3, 0100=Gen4)
[3]   Enable3Tx
[2]   Enable3Rx
[1]   Lane1Enabled
[0]   Lane0Enabled
```

