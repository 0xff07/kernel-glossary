---
topics: usb4
tags:
    - "usb4"
    - "host-if"
    - "verification-needed"
---

# Frame Mode (Receive Ring)

## LINUX KERNEL

- [`drivers/thunderbolt/nhi_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h): Ring options registers
- [`'\<REG_RX_OPTIONS_BASE\>':'drivers/thunderbolt/nhi_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L80): RX options include SOF/EOF mask

In frame mode, the RX ring options register at [`REG_RX_OPTIONS_BASE`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L80) contains the SOF/EOF mask that tells the NHI which packet boundaries to match:

```c
/*
 * 32 bytes per entry, one entry for every hop (REG_CAPS)
 * 00: enum ring_flags
 *     If RING_FLAG_E2E_FLOW_CONTROL is set then bits 13-23 must be set to
 *     the corresponding TX hop id.
 * 04: EOF/SOF mask (ignored for RING_FLAG_RAW rings)
 */
#define REG_RX_OPTIONS_BASE	0x29800
```

## SPECIFICATIONS

## DETAILS

Frame mode also works on the receive side. You tell the Host I/F what SOF/EOF to look for in the Receive Ring Descriptor, and the Host I/F will aggregate those packets and put them nicely in the Receive Ring Descriptor.
