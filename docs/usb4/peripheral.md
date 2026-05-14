---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# USB4 Peripherals

```
        ┌────────────┐
        ▼            │    +-------------------+---------------+-------------------+
┌─────────────────┐  │    |                   |               |                   |
│    (UFP)        │  │    +---------------+   |   USB4 UFP    |                   |
│                 │  └────|PCIe UP Adapter|   |   Lane0 (#1)  |   +---------------+
│   PCIe Endpoint │       |     (#3)      |   |   Lane1 (#2)  |   |USB3 UP Adapter|─────────────┐
│                 │       +---------------+   +---------------+   |     (#6)      |             ▼
│                 │       |                                       +---------------+     ┌──────────────┐
└─────────────────┘       |                                                       |     │              │
                          |                                                       |     │     USB3     │
                          |                                                       |     │   Function   │
┌─────────────────┐       |                                                       |     │              │
│                 │       +---------------+                                       |     └──────────────┘
│    DP Sink      │◄──────|DP Out Adapter |                                       |
│                 │       |               |                                       |
│                 │       +---------------+   +---------------+                   |
└─────────────────┘       |                   |Control Adapter|                   |
                          |                   |     (#0)      |                   |
                          |                   +---------------+                   |
                          |                       +-------+                       |
                          |                       |  TMU  |                       |
                          |                       +-------+                       |
                          +-------------------------------------------------------+
```

## SPECIFICATIONS

- USB4 Specification, section 2.1.1.4.1: USB4 Peripheral Device

## LINUX KERNEL

- [`drivers/thunderbolt/switch.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c): Switch (router) management
- [`drivers/thunderbolt/tunnel.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tunnel.c): Tunnel creation for protocols
- [`'\<tb_switch\>':'drivers/thunderbolt/tb.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L171): Represents any USB4 router
- [`'\<tb_port_type\>':'drivers/thunderbolt/tb_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L268): Adapter types

A USB4 peripheral router has only UP adapters (no DOWN adapters). In the kernel, a peripheral is identified when its [`struct tb_switch`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L171) only has upstream-facing protocol adapters (`TB_TYPE_PCIE_UP`, `TB_TYPE_USB3_UP`) and no downstream-facing counterparts.

## OTHER SOURCES

- [[PATCH] thunderbolt: Fix PCIe device enumeration with delayed rescan](https://lore.kernel.org/all/20260121052744.233517-1-acelan.kao@canonical.com/)

## DETAILS

### Router Structure

A USB4 Router in a USB4 Peripheral Device doesn't have a downstream port. It only send and receive USB4 traffic to and from its upsream-facing USB4 port.

The Router may optionally connect to PCIe endpoints (e.g. maybe this is a NVMe storage), USB3 function (e.g. some audio class function) or DP sink (e.g. maybe this is a USB4 monitor). Those are optional. But when they exists, the DOWN/UP adapter connecting rule applies here: an UP adapter always connects to a UFP of that protocol, while a DOWN adapter connects to the DFP of that protocl.

For example, an PCIe endpoint has only upstream PCIe port, so that must get connected to a PCIe UP Adapter on a USB4 Router.

### Observed USB4 Peripheral in dmesg

The following dmesg excerpts come from a Dell laptop with an AMD USB4 host router (vendor 0x438, device 0x20e) running Linux 6.19.0-rc6 with `thunderbolt.dyndbg=+pt` enabled on the kernel command line. A Dell Thunderbolt 4 Dock (Intel JHL8540, vendor 0x8087, device 0xb26) is connected as a hub at route 0x2 (depth 1). Two OWC Envoy Express Thunderbolt 3 NVMe enclosures (Intel JHL6540, 8086:15c0) are connected downstream of the dock as peripherals at routes 0x502 (through dock port 5) and 0x702 (through dock port 7). These NVMe enclosures are typical USB4 peripherals: they have only UP adapters (no DOWN adapters), meaning they are leaf devices in the topology.

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

#### Identifying a Peripheral in dmesg

A USB4 peripheral is identified by having only UP protocol adapters (no DOWN adapters). The OWC Envoy Express at route 0x502 shows this pattern clearly. Its switch config identifies it as a Thunderbolt 3 device:

```
[    5.393520] [12] thunderbolt 0000:c7:00.6: current switch config:
[    5.393523] [12] thunderbolt 0000:c7:00.6:  Thunderbolt 3 Switch: 8086:15c0 (Revision: 1, TB Version: 2)
[    5.393527] [12] thunderbolt 0000:c7:00.6:   Max Port Number: 5
[    5.393531] [12] thunderbolt 0000:c7:00.6:    Upstream Port Number: 0 Depth: 0 Route String: 0x0 Enabled: 0, PlugEventsDelay: 10ms
[    5.398730] [12] thunderbolt 0000:c7:00.6: initializing Switch at 0x502 (depth: 2, up port: 1)
```

"Thunderbolt 3 Switch" appears because TB Version is 2 (a legacy TB3 device, compared to TB Version 32 for USB4 devices). The CM sets the route to 0x502 and depth to 2. Route 0x502 encodes the path: port 5 of the router at route 0x2 (0x500 | 0x002). Max Port Number 5 indicates a simple router with few ports.

#### Peripheral Port Layout (UP Only)

The port enumeration reveals only UP adapters and disabled ports, confirming this is a peripheral:

```
[    5.734648] [12] thunderbolt 0000:c7:00.6:  Port 1: 8086:15c0 (Revision: 1, TB Version: 1, Type: Port (0x1))
[    5.734657] [12] thunderbolt 0000:c7:00.6:   Credits (total/control): 60/2
[    5.736084] [12] thunderbolt 0000:c7:00.6:  Port 2: 8086:15c0 (Revision: 1, TB Version: 1, Type: Port (0x1))
[    5.736090] [12] thunderbolt 0000:c7:00.6:   Credits (total/control): 60/2
```

Ports 1 and 2 are Lane adapters forming the upstream USB4 link back to the dock. Each has 60 credits. These are the only active Lane adapters (there are no downstream-facing Lane adapters because a peripheral has no downstream USB4 ports).

```
[    5.736091] [12] thunderbolt 0000:c7:00.6: 502:3: disabled by eeprom
```

Port 3 is disabled by the device's EEPROM (DROM). The hardware designer chose not to expose this port. This is common on peripherals that do not need all possible adapter slots.

```
[    5.736345] [12] thunderbolt 0000:c7:00.6:  Port 4: 8086:15c0 (Revision: 1, TB Version: 1, Type: PCIe (0x100102))
[    5.736349] [12] thunderbolt 0000:c7:00.6:   Max counters: 2
[    5.736350] [12] thunderbolt 0000:c7:00.6:   NFC Credits: 0x800000
[    5.736351] [12] thunderbolt 0000:c7:00.6:   Credits (total/control): 8/0
```

Port 4 is a PCIe UP adapter (Type 0x100102, subtype 02 = UP). This is the only protocol adapter on the device, and it is upstream-facing. It connects to the NVMe controller inside the enclosure. There is no PCIe DOWN adapter (subtype 01) because a peripheral does not forward PCIe traffic to further downstream devices.

```
[    5.736352] [12] thunderbolt 0000:c7:00.6: 502:5: disabled by eeprom
```

Port 5 is also disabled by EEPROM. Of the 5 possible port slots (max port number 5), only ports 1, 2, and 4 are active. This minimal port layout is typical of a peripheral: two Lane adapters for the upstream link, one PCIe UP adapter for the internal endpoint, and everything else disabled.

#### Peripheral Identity

After DROM read, the peripheral's identity is logged:

```
[    5.732845] [12] thunderbolt 0000:c7:00.6: 502: DROM version: 1
[    5.733214] [12] thunderbolt 0000:c7:00.6: 502: uid: 0x5a5b3a4ca1e700
[    5.738659] thunderbolt 1-502: new device found, vendor=0x5a device=0xde34
[    5.738661] thunderbolt 1-502: Other World Computing Envoy Express
```

The "1-502" notation means domain 1, route 0x502. DROM vendor 0x5a and device 0xde34 identify this as an OWC product. The UID is the factory-programmed unique 64-bit identifier.

#### PCIe Tunnel to a Peripheral

The CM creates a PCIe tunnel from the dock's PCIe DOWN adapter to the peripheral's PCIe UP adapter. The tunnel for the OWC device at route 502 connects dock port 2:11 (PCIe Down) to device port 502:4 (PCIe Up):

```
[    6.164271] [1337] thunderbolt 0000:c7:00.6: 2:11 <-> 502:4 (PCI): activating
[    6.164397] [1337] thunderbolt 0000:c7:00.6: 502:1: Writing hop 1
[    6.164398] [1337] thunderbolt 0000:c7:00.6: 502:1:  In HopID: 8 => Out port: 4 Out HopID: 8
[    6.164399] [1337] thunderbolt 0000:c7:00.6: 502:1:   Weight: 1 Priority: 3 Credits: 32 Drop: 0 PM: 0
```

The hop at the peripheral's upstream Lane adapter (502:1) routes incoming HopID 8 to output port 4 (the PCIe UP adapter). The PCIe traffic enters the USB4 fabric from the dock's PCIe DOWN adapter (port 11), travels through the USB4 link, arrives at the peripheral's Lane adapter (port 1), and gets routed to the PCIe UP adapter (port 4), which delivers it to the NVMe controller.

After the tunnel is active, the NVMe device appears on the PCI bus:

```
[    6.301716] pci 0000:61:00.0: [8086:0b26] type 01 class 0x060400 PCIe Switch Upstream Port
[    6.469968] pci 0000:63:00.0: [1344:5411] type 00 class 0x010802 PCIe Endpoint
[    6.475236] nvme nvme1: pci function 0000:63:00.0
```

The NVMe drive (vendor 0x1344, device 0x5411, NVMe class 0x010802) is now accessible at PCI bus 63:00.0 through the USB4 PCIe tunnel.

#### Contrast with a Hub

The key difference between a peripheral and a hub in dmesg is the port type pattern. The OWC NVMe enclosure has: 2 Lane ports (1-2), 1 PCIe UP port (4), and 2 disabled ports (3, 5). The Dell dock has: 8 Lane ports (1-8), 1 PCIe UP port (9), 3 PCIe DOWN ports (10-12), 2 DP OUT ports (13-14), 1 Inactive port (15), 1 USB3 UP port (16), and 3 USB3 DOWN ports (17-19). The presence of DOWN adapters (PCIe DOWN 10-12, USB3 DOWN 17-19) is what makes the dock a hub rather than a peripheral.

#### Second Peripheral at Route 0x702

A second identical OWC NVMe enclosure connected through dock port 7 appears at route 0x702 at t=28s with the same port layout:

```
[   28.073208] [393] thunderbolt 0000:c7:00.6: initializing Switch at 0x702 (depth: 2, up port: 1)
[   28.411886] thunderbolt 1-702: new device found, vendor=0x5a device=0xde34
[   28.411888] thunderbolt 1-702: Other World Computing Envoy Express
```

Route 0x702 encodes port 7 of the router at route 0x2 (0x700 | 0x002). Despite being the same hardware model, it gets a different route string because it connects through a different downstream port on the dock. The port enumeration is identical: ports 1-2 Lane, port 3 disabled, port 4 PCIe UP, port 5 disabled. This confirms the peripheral pattern is intrinsic to the device hardware, not dependent on the connection point.
