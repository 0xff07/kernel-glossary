---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Capability Structures

In the Router Configuration Space and the Adapter Configuration Space, the Next Capability Pointer points to a list of capability structures.

The USB4 specification defines "Required", "Optional" capability structures. In the capability structure list, they have to sorted in that order.

## SPECIFICATIONS

- 8.2.1 Router Configuration Space
- 8.2.2 Adapter Configuration Space

## LINUX KERNEL

- [`drivers/thunderbolt/cap.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/cap.c): Capability walking and lookup
- [`'\<tb_switch_find_cap\>':'drivers/thunderbolt/cap.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/cap.c#L198): Walks the router capability list
- [`'\<tb_port_find_cap\>':'drivers/thunderbolt/cap.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/cap.c#L124): Walks the adapter capability list

### [`tb_switch_find_cap()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/cap.c#L198)

[`tb_switch_find_cap()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/cap.c#L198) walks the capability linked list in the Router Configuration Space, starting from the `first_cap_offset` in DW1. Each capability header contains a next-pointer and a capability ID:

```c
int tb_switch_find_cap(struct tb_switch *sw, enum tb_switch_cap cap)
{
	int offset = 0;

	do {
		struct tb_cap_any header;
		int ret;

		offset = tb_switch_next_cap(sw, offset);
		if (offset < 0)
			return offset;

		ret = tb_sw_read(sw, &header, TB_CFG_SWITCH, offset, 1);
		if (ret)
			return ret;

		if (header.basic.cap == cap)
			return offset;
	} while (offset);

	return -ENOENT;
}
```

### [`tb_port_find_cap()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/cap.c#L124)

[`tb_port_find_cap()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/cap.c#L124) walks the capability list in the Adapter Configuration Space. It temporarily enables the TMU for the port read, which is required for accessing some adapter registers:

```c
int tb_port_find_cap(struct tb_port *port, enum tb_port_cap cap)
{
	int ret;

	ret = tb_port_enable_tmu(port, true);
	if (ret)
		return ret;

	ret = __tb_port_find_cap(port, cap);

	tb_port_dummy_read(port);
	tb_port_enable_tmu(port, false);

	return ret;
}
```

## DETAILS

### Required Capability Structure

For Router Config Space, the only required is the TMU. For Adapter Config Space, each protocol adapters has their own requirements.

### Optional Capability Structure

The only thing currently defined is the Vendor Specific Structure. There are 2 format. Vendor Specific Capability Structure and the Vendor Specific Extended Capability Structure.

In `tbman` they will be named `VCS_CS_m_CS_n`, where `m` and `n` depend on the numbering of capability structure and registers in that capability structure.
