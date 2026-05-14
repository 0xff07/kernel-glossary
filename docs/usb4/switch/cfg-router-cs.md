---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Router Config Space (Control Registers)

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

## SPECIFICATIONS

- USB4 Specification, section 8.2.1.1: Basic Configuration Registers

## LINUX KERNEL

- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): Register offset and bit field defines
- [`drivers/thunderbolt/usb4.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c): USB4 router operations
- [`'\<usb4_switch_setup\>':'drivers/thunderbolt/usb4.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L243): Programs ROUTER_CS_5 with tunneling enables

### ROUTER_CS_5 and ROUTER_CS_6 Defines

The control register bit fields from [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L203):

```c
#define ROUTER_CS_5                 0x05
#define ROUTER_CS_5_SLP             BIT(0)
#define ROUTER_CS_5_WOP             BIT(1)
#define ROUTER_CS_5_WOU             BIT(2)
#define ROUTER_CS_5_WOD             BIT(3)
#define ROUTER_CS_5_CNS             BIT(23)
#define ROUTER_CS_5_PTO             BIT(24)  /* PCIe Tunneling On */
#define ROUTER_CS_5_UTO             BIT(25)  /* USB Tunneling On */
#define ROUTER_CS_5_HCO             BIT(26)  /* Host Controller On */
#define ROUTER_CS_5_CV              BIT(31)  /* Configuration Valid */
#define ROUTER_CS_6                 0x06
#define ROUTER_CS_6_SLPR            BIT(0)   /* Sleep Ready */
#define ROUTER_CS_6_TNS             BIT(1)   /* TBT3 Not Supported */
#define ROUTER_CS_6_WOPS            BIT(2)   /* Wake on PCIe Status */
#define ROUTER_CS_6_WOUS            BIT(3)   /* Wake on USB3 Status */
#define ROUTER_CS_6_HCI             BIT(18)  /* Host Controller Implemented */
#define ROUTER_CS_6_CR              BIT(25)  /* Configuration Ready */
```

### [`usb4_switch_setup()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L243) Usage

[`usb4_switch_setup()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L243) reads [`ROUTER_CS_6`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L213) to check hardware capabilities, then programs [`ROUTER_CS_5`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L203) to enable the tunneling modes the parent supports:

```c
int usb4_switch_setup(struct tb_switch *sw)
{
	/* ... */
	ret = tb_sw_read(sw, &val, TB_CFG_SWITCH, ROUTER_CS_6, 1);
	/* ... */
	xhci = val & ROUTER_CS_6_HCI;
	tbt3 = !(val & ROUTER_CS_6_TNS);

	ret = tb_sw_read(sw, &val, TB_CFG_SWITCH, ROUTER_CS_5, 1);
	/* ... */
	if (tb_acpi_may_tunnel_usb3() && sw->link_usb4 &&
	    tb_switch_find_port(parent, TB_TYPE_USB3_DOWN))
		val |= ROUTER_CS_5_UTO;

	if (tb_acpi_may_tunnel_pcie() &&
	    tb_switch_find_port(parent, TB_TYPE_PCIE_DOWN)) {
		val |= ROUTER_CS_5_PTO;
		if (xhci)
			val |= ROUTER_CS_5_HCO;
	}

	val &= ~ROUTER_CS_5_CNS;
	return tb_sw_write(sw, &val, TB_CFG_SWITCH, ROUTER_CS_5, 1);
}
```

## OTHER SOURCES

- [[PATCH] thunderbolt: Fix PCIe device enumeration with delayed rescan](https://lore.kernel.org/all/20260121052744.233517-1-acelan.kao@canonical.com/)

## REGISTERS

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

### Observed Router Configuration Writes (dmesg)

The following dmesg excerpts come from a Dell laptop with an AMD USB4 host router (vendor 0x438, device 0x20e) running Linux 6.19.0-rc6 with `thunderbolt.dyndbg=+pt` enabled on the kernel command line. This system has two USB4 domains. A Dell Thunderbolt 4 Dock (Intel JHL8540, vendor 0x8087, device 0xb26, 19 ports) is connected to domain 1 at route 0x2 (depth 1). Two OWC Envoy Express Thunderbolt 3 NVMe enclosures (Intel JHL6540, 8086:15c0) are connected downstream of the dock at routes 0x502 and 0x702 (depth 2). The packet traces here show how the CM writes to ROUTER_CS_5 during tb_switch_configure() to enable tunneling and how it reads ROUTER_CS_6 to check device capabilities. These traces are from the hot-plug of the second OWC NVMe enclosure at route 0x702 at t=28s.

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

#### Write to ROUTER_CS_5 (Enabling Tunneling)

During tb_switch_configure(), the CM programs ROUTER_CS_5 to enable PCIe tunneling (PTO), USB tunneling (UTO), and mark the configuration as valid (CV). This is a write to Router Configuration Space offset 0x5 on route 702:

```
[   28.073212] tb_tx Write Request Domain 1 Route 702 Adapter 0
               0x00/---- 0x00000000 .... Route String High
               0x01/---- 0x00000702 .... Route String Low
               0x02/---- 0x0400200a ....
                 [00:12]        0xa Address
                 [13:18]        0x1 Write Size
                 [19:24]        0x0 Adapter Num
                 [25:26]        0x2 Configuration Space (CS) → Router Configuration Space
               0x03/000a 0x83000000 .... ROUTER_CS_5 (at absolute offset 0x5, but shown as 0xa in some traces)
[   28.073330] tb_rx Write Response Domain 1 Route 702 Adapter 0
               0x02/---- 0x0400200a ....
                 [00:12]        0xa Address
                 [13:18]        0x1 Write Size
                 [19:24]        0x0 Adapter Num
                 [25:26]        0x2 Configuration Space (CS) → Router Configuration Space
```

The CM writes 1 DWORD to Router Configuration Space. The value 0x83000000 has bit 31 (CV, Configuration Valid) and bit 24 (PTO, PCIe Tunneling On) set. The Write Response echoes the address fields without data, confirming success. This is performed by usb4_switch_setup() which checks ROUTER_CS_6 for capabilities (HCI, TNS) and then sets the appropriate tunneling enable bits in ROUTER_CS_5.

#### Read of ROUTER_CS_6 (Capability Check)

Before writing ROUTER_CS_5, the CM reads ROUTER_CS_6 to determine what the device supports:

```
[   28.068504] tb_rx Read Response Domain 1 Route 702 Adapter 1 / Lane
               0x03/0006 0x01000000 .... ROUTER_CS_6
                 [00:00]        0x0 Sleep Ready (SLPR)
                 [01:01]        0x0 TBT3 Not Supported (TNS)
                 [18:18]        0x0 Internal Host Controller Implemented (HCI)
                 [24:24]        0x1 Router Ready (RR)
                 [25:25]        0x0 Configuration Ready (CR)
```

ROUTER_CS_6 at offset 0x6 reports the router's current status and capabilities. Router Ready (RR) = 1 means the router is ready to accept configuration. TNS = 0 means Thunderbolt 3 backward compatibility is supported. HCI = 0 means no internal xHCI host controller is implemented (this is an NVMe enclosure, not a hub with USB ports). The CM uses these bits to decide which tunneling modes to enable in ROUTER_CS_5: since HCI = 0, the HCO bit is not set; since there is no USB3 adapter, UTO is not set for this particular device.

#### Kernel Log Showing Switch Configuration

After the config space writes, the kernel logs the switch initialization with the programmed route and depth:

```
[   28.073208] [393] thunderbolt 0000:c7:00.6: initializing Switch at 0x702 (depth: 2, up port: 1)
```

tb_switch_alloc() has set the route to 0x702 and depth to 2. The CM will subsequently write these values to ROUTER_CS_1 (depth field) and ROUTER_CS_2/3 (TopologyID fields) via tb_switch_configure(), which also writes ROUTER_CS_5 as shown in the packet trace above.

#### Security Level and NVM Version

```
[   28.411886] thunderbolt 1-702: new device found, vendor=0x5a device=0xde34
[   28.411888] thunderbolt 1-702: Other World Computing Envoy Express
[   28.413040] [393] thunderbolt 0000:c7:00.6: 702: NVM version 21.0
```

After full configuration and DROM read, the device is registered with the kernel device model. The "1-702" notation means domain 1, route 0x702. Vendor 0x5a and device 0xde34 are the DROM-reported identifiers (distinct from the chip-level Vendor/Product IDs 8086:15c0 in ROUTER_CS_0). NVM version 21.0 is the device firmware version read from the NVM authentication area.
