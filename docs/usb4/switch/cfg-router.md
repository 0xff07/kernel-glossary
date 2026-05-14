---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Router Config Space

```
+---------------------------+------------------+
|       Product ID(31:16)   | Vendor ID(15:0)  | DW00
+--------+----+-------+-------+--------+-------+
|Revision|(R) |Depth  |MaxAdp |UpstrAdp|NextCap|
|(31:24) |(23)|(22:20)|(19:14)|(13:8)  |(7:0)  | DW01
+--------+----+-------+-------+--------+-------+
|              Topology ID Low (31:0)          | DW02
+----+--------+--------------------------------+
|TIDV|Reserved|     Topology ID High (23:0)    | DW03
|(31)|(30:24) |            (23:0)              |
+---+---------+--------------------------------+
| USB4Ver | Reserved |    CMUV   |  Notify TO  | DW04
|(31:24)  |(23:16)   |   (15:8)  |    (7:0)    |
+---------+----------+-----------+-------------+
|               Router CS 5                    | DW05
+----------------------------------------------+
|               Router CS 6                    | DW06
+----------------------------------------------+
|             UUID High (31:0)                 | DW07
+----------------------------------------------+
|             UUID Low (31:0)                  | DW08
+----------------------------------------------+
|                 Data[n]                      | DW09 - DW24
+----------------------------------------------+
|                 Metadata                     | DW25
+----+----+---------+----------+---------------+
|OV  |ONS | STATUS  | Reserved | OPCODE(15:0)  | DW26
|(31)|(30)| (29:24) | (23:16)  |               |
+----+----+---------+----------+---------------+
```

## SUMMARY

The Router Configuration Space is the primary register space for managing a USB4 router. It contains device identification (Vendor/Product ID, UUID), topology information (Depth, Topology ID, Upstream Adapter), control registers (ROUTER_CS_5/6), and an operation region (Data, Metadata, Opcode). In the kernel, this space is represented by [`struct tb_regs_switch_header`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L166) and accessed via [`tb_sw_read()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L671) and [`tb_sw_write()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L685) with the `TB_CFG_SWITCH` config space selector.

## SPECIFICATIONS

- USB4 Specification, section 8.2.1: Router Configuration Space

## LINUX KERNEL

- [`drivers/thunderbolt/switch.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c): Switch (router) management
- [`drivers/thunderbolt/usb4.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c): USB4-specific router operations
- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): Register structures and defines
- [`'\<tb_regs_switch_header\>':'drivers/thunderbolt/tb_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L166): Router Config Space header (DW0-DW4)
- [`'\<tb_sw_read\>':'drivers/thunderbolt/tb.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L671): Read from router config space
- [`'\<tb_sw_write\>':'drivers/thunderbolt/tb.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L685): Write to router config space
- [`'\<usb4_switch_read_uid\>':'drivers/thunderbolt/usb4.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L342): Reads UUID from ROUTER_CS_7/8
- [`'\<usb4_switch_setup\>':'drivers/thunderbolt/usb4.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L243): Configures ROUTER_CS_5 tunneling enables

### [`struct tb_regs_switch_header`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L166)

[`struct tb_regs_switch_header`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L166) maps the first 5 DWORDs of the Router Configuration Space. It is cached in [`tb_switch.config`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L172) during [`tb_switch_alloc()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c#L2455):

```c
/* Present on port 0 in TB_CFG_SWITCH at address zero. */
struct tb_regs_switch_header {
	/* DWORD 0 */
	u16 vendor_id;
	u16 device_id;
	/* DWORD 1 */
	u32 first_cap_offset:8;
	u32 upstream_port_number:6;
	u32 max_port_number:6;
	u32 depth:3;
	u32 __unknown1:1;
	u32 revision:8;
	/* DWORD 2 */
	u32 route_lo;
	/* DWORD 3 */
	u32 route_hi:31;
	bool enabled:1;
	/* DWORD 4 */
	u32 plug_events_delay:8;
	u32 cmuv:8;
	u32 __unknown4:8;
	u32 thunderbolt_version:8;
};
```

### Router CS Register Defines

The kernel defines symbolic names for router config space offsets and bit fields in [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L196):

```c
#define ROUTER_CS_1                 0x01
#define ROUTER_CS_3                 0x03
#define ROUTER_CS_3_V               BIT(31)
#define ROUTER_CS_4                 0x04
#define ROUTER_CS_5                 0x05
#define ROUTER_CS_5_SLP             BIT(0)
#define ROUTER_CS_5_CNS             BIT(23)
#define ROUTER_CS_5_PTO             BIT(24)
#define ROUTER_CS_5_UTO             BIT(25)
#define ROUTER_CS_5_HCO             BIT(26)
#define ROUTER_CS_5_CV              BIT(31)
#define ROUTER_CS_6                 0x06
#define ROUTER_CS_6_SLPR            BIT(0)
#define ROUTER_CS_6_TNS             BIT(1)
#define ROUTER_CS_6_HCI             BIT(18)
#define ROUTER_CS_6_CR              BIT(25)
#define ROUTER_CS_7                 0x07
#define ROUTER_CS_9                 0x09
#define ROUTER_CS_25                0x19
#define ROUTER_CS_26                0x1a
#define ROUTER_CS_26_OV             BIT(31)
```

### Config Space Read/Write

[`tb_sw_read()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L671) sends a control packet to read from a router's config space. It wraps [`tb_cfg_read()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c#L750) with the switch's route string:

```c
static inline int tb_sw_read(struct tb_switch *sw, void *buffer,
			     enum tb_cfg_space space, u32 offset, u32 length)
{
	if (sw->is_unplugged)
		return -ENODEV;
	return tb_cfg_read(sw->tb->ctl, buffer, tb_route(sw),
			   0, space, offset, length);
}
```

## OTHER SOURCES

- [[PATCH] thunderbolt: Fix PCIe device enumeration with delayed rescan](https://lore.kernel.org/all/20260121052744.233517-1-acelan.kao@canonical.com/)

## REGISTERS

### Device information

```
+---------------------------+------------------+
|       Product ID(31:16)   | Vendor ID(15:0)  | DW00
+--------+----+-------+-------+--------+-------+
|Revision|    |       |       |        |       |
|(31:24) |    |       |       |        |       | DW01
+--------+----+-------+-------+--------+-------+
|                                              | DW02
+----+--------+--------------------------------+
|    |        |                                | DW03
|    |        |                                |
+---+---------+--------------------------------+
| USB4Ver |          |           |             | DW04
|(31:24)  |          |           |             |
+---------+----------+-----------+-------------+
|                                              | DW05
+----------------------------------------------+
|                                              | DW06
+----------------------------------------------+
|             UUID High (31:0)                 | DW07
+----------------------------------------------+
|             UUID Low (31:0)                  | DW08
+----------------------------------------------+
```

This is similar to that if the PCIe. Those fields are there to identify the device itself.

### For device enumeration

```
+---------------------------+------------------+
|                           |                  | DW00
+--------+----+-------+-------+--------+-------+
|        |    |Depth  |MaxAdp |UpstrAdp|       |
|        |    |(22:20)|(19:14)|(13:8)  |       | DW01
+--------+----+-------+-------+--------+-------+
|              Topology ID Low (31:0)          | DW02
+----+--------+--------------------------------+
| V  |        |     Topology ID High (23:0)    | DW03
|(31)|        |            (23:0)              |
+---+---------+--------------------------------+
|         |          |    CMUV   |             | DW04
|         |          |   (15:8)  |             |
+---------+----------+-----------+-------------+
```

### Control Registers

```
+---+---------+--------------------------------+
|         |          |           |  Notify TO  | DW04
|         |          |           |    (7:0)    |
+---------+----------+-----------+-------------+
|               Router CS 5                    | DW05
+----------------------------------------------+
|               Router CS 6                    | DW06
+----------------------------------------------+
```

## DETAILS

### Depth

How may Routers are there between devices connected to this Router and the Host Router. CM programs this during enumeration. The Host Router has a depth of `0`, and the depth increments with each router cascaded.

The depth is crucial because this decides how to interpret the route string.

### Topology ID and the Valid Bit

This is an ID that uniquely identifies a Router in a USB4 topology (similar to a BDF in PCI). CM program this during enumeration. Note that this ID only valid if the Valid in DW3 is set.

### CM USB4 Version

CM can program this field to tell the Router what version of USB4 it is compliant to.

### Upstream Adapter

The Adapter Number of the Lane 0 adapter on this router that is connected upstream. This is hard-coded by the hardware designer, because this apparently is decided by the hardware design. The CM can read this value to understand how things should be routed.

### Max Adapter

Max nubering that the Adapters in this Router have. This is also a hard-coded field. Once this is known to the CM, the CM can iterate over the Adapters by probing every adapters that has an ID no greater than this value.

### [`ROUTER_CS_5`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L203)

```
$ tbdump -d 0 -r 0 -N 1 ROUTER_CS_6 -vv
0x0006 0x01000000 0b00000001 00000000 00000000 00000000 .... ROUTER_CS_6
  [00:00]        0x0 Sleep Ready (SLPR)
  [01:01]        0x0 TBT3 Not Supported (TNS)
  [02:02]        0x0 Wake on PCIe Status
  [03:03]        0x0 Wake on USB3 Status
  [04:04]        0x0 Wake on DP Status
  [18:18]        0x0 Internal Host Controller Implemented (HCI)
  [19:19]        0x0 Partial DP Connectivity Implementation (PI)
  [20:20]        0x0 DPTX Discovery Support (DDS)
  [22:22]        0x0 Gen T Bundle Weight Mode (GTBW)
  [24:24]        0x1 Router Ready (RR)
  [25:25]        0x0 Configuration Ready (CR)
```

These are registers that control enable/disable certain behaviors of the Router. Use `tbman` to see this.

### [`ROUTER_CS_6`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L213)

```
tbdump -d 0 -r 0 -N 1 ROUTER_CS_6 -vv
0x0006 0x01000000 0b00000001 00000000 00000000 00000000 .... ROUTER_CS_6
  [00:00]        0x0 Sleep Ready (SLPR)
  [01:01]        0x0 TBT3 Not Supported (TNS)
  [02:02]        0x0 Wake on PCIe Status
  [03:03]        0x0 Wake on USB3 Status
  [04:04]        0x0 Wake on DP Status
  [18:18]        0x0 Internal Host Controller Implemented (HCI)
  [19:19]        0x0 Partial DP Connectivity Implementation (PI)
  [20:20]        0x0 DPTX Discovery Support (DDS)
  [22:22]        0x0 Gen T Bundle Weight Mode (GTBW)
  [24:24]        0x1 Router Ready (RR)
  [25:25]        0x0 Configuration Ready (CR)
```

This shows various event status from the Router.

### Notification Timeout

For some operations that requires ACK, this is the timeout value before this Router retries. This field is programmable by the CM.

### Observed Router Config Space Reads (dmesg)

The following dmesg excerpts come from a Dell laptop with an AMD USB4 host router (vendor 0x438, device 0x20e) running Linux 6.19.0-rc6 with `thunderbolt.dyndbg=+pt` enabled on the kernel command line. This system has two independent USB4 domains: domain 0 at PCI function 0000:c7:00.5 and domain 1 at PCI function 0000:c7:00.6. A Dell Thunderbolt 4 Dock (Intel JHL8540, vendor 0x8087, device 0xb26, 19 ports) is connected to domain 1 at route 0x2 (depth 1, through lane adapter port 2). Two OWC Envoy Express Thunderbolt 3 NVMe enclosures (Intel JHL6540, 8086:15c0) are connected downstream of the dock at routes 0x502 (depth 2, through dock port 5) and 0x702 (depth 2, through dock port 7). The Router Configuration Space reads shown here are from the hot-plug event at t=28s when the second OWC NVMe enclosure is connected at route 0x702.

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

#### Single-DWORD Read of ROUTER_CS_0

When the CM first discovers a new router, it reads ROUTER_CS_0 to check the Vendor/Product ID before doing a full read. This packet trace shows the Read Request and Response for route 0x702:

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

The Read Request targets Router Configuration Space (CS=2) at offset 0x0 (ROUTER_CS_0), reading 1 DWORD. Adapter Num is 0 because the Router Configuration Space is not port-specific (it belongs to the router as a whole, not to any individual adapter). The route string 0x702 in DW1 directs the packet through port 2 of the host router, then port 7 of the dock at route 2.

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

The response returns Vendor ID 0x8086 (Intel) and Product ID 0x15c0 (JHL6540, Alpine Ridge). Note that the response Adapter Num is 1 (the upstream lane adapter through which the response arrived), while the request used Adapter 0. The response has Route String High bit [31] set (0x80000000), marking it as an upstream (response) packet.

#### Full 5-DWORD Read of ROUTER_CS_0 through ROUTER_CS_4

After confirming the device identity, the CM reads the full switch configuration header (5 DWORDs):

```
[   28.067999] tb_rx Read Response Domain 1 Route 702 Adapter 1 / Lane
               0x02/---- 0x0408a000 ....
                 [00:12]        0x0 Address
                 [13:18]        0x5 Read Size
                 [19:24]        0x1 Adapter Num
                 [25:26]        0x2 Configuration Space (CS) → Router Configuration Space
               0x03/0000 0x15c08086 .... ROUTER_CS_0
                 [00:15]     0x8086 Vendor ID
                 [16:31]     0x15c0 Product ID
               0x04/0001 0x01014005 ..@. ROUTER_CS_1
                 [00:07]        0x5 Next Capability Pointer
                 [08:13]        0x0 Upstream Adapter
                 [14:19]        0x5 Max Adapter
                 [20:22]        0x0 Depth
                 [24:31]        0x1 Revision Number
               0x05/0002 0x00000000 .... ROUTER_CS_2
                 [00:31]        0x0 TopologyID Low
               0x06/0003 0x00000000 .... ROUTER_CS_3
                 [00:23]        0x0 TopologyID High
                 [31:31]        0x0 TopologyID Valid (V)
               0x07/0004 0x0200000a .... ROUTER_CS_4
                 [00:07]        0xa Notification Timeout
                 [08:15]        0x0 Connection Manager USB4 Version (CMUV)
                 [24:31]        0x2 USB4 Version
```

Read Size = 5 retrieves DW0 through DW4 in a single transaction. ROUTER_CS_1 shows Max Adapter = 5 (ports numbered 0 through 5), Depth = 0 (unconfigured; the CM will set this to 2), Upstream Adapter = 0 (port 0, this TB3 device uses port 0 as upstream rather than port 1 as the USB4 dock does), and Revision 1. ROUTER_CS_2 and ROUTER_CS_3 contain the TopologyID which is zero with Valid = 0 because the CM has not yet configured this switch. ROUTER_CS_4 shows USB4 Version = 2 (Thunderbolt 3 compatibility mode), Notification Timeout = 0xa (10ms debounce), and CMUV = 0 (this field was introduced in USB4 v2; a legacy device reports 0).

#### Kernel Log After Config Read

After reading the raw registers, the kernel logs a human-readable summary via tb_dump_switch():

```
[   28.068023] [393] thunderbolt 0000:c7:00.6: current switch config:
[   28.068028] [393] thunderbolt 0000:c7:00.6:  Thunderbolt 3 Switch: 8086:15c0 (Revision: 1, TB Version: 2)
[   28.068033] [393] thunderbolt 0000:c7:00.6:   Max Port Number: 5
[   28.068035] [393] thunderbolt 0000:c7:00.6:   Config:
[   28.068037] [393] thunderbolt 0000:c7:00.6:    Upstream Port Number: 0 Depth: 0 Route String: 0x0 Enabled: 0, PlugEventsDelay: 10ms
```

This is tb_dump_switch() printing the decoded struct tb_regs_switch_header. "Thunderbolt 3 Switch" (rather than "USB4 Switch") appears because TB Version is 2 (not 32 as seen on USB4 devices). The raw Depth and Route String are 0 because they reflect the unconfigured register values. The CM then calls tb_switch_alloc() which sets the correct depth (2) and route (0x702) programmatically.
