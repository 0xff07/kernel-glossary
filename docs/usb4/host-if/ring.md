---
topics: usb4
tags:
    - "usb4"
    - "host-if"
    - "verification-needed"
---

# Rings

## SPECIFICATIONS

- 12.6.3 Registers Description
- Figure 2-37. Descriptor Ring and Data Buffers

## LINUX KERNEL

- [`drivers/thunderbolt/nhi_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h)
- [`'\<ring_desc\>':'drivers/thunderbolt/nhi_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L34)
- [`'\<REG_CAPS\>':'drivers/thunderbolt/nhi_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L113)
- [`'\<REG_TX_RING_BASE\>':'drivers/thunderbolt/nhi_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L52)
- [`'\<REG_RX_RING_BASE\>':'drivers/thunderbolt/nhi_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L62)
- [`'\<REG_TX_OPTIONS_BASE\>':'drivers/thunderbolt/nhi_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L70)
- [`'\<REG_RX_OPTIONS_BASE\>':'drivers/thunderbolt/nhi_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L80)
- [`drivers/thunderbolt/nhi.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c)
- [`'\<ring_desc_base\>':'drivers/thunderbolt/nhi.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c#L177)
- [`'\<ring_options_base\>':'drivers/thunderbolt/nhi.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c#L185)
- [`'\<ring_full\>':'drivers/thunderbolt/nhi.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c#L225)
- [`'\<ring_empty\>':'drivers/thunderbolt/nhi.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c#L230)
- [`'\<nhi_probe\>':'drivers/thunderbolt/nhi.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c#L1365)
- [`'\<tb_nhi\>':'include/linux/thunderbolt.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L497)

## OTHER SOURCES

- [[PATCH] thunderbolt: Fix PCIe device enumeration with delayed rescan](https://lore.kernel.org/all/20260121052744.233517-1-acelan.kao@canonical.com/)

## DETAILS

```
Base Address --------------------> +--------------+          +-------------+
      ^                            |  Descriptor  |--------->| Data Buffer |
      |                            +--------------+          +-------------+
      |                            |  Descriptor  |---+
      |                            +--------------+   |
      |                            |      .       |   |
      |                            |      .       |   |      +-------------+
  Ring Size                        |      .       |   +----->| Data Buffer |
      |                            |      .       |          +-------------+
      |                            +--------------+
      |                            |  Descriptor  |---+
      |                            +--------------+   |      +-------------+
      V                            |  Descriptor  |   +----->| Data Buffer |
+----------------------------------+--------------+          +-------------+
```

### Host Interface Capability Register

This regsiter in Host I/F's MMIO space defines how many rings this host interface supports, Notably, in *Table 12-11. Host Interface Capabilities Register*. In kernel, this is [`REG_CAPS`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L113) in [`drivers/thunderbolt/nhi_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L113).

This value is read during probe, and is stored in `tb_nhi->hop_count`:

```
static int nhi_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
    [...]
    nhi->hop_count = ioread32(nhi->iobase + REG_CAPS) & 0x3ff;
    dev_dbg(dev, "total paths: %d\n", nhi->hop_count);
    [...]
}
```

See [`include/linux/thunderbolt.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L508)

### MMIO registers for rings

There are registers in the MMIO space containing meta data for those rings. This includes where the ring descriptor is and other characteristics of this ring. They are in *Table 12-10. Summary of Memory BAR Registers*. Each ring takes a set of registers in the MMIO space. They are grouped together in `0x00000` and `0x198000` for a transmit ring, `0x08000` and `0x29800` for a receive ring.

These can be found in [`drivers/thunderbolt/nhi_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L52) as well:

```c
/* NHI registers in bar 0 */

/*
 * 16 bytes per entry, one entry for every hop (REG_CAPS)
 * 00: physical pointer to an array of struct ring_desc
 * 08: ring tail (set by NHI)
 * 10: ring head (index of first non posted descriptor)
 * 12: descriptor count
 */
#define REG_TX_RING_BASE    0x00000

/*
 * 16 bytes per entry, one entry for every hop (REG_CAPS)
 * 00: physical pointer to an array of struct ring_desc
 * 08: ring head (index of first not posted descriptor)
 * 10: ring tail (set by NHI)
 * 12: descriptor count
 * 14: max frame sizes (anything larger than 0x100 has no effect)
 */
#define REG_RX_RING_BASE    0x08000

/*
 * 32 bytes per entry, one entry for every hop (REG_CAPS)
 * 00: enum_ring_flags
 * 04: isoch time stamp ?? (write 0)
 * ..: unknown
 */
#define REG_TX_OPTIONS_BASE 0x19800

/*
 * 32 bytes per entry, one entry for every hop (REG_CAPS)
 * 00: enum ring_flags
 *     If RING_FLAG_E2E_FLOW_CONTROL is set then bits 13-23 must be set to
 *     the corresponding TX hop id.
 * 04: EOF/SOF mask (ignored for RING_FLAG_RAW rings)
 * ..: unknown
 */
#define REG_RX_OPTIONS_BASE 0x29800
```

### Helper functions in kernel

The helper function in [`ring_desc_base()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c#L177) to access this structure for each ring:

```
static void __iomem *ring_desc_base(struct tb_ring *ring)
{
	void __iomem *io = ring->nhi->iobase;
	io += ring->is_tx ? REG_TX_RING_BASE : REG_RX_RING_BASE;
	io += ring->hop * 16;
	return io;
}
```

And helper [`ring_options_base()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c#L185)

```
static void __iomem *ring_options_base(struct tb_ring *ring)
{
	void __iomem *io = ring->nhi->iobase;
	io += ring->is_tx ? REG_TX_OPTIONS_BASE : REG_RX_OPTIONS_BASE;
	io += ring->hop * 32;
	return io;
}
```

### Ring Descriptor Structures

A rings contains one or more ring descriptors in main memory. The number of descriptors a ring has is defined in its *Ring Size Register* in the MMIO register of the ring. The format of a ring descriptor for a transfer ring and a receive ring are slightly different, but the usage of the common fields are the same. The specification list them separately, though.

In the Linux, these two types of rings are represented in the same [`struct ring_desc`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L34). This can be found in [`drivers/thunderbolt/nhi_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L34) of the Linux kernel source code:

```c
struct ring_desc {
    u64 phys;
    u32 length:12;
    u32 eof:4;
    u32 sof:4;
    enum ring_desc_flags flags:12;
    u32 time; /* write zero */
} __packed;
```

### Accessing the descriptors

The CM access array of ring descriptor structure in circular manner. See [`ring_full()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c#L225):

```
static bool ring_full(struct tb_ring *ring)
{
	return ((ring->head + 1) % ring->size) == ring->tail;
}
```

And [`ring_empty()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi.c#L230)

```
static bool ring_empty(struct tb_ring *ring)
{
	return ring->head == ring->tail;
}
```

### Observed Ring Operations (dmesg)

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

#### Hop count (ring capacity)

```
[    0.828606] [296] thunderbolt 0000:c7:00.5: total paths: 3
```

The hop count (3) is read from REG_CAPS[9:0] during nhi_probe(). This value determines that the NHI supports 3 TX rings and 3 RX rings. It is stored in tb_nhi->hop_count and used to allocate the tx_rings[] and rx_rings[] pointer arrays.

#### Ring allocation

```
[    0.829118] [296] thunderbolt 0000:c7:00.5: allocating TX ring 0 of size 10
[    0.829168] [296] thunderbolt 0000:c7:00.5: allocating RX ring 0 of size 10
```

Ring 0 (both TX and RX) is allocated for the control channel. "Size 10" means 10 ring descriptors (struct ring_desc) are allocated as a contiguous DMA-coherent array. The physical address of this array is written to the ring's MMIO register at REG_TX_RING_BASE (offset 0x00000) for TX and REG_RX_RING_BASE (offset 0x08000) for RX, each indexed by hop * 16 bytes.

#### Ring start and interrupt enable

```
[    0.841890] [296] thunderbolt 0000:c7:00.5: starting TX ring 0
[    0.841919] [296] thunderbolt 0000:c7:00.5: enabling interrupt at register 0x38200 bit 0 (0x0 -> 0x1)
[    0.841922] [296] thunderbolt 0000:c7:00.5: starting RX ring 0
[    0.841926] [296] thunderbolt 0000:c7:00.5: enabling interrupt at register 0x38200 bit 3 (0x1 -> 0x9)
```

tb_ring_start() enables each ring and its interrupt via ring_interrupt_active(). The interrupt register at 0x38200 (REG_RING_INTERRUPT_BASE) uses one bit per ring. TX rings occupy bits 0 through (hop_count-1), and RX rings occupy bits hop_count through (2*hop_count-1). With hop_count=3, TX ring 0 is bit 0 and RX ring 0 is bit 3. The register value transitions from 0x0 -> 0x1 (TX bit 0 set) -> 0x9 (0b1001, both TX bit 0 and RX bit 3 set).

#### Ring stop during suspend

```
[   24.229671] [170] thunderbolt 0000:c7:00.5: 0: suspending switch
[   24.230254] [170] thunderbolt 0000:c7:00.5: stopping RX ring 0
[   24.230260] [170] thunderbolt 0000:c7:00.5: disabling interrupt at register 0x38200 bit 3 (0x9 -> 0x1)
[   24.230269] [170] thunderbolt 0000:c7:00.5: stopping TX ring 0
[   24.230273] [170] thunderbolt 0000:c7:00.5: disabling interrupt at register 0x38200 bit 0 (0x1 -> 0x0)
[   24.230280] [170] thunderbolt 0000:c7:00.5: control channel stopped
```

During system suspend, tb_ring_stop() disables each ring's interrupt by clearing the corresponding bit in REG_RING_INTERRUPT_BASE. The reverse order (RX first, then TX) ensures no new packets arrive while the TX ring is still active. The register value transitions from 0x9 -> 0x1 (RX bit 3 cleared) -> 0x0 (TX bit 0 cleared). Domain 0 is suspended at t=24s because it has no connected devices (the dock is on domain 1). This demonstrates the symmetry between ring start and stop.
