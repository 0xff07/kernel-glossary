---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Path Config Space

Each Protocol Adapter, other than their own Adapter Config Space, also owns a Path Config Space. The Path Config Spaces is the routing tables for various traffic types in the USB4. Each entry inside describes the output adapter and HopID the ingress traffic should be routed to, as well as some flow control information.

Note that Path Config Space for an adapter is its own address space. It's a separate space from the Adapter Config Space and is NOT in some offset inside the Adapter Config Space.

## SPECIFICATIONS

- USB4 Specification, section 8.2.3: Path Configuration Space

## LINUX KERNEL

- [`drivers/thunderbolt/path.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/path.c): Path allocation and activation
- [`drivers/thunderbolt/tunnel.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tunnel.c): Tunnel management (groups of paths)
- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): Hop register structure
- [`'\<tb_path\>':'drivers/thunderbolt/tb.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L430): A unidirectional path between two ports
- [`'\<tb_path_hop\>':'drivers/thunderbolt/tb.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L381): Routing info for each hop in a path
- [`'\<tb_regs_hop\>':'drivers/thunderbolt/tb_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L502): Hardware hop register (Path Entry)
- [`'\<tb_path_activate\>':'drivers/thunderbolt/path.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/path.c#L505): Writes hop registers to activate a path
- [`'\<tb_tunnel\>':'drivers/thunderbolt/tunnel.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tunnel.h#L73): A tunnel (collection of paths between two ports)

### [`struct tb_regs_hop`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L502)

[`struct tb_regs_hop`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L502) maps a Path Entry Structure (2 DWORDs). This is what [`tb_path_activate()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/path.c#L505) writes to the Path Config Space to program routing:

```c
/* Hop register from TB_CFG_HOPS. 8 bytes per entry. */
struct tb_regs_hop {
	/* DWORD 0 */
	u32 next_hop:11;       /* Output HopID */
	u32 out_port:6;        /* Output Adapter */
	u32 initial_credits:7; /* Path Credits Allocated */
	u32 pmps:1;
	u32 unknown1:6;
	bool enable:1;         /* Valid */
	/* DWORD 1 */
	u32 weight:4;
	u32 unknown2:4;
	u32 priority:3;
	bool drop_packages:1;
	u32 counter:11;        /* Counter ID */
	bool counter_enable:1; /* Counter Enable (CE) */
	bool ingress_fc:1;     /* Ingress Flow Control (IFC) */
	bool egress_fc:1;      /* Egress Flow Control (EFC) */
	bool ingress_shared_buffer:1; /* ISE */
	bool egress_shared_buffer:1;  /* ESE */
	bool pending:1;        /* Pending Packets (PP) */
	u32 unknown3:3;
};
```

### [`struct tb_path_hop`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L381)

[`struct tb_path_hop`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L381) is the kernel's in-memory representation of a path hop, before it is written to hardware:

```c
struct tb_path_hop {
	struct tb_port *in_port;
	struct tb_port *out_port;
	int in_hop_index;
	int in_counter_index;
	int next_hop_index;
	unsigned int initial_credits;
	unsigned int nfc_credits;
	bool pm_support;
};
```

### [`struct tb_path`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L430)

[`struct tb_path`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L430) represents a unidirectional path. A PCIe tunnel requires two paths (upstream and downstream). A DP tunnel uses three paths (video, AUX TX, AUX RX):

```c
struct tb_path {
	struct tb *tb;
	const char *name;
	unsigned int priority:3;
	int weight:4;
	bool drop_packages;
	bool activated;
	bool clear_fc;
	struct tb_path_hop *hops;
	int path_length;
	bool alloc_hopid;
	/* ... flow control fields ... */
};
```

## OTHER SOURCES

- [[PATCH] thunderbolt: Fix PCIe device enumeration with delayed rescan](https://lore.kernel.org/all/20260121052744.233517-1-acelan.kao@canonical.com/)

## REGISTERS

### Path Entry Structure

The Path Address Space contains Path Entry Structures. Each of the Path Entry Structure describes routing information for the traffic type that adapter is associated to:

```
0x0010 0x000e0000 0b00000000 00001110 00000000 00000000 .... PATH_CS_0
  [00:06]        0x0 Output HopID
  [11:16]        0x0 Output Adapter
  [17:23]        0x7 Path Credits Allocated
  [31:31]        0x0 Valid
0x0011 0x01000000 0b00000001 00000000 00000000 00000000 .... PATH_CS_1
  [00:03]        0x0 Weight
  [08:10]        0x0 Priority
  [12:22]        0x0 Counter ID
  [23:23]        0x0 Counter Enable (CE)
  [24:24]        0x1 Ingress Flow Control Flag (IFC)
  [25:25]        0x0 Egress Flow Control Flag (EFC)
  [26:26]        0x0 Ingress Shared Buffering Enable Flag (ISE)
  [27:27]        0x0 Egress Shared Buffering Enable Flag (ESE)
  [28:28]        0x0 Pending Packets (PP)
```

## DETAILS

### For Protocol Adapter (non-DP)

For a downstram/upstream adapter (in USB3 and PCIe), they only serve traffic from a single direction, so there's only one Path Entry Structure inside. Note that the beginning offset into the Path Config Space in those adapters are `0x0010` instead of `0x0000`, because Path 0 to Path 7 are for control traffic or simply reserved (only Host Interface can use Path 1 to Path 7).

For example, even if the attempt is to dump 8 DWs, there's still only a pair of registers:

```
$ tbdump --route 0 --adapter 8 --nregs 8 --path 0 -v
0x0010 0x000e0000 PATH_CS_0
0x0011 0x01000000 PATH_CS_1
```

### For Protocol Adapter (DP)

For each DP Adapter, there are both traffic and the AUX channel traffic, so there are 2 Path Entry Structures in each adapter:

```
$ tbdump --route 0 --adapter 5 --nregs 8 --path 0 -v
0x0010 0x00060000 PATH_CS_0
0x0011 0x01000000 PATH_CS_1
0x0012 0x001a0000 PATH_CS_0
0x0013 0x00000000 PATH_CS_1
```

Path 8 for AUX and Path 9 for video.

### For Lane Adapter

For Lane Adapter, most of the time there are more than 1 path, notably Path 0 for control traffic plus Path 8 or above from other protocols. But because that the Path 1 to Path 7 are reserved, there's seemingly a gap (`0x0001` to `0x0010`) in the Path Config Space of the Lane Adapter:

```
$ tbdump --route 0 --adapter 1 --nregs 8 --path 0 -v
0x0000 0x80040001 PATH_CS_0
0x0001 0x21000001 PATH_CS_1
0x0010 0x00000000 PATH_CS_0
0x0011 0x00000000 PATH_CS_1
0x0012 0x00000000 PATH_CS_0
0x0013 0x00000000 PATH_CS_1
0x0014 0x00000000 PATH_CS_0
0x0015 0x00000000 PATH_CS_1
...
```

The number of Paths supported by the Lane Adapter is in the Max Input HopID of the Adapter Config Space.

Note that the Path Entry Structure for the Path 0 has its own structure. It's special in that it doesn't need Output Adapter and Output HopID, nor does it need the Priority and Weight, because the control traffic simply has the priviledge to use Path 0 whatever it likes. Those unused fields either become vendor-defined field, or reserved.

### For Host Interface

It can use all Path ID:

```
$ tbdump --route 0 --adapter 7 --nregs 9 --path 0 -v
0x0000 0x80040007 PATH_CS_0
0x0001 0x21000001 PATH_CS_1
0x0002 0x00200000 PATH_CS_0
0x0003 0x05000000 PATH_CS_1
0x0004 0x00200000 PATH_CS_0
0x0005 0x05000000 PATH_CS_1
0x0006 0x00200000 PATH_CS_0
0x0007 0x05000000 PATH_CS_1
0x0008 0x00200000 PATH_CS_0
...
```

### Observed Path Activation (dmesg)

The following dmesg excerpts come from a Dell laptop with an AMD USB4 host router (vendor 0x438, device 0x20e) running Linux 6.19.0-rc6 with `thunderbolt.dyndbg=+pt` enabled on the kernel command line. A Dell Thunderbolt 4 Dock (Intel JHL8540, vendor 0x8087, device 0xb26, 19 ports) is connected to domain 1 at route 0x2 (depth 1, through lane adapter port 2). Two OWC Envoy Express Thunderbolt 3 NVMe enclosures (Intel JHL6540, 8086:15c0) are connected downstream of the dock at routes 0x502 (through dock port 5) and 0x702 (through dock port 7). The hop programming logs shown here demonstrate how tb_path_activate() writes Path Entry Structures (2 DWORDs each from struct tb_regs_hop) into the Path Configuration Space of each router along the tunnel path.

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

#### PCIe Tunnel Down Path (Dock to OWC NVMe at Route 702)

When the second OWC NVMe enclosure is hot-plugged at t=28s, the CM creates a PCIe tunnel between dock port 2:12 (PCIe downstream adapter) and device port 702:4 (PCIe upstream adapter). tb_path_activate() programs hops in reverse order (last hop first) to ensure the downstream router is ready before the upstream starts sending:

```
[   28.835999] [3458] thunderbolt 0000:c7:00.6: activating PCIe Down path from 2:12 to 702:4
[   28.836120] [3458] thunderbolt 0000:c7:00.6: 702:1: Writing hop 1
[   28.836122] [3458] thunderbolt 0000:c7:00.6: 702:1:  In HopID: 8 => Out port: 4 Out HopID: 8
[   28.836124] [3458] thunderbolt 0000:c7:00.6: 702:1:   Weight: 1 Priority: 3 Credits: 32 Drop: 0 PM: 0
[   28.836125] [3458] thunderbolt 0000:c7:00.6: 702:1:    Counter enabled: 0 Counter index: 2047
[   28.836127] [3458] thunderbolt 0000:c7:00.6: 702:1:   Flow Control (In/Eg): 1/0 Shared Buffer (In/Eg): 0/0
[   28.836374] [3458] thunderbolt 0000:c7:00.6: 2:12: Writing hop 0
[   28.836377] [3458] thunderbolt 0000:c7:00.6: 2:12:  In HopID: 8 => Out port: 7 Out HopID: 8
[   28.836379] [3458] thunderbolt 0000:c7:00.6: 2:12:   Weight: 1 Priority: 3 Credits: 7 Drop: 0 PM: 0
[   28.836381] [3458] thunderbolt 0000:c7:00.6: 2:12:   Flow Control (In/Eg): 1/1 Shared Buffer (In/Eg): 0/0
[   28.836507] [3458] thunderbolt 0000:c7:00.6: PCIe Down path activation complete
```

Hop 1 (the last hop, programmed first) is at the OWC device's upstream lane adapter (702:1). It routes incoming packets with In HopID 8 to Out port 4 (the PCIe upstream adapter) with Out HopID 8. This entry is written to the Path Configuration Space of adapter 702:1 at HopID 8 (which maps to path entry offset 0x0010 in the Path CS, since protocol adapter paths start at offset 0x0010). The hop has Weight 1 (PCIe uses lower weight than USB3's weight of 2), Priority 3 (highest scheduling priority), and 32 credits (matching the device's max_pcie_credits). Ingress flow control is enabled (In: 1) for back-pressure from the downstream endpoint.

Hop 0 (the first hop, programmed second) is at the dock's PCIe downstream adapter (2:12). It routes In HopID 8 to Out port 7 (the dock's downstream lane adapter facing the OWC device) with Out HopID 8. Credits are 7 (fewer than the device side, reflecting the PCIe adapter's smaller buffer). Both ingress and egress flow control are enabled (In/Eg: 1/1) because this is the source hop.

#### PCIe Tunnel Up Path (OWC NVMe to Dock)

The up path carries PCIe traffic from the device back to the dock:

```
[   28.836509] [3458] thunderbolt 0000:c7:00.6: activating PCIe Up path from 702:4 to 2:12
[   28.836638] [3458] thunderbolt 0000:c7:00.6: 2:7: Writing hop 1
[   28.836641] [3458] thunderbolt 0000:c7:00.6: 2:7:  In HopID: 8 => Out port: 12 Out HopID: 8
[   28.836642] [3458] thunderbolt 0000:c7:00.6: 2:7:   Weight: 1 Priority: 3 Credits: 32 Drop: 0 PM: 0
[   28.836899] [3458] thunderbolt 0000:c7:00.6: 702:4: Writing hop 0
[   28.836901] [3458] thunderbolt 0000:c7:00.6: 702:4:  In HopID: 8 => Out port: 1 Out HopID: 8
[   28.836903] [3458] thunderbolt 0000:c7:00.6: 702:4:   Weight: 1 Priority: 3 Credits: 7 Drop: 0 PM: 0
[   28.837029] [3458] thunderbolt 0000:c7:00.6: PCIe Up path activation complete
```

The up path mirrors the down path with reversed source/destination. Hop 1 is at the dock's downstream lane adapter (2:7, the port facing the OWC device), routing In HopID 8 to Out port 12 (the PCIe downstream adapter on the dock) with Out HopID 8. Hop 0 is at the OWC device's PCIe upstream adapter (702:4), routing to Out port 1 (the upstream lane adapter). The HopID stays at 8 throughout because this is a single-hop tunnel (within the dock-to-device link) and the PCIe adapter's fixed HopID is 8.

#### USB3 Tunnel Hops (Host to Dock)

For comparison, the USB3 tunnel between the host and dock shows different credit and weight parameters:

```
[    5.391406] [12] thunderbolt 0000:c7:00.6: activating USB3 Down path from 0:4 to 2:16
[    5.391539] [12] thunderbolt 0000:c7:00.6: 2:1: Writing hop 1
[    5.391541] [12] thunderbolt 0000:c7:00.6: 2:1:  In HopID: 8 => Out port: 16 Out HopID: 8
[    5.391543] [12] thunderbolt 0000:c7:00.6: 2:1:   Weight: 2 Priority: 3 Credits: 14 Drop: 0 PM: 0
[    5.391547] [12] thunderbolt 0000:c7:00.6: 2:1:   Flow Control (In/Eg): 1/0 Shared Buffer (In/Eg): 0/0
[    5.391800] [12] thunderbolt 0000:c7:00.6: 0:4: Writing hop 0
[    5.391801] [12] thunderbolt 0000:c7:00.6: 0:4:  In HopID: 8 => Out port: 2 Out HopID: 8
[    5.391803] [12] thunderbolt 0000:c7:00.6: 0:4:   Weight: 2 Priority: 3 Credits: 7 Drop: 0 PM: 0
[    5.391805] [12] thunderbolt 0000:c7:00.6: 0:4:   Flow Control (In/Eg): 1/1 Shared Buffer (In/Eg): 0/0
[    5.391930] [12] thunderbolt 0000:c7:00.6: USB3 Down path activation complete
```

USB3 hops use Weight 2 (higher than PCIe's Weight 1, giving USB3 more scheduling bandwidth share). Credits at the dock's lane adapter (2:1) are 14, matching the dock's max_usb3_credits. The host-side USB3 adapter (0:4) has 7 credits. The USB3 tunnel routes from host port 0:4 (USB3 downstream adapter) through host lane adapter 0:2 to dock lane adapter 2:1 and then to dock port 2:16 (USB3 upstream adapter).

#### Mapping Hop Fields to struct tb_regs_hop

Each hop write corresponds to programming a Path Entry Structure (struct tb_regs_hop) which is 2 DWORDs. The fields logged map directly to the struct:

```
DWORD 0: next_hop (Out HopID) [10:0], out_port [16:11], initial_credits [23:17], enable [31]
DWORD 1: weight [3:0], priority [10:8], drop_packages [11], counter [22:12],
         counter_enable [23], ingress_fc [24], egress_fc [25],
         ingress_shared_buffer [26], egress_shared_buffer [27]
```

For example, the hop "2:1: In HopID: 8 => Out port: 16 Out HopID: 8, Weight: 2, Priority: 3, Credits: 14" translates to: DWORD 0 has next_hop=8, out_port=16, initial_credits=14, enable=1. DWORD 1 has weight=2, priority=3. These DWORDs are written via tb_cfg_write() to the Path Configuration Space of adapter 2:1 at the entry corresponding to In HopID 8 (path entry at offset 0x0010, since HopID 8 maps to offset (8 * 2) = 0x0010 in the Path CS).
