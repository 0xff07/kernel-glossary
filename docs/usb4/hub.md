---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# USB4 Hub

```
        ┌────────────┐                                                                                   ┌────────────┐
        │            │                                                                                   │            ▼
        ▼            │    +-------------------------+--------------------+------------------------+      │     ┌──────────────┐
┌─────────────────┐  │    |                         |                    |                        |      │     │     (UFP)    │
│    (UFP)        │  │    +----------------+        |     USB4 UFP       |                        |      │     │   USB3 HUB   │
│                 │  └────| PCIe UP Adapter|        |     Lane0 (#1)     |     +------------------+      │     │              │
│   PCIe Switch   │       |      (#3)      |        |     Lane1 (#2)     |     | USB3 UP Adapter  |──────┘     │ (DFP)  (DFP) │
│                 │       +----------------+        +--------------------+     |      (#6)        |            └───┬──────┬───┘
│  (DFP)  (DFP)   │       |                                                    +------------------+                │      │
└────┬───────┬────┘       |                                                    | USB3 DN Adapter  |◄───────────────┘      │
     │       │            +----------------+                                   |      (#7)        |                       │
     │       └───────────►| PCIe DN Adapter|                                   +------------------+                       │
     │                    |     (#4)       |                                   | USB3 DN Adapter  |◄──────────────────────┘
     │                    +----------------+                                   |      (#8)        |
     │                    | PCIe DN Adapter|                                   +------------------+
     └───────────────────►|     (#5)       |                                                      |
                          +----------------+        +--------------------+                        |
                          |                         |  Control Adapter   |                        |
                          |                         |       (#0)         |                        |
                          |                         +--------------------+                        |
                          |                                 +-------+                             |
                          |                                 |  TMU  |                             |
                          |                                 +-------+                             |
                          |  +--------------------+ +--------------------+                        |
                          |  |     USB4 DFP       | |     USB4 DFP       |                        |
                          |  |  Lane0 (#9)        | |  Lane0 (#11)       |                        |
                          |  |  Lane1 (#10)       | |  Lane1 (#12)       |                        |
                          +--+--------------------+-+---------------------------------------------+
```

## SPECIFICATIONS

- USB4 Specification, section 2.1.1.4.2: USB4 Hub

## LINUX KERNEL

- [`drivers/thunderbolt/tb.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.c): Connection manager, handles hotplug of hubs
- [`drivers/thunderbolt/switch.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c): Switch (router) management
- [`drivers/thunderbolt/tunnel.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tunnel.c): Tunnel creation for PCIe, USB3, DP through hubs
- [`'\<tb_switch\>':'drivers/thunderbolt/tb.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L171): Represents any USB4 router (host, hub, or peripheral)
- [`'\<tb_port_type\>':'drivers/thunderbolt/tb_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L268): Adapter types (UP/DOWN for each protocol)

A USB4 hub is represented by the same [`struct tb_switch`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L171) as any other router. The kernel distinguishes hub adapters by their [`tb_port_type`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L268): a hub has both UP and DOWN adapters for PCIe and USB3 (e.g., `TB_TYPE_PCIE_UP` and `TB_TYPE_PCIE_DOWN`).

## OTHER SOURCES

- [[PATCH] thunderbolt: Fix PCIe device enumeration with delayed rescan](https://lore.kernel.org/all/20260121052744.233517-1-acelan.kao@canonical.com/)

## DETAILS

### UP and DOWN Adapters

Protocol Adapters for USB3 and PCIe are further categorized into the UP Adapters and the DOWN adapters, where:

1. An UP Adapter always connect to the upstream facing port of the integrated hub/switch of that protocol.
2. AN DOWN Adapter always connect to the downstream facing port of a hub/switch

### PCIe

The hub has a integrated switch. There's a very specific way on how the UFP and the DFP ports on PCIe are connected to the Router.

#### UFP: PCIe UP Adapters

The PCIe UP Adapter always connects to the upstream facing port of the integrated PCIe switch. In this example, the UFP of the integrated PCIe switch is connected Adapter #3, which is a PCIe UP Adapter.

#### DFP: PCIe DOWN Adapters

THe DFPs of the PCIe switch always connect to the DOWN adapters. In this example, 2 of the ports on the integrated PCIe switch are connected to Adapter #4 and Adapter #5 respectively.

Note that it doesn't always mean that all the DFP on the integrated PCIe switch have to connect to the Router. Some of the ports on the PCIe switch can connect to other PCIe devices (e.g. ethernet controller). This essentially makes this hub similar to a dock having a ethernet connector.

### USB3

Ports on the integrated USB3 hub also connects to the UP/DOWN adapters in a very specific way.

#### USB3 UFP: USB3 UP Adapter

The UFPs of the integrated USB3 hub always connect to the USB3 UP Adapter on the Router. Here it is the Adapter #6.

#### USB3 DFP: USB3 DOWN Adapter

On the other hand, DFPs of the integrated USB3 hub always connect to the USB3 DOWN adapters.

### Protocol Adapter Behaviors

#### Tunneled traffic routing

"Tunneled" traffic referes to traffic from other protocols that is disguised as USB4 packet. When a packet from a native protocol (e.g. USB3, PCIe, DP) arrives a Router's protocol adapter, the router packs that packet into a USB4 packet. All USB4 packets, regardless of what they originated from, USB3, PCIe, DP, can now be sent through the Lane Adapter to other routers.

Let's assume that there's a tunneled PCIe packet received from the UFP of the USB4 port. A tunneled PCIe packet here means a PCIe packet that gets packed into a USB4 packet and can move through USB4 link.

When a tunneled packet arrives the Lane Adapter of a USB4 Upstream Port of a Router on a USB4 hub, the packet gets routed to the UP Adapter of that protocol, and subsequently the UFP of a hub/switch of that protocol. For example, a tunneled PCIe packet arrives at the Lane Adapter in the Router first. The Lane Adapter routes this packet according to its Patch Config Space, which points this PCIe packet to an PCIe UP Adapter (let's say #3). This tunneled PCIe packet then gets recovered to native PCIe packet, and send to the UFP of the integrated PCIe switch. From the switch's point of view, this packet is no different than all the other PCIe packets. The switch then processes this packet, send it to its downstream PCIe port.

Once the packet get sent out from the DFP of the PCIe port, if that DFP is connected to a PCIe DOWN adapter, the packet will gets tunneled again, meaning: the router packs the PCIe packet into a USB4 packet, and routes it to the Lane Adapter.

#### UP and DOWN are intrinsic

The UP and DOWN attribute is intrinsic to any Adapter on USB4 Router. This attribite is determined by its hardware designer and can't be changed.

#### Side note on DisplayPort

DisplayPort protocol doesn't define any switch/hub-like component. The DP traffic doesn't goes through any hub/switch because non of then exists in the specification. Instead, the get routed directly from one Lane Adapter to another.

### Observed USB4 Hub in dmesg

The following dmesg excerpts come from a Dell laptop with an AMD USB4 host router (vendor 0x438, device 0x20e) running Linux 6.19.0-rc6 with `thunderbolt.dyndbg=+pt` enabled on the kernel command line. A Dell Thunderbolt 4 Dock (Intel JHL8540, vendor 0x8087, device 0xb26) is connected to domain 1 at route 0x2 (depth 1, through lane adapter port 2). This dock functions as a USB4 hub because it has both UP and DOWN protocol adapters for PCIe and USB3, allowing it to forward tunneled traffic to downstream devices. Two OWC Envoy Express Thunderbolt 3 NVMe enclosures (Intel JHL6540, 8086:15c0) are connected downstream of the dock as peripherals at routes 0x502 (through dock port 5) and 0x702 (through dock port 7).

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

#### Identifying a Hub in dmesg

A USB4 hub is identified by having both UP and DOWN protocol adapters for at least one protocol (PCIe or USB3). The Dell dock's switch config shows it is a USB4 device at depth 1:

```
[    3.858742] [12] thunderbolt 0000:c7:00.6:  USB4 Switch: 8087:b26 (Revision: 3, TB Version: 32)
[    3.858745] [12] thunderbolt 0000:c7:00.6:   Max Port Number: 19
[    3.858748] [12] thunderbolt 0000:c7:00.6:    Upstream Port Number: 1 Depth: 0 Route String: 0x0 Enabled: 0, PlugEventsDelay: 10ms
[    3.863981] [12] thunderbolt 0000:c7:00.6: initializing Switch at 0x2 (depth: 1, up port: 1)
```

The raw Depth and Route String show 0 because the CM has not yet configured the switch. The CM then programs route 0x2 and depth 1. Max Port Number 19 indicates this is a complex router with many adapters. "Enabled: 0" means the firmware had not yet enabled this router (unlike the host router which shows "Enabled: 1").

#### Hub Port Layout (the UP/DOWN Pattern)

The defining characteristic of a hub is the presence of both UP and DOWN adapters for PCIe and USB3. Here is the full port enumeration showing this pattern:

Lane adapters (ports 1-8) provide upstream and downstream USB4 links:

```
[    3.873518] [12] thunderbolt 0000:c7:00.6:  Port 1: 8086:b26 (Revision: 3, TB Version: 1, Type: Port (0x1))
[    3.873526] [12] thunderbolt 0000:c7:00.6:   Credits (total/control): 60/2
```

Port 1 is the upstream Lane adapter connecting back to the host router. Ports 2-8 are additional Lane adapters for downstream connections. The vendor:device shows 8086:b26 (Intel JHL8540 chip-level IDs).

PCIe UP adapter (port 9) connects to the upstream-facing port of the dock's internal PCIe switch:

```
[    3.883451] [12] thunderbolt 0000:c7:00.6:  Port 9: 8086:b26 (Revision: 3, TB Version: 1, Type: PCIe (0x100102))
[    3.883456] [12] thunderbolt 0000:c7:00.6:   Credits (total/control): 8/0
```

Type 0x100102: protocol 0x10 (PCIe), version 01, subtype 02 (UP/upstream-facing). This is the receiving end for PCIe tunnels from the host.

PCIe DOWN adapters (ports 10-12) connect to downstream-facing ports of the dock's internal PCIe switch:

```
[    3.883694] [12] thunderbolt 0000:c7:00.6:  Port 10: 8086:b26 (Revision: 3, TB Version: 1, Type: PCIe (0x100101))
[    3.883975] [12] thunderbolt 0000:c7:00.6:  Port 11: 8086:b26 (Revision: 3, TB Version: 1, Type: PCIe (0x100101))
[    3.884236] [12] thunderbolt 0000:c7:00.6:  Port 12: 8086:b26 (Revision: 3, TB Version: 1, Type: PCIe (0x100101))
```

Type 0x100101: subtype 01 (DOWN/downstream-facing). These carry PCIe tunnels to downstream devices. Some connect to downstream USB4 ports (for tunneling to the OWC NVMe enclosures), while others may connect to internal PCIe devices on the dock (ethernet controller, audio codec, etc.).

DP OUT adapters (ports 13-14) are the exit points for DisplayPort tunnels:

```
[    3.884747] [12] thunderbolt 0000:c7:00.6:  Port 13: 8086:b26 (Revision: 1, TB Version: 1, Type: DP/HDMI (0xe0102))
[    3.885256] [12] thunderbolt 0000:c7:00.6:  Port 14: 8086:b26 (Revision: 1, TB Version: 1, Type: DP/HDMI (0xe0102))
```

Type 0xe0102: subtype 02 (OUT). DP OUT adapters drive physical display outputs on the dock. The host's DP IN adapters (ports 6-7) pair with these DP OUT adapters to form DP tunnels.

Port 15 is Inactive (hardware slot exists but is unused):

```
[    3.886039] [12] thunderbolt 0000:c7:00.6:  Port 15: 8086:b26 (Revision: 1, TB Version: 1, Type: Inactive (0x0))
```

USB3 UP adapter (port 16) connects to the upstream-facing port of the dock's internal USB3 hub:

```
[    3.886301] [12] thunderbolt 0000:c7:00.6:  Port 16: 8086:b26 (Revision: 3, TB Version: 1, Type: USB (0x200102))
```

Type 0x200102: protocol 0x20 (USB3), subtype 02 (UP). This receives USB3 tunnel traffic from the host.

USB3 DOWN adapters (ports 17-19) connect to downstream-facing ports of the dock's USB3 hub:

```
[    3.886562] [12] thunderbolt 0000:c7:00.6:  Port 17: 8086:b26 (Revision: 3, TB Version: 1, Type: USB (0x200101))
[    3.886823] [12] thunderbolt 0000:c7:00.6:  Port 18: 8086:b26 (Revision: 3, TB Version: 1, Type: USB (0x200101))
[    3.887085] [12] thunderbolt 0000:c7:00.6:  Port 19: 8086:b26 (Revision: 3, TB Version: 1, Type: USB (0x200101))
```

Type 0x200101: subtype 01 (DOWN). These drive the dock's physical USB-A/USB-C ports.

#### The Subtype Pattern for UP vs DOWN

The type encoding for protocol adapters follows a consistent pattern: the last two hex digits encode subtype 01 (DOWN/downstream-facing) or 02 (UP/upstream-facing). This is how to decode the type field:

```
    Type encoding: 0xPPVVSS
      PP = Protocol (0x10 PCIe, 0x20 USB3, 0x0e DP)
      VV = Version (01)
      SS = Subtype (01 = DOWN, 02 = UP)

    Host router ports:  PCIe 0x100101 (Down), USB3 0x200101 (Down), DP 0x0e0101 (IN)
    Hub ports:          PCIe 0x100102 (Up) + 0x100101 (Down)
                        USB3 0x200102 (Up) + 0x200101 (Down)
                        DP   0x0e0102 (OUT)
    Peripheral ports:   PCIe 0x100102 (Up), USB3 0x200102 (Up), DP 0x0e0102 (OUT)
```

A hub has both subtypes (01 and 02) for PCIe and/or USB3. A host has only subtype 01 (Down/IN). A peripheral has only subtype 02 (Up/OUT).

#### Device Identity

After DROM read, the dock's identity is announced:

```
[    3.898390] thunderbolt 1-2: new device found, vendor=0xd4 device=0xb0a1
[    3.898392] thunderbolt 1-2: Dell Dell Thunderbolt 4 Dock
```

The "1-2" notation means domain 1, route 0x2. The DROM vendor (0xd4) and device (0xb0a1) are distinct from the chip vendor:device (8087:b26 in the switch config).

#### Tunnel Routing Through a Hub

When a PCIe tunnel is created from the host to the OWC NVMe at route 502, the hub's role is visible in the hop programming. The USB3 tunnel from host port 0:4 to dock port 2:16 uses the hub's UP adapter as the tunnel endpoint:

```
[    5.389850] [12] thunderbolt 0000:c7:00.6: 0:4 <-> 2:16 (USB3): activating
```

Port 0:4 is the host's USB3 Down adapter, and port 2:16 is the dock's USB3 Up adapter. The tunnel connects the host's xHCI controller to the dock's internal USB3 hub through the USB4 fabric.
