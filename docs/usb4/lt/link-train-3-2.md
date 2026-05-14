---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Link Training Phase 3 (The Bonding attribute)

## SPECIFICATIONS

- 4.1.2.3 Phase 3 - Determination of USB4 Port Characteristics

## REGISTERS

### Link Configuration Register

```
+-------------------------------------------------------------------------------------------------------------+
|  (Reserved)  | Req Asym Rx  | Req Asym Tx | Asym Sup 3Rx | Asym Sup 3Tx | Gen4 Sup    | TBT3-Comp| SBCh Sup |
|     [23]     |      [22]    |      [21]   |      [20]    |      [19]    |   [18]      |   [17]   |   [16]   |
+-------------------------------------------------------------------------------------------------------------+
| RS-FEC Req G3| RS-FEC Req G2| Gen3 Sup    | Bonding Sup  |           Reserved         | EnReq L1 | EnReq L0 |
|      [15]    |      [14]    |     [13]    |     [12]     |            [11:10]         |    [9]   |    [8]   |
+-------------------------------------------------------------------------------------------------------------+
|                           (Reserved)                     | Asym Dec Rx  | Asym Dec Tx | EnDec L1 | EnDec L0 |
|                             [7:4]                        |      [3]     |     [2]     |    [1]   |    [0]   |
+-------------------------------------------------------------------------------------------------------------+
```

### PORT_CS_18

If both sides support Lane Bonding, they will do Lane Bonding at the end of the Link trainig. Also, the Router will set the `Bonding Enabled (BE)` in the [`PORT_CS_18`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L384) in the `USB4 Port Capability (in Lane 0 Config Space) to `1` to indicate that:

```diff
$ tbdump --route 0 --adapter 1 --nregs 1 PORT_CS_18 -vv
0x00ae 0x00000410 0b00000000 00000000 00000100 00010000 .... PORT_CS_18
  [00:07]       0x10 Cable USB4 Version
+ [08:08]        0x0 Bonding Enabled (BE)
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

## LINUX KERNEL

- [`drivers/thunderbolt/sb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h): Sideband register defines including [`USB4_SB_LINK_CONF`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h#L43) (Link Configuration register at 0x0C)
- [`drivers/thunderbolt/usb4.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c): [`usb4_port_sb_read()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L1354) and [`usb4_port_sb_write()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L1407) access sideband registers

Phase 3 is handled in hardware. The CM can read the results of the negotiation through the sideband registers and the Lane Adapter Configuration registers.

## DETAILS

### Lane Bonding decision

This decision is about whether to make 2 Lanes into a x2 link. The prerequisite of this is that:

1. Both Lane are planned to be trained: this is determined by the `Enabling Request Lane 0/1` bits.
1. Both port support Bonding: by the value of `Bonding Support` bit

Bonding capability of the ports is exchanged through reading the `Binding Support` bit in the Link Configuration register.

```
Sideband Channel
 ┌────────┐
 │ SB Regs│ SBTX =============▶ SBU1 Bonding Support (bit 12)
 │        │ SBRX ◀============= SBU2 Bonding Support (bit 12)
 └────────┘
```
