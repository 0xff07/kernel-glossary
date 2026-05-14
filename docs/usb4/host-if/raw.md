---
topics: usb4
tags:
    - "usb4"
    - "host-if"
    - "verification-needed"
---

# Raw Mode

## SPECIFICATIONS

- USB4 Specification, section 12.2: Ring Operation Modes

## LINUX KERNEL

- [`drivers/thunderbolt/nhi_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h): Ring flags and descriptor structures
- [`drivers/thunderbolt/ctl.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c): Control channel uses raw mode for Ring 0

### [`enum ring_flags`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L14)

From [`drivers/thunderbolt/nhi_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L14), [`RING_FLAG_RAW`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L18) selects raw mode for a ring:

```c
enum ring_flags {
	RING_FLAG_ISOCH_ENABLE = 1 << 27,       /* TX only? */
	RING_FLAG_E2E_FLOW_CONTROL = 1 << 28,
	RING_FLAG_PCI_NO_SNOOP = 1 << 29,
	RING_FLAG_RAW = 1 << 30,    /* ignore EOF/SOF mask, include checksum */
	RING_FLAG_ENABLE = 1 << 31,
};
```

When [`RING_FLAG_RAW`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L18) is set, the NHI ignores the SOF/EOF masks and the CM is responsible for crafting complete packets, including headers and CRC.

## DETAILS

You hand-craft the packet, including headers and CRC, and put that into the buffer area pointed by the ring descriptor.

The control channel ([`struct tb_ctl`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c#L39)) always uses raw mode for Ring 0 TX and Ring 0 RX, because control packets require the CM to build the complete packet with route string, PDF, and CRC.
