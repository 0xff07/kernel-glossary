---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# USB4 Host

```
┌─────────────────┐
│                 │
│   DP Source     ├──────┐
│                 │      │
└─────────────────┘      │      +-------------------------+---------------------+----------------+   ┌─────────────────┐
                         │      |                         |    USB4 HIST I/F    |                |   │                 │
                         │      +----------------+        +---------------------+                |   │   USB3 Host     │
┌─────────────────┐      │      |  DP IN Adapter |                                               |   │    Controller   │
│                 │      └────► |      (#3)      |                                               |   │                 │
│                 │             +----------------+                                               |   └───┬──────┬──────┘
│  PCIe Root Port ├──────┐      |                                                +---------------+       │      │
│                 │      │      |                                                |USB3 DN Adapter|◄──────┘      │
└─────────────────┘      │      +----------------+                               |     (#7)      |              │
                         │      | PCIe DN Adapter|                               +---------------+              │
                         └────► |     (#5)       |                               |USB3 DN Adapter|◄─────────────┘
                                +----------------+                               |     (#8)      |
                                |                                                +---------------+
                                |                          +--------------------+                |
                                |                          |  Control Adapter   |                |
                                |                          |       (#0)         |                |
                                |                          +--------------------+                |
                                |                              +-------+                         |
                                |                              |  TMU  |                         |
                                |                              +-------+                         |
                                |  +--------------------+  +--------------------+                |
                                |  |     USB4 DFP       |  |     USB4 DFP       |                |
                                |  |  Lane0 (#9)        |  |  Lane0 (#11)       |                |
                                |  |  Lane1 (#10)       |  |  Lane1 (#12)       |                |
                                +--+--------------------+--+--------------------+-----------+----+
```

## SUMMARY

The USB4 Host Router is the root of a USB4 domain. It connects the host system's protocol controllers (PCIe root ports, xHCI, DP sources) to the USB4 fabric via protocol adapters. The kernel represents a USB4 router as [`struct tb_switch`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L171), and the native host interface (NHI) as [`struct tb_nhi`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L497). During probe, [`nhi_probe()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c#L1365) initializes the NHI PCI device, and [`tb_switch_alloc()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c#L2455) allocates the root switch (route = 0).

## SPECIFICATIONS

- USB4 Specification, section 2.1.1.5: Host

## LINUX KERNEL

- [`drivers/thunderbolt/switch.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c): Switch (router) management
- [`drivers/thunderbolt/usb4.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c): USB4 router operations
- [`drivers/thunderbolt/nhi.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c): Native Host Interface (NHI) driver
- [`drivers/thunderbolt/tb.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h): Core data structures
- [`'\<tb_switch\>':'drivers/thunderbolt/tb.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L171): Represents a USB4 router
- [`'\<tb_nhi\>':'include/linux/thunderbolt.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L497): Native Host Interface structure
- [`'\<tb_switch_alloc\>':'drivers/thunderbolt/switch.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c#L2455): Allocates and initializes a router
- [`'\<tb_switch_add\>':'drivers/thunderbolt/switch.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c#L3297): Adds a router to the domain
- [`'\<usb4_switch_setup\>':'drivers/thunderbolt/usb4.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L243): Additional setup for USB4 router (tunneling enables)
- [`'\<nhi_probe\>':'drivers/thunderbolt/nhi.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c#L1365): PCI probe for the NHI device

### [`struct tb_switch`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L171)

In USB4 terminology, [`struct tb_switch`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L171) represents a router. Key fields:

```c
struct tb_switch {
	struct device dev;
	struct tb_regs_switch_header config;
	struct tb_port *ports;
	struct tb_dma_port *dma_port;
	struct tb_switch_tmu tmu;
	struct tb *tb;
	u64 uid;
	uuid_t *uuid;
	u16 vendor;
	u16 device;
	unsigned int link_speed;
	enum tb_link_width link_width;
	bool link_usb4;
	unsigned int generation;
	int cap_plug_events;
	int cap_vsec_tmu;
	int cap_lc;
	int cap_lp;
	bool is_unplugged;
	unsigned int clx;
	/* ... */
};
```

### [`struct tb_nhi`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L497)

[`struct tb_nhi`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L497) represents the Native Host Interface, the PCIe device through which the CM interacts with the USB4 domain:

```c
struct tb_nhi {
	spinlock_t lock;
	struct pci_dev *pdev;
	const struct tb_nhi_ops *ops;
	void __iomem *iobase;
	struct tb_ring **tx_rings;
	struct tb_ring **rx_rings;
	struct ida msix_ida;
	bool going_away;
	bool iommu_dma_protection;
	struct work_struct interrupt_work;
	u32 hop_count;
	unsigned long quirks;
};
```

## KERNEL DOCUMENTATION

- [`Documentation/admin-guide/thunderbolt.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/admin-guide/thunderbolt.rst): Thunderbolt/USB4 administration guide

## OTHER SOURCES

- [[PATCH] thunderbolt: Fix PCIe device enumeration with delayed rescan](https://lore.kernel.org/all/20260121052744.233517-1-acelan.kao@canonical.com/)

## DETAILS

### Down Adapters (PCIe, USB)

The Down Adapters on a USB4 router are adapters connected to the downsteam-facing port of a given protocol.

In this example, here are a PCIe Down Adapter as well as two USB3 Down Adapters. The PCIe Down Adapter is connected to the root hub port from the PCIe controller, while the USB3 Down Adapters are connected to the root hub ports from the xHCI controller.

### IN Adapters (DP)

The DisplayPort Protocole doesn't have abstractions of hubs or switches like USB or PCIe does. So the Adapter accepting traffic from a DisplayPort Source is called IN adapters instead.

### Control Adapter

The Control Adapter is the Adapter where the Control Packets are routed to. The Control Packtes are packets that manages a Router, including the Adapters on it. The CM manages a Router by sending Control Packets reading/writing the USB4 Config Spaces of the components on a Router.

### Host Interface

The Host Interface is where the CM interact with the Host Router, and subsequently, the USB4 domain spanned from this Host Router. Usually it is recognized as a PCIe device. The CM can interact with the domain by queueing packets in the memory, and ring the door bell.

In the kernel, the NHI is a PCI device managed by [`nhi_probe()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c#L1365). The probe function reads [`REG_CAPS`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L113) to determine the number of rings (`hop_count`), then sets up TX/RX ring arrays in [`struct tb_nhi`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L497).

### Lane Adapters (on DFPs)

The Lane Adapters manages USB4 lanes between other Routers. Each Lane has its own Lane Adapter.

The Lane 0 Adapters is special. It manages the Lane itself and also manages various aspects of the Link.

Multiple Lanes can be bonded to a single link. The decision is made during Link Training, and if the decision is to bond, say, 2 Lanes to a single Link, then after the Link Training the link state goes to a detour to the Bonding state first before entering CL0.

### Observed Host Router in dmesg

The following dmesg excerpts come from a Dell laptop with an AMD USB4 host router (vendor 0x438, device 0x20e) running Linux 6.19.0-rc6 with `thunderbolt.dyndbg=+pt` enabled on the kernel command line. This system has two independent USB4 domains: domain 0 at PCI function 0000:c7:00.5 (behind root port 00:01.1) and domain 1 at PCI function 0000:c7:00.6 (behind root port 00:01.2). A Dell Thunderbolt 4 Dock (Intel JHL8540) is connected to domain 1 as a hub at route 0x2, and two OWC Envoy Express NVMe enclosures (Intel JHL6540) are connected as peripherals at routes 0x502 and 0x702.

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

#### Identifying a Host Router in dmesg

A host router is identified by three characteristics in the dmesg output: (1) it is at route 0x0 and depth 0, (2) its upstream port is the NHI adapter, and (3) it is the first router enumerated during boot (before any hot-plug events). The switch config dump at boot shows these values:

```
[    0.872605] [296] thunderbolt 0000:c7:00.6: current switch config:
[    0.872605] [296] thunderbolt 0000:c7:00.6:  USB4 Switch: 438:20e (Revision: 0, TB Version: 32)
[    0.872606] [296] thunderbolt 0000:c7:00.6:   Max Port Number: 7
[    0.872607] [296] thunderbolt 0000:c7:00.6:   Config:
[    0.872607] [296] thunderbolt 0000:c7:00.6:    Upstream Port Number: 1 Depth: 0 Route String: 0x0 Enabled: 1, PlugEventsDelay: 254ms
```

The key identifiers are: Route String 0x0 (always the host router), Depth 0 (root of the topology), Enabled 1 (the router was already active when the CM read it, set by firmware), and Upstream Port Number 1 (the NHI adapter). TB Version 32 indicates USB4 v2 (decimal 32 = 0x20). The "USB4 Switch" label (as opposed to "Thunderbolt 3 Switch") appears because TB Version >= 32.

#### Host Router Port Layout

The host router's ports reveal its role as the root. Port 1 is the NHI (the interface between the CM software and the USB4 fabric). Ports 2-3 are Lane adapters forming a dual-link pair for downstream connections. Ports 4-7 are protocol adapters connecting to the host system's controllers:

```
[    0.877834] [296] thunderbolt 0000:c7:00.6:  Port 1: 0:0 (Revision: 0, TB Version: 1, Type: NHI (0x2))
[    0.877834] [296] thunderbolt 0000:c7:00.6:   Max hop id (in/out): 2/2
[    0.877835] [296] thunderbolt 0000:c7:00.6:   Max counters: 3
[    0.877835] [296] thunderbolt 0000:c7:00.6:   NFC Credits: 0x0
[    0.877836] [296] thunderbolt 0000:c7:00.6:   Credits (total/control): 0/0
```

Port 1 Type NHI (0x2) is unique to host routers. No hub or peripheral has an NHI port. The "0:0" after Port 1 means vendor:device are 0:0 (the NHI adapter does not report chip-level IDs). Max hop id 2/2 matches the hop_count of 3 (hops 0, 1, 2).

```
[    0.879140] [296] thunderbolt 0000:c7:00.6:  Port 2: 0:0 (Revision: 0, TB Version: 1, Type: Port (0x1))
[    0.879143] [296] thunderbolt 0000:c7:00.6:   Credits (total/control): 128/2
[    0.880314] [296] thunderbolt 0000:c7:00.6:  Port 3: 0:0 (Revision: 0, TB Version: 1, Type: Port (0x1))
[    0.880320] [296] thunderbolt 0000:c7:00.6:   Credits (total/control): 128/2
```

Ports 2 and 3 are Lane adapters (Type: Port, 0x1) with 128 credits each. These form the host router's downstream-facing USB4 link. Lane adapters on the host router have large credit pools (128) compared to device routers (typically 60) because the host is the root of the credit tree.

```
[    0.880571] [296] thunderbolt 0000:c7:00.6:  Port 4: 0:0 (Revision: 0, TB Version: 1, Type: USB (0x200101))
[    0.880832] [296] thunderbolt 0000:c7:00.6:  Port 5: 0:0 (Revision: 0, TB Version: 1, Type: PCIe (0x100101))
[    0.881093] [296] thunderbolt 0000:c7:00.6:  Port 6: 0:0 (Revision: 0, TB Version: 1, Type: DP/HDMI (0xe0101))
[    0.881355] [296] thunderbolt 0000:c7:00.6:  Port 7: 0:0 (Revision: 0, TB Version: 1, Type: DP/HDMI (0xe0101))
```

The protocol adapters on a host router are all downstream-facing (subtype 01 in the type encoding). Port 4 is USB3 Down (0x200101), port 5 is PCIe Down (0x100101), and ports 6-7 are DP IN (0xe0101). The "Down" direction means traffic flows from the host system's controllers (xHCI, PCIe root port, GPU) into the USB4 fabric. A host router has only Down/IN adapters for protocols because the host is the source of PCIe root ports, USB3 host controllers, and DP sources.

```
[    0.881357] [296] thunderbolt 0000:c7:00.6: 0: linked ports 2 <-> 3
```

The dual-link pairing confirms ports 2 and 3 can be bonded for 40 Gb/s operation. This is logged by tb_switch_default_link_ports() during switch initialization.

#### Host Router Initialization Sequence

The host router's initialization proceeds in this order: NHI probe reads the hop count, allocates control channel rings, selects the software CM, then reads the switch config and enumerates all ports. The complete sequence for domain 1 is visible in the timestamps (all within 0.871-0.886 seconds after boot):

```
[    0.871744] [296] thunderbolt 0000:c7:00.6: total paths: 3
[    0.871745] [296] thunderbolt 0000:c7:00.6: IOMMU DMA protection is enabled
[    0.872280] [296] thunderbolt 0000:c7:00.6: control channel created
[    0.872280] [296] thunderbolt 0000:c7:00.6: using software connection manager
[    0.872422] [296] thunderbolt 0000:c7:00.6: NHI initialized, starting thunderbolt
[    0.874178] [296] thunderbolt 0000:c7:00.6: initializing Switch at 0x0 (depth: 0, up port: 1)
[    0.875349] [296] thunderbolt 0000:c7:00.6: 0: credit allocation parameters:
[    0.875349] [296] thunderbolt 0000:c7:00.6: 0:  USB3: 24
[    0.875350] [296] thunderbolt 0000:c7:00.6: 0:  DP AUX: 1
[    0.875350] [296] thunderbolt 0000:c7:00.6: 0:  PCIe: 32
[    0.875351] [296] thunderbolt 0000:c7:00.6: 0:  DMA: 32
[    0.877573] [296] thunderbolt 0000:c7:00.6: 0: uid: 0x5dde05610438ba11
```

The "initializing Switch at 0x0 (depth: 0, up port: 1)" line confirms this is the host router. The credit allocation parameters (USB3: 24, PCIe: 32, DMA: 32) define how many flow control credits the host router can dedicate to each tunnel type. The UID is the factory-programmed unique identifier read from the DROM.

#### Distinguishing Host from Hub and Peripheral

Three dmesg patterns uniquely identify each router type in this topology:

The host router has: route 0x0, depth 0, NHI adapter (Type: NHI, 0x2), only Down/IN protocol adapters (subtypes ending in 01), and "Enabled: 1" at boot.

The hub (Dell dock at route 0x2) has: both UP and DOWN adapters for PCIe (ports 9 PCIe Up, 10-12 PCIe Down) and USB3 (port 16 USB3 Up, 17-19 USB3 Down), plus DP OUT adapters (13-14). The presence of both UP and DOWN protocol adapters is what makes it a hub.

The peripheral (OWC NVMe at route 0x502 or 0x702) has: only UP adapters (port 4 PCIe Up), no DOWN adapters. Ports 3 and 5 are "disabled by eeprom". The absence of DOWN adapters means it is a leaf device.
