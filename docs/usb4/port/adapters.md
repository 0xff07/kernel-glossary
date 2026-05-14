---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Adapters

## SPECIFICATIONS

- USB4 Specification, section 8.2.2: Adapter Configuration Space

## LINUX KERNEL

- [`drivers/thunderbolt/tb.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h): Core data structures
- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): Register structures and adapter type enums
- [`drivers/thunderbolt/switch.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c): Port initialization
- [`'\<tb_port\>':'drivers/thunderbolt/tb.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L280): Represents a USB4 adapter (protocol or lane)
- [`'\<tb_regs_port_header\>':'drivers/thunderbolt/tb_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L283): Adapter Config Space header (DW0-DW7)
- [`'\<tb_port_type\>':'drivers/thunderbolt/tb_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L268): Enum identifying adapter type (Lane, NHI, PCIe, USB3, DP)

### [`struct tb_port`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L280)

[`struct tb_port`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L280) represents a USB4 adapter. In USB4 terminology, each port on a router is an adapter (protocol adapter or lane adapter):

```c
struct tb_port {
	struct tb_regs_port_header config;
	struct tb_switch *sw;
	struct tb_port *remote;
	struct tb_xdomain *xdomain;
	int cap_phy;
	int cap_tmu;
	int cap_adap;
	int cap_usb4;
	struct usb4_port *usb4;
	u8 port;
	bool disabled;
	bool bonded;
	struct tb_port *dual_link_port;
	u8 link_nr:1;
	struct ida in_hopids;
	struct ida out_hopids;
	/* ... */
};
```

### [`enum tb_port_type`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L268)

[`enum tb_port_type`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L268) identifies the adapter type, encoded in DW2 of the Adapter Configuration Space (the Type Protocol, Type Version, and Type SubType combined):

```c
enum tb_port_type {
	TB_TYPE_INACTIVE    = 0x000000,
	TB_TYPE_PORT        = 0x000001,  /* Lane adapter */
	TB_TYPE_NHI         = 0x000002,  /* Host Interface adapter */
	TB_TYPE_DP_HDMI_IN  = 0x0e0101,  /* DP IN adapter */
	TB_TYPE_DP_HDMI_OUT = 0x0e0102,  /* DP OUT adapter */
	TB_TYPE_PCIE_DOWN   = 0x100101,  /* PCIe downstream adapter */
	TB_TYPE_PCIE_UP     = 0x100102,  /* PCIe upstream adapter */
	TB_TYPE_USB3_DOWN   = 0x200101,  /* USB3 downstream adapter */
	TB_TYPE_USB3_UP     = 0x200102,  /* USB3 upstream adapter */
};
```

## OTHER SOURCES

- [[PATCH] thunderbolt: Fix PCIe device enumeration with delayed rescan](https://lore.kernel.org/all/20260121052744.233517-1-acelan.kao@canonical.com/)

## DETAILS

### Adapter Numbering

In a USB4 Router, every Adapter has a hard-coded unitque Adapter Number within that Router. This Adapter Number is part of the hardware design and can't be changed.

The numbering of adapter within a Router follows certain convention. For example, a Control Adapter always has Adapter Number `0`. Another rule is that all the upstream-facing Lane Adapters must have smaller Adapter Numbers than the downstream-facing Lane Alapters, and all the Up-Adapters must have lower Adapter Numbers than all the Down-Adapters. Also, a pair of Lane 0/Lane 1 Adapters must have consecutive Adapter Number, with the Lane 0 Adapter having the smaller Adapter Number.

A Router can have up to 64 Adapters.

### Observed Adapter Enumeration (dmesg)

The following dmesg excerpts come from a Dell laptop with an AMD USB4 host router (vendor 0x438, device 0x20e) running Linux 6.19.0-rc6 with `thunderbolt.dyndbg=+pt` enabled on the kernel command line. This system has two independent USB4 domains: domain 0 at PCI function 0000:c7:00.5 and domain 1 at PCI function 0000:c7:00.6. A Dell Thunderbolt 4 Dock (Intel JHL8540, vendor 0x8087, device 0xb26, 19 ports) is connected to domain 1 at route 0x2 (depth 1, through lane adapter port 2). Two OWC Envoy Express Thunderbolt 3 NVMe enclosures (Intel JHL6540, 8086:15c0) are connected downstream of the dock at routes 0x502 and 0x702 (depth 2). The port enumeration logs shown here illustrate every adapter type defined in enum tb_port_type and demonstrate the adapter numbering conventions.

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

#### NHI Adapter

```
[    0.847311] [296] thunderbolt 0000:c7:00.5:  Port 1: 0:0 (Revision: 0, TB Version: 1, Type: NHI (0x2))
[    0.847312] [296] thunderbolt 0000:c7:00.5:   Max hop id (in/out): 2/2
[    0.847313] [296] thunderbolt 0000:c7:00.5:   Max counters: 3
[    0.847313] [296] thunderbolt 0000:c7:00.5:   NFC Credits: 0x0
[    0.847314] [296] thunderbolt 0000:c7:00.5:   Credits (total/control): 0/0
```

Port 1 is the NHI adapter (Type 0x2, TB_TYPE_NHI). It is the upstream adapter of the host router. The Max hop id of 2/2 matches the hop_count (3 paths: hops 0, 1, 2). NFC Credits and total credits are 0 because the NHI adapter does not participate in credit-based flow control the same way lane adapters do. 3 counters are available for performance monitoring.

#### Lane Adapter

```
[    0.848617] [296] thunderbolt 0000:c7:00.5:  Port 2: 0:0 (Revision: 0, TB Version: 1, Type: Port (0x1))
[    0.848618] [296] thunderbolt 0000:c7:00.5:   Max hop id (in/out): 16/16
[    0.848619] [296] thunderbolt 0000:c7:00.5:   Max counters: 10
[    0.848619] [296] thunderbolt 0000:c7:00.5:   NFC Credits: 0x88000000
[    0.848620] [296] thunderbolt 0000:c7:00.5:   Credits (total/control): 128/2
```

Port 2 is a Lane adapter (Type 0x1, TB_TYPE_PORT). Lane adapters are the physical USB4 links that carry tunneled traffic. 16 hop IDs allow multiplexing USB3 (HopID 8), PCIe (HopID 9), DP (HopID 8+9), and control (HopID 0) simultaneously. 128 total buffer credits support high-throughput data transfer, with 2 credits reserved for control path. NFC Credits 0x88000000 encodes the non-flow-controlled credit allocation.

#### USB3 Adapter

```
[    0.850053] [296] thunderbolt 0000:c7:00.5:  Port 4: 0:0 (Revision: 0, TB Version: 1, Type: USB (0x200101))
[    0.850054] [296] thunderbolt 0000:c7:00.5:   Max hop id (in/out): 8/8
[    0.850055] [296] thunderbolt 0000:c7:00.5:   Max counters: 1
[    0.850055] [296] thunderbolt 0000:c7:00.5:   NFC Credits: 0x0
[    0.850056] [296] thunderbolt 0000:c7:00.5:   Credits (total/control): 0/0
```

Port 4 is a USB3 downstream adapter (Type 0x200101, TB_TYPE_USB3_DOWN). The type encoding breaks down as: protocol 0x20 (USB3), version 01, subtype 01 (downstream-facing). On the dock side, port 16 is the corresponding upstream USB3 adapter (Type 0x200102, subtype 02). The USB3 tunnel connects host port 0:4 to dock port 2:16.

#### PCIe Adapter

```
[    0.850313] [296] thunderbolt 0000:c7:00.5:  Port 5: 0:0 (Revision: 0, TB Version: 1, Type: PCIe (0x100101))
[    0.850314] [296] thunderbolt 0000:c7:00.5:   Max hop id (in/out): 8/8
[    0.850315] [296] thunderbolt 0000:c7:00.5:   Max counters: 1
```

Port 5 is a PCIe downstream adapter (Type 0x100101, TB_TYPE_PCIE_DOWN). Protocol 0x10 = PCIe. The PCIe tunnel connects host port 0:5 to dock port 2:9 (PCIe upstream adapter, Type 0x100102).

#### DP/HDMI Adapter

```
[    0.850576] [296] thunderbolt 0000:c7:00.5:  Port 6: 0:0 (Revision: 0, TB Version: 1, Type: DP/HDMI (0xe0101))
[    0.850576] [296] thunderbolt 0000:c7:00.5:   Max hop id (in/out): 9/8
[    0.850577] [296] thunderbolt 0000:c7:00.5:   Max counters: 2
```

Port 6 is a DP IN adapter (Type 0xe0101, TB_TYPE_DP_HDMI_IN). Protocol 0x0e = DisplayPort. The asymmetric hop ID limits (9 in, 8 out) accommodate the AUX channel which needs an extra ingress hop. The dock has DP OUT adapters (ports 13-14, Type 0xe0102) that are the tunnel endpoints for driving physical display outputs.

#### Dual-Link Port Pairing

```
[    0.850839] [296] thunderbolt 0000:c7:00.5: 0: linked ports 2 <-> 3
```

Ports 2 and 3 form a dual-link pair. This follows the numbering convention: Lane 0 and Lane 1 adapters in a pair have consecutive adapter numbers, with Lane 0 having the smaller number. When lane bonding is enabled, both lanes operate as a single logical link with doubled bandwidth (40 Gb/s instead of 20 Gb/s).

#### Inactive and EEPROM-Disabled Ports

```
[    3.886039] [12] thunderbolt 0000:c7:00.6:  Port 15: 8086:b26 (Revision: 1, TB Version: 1, Type: Inactive (0x0))
[    3.886041] [12] thunderbolt 0000:c7:00.6:   Max hop id (in/out): 0/0
```

```
[    5.736091] [12] thunderbolt 0000:c7:00.6: 502:3: disabled by eeprom
[    5.736352] [12] thunderbolt 0000:c7:00.6: 502:5: disabled by eeprom
```

Port 15 on the dock is an Inactive adapter (Type 0x0, TB_TYPE_INACTIVE) with no hop IDs or credits, meaning this adapter slot exists in hardware but is not implemented. On the OWC NVMe enclosure at route 502, ports 3 and 5 are disabled by the device's EEPROM (DROM), meaning the hardware designer chose not to expose these ports.
