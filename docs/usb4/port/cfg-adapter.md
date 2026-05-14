---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Adapter Config Space

## SPECIFICATIONS

- USB4 Specification, section 8.2.2: Adapter Configuration Space

## LINUX KERNEL

- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): Register structures and defines
- [`drivers/thunderbolt/switch.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/switch.c): Port initialization
- [`'\<tb_regs_port_header\>':'drivers/thunderbolt/tb_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L283): Adapter Config Space header (DW0-DW7)
- [`'\<tb_port_type\>':'drivers/thunderbolt/tb_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L268): Adapter type enum
- [`'\<tb_port_find_cap\>':'drivers/thunderbolt/cap.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/cap.c#L124): Walk adapter capability list

### [`struct tb_regs_port_header`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L283)

[`struct tb_regs_port_header`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L283) maps DW0 through DW7 of the Adapter Configuration Space. It is cached in [`tb_port.config`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L281):

```c
struct tb_regs_port_header {
	/* DWORD 0 */
	u16 vendor_id;
	u16 device_id;
	/* DWORD 1 */
	u32 first_cap_offset:8;
	u32 max_counters:11;
	u32 counters_support:1;
	u32 __unknown1:4;
	u32 revision:8;
	/* DWORD 2 */
	enum tb_port_type type:24;
	u32 thunderbolt_version:8;
	/* DWORD 3 */
	u32 __unknown2:20;
	u32 port_number:6;
	u32 __unknown3:6;
	/* DWORD 4 */
	u32 nfc_credits;
	/* DWORD 5 */
	u32 max_in_hop_id:11;
	u32 max_out_hop_id:11;
	u32 __unknown4:10;
	/* DWORD 6-7 */
	u32 __unknown5;
	u32 __unknown6;
};
```

### Adapter CS Register Defines

From [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L314):

```c
#define ADP_CS_4                        0x04
#define ADP_CS_4_NFC_BUFFERS_MASK       GENMASK(9, 0)
#define ADP_CS_4_TOTAL_BUFFERS_MASK     GENMASK(29, 20)
#define ADP_CS_4_TOTAL_BUFFERS_SHIFT    20
#define ADP_CS_4_LCK                    BIT(31)
#define ADP_CS_5                        0x05
#define ADP_CS_5_LCA_MASK               GENMASK(28, 22)
#define ADP_CS_5_LCA_SHIFT              22
#define ADP_CS_5_DHP                    BIT(31)
```

## REGISTERS

```
+-------------------------------------------------------------------------+
|                        Vendor Defined [31:0]                            | DW0
+-------------------------------------------------------------------------+
| Reserved                    | CCS |   Max CounterSets  | Next Cap Ptr   | DW1
| [31:20]                     | [19]|       [18:8]       | [7:0]          |
+-------------------------------------------------------------------------+
| Reserved       | Type Protocol   | Type Version    | Type SubType       | DW2
| [31:24]        | [23:16]         | [15:8]          | [7:0]              |
+-------------------------------------------------------------------------+
|SBC |FCEE|HEE |  Rsvd | Adapter Number  |  Reserved                      | DW3
|[31]|[30]|[29]|[28:26]| [25:20]         |  [19:0]                        |
+-------------------------------------------------------------------------+
|LCK |Plug|         Total Buffers        | Reserved      |   NFC Buffers  | DW4
|[31]|[30]|            [29:20]           | [19:10]       |      [9:0]     |
+-------------------------------------------------------------------------+
|DHP |FCEE|HEE | Link Credits | Max Output HopID   |  Max Input HopID     | DW5
|[31]|[30]|[29]|   [28:22]    | [21:11]            |       [10:0]         |
+-------------------------------------------------------------------------+
|                            HEC Errors [31:0]                            | DW6
+-------------------------------------------------------------------------+
|                       Invalid HopID Errors [31:0]                       | DW7
+-------------------------------------------------------------------------+
|                            ECC Errors [31:0]                            | DW8
+-------------------------------------------------------------------------+
```

## DETAILS

### The Adapter Type

```
+-------------------------------------------------------------------------+
| Reserved       | Type Protocol   | Type Version    | Type SubType       | DW2
| [31:24]        | [23:16]         | [15:8]          | [7:0]              |
+-------------------------------------------------------------------------+
```

These are all RO fields. This shows:

1. The type of protocol this adapter tunnels.
2. Version of the protocol that this protocol adapter supports.
3. Directions or other protocol-specific traffic type. For example for a DP adapter this may indicate In or OUT. For USB3 and PCIe this indicates downstream or upstream traffic. For USB4 this could indicate that it's a Howst Interface Adapter or a Lane Adapter.

### Fields a Protocol Adapter uses

As complicated as the Adapter Config Space looks like, a Protocol Adapter (USB3, PCIe, DP) doesn't really use all of those fields. Below is the fields that a Protocol Adapter uses:

```
+-------------------------------------------------------------------------+
|                                                                         | DW0
+-------------------------------------------------------------------------+
|                             | CCS |   Max CounterSets  | Next Cap Ptr   | DW1
|                             | [19]|       [18:8]       | [7:0]          |
+-------------------------------------------------------------------------+
|                | Type Protocol   | Type Version    | Type SubType       | DW2
|                | [23:16]         | [15:8]          | [7:0]              |
+-------------------------------------------------------------------------+
|    |    |    |       | Adapter Number  |                                | DW3
|    |    |    |       | [25:20]         |                                |
+-------------------------------------------------------------------------+
|    |Plug|                              |               |   NFC Buffers  | DW4
|    |[30]|                              |               |      [9:0]     |
+-------------------------------------------------------------------------+
|DHP |    |    |              | Max Output HopID   |  Max Input HopID     | DW5
|[31]|    |    |              | [21:11]            |       [10:0]         |
+-------------------------------------------------------------------------+
|                                                                         | DW6
+-------------------------------------------------------------------------+
|                       Invalid HopID Errors [31:0]                       | DW7
+-------------------------------------------------------------------------+
|                                                                         | DW8
+-------------------------------------------------------------------------+
```

### Capability Structure

Each adapter-type has capability structures specific to that type.
