---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Link Training Phase 3 (The Enabling attribute)

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

## LINUX KERNEL

- [`drivers/thunderbolt/sb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h): Sideband register defines including [`USB4_SB_LINK_CONF`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h#L43) (Link Configuration register at 0x0C)
- [`drivers/thunderbolt/usb4.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c): [`usb4_port_sb_read()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L1354) and [`usb4_port_sb_write()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L1407) access sideband registers

Phase 3 is handled in hardware. The CM can read the results of the negotiation through the sideband registers and the Lane Adapter Configuration registers.

## DETAILS

### Lane Enabling Decision

The Router set the the `Enable Request Lane 0` (bit 8) and `Enable Request Lane 1` (bit 9) on its own Link Configuration register to indicate which it'd like to train:

Meanwhile, the Router on the link partner does the same thing. So both Routers can simply read those bits in each other's Link Configuration register to find the common ground on which lanes to train:

```
Sideband Channel
 ┌────────┐
 │ SB Regs│ SBTX =============▶ SBU1 Lane 0/1 Enable Request (bit 8, bit 9)
 │        │ SBRX ◀============= SBU2 Lane 0/1 Enable Request (bit 8, bit 9)
 └────────┘
```

### The Enabling Request Lane 0/1 bits

A Lane can only be trained if both Router set their own Enabling Request bit for that Lane.

### The Enabling Decision Lane 0/1 bits

If a Router sees both the Enabling Request bit of a Lane is set on both its own Link Configuration register and the link partner's, the Router will set its Enabling Decision bit (`Enabling Decision Lane 0` or `Enabling Decision Lane 1` for the respective Lane) to `1`. (They are bit 0 and bit 1 respectively in the Link Configuration register.)

On the other hand, either side of the link set the Enabling Request of a Lane to `0`, that Lane will not be trained.
