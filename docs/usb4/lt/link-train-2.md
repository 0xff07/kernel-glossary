---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Link Training Phase 2

## SPECIFICATIONS

- 4.1.2.2 Router Detection

## REGISTERS

### PORT_CS_18

Command that dump the register:

```
$ tbdump --route 0 --adapter 1 --nregs 1 PORT_CS_18 -vv
```

And example output:

```
0x00ae 0x00000410 0b00000000 00000000 00000100 00010000 .... PORT_CS_18
  [00:07]       0x10 Cable USB4 Version
  [08:08]        0x0 Bonding Enabled (BE)
  [09:09]        0x0 TBT3-Compatible Mode (TCM)
  [10:10]        0x1 CLx Protocol Support (CPS)
  [11:11]        0x0 RS-FEC Enabled (Gen 2) (RE2)
  [12:12]        0x0 RS-FEC Enabled (Gen 3) (RE3)
  [13:13]        0x0 Router Detected (RD)
  [16:16]        0x0 Wake on Connect Status
  [17:17]        0x0 Wake on Disconnect Status
  [18:18]        0x0 Wake on USB4 Wake Status
  [19:19]        0x0 Wake on Inter-Domain Status
  [20:20]        0x0 Cable Gen 3 Support (CG3)
  [21:21]        0x0 Cable Gen 4 Support (CG4)
  [22:22]        0x0 Cable Asymmetric Support (CSA)
  [23:23]        0x0 Cable CLx Support (CSC)
  [24:24]        0x0 AsymmetricTransitionInProgress (TIP)
```

Note that for this output there's no other USB4 device connected to it. This is just an illustration of register format.

## LINUX KERNEL

- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): [`PORT_CS_18`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L384) contains the Router Detected (RD) bit

Router detection is handled in hardware. The CM reads [`PORT_CS_18`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L384) to check if a router was detected on the link.

## DETAILS

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
           │   │     Lane0_TX± ────────┼───┼──────▶ TX1± / TX2± pins
           │   │     Lane0_RX± ◀───────┼───┼─────── RX1± / RX2± pins
           │   │     Lane1_TX± ────────┼───┼──────▶ TX1± / TX2± pins
           │   │     Lane1_RX± ◀───────┼───┼─────── RX1± / RX2± pins
           │   │                       │   │
           │   │  Sideband Channel     │   │
           │   │   ┌────────┐          │   │
           │   │   │ SB Regs│ SBTX ────┼───┼──────▶ SBU1 __LOW__->__HIGH__  [1]
           │   │   │        │ SBRX ◀───┼───┼─────── SBU2 __LOW__->__HIGH__  [2]
           │   │   └────────┘          │   │
           │   ├───────────────────────┤   │
           │   │  CC / Power Ctrl      │   │
           │   │     CC (detect, mux)──┼───┼──────▶ CC1/CC2
           │   │     VBUS/VCONN/GND ───┼───┼──────▶ Power pins
           │   └───────────────────────┘   │
           │                               │
           └───────────────────────────────┘
```

### Router Detection

The Host Router initiate Router detection by driving the `SBTX` to high. When the link partner sees this signal remains high for more the `tConnectRx`, the Router on the link partner will also raise its `SBTX` to high ON ALL ITS PORTS.

### PORT_CS_18

When a Router find out both of its `SBTX` and `SBRX` remains high for more than `tConnectRx`, this Router will set its **Router Detected** bit in the [`PORT_CS_18`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L384) (in USB4 Port Capability in the Config Space of the Lane 0 Adapter) and proceed with the Phase 3 of Link training.
