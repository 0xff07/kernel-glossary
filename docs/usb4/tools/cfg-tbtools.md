---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# `tbtools`

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

## LINUX KERNEL

- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): Register offset defines used by `tbdump` naming convention
- [`drivers/thunderbolt/debugfs.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/debugfs.c): Kernel debugfs interface for thunderbolt

The register names used by `tbdump` and `tbman` (e.g., [`ROUTER_CS_4`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L199), [`PORT_CS_18`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L384), [`ADP_PCIE_CS_0`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L475)) follow the same naming convention as the kernel defines in [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h).

## SUMMARY

The Config Space can be accessed with the [`tbtools`](https://github.com/intel/tbtools)

## DETAILS

### By `tbman`

`tbman` is a tool that allows user tweaking the USB4 subsystem interactively.

Run `tbman` the hit `F8` to enter the `F8 Regs` pop-up menu. In the pop-up menu:

1. Hit enter in the `None` in the `Config space` to switch to the Config Space you'd like to access (in this case, choose the `Router`).
2. Select proper `ROUTER_CS_n` and hit enter to see detailed fields in that register.

The postfix (`n` in the `ROUTER_CS_n`) suggest which DW it is in the Config Space. For example, `ROUTER_CS_0` shows the Produce ID and the Vendor ID, while [`ROUTER_CS_7`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L220) is the upper 32-bit of UUID.

### By `tbdump`

`tbdump` allows user to dump Config Space register in command line. For a Router:

```
$ tbdump -d ${DOMAIN#} --router ${ROUTER#} --nregs 1 ${REG_NAME} -vv
```

For example, for Router `0` on Domain `0` and register [`ROUTER_CS_4`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L199):

```
$ tbdump -d 0 --router 0 -nregs 1 ROUTER_CS_4 -vv
```

This shows the content of `DW4` in the Router Config Space:`

```
0x0004 0x2000100a 0b00100000 00000000 00010000 00001010 .... ROUTER_CS_4
  [00:07]        0xa Notification Timeout
  [08:15]       0x10 Connection Manager USB4 Version (CMUV)
  [24:31]       0x20 USB4 Version
```

Again, the postfix in the register name suggests the offset in the Config Space. For exanple, [`ROUTER_CS_7`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L220) register is in `DW7` of the Config Space.
