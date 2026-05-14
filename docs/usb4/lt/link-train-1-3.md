---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Link Training Phase 1, Part 3 (`Discover_Identity` from `SOP'`)

## SPECIFICATIONS

- *C.1.1 Discover Identity Command request*, Universal Serial Bus Power Delivery Specification, Revision 3.2, Version 1.1, 2024-10
- *8.3.2.14.1.1 Initiator to Responder Discover Identity (ACK)*, Universal Serial Bus Power Delivery Specification, Revision 3.2, Version 1.1, 2024-10

## LINUX KERNEL

Link Training Phase 1 is handled by the USB Type-C port controller and PD stack, outside the thunderbolt driver. The thunderbolt driver begins its work after Phase 1 completes.

## OTHER INFORMATION

- [USB Power Delivery](https://www.usb.org/sites/default/files/D2T2-1%20-%20USB%20Power%20Delivery.pdf)

## DETAILS

```
+---------+                                          +--------------+
| Source  |                                          |  Cable Plug  |
|  (Host) |                                          |   (SOP′)     |
+---------+                                          +--------------+
     |                                                       |
     |  Structured VDM: Discover_Identity (CMD=0001)         |
     |------------------------------------------------------>|
     |                                                       |
     |   ACK with VDOs:                                      |
     |   - ID Header VDO                                     |
     |   - Cert Stat VDO                                     |
     |   - Product VDO                                       |
     |   - Cable VDO(s)                                      |
     |<------------------------------------------------------|
     |                                                       |
     |                                                       |

```

### `Discovery_Identity` to `SOP'`

See *C.1.1 Discover Identity Command request* of the PD specification.

```
[Preamble: 64b alternating 1/0 for sync]
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

[SOP′: 4 K-code symbols]
+--K-code--+--K-code--+--K-code--+--K-code--+

[Message Header: 16 bits]
+-------+-----------+--------+--------+---------+-------+-------------+
| Rsv   |   NDO     | MsgID  |CablePlg| SpecRev | Rsv   |  MsgType    |
+-------+-----------+--------+--------+---------+-------+-------------+

[VDM Header: 32 bits]
+---------------+--------+------+------+-------+-----+------+-----------+
|      SVID     | VDMType| Ver1 | Ver2 | ObjPos|CType| Rsv  |  Command  |
+---------------+--------+------+------+-------+-----+------+-----------+
```

#### The Message Header

1. `Type`: this will be `1111b` (VDM, Vendor-defined message)
2. `NDO`: 1, the VDM Header.

#### Data Objects: The VDM Header

```
[31:16]         [15]     [14:13][12:11][10:8]  [7:6] [5]    [4:0]
+---------------+--------+------+------+-------+-----+------+-----------+
|      SVID     | VDMType| Ver1 | Ver2 | ObjPos|CType| Rsv  |  Command  |
+---------------+--------+------+------+-------+-----+------+-----------+
```

1. `CType`: Initiator (0)
2. `Command`: `Discover_Identity` (0001)


### Response from `SOP'`

See *C.1.2 Discover Identity Command response - Active Cable.* from the USB PD specification

```
[Preamble: 64b alternating 1/0 for sync]
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

[SOP]

[Message Header: 16 bits][NDO = 5]
+-------+-----------+--------+--------+---------+-------+-------------+
| Rsv   |   NDO     | MsgID  |CablePlg| SpecRev | Rsv   |  MsgType    |
+-------+-----------+--------+--------+---------+-------+-------------+

[VDM Header: 32 bits]
+---------------+--------+------+------+-------+-----+------+-----------+
|      SVID     | VDMType| Ver1 | Ver2 | ObjPos|CType| Rsv  |  Command  |
+---------------+--------+------+------+-------+-----+------+-----------+

[ID Header VDO: 32 bits]
+----------+----------+-----------+-------------+---------------------+------------------+
| USB Host | USB Dev  | Product   | Modal       |       Reserved      | USB-IF Vendor ID |
| capable  | capable  | Type      | Operation   | (shall be zero)     | (VID, 16-bit)    |
+----------+----------+-----------+-------------+---------------------+------------------+

[Cert Stat VDO: 32 bits]
+-------------------------------+
|        USB-IF TID/XID         |
+-------------------------------+

[Product VDO: 32 bits]
+------------+------------------+
|    PID     |    bcdDevice     |
+------------+------------------+

[Cable VDO1: 32 bits]
+-----+-----+-----+-----+--+-----+-----+-----+----+----+----+--+--+----+
| HWV | FWV |Rsv  |Conn |PR|Lat  |Term |Vmax |SBUS|SBUT|Imax|VB|S"| Rsv|
+-----+-----+-----+-----+--+-----+-----+-----+----+----+----+--+--+----+

[Cable VDO2: 32 bits]
+--------+--------+----+------+----+----+----+-----+------+----+----+----+---+----+---+
| MaxOpT | ShutT  |Rsv |U3/CLd|U3→0|Phys|Elem|USB4 |HubHop|U2  |U3.2|Lanes|Iso|R  |Gen|
| (°C)   | (°C)   | 0  |Power |Mode|Conn|Act |Supp |(U2)  |Supp|Supp|Supp |   |sv |   |
+--------+--------+----+------+----+----+----+-----+------+----+----+-----+---+---+---+
```

#### The Message Header

```
+-------+-----------+--------+--------+---------+-------+-------------+
| Rsv   |   NDO     | MsgID  |CablePlg| SpecRev | Rsv   |  MsgType    |
+-------+-----------+--------+--------+---------+-------+-------------+
```

#### VDM Header

```
[VDM Header: 32 bits]
+---------------+--------+------+------+-------+-----+------+-----------+
|      SVID     | VDMType| Ver1 | Ver2 | ObjPos|CType| Rsv  |  Command  |
+---------------+--------+------+------+-------+-----+------+-----------+

- Command type: Responser ACK
- Command: Discover_Identity
```

#### ID Header VDO

```
31        30        29:27        26            25:16                 15:0
+----------+----------+-----------+-------------+---------------------+------------------+
| USB Host | USB Dev  | Product   | Modal       |       Reserved      | USB-IF Vendor ID |
| capable  | capable  | Type      | Operation   | (shall be zero)     | (VID, 16-bit)    |
+----------+----------+-----------+-------------+---------------------+------------------+
```

#### Cert stat VDO

```
31:20        19:0
+------------+-------------------+
|  Reserved  |   TID / XID       |
+------------+-------------------+
|   0000     |   0x0ABCDE        |
+------------+-------------------+

- USB-IF Test ID = 0x0ABCDE
```

#### Product VDO

```
31:16            15:0
+----------------+----------------+
|     USB PID    |   bcdDevice    |
+----------------+----------------+
|     0x5678     |     0x0200     |
+----------------+----------------+

- PID = 0x5678
- bcdDevice = 0x0200 (rev 2.00)
```

#### Cable VDO1

```
31:28 27:24 23:20 19:18 17 16:13 12:11 10:9   8    7   6:5   4  3  2:0
+-----+-----+-----+-----+--+-----+-----+-----+----+----+----+--+--+----+
| HWV | FWV |Rsv  |Conn |PR|Lat  |Term |Vmax |SBUS|SBUT|Imax|VB|S"| Rsv|
+-----+-----+-----+-----+--+-----+-----+-----+----+----+----+--+--+----+
|0001 |0001 |0000 |10b  |0 |0010 | 10b |00   |0   |0   |10b |1 |1 |000 |
+-----+-----+-----+-----+--+-----+-----+--+--+----+----+----+--+--+----+

- HWV/FWV   = 0001/0001 → rev 1.1
- Conn=10b  → Type-C plug
- PR=0      → Plug
- Lat=0010  → <20 ns
- Term=10b  → Retimer, VCONN powered
- Vmax=00b  → 20V
- SBUS=0b   → SBU connect supported
- SBUT=0b   → SBU is passive type
- Imax=10b  → 5 A
- VBUS=1    → VBUS through cable
- SOP″=1    → has SOP″ controller
- Reserved
```

#### Cable VDO2

```
[31:24]  [23:16]  [15] [14:12] [11] [10] [9]  [8]  [7:6]  [5]   [4]  [3]  [2] [1] [0]
+--------+--------+----+------+----+----+----+-----+------+----+----+-----+---+---+---+
| MaxOpT | ShutT  |Rsv |U3/CLd|U3→0|Phys|Elem|USB4 |HubHop|U2  |U3.2|Lanes|Iso|R  |Gen|
| (°C)   | (°C)   | 0  |Power |Mode|Conn|Act |Supp |(U2)  |Supp|Supp|Supp |   |sv |   |
+--------+--------+----+------+----+----+----+-----+------+----+----+-----+---+---+---+
|  70    |  80    | 0  | 010  | 0  | 0  | 0  |  0  |  10  | 0  | 0  | 1   | 0 | 0 | 1 |
+--------+--------+----+------+----+----+----+-----+------+----+----+-----+---+---+---+
```

