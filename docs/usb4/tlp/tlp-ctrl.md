---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Control Packets

Note that Config Spaces are also accessed through this type of packet.

## SPECIFICATIONS

- USB4 Specification, section 6.4: Control Packet Protocol

## LINUX KERNEL

The Control Packets are sent and received via Transmit Ring 0 and Receive Ring 0 in Raw Mode respectively. This is implemented in [`ctl.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c). Because they operate in Raw Mode, so the CM needs to hand-craft the packets.

- [`drivers/thunderbolt/ctl.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c): Control channel implementation
- [`drivers/thunderbolt/ctl.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.h): Control channel structures
- [`drivers/thunderbolt/tb_msgs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_msgs.h): Packet structures
- [`'\<tb_cfg_header\>':'drivers/thunderbolt/tb_msgs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_msgs.h#L43): Route string header
- [`'\<cfg_read_pkg\>':'drivers/thunderbolt/tb_msgs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_msgs.h#L60): Config read request/response packet
- [`'\<tb_cfg_pkg_type\>':'include/linux/thunderbolt.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L30): PDF values (Read=1, Write=2, Error=3, etc.)
- [`'\<tb_cfg_request\>':'drivers/thunderbolt/ctl.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.h#L77): Queued control channel request

### [`struct cfg_read_pkg`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_msgs.h#L60)

[`struct cfg_read_pkg`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_msgs.h#L60) is the packet structure for config space read requests and responses (PDF = 0x01):

```c
struct cfg_read_pkg {
	struct tb_cfg_header header;
	struct tb_cfg_address addr;
};
```

### [`struct tb_cfg_header`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_msgs.h#L43)

[`struct tb_cfg_header`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_msgs.h#L43) encodes the route string in the control packet:

```c
struct tb_cfg_header {
	u32 route_hi:22;
	u32 unknown:10; /* highest order bit is set on replies */
	u32 route_lo;
};
```

## OTHER SOURCES

- [[PATCH] thunderbolt: Fix PCIe device enumeration with delayed rescan](https://lore.kernel.org/all/20260121052744.233517-1-acelan.kao@canonical.com/)

## DETAILS

### Packet Format

```
Bit index:   31    28   27    26   23 22        16 15              8 7                0
             |-------|-------|-------|------------|-----------------|-----------------|
DW0 (Hdr)    |  PDF  |SuppID*|Rsvd   |  HopID=0   |      Length     |       HEC       |
             |(31:28)| (27)  |(26:23)|  (22:16)   |     (15:8)      |      (7:0)      |
             |-------|-------|-------|------------|-----------------|-----------------|
DW1          |                     Route String (Low)                                 |
             |------------------------------------------------------------------------|
DW2          |                     Route String (High)                                |
             |------------------------------------------------------------------------|
DW2...DWn    |                     Control Data (Protocol-defined)                    |
             |------------------------------------------------------------------------|
             |                               CRC (31:0)                               |
             |------------------------------------------------------------------------|

* SuppID = Supplemental ID

```

### Characteristics

#### HopID: always 0

The Control Packets always has a `0` HopID.

#### Always sent to Control Adapter

The Control Adapter handles the the Control Packets. The Control packets are routed to the Countrol Adapter, and the Control Adapter then make decision on whether itself is the receiver, or if not, which direction to forward it to.

#### Always has a Route String

A Route String is a TopologyID prefixed with a `CM` bit indicating it's a downstream packet or a upstream one:

```
             |--|--------|--|----------|--|----------|--|----------|
DW1 (63:32)  |CM| Rsvd   |R | Lv6 ID   |R | Lv5 ID   |R | Lv4 ID   |
             |1b|  7b    |2b|   6b     |2b|   6b     |2b|   6b     |
             |--|--------|--|----------|--|----------|--|----------|

             |--|--------|--|----------|--|----------|--|----------|
DW2 (31:0)   |R | Lv3 ID |R |  Lv2 ID  |R |  Lv1 ID  |R |  Lv0 ID  |
             |2b|   6b   |2b|   6b     |2b|   6b     |2b|   6b     |
             |--|--------|--|----------|--|----------|--|----------|
```

Note that Route String is a field in the Control Packet, and a Control Packet is always forwarded to and terminated at the Control Adapter, so in terms of packet routing this is all it takes.

### PDF

See the specification for the PDF values. Here are some of the examples:

#### Config Spcace Access

1. `0x01` (Read Requests/Read Responses): for reading from Config Spaces
2. `0x02` (Write Requests/Write Responses): for writing to Config Spaces

Whether it is a Request or a Response depends on the direction of packet. Requests are always issued from the CM downstream, so a Control Packet with a `0x01` PDF going downstream is considered a Read Request. On the contrary, a Control Packet with `PDF=0x01` going upstream is considered a Read Response.

#### Event Notification

1. `0x03` (Notification)
2. `0x04` (Notification ACK)
5. `0x05` (Hotplug)

### Observed Control Packets (dmesg)

The following dmesg excerpts come from a Dell laptop with an AMD USB4 host router (vendor 0x438, device 0x20e) running Linux 6.19.0-rc6 with `thunderbolt.dyndbg=+pt` enabled on the kernel command line. A Dell Thunderbolt 4 Dock (Intel JHL8540, vendor 0x8087, device 0xb26, 19 ports) is connected to domain 1 at route 0x2 (depth 1, through lane adapter port 2). Two OWC Envoy Express Thunderbolt 3 NVMe enclosures (Intel JHL6540, 8086:15c0) are connected downstream of the dock at routes 0x502 (through dock port 5) and 0x702 (through dock port 7). The tb_tx (transmitted) and tb_rx (received) traces shown here are from the hot-plug event at t=28s when the second OWC NVMe enclosure is connected through dock port 7, demonstrating each of the control packet types in action.

```
    Domain 1 (NHI 0000:c7:00.6)

    Host Router (AMD 438:20e, route 0, depth 0, max port number 7)
      Ports: 1 NHI, 2-3 Lane, 4 USB3 Down, 5 PCIe Down, 6-7 DP IN
        │
        │ port 2+3 Link (dual-lane bond, 40 Gb/s)
        │
    Dell TB4 Dock (Intel 8087:b26, route 0x2, depth 1, max port number 19)
      Ports: 1-8 Lane, 9 PCIe Up, 10-12 PCIe Down,
             13-14 DP OUT, 15 Inactive, 16 USB3 Up, 17-19 USB3 Down
        │
        ├── port 5+6 Link ── OWC Envoy Express (8086:15c0, route 0x502, depth 2)
        │                     NVMe enclosure (TB3, max port number 5)
        │
        └── port 7+8 Link ── OWC Envoy Express (8086:15c0, route 0x702, depth 2)
                              NVMe enclosure (TB3, max port number 5)
```

#### Notification Packet (HP_ACK)

When a hot-plug event occurs on dock port 7, the CM sends a Notification Packet to acknowledge it. This uses PDF=4 (Notification ACK):

```
[   28.067373] tb_tx Notification Packet Domain 1 Route 2
               0x00/---- 0x00000000 .... Route String High
               0x01/---- 0x00000002 .... Route String Low
               0x02/---- 0x80000707 ....
                 [00:07]        0x7 Event Code → HP_ACK
                 [08:13]        0x7 Event Info
                 [14:14]        0x0 Sequence
                 [30:31]        0x2 PG
```

This Notification Packet targets route 0x2 (the Dell dock). Event Code 0x7 (HP_ACK) tells the dock's Control Adapter that the CM has processed the hot-plug event. Event Info 0x7 identifies adapter 7 as the port that generated the event. The CM bit in Route String High is 0, meaning this packet travels downstream. The PG (Packet Generation) field is 2. The packet is crafted by the kernel's control channel code and sent through TX Ring 0 in Raw Mode.

#### Read Request (PDF=1)

After acknowledging the hot-plug, the CM reads the new device's Router Configuration Space. This uses PDF=1 (Read Request):

```
[   28.067757] tb_tx Read Request Domain 1 Route 702 Adapter 0
               0x00/---- 0x00000000 .... Route String High
               0x01/---- 0x00000702 .... Route String Low
               0x02/---- 0x04002000 ....
                 [00:12]        0x0 Address
                 [13:18]        0x1 Read Size
                 [19:24]        0x0 Adapter Num
                 [25:26]        0x2 Configuration Space (CS) → Router Configuration Space
                 [27:28]        0x0 Sequence Number (SN)
```

The Read Request targets route 0x702 (the second OWC NVMe enclosure, reachable through host port 2 then dock port 7). The request reads 1 DWORD from Router Configuration Space (CS=2) at offset 0 (ROUTER_CS_0, which contains Vendor/Product ID). Adapter Num is 0 because Router CS belongs to the router as a whole. DW2 encodes the address, size, adapter, CS selector, and sequence number in a packed format. The Control Adapter at each intermediate router (the host router and the dock) uses the route string to forward this packet downstream.

#### Read Response (PDF=1, response bit set)

The device responds with a Read Response:

```
[   28.067864] tb_rx Read Response Domain 1 Route 702 Adapter 1 / Lane
               0x00/---- 0x80000000 .... Route String High
               0x01/---- 0x00000702 .... Route String Low
               0x02/---- 0x04082000 ....
                 [00:12]        0x0 Address
                 [13:18]        0x1 Read Size
                 [19:24]        0x1 Adapter Num
                 [25:26]        0x2 Configuration Space (CS) → Router Configuration Space
               0x03/0000 0x15c08086 .... ROUTER_CS_0
                 [00:15]     0x8086 Vendor ID
                 [16:31]     0x15c0 Product ID
```

The Read Response has Route String High bit [31] set (0x80000000), which is the CM bit indicating this packet travels upstream toward the CM. The Adapter Num in the response is 1 (the upstream lane adapter through which the response exits the device), rather than 0 as in the request. The data DWORD at position 0x03 contains ROUTER_CS_0: Vendor ID 0x8086 (Intel) and Product ID 0x15c0 (JHL6540 Alpine Ridge). The notation "0x03/0000" means DWORD index 3 in the packet, corresponding to register offset 0x0000.

#### Write Request and Response (PDF=2)

The CM writes to ADP_CS_4 on dock adapter 7 to clear the Lock bit:

```
[   28.067628] tb_tx Write Request Domain 1 Route 2 Adapter 7 / Lane
               0x02/---- 0x02382004 .8..
                 [00:12]        0x4 Address
                 [13:18]        0x1 Write Size
                 [19:24]        0x7 Adapter Num
                 [25:26]        0x1 Configuration Space (CS) → Adapter Configuration Space
               0x03/0004 0x43c00000 C... ADP_CS_4
                 [00:09]        0x0 Non-Flow Controlled Buffers
                 [20:29]       0x3c Total Buffers
                 [30:30]        0x1 Plugged
                 [31:31]        0x0 Lock (LCK)
```

The Write Request (PDF=2) targets Adapter Configuration Space (CS=1) on route 2, adapter 7, at offset 0x4 (ADP_CS_4). The data DWORD sets Lock = 0 (clearing the lock to allow reconfiguration) while preserving Plugged = 1 and Total Buffers = 0x3c (60 credits). Unlike Read Requests, Write Requests include data DWORDs in the packet.

```
[   28.067733] tb_rx Write Response Domain 1 Route 2 Adapter 7 / Lane
               0x02/---- 0x02382004 .8..
                 [00:12]        0x4 Address
                 [13:18]        0x1 Write Size
                 [19:24]        0x7 Adapter Num
                 [25:26]        0x1 Configuration Space (CS) → Adapter Configuration Space
```

The Write Response echoes the address header (route, adapter, CS, offset) but contains no data DWORDs. The absence of data in the response confirms the write was accepted. If the write had failed, the device would have returned an Error packet (PDF=3) instead.

#### Multi-DWORD Read (5 DWORDs)

The CM reads the full switch config header (5 DWORDs from ROUTER_CS_0 through ROUTER_CS_4) in a single transaction:

```
[   28.067887] tb_tx Read Request Domain 1 Route 702 Adapter 0
               0x02/---- 0x0400a000 ....
                 [00:12]        0x0 Address
                 [13:18]        0x5 Read Size
                 [19:24]        0x0 Adapter Num
                 [25:26]        0x2 Configuration Space (CS) → Router Configuration Space
```

Read Size = 5 (bits [18:13]) requests 5 DWORDs starting from offset 0. The response will contain DWORDs for ROUTER_CS_0 through ROUTER_CS_4, allowing the CM to read the full device identity, topology configuration, and USB4 version in a single round-trip. This is more efficient than issuing 5 separate single-DWORD reads. The maximum read size is limited by the control channel frame size (TB_FRAME_SIZE).
