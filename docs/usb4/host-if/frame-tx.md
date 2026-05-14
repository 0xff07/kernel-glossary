---
topics: usb4
tags:
    - "usb4"
    - "host-if"
    - "verification-needed"
---

# Frame Mode (Transmit Rings)

## LINUX KERNEL

- [`drivers/thunderbolt/nhi_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h): Ring descriptor with `sof` and `eof` fields
- [`'\<ring_desc\>':'drivers/thunderbolt/nhi_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L34): Descriptor structure used in frame mode

In frame mode (when [`RING_FLAG_RAW`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L18) is not set), the `sof` and `eof` fields in [`struct ring_desc`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/nhi_regs.h#L34) tell the NHI how to segment data into packets:

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

## SPECIFICATIONS

## DETAILS

You put a chunk of data in the transmit descriptor, and the SOF/EOF number you'd like to use, and let the Host I/F figure out how to segment that chunk of data into packets.
