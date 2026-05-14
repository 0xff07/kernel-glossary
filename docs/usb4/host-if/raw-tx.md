---
topics: usb4
tags:
    - "usb4"
    - "host-if"
    - "verification-needed"
---

# Raw Mode (Transmit Ring)

## LINUX KERNEL

- [`drivers/thunderbolt/nhi_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h): [`RING_FLAG_RAW`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L18) selects raw mode
- [`drivers/thunderbolt/ctl.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/ctl.c): Control channel TX uses raw mode on Ring 0

## SPECIFICATIONS
