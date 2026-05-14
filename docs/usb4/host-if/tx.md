---
topics: usb4
tags:
    - "usb4"
    - "host-if"
    - "verification-needed"
---

# Transfer Rings

## SPECIFICATIONS

- 12.3.1 Transmit Descriptor Structure
- 12.6.3.2 Transmit Descriptor Rings

## LINUX KERNEL

- [`drivers/thunderbolt/nhi_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h)
- [`'\<REG_TX_RING_BASE\>':'drivers/thunderbolt/nhi_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L52)
- [`'\<REG_TX_OPTIONS_BASE\>':'drivers/thunderbolt/nhi_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L70)

## DETAILS

### MMIO registers for transfer rings

See *Table 12-10. Summary of Memory BAR Registers*, notable the *Transmit Descriptor Rings*. Also [`drivers/thunderbolt/nhi_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L52) in the Linux kernel. Notably:

```
/*
 * 16 bytes per entry, one entry for every hop (REG_CAPS)
 * 00: physical pointer to an array of struct ring_desc
 * 08: ring tail (set by NHI)
 * 10: ring head (index of first non posted descriptor)
 * 12: descriptor count
 */
#define REG_TX_RING_BASE    0x00000
```

And:

```
/*
 * 32 bytes per entry, one entry for every hop (REG_CAPS)
 * 00: enum_ring_flags
 * 04: isoch time stamp ?? (write 0)
 * ..: unknown
 */
#define REG_TX_OPTIONS_BASE 0x19800
```

### Transmit Descriptor Structure

This is the memory structure pointed to by the Base Address High/Base Address Low in the MMIO register above:

```
31                                                                    0
+---------------------------------------------------------------------+
|                             Address Low                             |
+---------------------------------------------------------------------+
|                             Address High                            |
+-----------------+-+-+-+----+-------+-------+------------------------+
|                 | | | |    |  SOF  |  EOF  |                        |
|      Offset     |I|R|D| Rs |       |       |      Data Length       |
|                 |E|S|D| vd |  PDF  |  PDF  |                        |
+-----------------+-+-+-+----+-------+-------+------------------------+
|                                Rsvd                                 |
+---------------------------------------------------------------------+
```

The CM needs to initialize this structure first. The Host I/F consumes this. Once the transmission is finished, this memory structure is consideded used and needs to re-initialized by the CM again before being used again by the Host I/F.
