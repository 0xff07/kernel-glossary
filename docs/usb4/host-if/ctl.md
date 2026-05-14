---
topics: usb4
tags:
    - "usb4"
    - "host-if"
    - "verification-needed"
---

# Transmit Ring 0 and Receive Ring 0

## SPECIFICATIONS

- USB4 Specification, section 12.3.1: Transmit Descriptor Structure
- USB4 Specification, section 12.4.1: Receive Descriptor Structure

## LINUX KERNEL

- [`drivers/thunderbolt/ctl.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c): Control channel implementation
- [`drivers/thunderbolt/ctl.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.h): Control channel structures
- [`'\<tb_ctl\>':'drivers/thunderbolt/ctl.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c#L39): Control channel uses `tx` (Ring 0 TX) and `rx` (Ring 0 RX)
- [`'\<tb_cfg_request\>':'drivers/thunderbolt/ctl.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.h#L77): A control channel request
- [`'\<tb_cfg_header\>':'drivers/thunderbolt/tb_msgs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_msgs.h#L43): Control packet header (route string)
- [`'\<cfg_read_pkg\>':'drivers/thunderbolt/tb_msgs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_msgs.h#L60): Config read packet structure

### Control Packet Header

[`struct tb_cfg_header`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_msgs.h#L43) represents the route string portion of a control packet:

```c
struct tb_cfg_header {
	u32 route_hi:22;
	u32 unknown:10; /* highest order bit is set on replies */
	u32 route_lo;
};
```

### Config Space Read Packet

[`struct cfg_read_pkg`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_msgs.h#L60) is the packet sent for config space read requests (PDF = 0x01):

```c
struct cfg_read_pkg {
	struct tb_cfg_header header;
	struct tb_cfg_address addr;
};
```

## OTHER SOURCES

- [[PATCH] thunderbolt: Fix PCIe device enumeration with delayed rescan](https://lore.kernel.org/all/20260121052744.233517-1-acelan.kao@canonical.com/)

## DETAILS

Transmit Ring 0 and Receive Ring 0 are special. They are dedicated for control packets. Also they must use Raw mode. They also have interesting routing rules.

In the kernel, [`struct tb_ctl`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c#L39) manages the control channel. Its `tx` and `rx` fields point to [`struct tb_ring`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L539) instances for Ring 0. The control channel crafts raw packets using the structures from [`drivers/thunderbolt/tb_msgs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_msgs.h).

### Observed Control Channel Operations (dmesg)

The following dmesg excerpts come from a Dell laptop with an AMD USB4 host router (vendor 0x438, device 0x20e) running Linux 6.19.0-rc6 with `thunderbolt.dyndbg=+pt` enabled on the kernel command line. This system has two independent USB4 domains: domain 0 at PCI function 0000:c7:00.5 (behind root port 00:01.1) and domain 1 at PCI function 0000:c7:00.6 (behind root port 00:01.2). A Dell Thunderbolt 4 Dock (Intel JHL8540, vendor 0x8087, device 0xb26, 19 ports) is connected to domain 1 at route 0x2 (depth 1, through lane adapter port 2). Two OWC Envoy Express Thunderbolt 3 NVMe enclosures (Intel JHL6540, 8086:15c0) are connected downstream of the dock at routes 0x502 (depth 2, port 5) and 0x702 (depth 2, port 7).

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

#### Control Channel Lifecycle

```
[    0.829118] [296] thunderbolt 0000:c7:00.5: allocating TX ring 0 of size 10
[    0.829168] [296] thunderbolt 0000:c7:00.5: allocating RX ring 0 of size 10
[    0.829207] [296] thunderbolt 0000:c7:00.5: control channel created
[    0.841890] [296] thunderbolt 0000:c7:00.5: control channel starting...
```

[`tb_ctl_alloc()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c#L653) allocates Ring 0 TX and Ring 0 RX, each with 10 descriptors, both operating in Raw Mode. Raw Mode is required for Ring 0 because control packets carry their own routing information (the route string in the packet header) rather than relying on the Path-Based Routing used by higher-numbered rings. The size of 10 descriptors per ring is hardcoded in [`TB_CTL_RX_PKG_COUNT`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c#L21) and means up to 10 control packets can be in flight in each direction before the ring must be serviced. When [`tb_ctl_start()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c#L730) runs, it starts both rings and pre-submits all 10 RX descriptors so the hardware has buffers ready to receive incoming control messages from the fabric.

```
[   24.230280] [170] thunderbolt 0000:c7:00.5: control channel stopped
```

During suspend (or domain teardown), [`tb_ctl_stop()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c#L751) halts both rings and drains any pending requests. All outstanding [`tb_cfg_request`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.h#L77) structures waiting for responses are cancelled with an error so their callers do not block indefinitely. The rings are not freed here (only stopped), so they can be restarted on resume without reallocating DMA buffers.

#### Hot-Plug Notification Packet (tb_tx)

When a device is connected or disconnected, the router that detects the physical event sends a Hot-Plug Event notification upstream to the host. The Connection Manager (CM) in the kernel processes the event and then sends back an acknowledgment so the router knows it can release the event and accept new ones on that adapter. In this example, at t=28s, a second OWC NVMe enclosure is connected through port 7 of the Dell dock (route 0x2). The CM sends this HP_ACK notification:

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

This is a Notification Packet (PDF=4) sent downstream to route 0x2 (the Dell dock). The Event Code 0x7 is HP_ACK, telling the dock that the CM has processed the hot-plug event on adapter 7. Event Info = 0x7 identifies the adapter that generated the event. The Route String in DW0 and DW1 encodes route 0x00000002 (the dock). PG=2 is the Packet Generation counter, which the host router uses to detect lost or reordered packets. The `tb_tx` prefix means this packet was transmitted from the host to the fabric via TX Ring 0.

#### Config Space Read Request/Response (tb_tx/tb_rx)

After acknowledging the hot-plug, the CM reads the Lane Adapter Configuration Space on the newly active port to check its link state. This reads LANE_ADP_CS_0 and LANE_ADP_CS_1 from adapter 7 on the dock (route 2):

```
[   28.067442] tb_tx Read Request Domain 1 Route 2 Adapter 7 / Lane
               0x00/---- 0x00000000 .... Route String High
               0x01/---- 0x00000002 .... Route String Low
               0x02/---- 0x02384036 .8@6
                 [00:12]       0x36 Address
                 [13:18]        0x2 Read Size
                 [19:24]        0x7 Adapter Num
                 [25:26]        0x1 Configuration Space (CS) → Adapter Configuration Space
                 [27:28]        0x0 Sequence Number (SN)
```

This is a Read Request (PDF=1) targeting Route 2, Adapter 7, Adapter Configuration Space (CS=1), at DWORD offset 0x36, reading 2 DWORDs. Offset 0x36 is where LANE_ADP_CS_0 begins in the Lane Adapter capability block. The Address field encodes a DWORD-granular offset into the selected configuration space. The Read Size field specifies how many DWORDs to return (here, 2). The Sequence Number starts at 0 and increments on retries, allowing the responder to distinguish retransmissions from new requests.

```
[   28.067478] tb_rx Read Response Domain 1 Route 2 Adapter 7 / Lane
               0x00/---- 0x80000000 .... Route String High
               0x01/---- 0x00000002 .... Route String Low
               0x02/---- 0x02384036 .8@6
                 [00:12]       0x36 Address
                 [13:18]        0x2 Read Size
                 [19:24]        0x7 Adapter Num
                 [25:26]        0x1 Configuration Space (CS) → Adapter Configuration Space
                 [27:28]        0x0 Sequence Number (SN)
               0x03/0036 0x003c013e .<.> LANE_ADP_CS_0
                 [00:07]       0x3e Next Capability Pointer
                 [08:15]        0x1 Capability ID
                 [16:19]        0xc Supported Link Speeds
                 [20:21]        0x3 Supported Link Widths (SLW)
                 [26:26]        0x0 CL0s Support
                 [27:27]        0x0 CL1 Support
                 [28:28]        0x0 CL2 Support
               0x04/0037 0x4814001c H... LANE_ADP_CS_1
                 [00:03]        0xc Target Link Speed → Router shall attempt Gen 3 speed
                 [04:05]        0x1 Target Link Width → Establish two Single-Lane Links
                 [16:19]        0x4 Current Link Speed → Gen 3
                 [20:25]        0x1 Negotiated Link Width → Single-Lane Link (x1)
                 [26:29]        0x2 Adapter State → CL0
                 [30:30]        0x1 PM Secondary (PMS)
```

The response has Route String High bit [31] set (0x80000000), marking it as a response packet. This is the mechanism the control channel uses to distinguish responses from requests: the highest bit in `tb_cfg_header.route_hi` (the `unknown` field in the kernel struct) is set to 1 in all response packets. The data DWORDs decode LANE_ADP_CS_0 (offset 0x36) and LANE_ADP_CS_1 (offset 0x37). The adapter state is CL0 (link up and active), Current Link Speed is Gen 3 (20 Gb/s per lane), and Negotiated Width is single lane (x1). The PMS (PM Secondary) bit being set means this adapter is the secondary end of the link for power management purposes. The `tb_rx` prefix means this packet was received from the fabric via RX Ring 0. The notation "0x03/0036" means DWORD index 3 in the raw packet buffer, corresponding to register offset 0x36 in the adapter configuration space.

#### Config Space Write Request/Response (tb_tx/tb_rx)

After reading the lane adapter state, the CM writes to ADP_CS_4 on the same adapter to clear the Lock bit. The Lock bit prevents concurrent access to adapter resources during enumeration, and clearing it releases the adapter for normal operation:

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
[   28.067733] tb_rx Write Response Domain 1 Route 2 Adapter 7 / Lane
               0x02/---- 0x02382004 .8..
                 [00:12]        0x4 Address
                 [13:18]        0x1 Write Size
                 [19:24]        0x7 Adapter Num
                 [25:26]        0x1 Configuration Space (CS) → Adapter Configuration Space
```

The Write Request (PDF=2) targets ADP_CS_4 at offset 0x4 in the Adapter Configuration Space, writing 1 DWORD. The Lock bit (bit [31]) is cleared to 0 in the written value (it was 1 in a prior read, now 0 in this write). The Plugged bit (bit [30]) remains 1 because a device is physically connected to this adapter. Total Buffers = 0x3c (60 decimal) indicates the number of flow-controlled credits available on this adapter. The Write Response echoes the address fields (Address, Write Size, Adapter Num, CS) but contains no data DWORDs, confirming the write succeeded. A write failure would instead produce an Error Response (PDF=3) with a status code indicating the reason.
