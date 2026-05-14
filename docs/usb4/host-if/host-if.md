---
topics: usb4
tags:
    - "usb4"
    - "host-if"
    - "verification-needed"
---

# Host Interface

## SUMMARY

The Host Interface is the NHI (Native Host Interface), a PCIe device that connects the Connection Manager (CM) to the USB4 fabric. It exposes MMIO registers and uses transmit/receive rings in host memory to send and receive packets. In the kernel, [`struct tb_nhi`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L497) represents the NHI, and [`struct tb_ring`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L539) represents each ring. The NHI is probed as a PCI device by [`nhi_probe()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c#L1365).

## SPECIFICATIONS

- 12.3.1 Transmit Descriptor Structure
- 12.4.1 Receive Descriptor Structure
- 12.6.2 Registers Summary
- Table 12-10. Summary of Memory BAR Registers

## LINUX KERNEL

- [`drivers/thunderbolt/nhi.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c): NHI PCI driver
- [`drivers/thunderbolt/nhi_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h): NHI register defines
- [`drivers/thunderbolt/ctl.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c): Control channel (Ring 0)
- [`include/linux/thunderbolt.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h): Public API (tb_nhi, tb_ring)
- [`'\<tb_nhi\>':'include/linux/thunderbolt.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L497): NHI structure
- [`'\<tb_ring\>':'include/linux/thunderbolt.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L539): Ring structure (TX or RX)
- [`'\<ring_desc\>':'drivers/thunderbolt/nhi_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L34): Ring descriptor (DMA)
- [`'\<nhi_probe\>':'drivers/thunderbolt/nhi.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c#L1365): PCI probe function
- [`'\<tb_ctl\>':'drivers/thunderbolt/ctl.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c#L39): Control channel (uses Ring 0)

### [`struct tb_ring`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L539)

[`struct tb_ring`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L539) represents a TX or RX ring associated with the NHI. The `hop` field corresponds to the HopID in the Host I/F's Path Configuration Space:

```c
struct tb_ring {
	spinlock_t lock;
	struct tb_nhi *nhi;
	int size;
	int hop;
	int head;
	int tail;
	struct ring_desc *descriptors;
	dma_addr_t descriptors_dma;
	struct list_head queue;
	struct list_head in_flight;
	struct work_struct work;
	bool is_tx:1;
	bool running:1;
	int irq;
	u8 vector;
	unsigned int flags;
	int e2e_tx_hop;
	u16 sof_mask;
	u16 eof_mask;
	void (*start_poll)(void *data);
	void *poll_data;
};
```

### [`struct tb_ctl`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c#L39)

[`struct tb_ctl`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c#L39) manages the control channel, using Ring 0 TX and Ring 0 RX in raw mode:

```c
struct tb_ctl {
	struct tb_nhi *nhi;
	struct tb_ring *tx;
	struct tb_ring *rx;
	struct dma_pool *frame_pool;
	struct ctl_pkg *rx_packets[TB_CTL_RX_PKG_COUNT];
	struct mutex request_queue_lock;
	struct list_head request_queue;
	bool running;
	int timeout_msec;
	event_cb callback;
	void *callback_data;
	int index;
};
```

## OTHER SOURCES

- [[PATCH] thunderbolt: Fix PCIe device enumeration with delayed rescan](https://lore.kernel.org/all/20260121052744.233517-1-acelan.kao@canonical.com/)

## DETAILS

Host Interaface uses rings to communicate with the fabric. To the USB4 fabric, Host I/F looks like a adapter (called Host I/F Adapter) and has its own Path Configuration Space However, packets routed to this adapter will pop up on the ring buffers (or "rings") in the main memory. A ring is either a trasmit ring or a receive ring. There can be at most 21 transmit rings and 21 receive rings. The HopID in the Path Configuration Space indicates which ring the Host I/F should put this packet into. CM set up those rings through Host Interface's MMIO registers, as well as memory structures defined by the specification.

CM can also transmit packets through rings. A packet sent through `i`'th "transmit ring" is considered to have HopID `i`. This is then routed through the Path Configuration Space of the Host I/F Adapter.

The control packets are also sent/received through those rings. Transmit Ring 0 and Receive Ring 0 in particular.

### Host interface itself is also an adapter

Its adapter number is implementation defined, but it can be found in the `ROUTER_ADP_CS_1`. The Host I/F adapter is the upstream adapter of the Host Router.

### Receiving control traffic

Every router routes upstream control packets (i.e. control packets where CM bit is set to `1`) to its upstream adapter. The adapter number of the upstream adapter is coded in `ROUTER_ADP_CS_1`. For Host Router, its upstream adapter is always the Host I/F, so all the upstream control traffic is eventually routed to the Host I/F. In particular, the 0'th receive ring is exclusively reserved for receiving those control packets.

### Sending control traffic

To send control packet downstream, craft the packet and put that to transmit ring 0. Note that the routing information is in the route string field of the crafted packet.

### Receiving/sending other tunneled traffic

The i'th ring, where i >= 1, can be used to send/receive tunneled packets (depending on if it's a transmit ring or a receive ring). The packet routing follows the Path Configuration Space of the Host I/F. Packets targeting HopID `i` of the Host I/F will get routed to `i`th receive ring. On the other hand, packets written to `i`'th transmit ring will be treated as a packet with HopID `i` and routed according to the Host I/F's Path Configuration Space.

### Observed NHI Initialization (dmesg)

The following dmesg excerpts come from a Dell laptop with an AMD USB4 host router (vendor 0x438, device 0x20e) running Linux 6.19.0-rc6 with `thunderbolt.dyndbg=+pt` enabled on the kernel command line. This system has two independent USB4 domains: domain 0 at PCI function 0000:c7:00.5 (behind root port 00:01.1) and domain 1 at PCI function 0000:c7:00.6 (behind root port 00:01.2). Each domain has its own NHI, control channel, and topology. A Dell Thunderbolt 4 Dock (Intel JHL8540 controller, vendor 0x8087, device 0xb26, with 19 ports) is connected to domain 1 at route 0x2 (depth 1, through lane adapter port 2). Two OWC Envoy Express Thunderbolt 3 NVMe enclosures (Intel JHL6540, vendor 0x8086, device 0x15c0) are connected downstream of the dock at routes 0x502 and 0x702 (depth 2).

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

During NHI probe, the driver reads REG_CAPS from the NHI's MMIO BAR to determine the hop count (the total number of TX/RX ring pairs the hardware supports). This value controls how many paths can be established through the host interface. The IOMMU DMA protection line indicates that external Thunderbolt/USB4 devices are placed behind an IOMMU, so their DMA transactions go through address translation and access control rather than having unrestricted access to host memory.

```
[    0.828606] [296] thunderbolt 0000:c7:00.5: total paths: 3
[    0.828609] [296] thunderbolt 0000:c7:00.5: IOMMU DMA protection is enabled
```

Ring 0 (both TX and RX) is reserved exclusively for control channel traffic. The driver allocates these rings with a size of 10, meaning 10 DMA descriptors per ring (each descriptor points to one frame buffer in host memory). These are Raw Mode rings, which means the NHI does not interpret the packet contents or perform any HopID-based routing on them; instead, the control channel code in ctl.c handles framing, sequencing, and retransmission of control packets directly.

```
[    0.829118] [296] thunderbolt 0000:c7:00.5: allocating TX ring 0 of size 10
[    0.829168] [296] thunderbolt 0000:c7:00.5: allocating RX ring 0 of size 10
[    0.829207] [296] thunderbolt 0000:c7:00.5: control channel created
```

The connection manager (CM) selection determines who manages the USB4 topology. The software connection manager means the Linux kernel has full control over router enumeration, path setup, tunnel creation, and security policy. The alternative is ICM (Intel Connection Manager), where firmware running on the host controller handles these tasks and the kernel communicates with it through a mailbox protocol. AMD USB4 host routers use the software CM.

```
[    0.829209] [296] thunderbolt 0000:c7:00.5: using software connection manager
```

The ACPI domain linkage maps the NHI to the PCI root port that sits above it in the PCI hierarchy. This association is used for power management coordination and for matching ACPI companion devices (_SB scope entries) to the correct USB4 domain. In this case, the NHI at 0000:c7:00.5 is reached through root port 0000:00:01.1.

```
[    0.841709] [296] thunderbolt 0000:c7:00.5: created link from 0000:00:01.1
```

Once the NHI is fully initialized, it starts the control channel by activating both Ring 0 TX and Ring 0 RX. Each ring gets its own interrupt bit in the NHI interrupt register at REG_RING_INTERRUPT_BASE (offset 0x38200 in the MMIO BAR). TX ring N uses bit N, and RX ring N uses bit (N + hop_count). Since this NHI has hop_count=3, TX ring 0 gets bit 0 and RX ring 0 gets bit 3. The register value transitions from 0x0 to 0x1 (TX ring 0 enabled) and then from 0x1 to 0x9 (both TX ring 0 at bit 0 and RX ring 0 at bit 3 enabled, since 0x9 = 0b1001).

```
[    0.841889] [296] thunderbolt 0000:c7:00.5: NHI initialized, starting thunderbolt
[    0.841890] [296] thunderbolt 0000:c7:00.5: control channel starting...
[    0.841890] [296] thunderbolt 0000:c7:00.5: starting TX ring 0
[    0.841919] [296] thunderbolt 0000:c7:00.5: enabling interrupt at register 0x38200 bit 0 (0x0 -> 0x1)
[    0.841922] [296] thunderbolt 0000:c7:00.5: starting RX ring 0
[    0.841926] [296] thunderbolt 0000:c7:00.5: enabling interrupt at register 0x38200 bit 3 (0x1 -> 0x9)
```
