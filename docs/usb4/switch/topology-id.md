---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# TopologyID

A TopologyID uniquely describes a Router's location in a domain, by the sequence of Downstream Lane Adapter numbers that a packet needs to go through in order to reach that Router.

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

The Control Adapter makes Routing decision for a Control Pcaket according to this Route String.

## SPECIFICATIONS

- 6.2 Router Addressing
- Figure 6-1. Example of TopologyID Assignment
- CONNECTION MANAGER NOTE, 8.2.1.1 Basic Configuration Registers

## LINUX KERNEL

- [`drivers/thunderbolt/switch.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c): Switch (router) management
- [`drivers/thunderbolt/tb.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h): Core data structures and inline helpers
- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): Register structures
- [`'\<tb_route\>':'drivers/thunderbolt/tb.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L582): Extracts the 64-bit route from a switch's config
- [`'\<tb_switch_find_by_route\>':'drivers/thunderbolt/switch.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c#L3829): Finds a switch in the domain by its route string
- [`'\<tb_switch_match\>':'drivers/thunderbolt/switch.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c#L3740): Matches a switch device by route
- [`'\<tb_regs_switch_header\>':'drivers/thunderbolt/tb_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L166): Contains route_lo, route_hi, depth fields

### [`tb_route()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L582)

[`tb_route()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L582) reconstructs the 64-bit route string from the cached config header. The route string encodes the Topology ID (the sequence of downstream lane adapter numbers):

```c
static inline u64 tb_route(const struct tb_switch *sw)
{
	return ((u64) sw->config.route_hi) << 32 | sw->config.route_lo;
}
```

### Route String in [`struct tb_regs_switch_header`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L166)

The route string is stored in DW2 (`route_lo`) and DW3 (`route_hi`) of the router config space. These are cached in [`struct tb_regs_switch_header`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L166) during [`tb_switch_alloc()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c#L2455):

```c
struct tb_regs_switch_header {
	/* ... */
	/* DWORD 2 */
	u32 route_lo;
	/* DWORD 3 */
	u32 route_hi:31;
	bool enabled:1;
	/* ... */
};
```

The Host Router always has a route of `0`. During [`tb_switch_alloc()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c#L2455), the route and depth are set from the provided route parameter:

```c
sw->config.depth = tb_route_length(route);
sw->config.route_hi = upper_32_bits(route);
sw->config.route_lo = lower_32_bits(route);
```

## OTHER SOURCES

- [[PATCH] thunderbolt: Fix PCIe device enumeration with delayed rescan](https://lore.kernel.org/all/20260121052744.233517-1-acelan.kao@canonical.com/)

## DETAILS

### Depth of a Router

The "depth" of a Host Router is defined to be `0`. A Host Router is a Router that has a Host Interface Adapter. There's only a Host Router in each Domain. A Router connected immediately to a Router of depth `n` has a depth `(n+1)` In USB4, the depth can be no more than `5`.

On hotplug, the CM has to tell the hot-plugged router what it depth is, by doing Configuration Write to its Config Space.

### Sequence of Lane Adapter Number

This is the downstream lane adapter number on each level of the Router in order for this packet to reach the target Router. Note that Route String is a field in the Control Packet, and a Contro Packet is always handled by the Control Adapter, so in terms of packet routing the TopologyID is all it takes.

### `CM`: is this packet targeting CM?

If this is set, this packet is targeting the `CM`, and the TopologyID is the Router this packet originated from. Otherwise this is targeting the downstream routers, where its TopologyID is the target Router.

### Observed Topology IDs (dmesg)

The following dmesg excerpts come from a Dell laptop with an AMD USB4 host router (vendor 0x438, device 0x20e) running Linux 6.19.0-rc6 with `thunderbolt.dyndbg=+pt` enabled on the kernel command line. This system has two independent USB4 domains: domain 0 at PCI function 0000:c7:00.5 and domain 1 at PCI function 0000:c7:00.6. A Dell Thunderbolt 4 Dock (Intel JHL8540, vendor 0x8087, device 0xb26, 19 ports) is connected to domain 1 at route 0x2 (depth 1, through lane adapter port 2). Two OWC Envoy Express Thunderbolt 3 NVMe enclosures (Intel JHL6540, 8086:15c0) are connected downstream of the dock at routes 0x502 (depth 2, through dock port 5) and 0x702 (depth 2, through dock port 7). The route string encoding directly reflects the physical path through the topology: route 0x502 means "port 5 of the router at route 0x2" (0x500 | 0x2 = 0x502), and route 0x702 means "port 7 of the router at route 0x2" (0x700 | 0x2 = 0x702).

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

#### Host router (route 0x0)

```
[    0.842092] [296] thunderbolt 0000:c7:00.5:    Upstream Port Number: 1 Depth: 0 Route String: 0x0 Enabled: 1, PlugEventsDelay: 254ms
[    0.843647] [296] thunderbolt 0000:c7:00.5: initializing Switch at 0x0 (depth: 0, up port: 1)
```

Route 0x0 and depth 0 identify the host router. The host router is always at the root of the topology with an empty route string. Upstream port 1 is the NHI adapter.

#### Dell dock (route 0x2)

```
[    3.858748] [12] thunderbolt 0000:c7:00.6:    Upstream Port Number: 1 Depth: 0 Route String: 0x0 Enabled: 0, PlugEventsDelay: 10ms
[    3.863981] [12] thunderbolt 0000:c7:00.6: initializing Switch at 0x2 (depth: 1, up port: 1)
```

The raw register read shows Depth 0 and Route String 0x0 because the CM has not yet configured the switch. The CM then programs route 0x2 and depth 1. Route 0x2 means the packet reaches this router through lane adapter port 2 of the host router. In the Lv0 ID field of the TopologyID, port 2 is encoded.

#### OWC NVMe at route 0x502

```
[    5.398730] [12] thunderbolt 0000:c7:00.6: initializing Switch at 0x502 (depth: 2, up port: 1)
```

Route 0x502 encodes the path: port 5 of the router at route 0x2. In hex, 0x502 = 0x500 | 0x002. The Lv0 ID field contains 0x2 (port 2 of host router) and the Lv1 ID field contains 0x5 (port 5 of dock). Depth 2 means two hops from the host.

#### OWC NVMe at route 0x702

```
[   28.073208] [393] thunderbolt 0000:c7:00.6: initializing Switch at 0x702 (depth: 2, up port: 1)
```

Route 0x702 encodes port 7 of the router at route 0x2 (0x700 | 0x002). This is a different downstream port on the same dock. Each downstream lane adapter on the dock has its own level ID in the route string.

#### Packet trace: Read Request to route 0x702

```
[   28.067757] tb_tx Read Request Domain 1 Route 702 Adapter 0
               0x00/---- 0x00000000 .... Route String High
               0x01/---- 0x00000702 .... Route String Low
               0x02/---- 0x04002000 ....
                 [00:12]        0x0 Address
                 [13:18]        0x1 Read Size
                 [19:24]        0x0 Adapter Num
                 [25:26]        0x2 Configuration Space (CS) → Router Configuration Space
```

The Route String Low (DW1) is 0x00000702. Parsed as a TopologyID: Lv0 ID = 0x2 (bits [5:0], the host router's downstream lane adapter port 2), Lv1 ID = 0x7 (bits [13:8], the dock's downstream lane adapter port 7). Route String High (DW0) is all zeros because the topology is only 2 levels deep (Lv2 through Lv6 are unused). The CM bit is 0, meaning this packet is directed downstream to the target router.

#### Packet trace: Read Response from route 0x702

```
[   28.067864] tb_rx Read Response Domain 1 Route 702 Adapter 1 / Lane
               0x00/---- 0x80000000 .... Route String High
               0x01/---- 0x00000702 .... Route String Low
```

In the response, Route String High bit [31] is set (0x80000000). This is the CM bit, indicating the packet is traveling upstream toward the CM. The route string itself is unchanged (0x702), identifying where the response originated.

#### Packet trace: unconfigured TopologyID in ROUTER_CS_2/3

```
[   28.067999] tb_rx Read Response Domain 1 Route 702 Adapter 1 / Lane
               0x05/0002 0x00000000 .... ROUTER_CS_2
                 [00:31]        0x0 TopologyID Low
               0x06/0003 0x00000000 .... ROUTER_CS_3
                 [00:23]        0x0 TopologyID High
                 [31:31]        0x0 TopologyID Valid (V)
```

Before the CM configures the switch, ROUTER_CS_2 (TopologyID Low) and ROUTER_CS_3 (TopologyID High) are both zero, and the Valid bit is 0. The CM will write the correct topology ID (matching the route string 0x702) and set the Valid bit during tb_switch_configure(). Until the Valid bit is set, the router uses the route from the incoming packet header rather than its stored topology ID.
