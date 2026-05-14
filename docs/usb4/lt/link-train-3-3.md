---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Link Training Phase 3 (The Lane Speed attribute)

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

### LANE_ADP_CS_1

The negotiated result will be shown in the `Current Link Speed` field of the [`LANE_ADP_CS_1`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L348) regsiter:

```diff
$ tbdump --route 0 --adapter 1 --nregs 1 LANE_ADP_CS_1 -vv
0x0037 0x5c18001c 0b01011100 00011000 00000000 00011100 \... LANE_ADP_CS_1
  [00:03]        0xc Target Link Speed → Router shall attempt Gen 3 speed
  [04:05]        0x1 Target Link Width → Establish two Single-Lane Links
  [06:07]        0x0 Target Asymmetric Link → Establish Symmetric Link
  [10:10]        0x0 CL0s Enable
  [11:11]        0x0 CL1 Enable
  [12:12]        0x0 CL2 Enable
  [14:14]        0x0 Lane Disable (LD)
  [15:15]        0x0 Lane Bonding (LB)
+ [16:19]        0x8 Current Link Speed → Gen 2
  [20:25]        0x1 Negotiated Link Width → Single-Lane Link (x1)
  [26:29]        0x7 Adapter State → CLd
  [30:30]        0x1 PM Secondary (PMS)
```

## LINUX KERNEL

- [`drivers/thunderbolt/sb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h): Sideband register defines including [`USB4_SB_LINK_CONF`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h#L43) (Link Configuration register at 0x0C)
- [`drivers/thunderbolt/usb4.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c): [`usb4_port_sb_read()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L1354) and [`usb4_port_sb_write()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L1407) access sideband registers

Phase 3 is handled in hardware. The CM can read the results of the negotiation through the sideband registers and the Lane Adapter Configuration registers.

## DETAILS

### Lane Speed Selection

This is the decision to select between Gen4/Gen3/Gen2 speed. Again, this depends on the Link Configuration for both side. Another important factor is whether the cable support that speed. For Gen4 speed, it requires both Lanes to work, which is another factor.

#### Gen4 Speed

First, Router on the both sides of the link should have Gen4 Support bit set in the Link Configuration register:

```
Sideband Channel
 ┌────────┐
 │ SB Regs│ SBTX =============▶ SBU1 Gen4 Supported (bit 18)
 │        │ SBRX ◀============= SBU2 Gen4 Supported (bit 18)
 └────────┘
```

Also, the cable should be able to carry the Gen4 speed. This would have been known when doing Phase 1 of the link trainig, where the `Discovery_Identity` PD message already queried the speeds the `SOP'` is capable of.

Finally, Gen4 speed must run on dual Lanes, so both Lane must be enabled.

#### Gen3 Speed

```
Sideband Channel
 ┌────────┐
 │ SB Regs│ SBTX =============▶ SBU1 Gen3 Supported (bit 13)
 │        │ SBRX ◀============= SBU2 Gen3 Supported (bit 13)
 └────────┘
```

This is somewhat similar to Gen4, with a slightly different bits in the Link Configuration register:

1. Both sides annouce that they support Gen3 speed.
2. The cable is capable of carrying Gen3 speed.

Note that the Gen3 speed doesn't require dual Lanes.

#### Gen2 Speed

If both Gen4 and Gen3 speeds are not available, the link uses Gen2 speed.
