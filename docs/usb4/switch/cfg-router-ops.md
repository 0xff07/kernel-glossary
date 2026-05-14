---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Router Config Space (Operation Region)

```
+---------------------------+------------------+
|       Product ID(31:16)   | Vendor ID(15:0)  | DW00
+--------+----+-------+-------+--------+-------+
|Revision|(R) |Depth  |MaxAdp |UpstrAdp|NextCap|
|(31:24) |(23)|(22:20)|(19:14)|(13:8)  |(7:0)  | DW01
+--------+----+-------+-------+--------+-------+
|              Topology ID Low (31:0)          | DW02
+----+--------+--------------------------------+
|TIDV|Reserved|     Topology ID High (23:0)    | DW03
|(31)|(30:24) |            (23:0)              |
+---+---------+--------------------------------+
| USB4Ver | Reserved |    CMUV   |  Notify TO  | DW04
|(31:24)  |(23:16)   |   (15:8)  |    (7:0)    |
+---------+----------+-----------+-------------+
|               Router CS 5                    | DW05
+----------------------------------------------+
|               Router CS 6                    | DW06
+----------------------------------------------+
|             UUID High (31:0)                 | DW07
+----------------------------------------------+
|             UUID Low (31:0)                  | DW08
+----------------------------------------------+
|                 Data[n]                      | DW09 - DW24
+----------------------------------------------+
|                 Metadata                     | DW25
+----+----+---------+----------+---------------+
|OV  |ONS | STATUS  | Reserved | OPCODE(15:0)  | DW26
|(31)|(30)| (29:24) | (23:16)  |               |
+----+----+---------+----------+---------------+
```

## SPECIFICATIONS

- USB4 Specification, section 8.2.1: Router Configuration Space

## LINUX KERNEL

- [`drivers/thunderbolt/usb4.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c): USB4 router operations
- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): Register offset and bit field defines
- [`'\<usb4_native_switch_op\>':'drivers/thunderbolt/usb4.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L54): Issues a router operation via the operation region
- [`'\<tb_sw_read\>':'drivers/thunderbolt/tb.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L671): Read from router config space
- [`'\<tb_sw_write\>':'drivers/thunderbolt/tb.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L685): Write to router config space

### Operation Region Register Defines

From [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L221):

```c
#define ROUTER_CS_9                 0x09  /* Data[0] */
#define ROUTER_CS_25                0x19  /* Metadata */
#define ROUTER_CS_26                0x1a  /* Opcode / Status / OV */
#define ROUTER_CS_26_OPCODE_MASK    GENMASK(15, 0)
#define ROUTER_CS_26_STATUS_MASK    GENMASK(29, 24)
#define ROUTER_CS_26_STATUS_SHIFT   24
#define ROUTER_CS_26_ONS            BIT(30)  /* Opcode Not Supported */
#define ROUTER_CS_26_OV             BIT(31)  /* Opcode Valid (doorbell) */
```

### [`usb4_native_switch_op()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L54)

[`usb4_native_switch_op()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L54) implements the router operation protocol. It writes parameters to [`ROUTER_CS_9`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L221) (Data), [`ROUTER_CS_25`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L222) (Metadata), sets the opcode and doorbell in [`ROUTER_CS_26`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L223), then polls for the doorbell to clear:

```c
static int usb4_native_switch_op(struct tb_switch *sw, u16 opcode,
				 u32 *metadata, u8 *status,
				 const void *tx_data, size_t tx_dwords,
				 void *rx_data, size_t rx_dwords)
{
	u32 val;
	int ret;

	if (metadata) {
		ret = tb_sw_write(sw, metadata, TB_CFG_SWITCH,
				  ROUTER_CS_25, 1);
		if (ret)
			return ret;
	}
	if (tx_dwords) {
		ret = tb_sw_write(sw, tx_data, TB_CFG_SWITCH,
				  ROUTER_CS_9, tx_dwords);
		if (ret)
			return ret;
	}

	val = opcode | ROUTER_CS_26_OV;
	ret = tb_sw_write(sw, &val, TB_CFG_SWITCH, ROUTER_CS_26, 1);
	if (ret)
		return ret;

	ret = tb_switch_wait_for_bit(sw, ROUTER_CS_26,
				     ROUTER_CS_26_OV, 0, 500);
	if (ret)
		return ret;

	ret = tb_sw_read(sw, &val, TB_CFG_SWITCH, ROUTER_CS_26, 1);
	if (ret)
		return ret;

	if (val & ROUTER_CS_26_ONS)
		return -EOPNOTSUPP;

	if (status)
		*status = (val & ROUTER_CS_26_STATUS_MASK) >>
			ROUTER_CS_26_STATUS_SHIFT;

	if (metadata) {
		ret = tb_sw_read(sw, metadata, TB_CFG_SWITCH,
				 ROUTER_CS_25, 1);
		if (ret)
			return ret;
	}
	if (rx_dwords) {
		ret = tb_sw_read(sw, rx_data, TB_CFG_SWITCH,
				 ROUTER_CS_9, rx_dwords);
		if (ret)
			return ret;
	}

	return 0;
}
```

## REGISTERS

```
+----------------------------------------------+
|                 Data[n]                      | DW09 - DW24
+----------------------------------------------+
|                 Metadata                     | DW25
+----+----+---------+----------+---------------+
|OV  |ONS | STATUS  | Reserved | OPCODE(15:0)  | DW26
|(31)|(30)| (29:24) | (23:16)  |               |
+----+----+---------+----------+---------------+
```

## DETAILS

This is another mechanism in USB4 to perform some extra operations to the Router. This mechanism uses the `OV` as doorbell, where:

1. The CM place Opcode and the parameters to be consumed by the Router in `Data[0]` to `Data[15]` and `Metadata`.
2. CM set `OV` to `1`. This triggers Router's operation.
3. When the Router is done, it clears the `OV` field back to `0`.
