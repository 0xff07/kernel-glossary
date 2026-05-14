---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Link Training Phase 1, Part 5

## SPECIFICATIONS

- *6.4.8 Enter_USB Message*, Universal Serial Bus Power Delivery Specification, Revision 3.2, Version 1.1, 2024-10

## LINUX KERNEL

Link Training Phase 1 is handled by the USB Type-C port controller and PD stack, outside the thunderbolt driver. The thunderbolt driver begins its work after Phase 1 completes.

## DETAILS

### `Enter_USB`

Send the `Enter_USB` PD message to `SOP'` and `SOP`

#### Message Header

```
+-------+-----------+--------+--------+---------+-------+-------------+
| Rsv   |   NDO=1   | MsgID  |CablePlg| SpecRev | Rsv   |  MsgType    |
+-------+-----------+--------+--------+---------+-------+-------------+
```

#### EUDO

```
31 [30:28]        [27][26]    [25]     [24]  [23:21]     [20:19]   [18:17] [16]   [15]    [14]    [13]    [12:0]
+--+--------------+---+-------+--------+-----+-----------+---------+-------+------+-------+-------+-------+------+
|RS|  USB Mode    |RS | USB4  | USB3   | RS  | Cable     | Cable   | Cable | PCIe |  DP   |  TBT3 | Host  | RSVD |
|  | (000:2.0     |   |  DRD  |  DRD   |     | Highest   |  Type   |Current| Tunn |Tunn   |Support|Present| (=0) |
|  | 001:3.2      |   | (Dev) | (Dev)  |     | Speed     |         |       | Supp |Supp   |       |       |      |
|  | 010:USB4)    |   |       |        |     | (Gen info)|         |(3A/5A)|      |       |       |       |      |
+--+--------------+---+-------+--------+-----+-----------+---------+-------+------+-------+-------+-------+------+
```

Notably this also advertise if host support PCIe/DP/TBT3 tunneling on 16:14 bits.
