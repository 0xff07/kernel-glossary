---
topics: usb4
tags:
    - "usb4"
    - "host-if"
    - "verification-needed"
---

# Receive Rings

## SPECIFICATIONS

- 12.6.3.3 Receive Descriptor Rings

## LINUX KERNEL

- [`drivers/thunderbolt/nhi_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h)
- [`'\<REG_RX_RING_BASE\>':'drivers/thunderbolt/nhi_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L62)
- [`'\<REG_RX_OPTIONS_BASE\>':'drivers/thunderbolt/nhi_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L80)

## DETAILS

### MMIO registers for receive rings

See *Table 12-10. Summary of Memory BAR Registers*, notably the *Receive Descriptor Rings*. Also [`drivers/thunderbolt/nhi_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L62) in the Linux kernel. Notably `0x80000` and `0x29800`:

```
/*
 * 16 bytes per entry, one entry for every hop (REG_CAPS)
 * 00: physical pointer to an array of struct ring_desc
 * 08: ring head (index of first not posted descriptor)
 * 10: ring tail (set by NHI)
 * 12: descriptor count
 * 14: max frame sizes (anything larger than 0x100 has no effect)
 */
#define REG_RX_RING_BASE	0x08000
```

And:

```
/*
 * 32 bytes per entry, one entry for every hop (REG_CAPS)
 * 00: enum ring_flags
 *     If RING_FLAG_E2E_FLOW_CONTROL is set then bits 13-23 must be set to
 *     the corresponding TX hop id.
 * 04: EOF/SOF mask (ignored for RING_FLAG_RAW rings)
 * ..: unknown
 */
#define REG_RX_OPTIONS_BASE	0x29800
```

### Receive Descriptor Structure

This is the memory structure pointed to by the Base Address High/Base Address Low in the MMIO register above:

```

31                                                            0
+---------------------------------------------------------------+
|                          Address Low                          |
+---------------------------------------------------------------+
|                          Address High                         |
+----------+--+--+--+-------------------------------------------+
|          |  |  |  |                                           |
|          | I| R| D|                                           |
|  Offset  | E| S| D|             Reserved                      |
|          |  |  |  |                                           |
|          |  |  |  |                                           |
+----------+--+--+--+-------------------------------------------+
|                             Rsvd                              |
+---------------------------------------------------------------+
```

The CM needs to initialize this structure first. The Host I/F consumes this. Once the receive is finished, this memory structure is consideded used and needs to re-initialized by the CM again before being used again by the Host I/F.

A "depleted" descriptor contains extra information left by the Host I/F.

```
31                                                            0
+---------------------------------------------------------------+
|                          Address Low                          |
+---------------------------------------------------------------+
|                          Address High                         |
+----------+--+--+--+--+-------+-------+------------------------+
|          |  |  |  |  |       |       |                        |
|          | I| B| D| C|  SOF  |  EOF  |                        |
|  Offset  | E| O| D| R|  PDF  |  PDF  |      Data Length       |
|          |  | D|  | C|       |       |                        |
|          |  |  |  |  |       |       |                        |
+----------+--+--+--+--+-------+-------+------------------------+
|                             Rsvd                              |
+---------------------------------------------------------------+
```
