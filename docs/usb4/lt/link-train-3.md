---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Link Training Phase 3 (Determination of USB4 Port Characteristics)

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

In this phase, the Routers between the link exchange link training preferences and port characteristics by reading each other's Sideband Registers through the `SBTX` and `SBRX`.

The Router reads Link Configuration register from the partner as well as stating its own preference by setting its own Link Configuration Register.

### Decisions to make

1. Lane enabling: decide which Lanes to train.
2. Bonding: decide whether it's there will be two x1 link, or a single x2 link.
3. Lane speed: whether the lane should run on Gen2/Gen3/Gen4 speed.
4. RS-FEC: whether to enable the RS-FEC error correction.
