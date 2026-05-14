---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Counter Config Space

## SPECIFICATIONS

- USB4 Specification, section 8.2.4: Counter Configuration Space

## LINUX KERNEL

- [`drivers/thunderbolt/path.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/path.c): Path activation writes counter index into hop registers
- [`'\<tb_path_activate\>':'drivers/thunderbolt/path.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/path.c#L505): Clears counters and configures counter_enable in hop register
- [`'\<tb_regs_hop\>':'drivers/thunderbolt/tb_regs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L502): Hop register with `counter` and `counter_enable` fields
- [`'\<tb_path_hop\>':'drivers/thunderbolt/tb.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L381): `in_counter_index` selects which counter set to use

### Counter Fields in [`struct tb_regs_hop`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L502)

The counter index is stored in each hop register. From [`struct tb_regs_hop`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L502):

```c
struct tb_regs_hop {
	/* DWORD 0 */
	u32 next_hop:11;
	u32 out_port:6;
	u32 initial_credits:7;
	u32 pmps:1;
	u32 unknown1:6;
	bool enable:1;
	/* DWORD 1 */
	u32 weight:4;
	u32 unknown2:4;
	u32 priority:3;
	bool drop_packages:1;
	u32 counter:11;        /* index into TB_CFG_COUNTERS */
	bool counter_enable:1;
	bool ingress_fc:1;
	bool egress_fc:1;
	bool ingress_shared_buffer:1;
	bool egress_shared_buffer:1;
	bool pending:1;
	u32 unknown3:3;
};
```

## REGISTERS

The Counter Config Space consists of multuple Counter Sets. Each Counter Set contains 3 DWs:

```
+-----------------------------------------------+
|         Received Packets Low [31:0]           | CS[0]
+-----------------------------------------------+
|        Received Packets High [31:0]           | CS[0]
+-----------------------------------------------+
|        Dropped Packets High [31:0]            | CS[0]
+-----------------------------------------------+

                      ...

+-----------------------------------------------+
|         Received Packets Low [31:0]           | CS[n]
+-----------------------------------------------+
|        Received Packets High [31:0]           | CS[n]
+-----------------------------------------------+
|        Dropped Packets High [31:0]            | CS[n]
+-----------------------------------------------+
```

## DETAILS

The Counter Config Space is a separate Config Space that an adapter may own. It is an optional Config Space. It contains registers showing statistics of Paths on this Adapter.

### The Counter Config Sets

The Counter Config Sets shows statistic numbers of a Path. These registers are not writable by the CM, but the CM can assocate them with Paths. They provide statictics of a given path.

### The `CCS` and the Max Counter Sets

Not all Adapter implement the Counter Config Space. If the `CCS` bit in Adapter Config Space is set, then this Adapter has a Counter Config Space. How large this Config Space is determined in the Max Counter Sets field.

For example, here's the relevant fields in the Adapter 8, Router 0:

```
$ tbdump --route 0 --adapter 8 ADP_CS_1 --nregs 1 -vv
0x0001 0x00080239 0b00000000 00001000 00000010 00111001 ...9 ADP_CS_1
  [00:07]       0x39 Next Capability Pointer
  [08:18]        0x2 Max Counter Sets
  [19:19]        0x1 Counters Configuration Space Flag (CCS)
  [20:20]        0x0 Bytes Counter Supported (BCS)
  [21:21]        0x0 Received Bytes Counter Enable (RBE)
  [22:22]        0x0 Lock Bytes Counter with TimeOffsetFromHR Low Supported (LBS)
  [23:23]        0x0 Lock Bytes Counter with TimeOffsetFromHR Low Enable (LBE)
```

The `CCS` is `1`, meaning that this Adapter has a Counter Config Space. The Max Counter Sets is `0x02`, meaning that there are 2 "sets" of Counter register. Each Counter Register Sets contains 3 DW. Dumping the Counter Config Space with the `--counter` options in `tbdump` shows exactly 6 DWs:

```
$ tbdump --route 0 --adapter 8 --counter --nregs 999 -v
0x0000 0x00000000
0x0001 0x00000000
0x0002 0x00000000
0x0003 0x00000000
0x0004 0x00000000
0x0005 0x00000000
```

Where the first 3 DW is the 0th Counter Set, and the next 3DW being the 1st Counter Set. For Counter Config Space larger than this the Counter Sets are indexed in similar manner.

### Associate a Counter Set with a Path

Each Path can be associated to at most one Counter Set. To do this, set the Counter ID field in the Path Entry Structure.

```diff
0x0010 0x000e0000 0b00000000 00001110 00000000 00000000 .... PATH_CS_0
  [00:06]        0x0 Output HopID
  [11:16]        0x0 Output Adapter
  [17:23]        0x7 Path Credits Allocated
  [31:31]        0x0 Valid
0x0011 0x01000000 0b00000001 00000000 00000000 00000000 .... PATH_CS_1
  [00:03]        0x0 Weight
  [08:10]        0x0 Priority
+ [12:22]        0x0 Counter ID
  [23:23]        0x0 Counter Enable (CE)
  [24:24]        0x1 Ingress Flow Control Flag (IFC)
  [25:25]        0x0 Egress Flow Control Flag (EFC)
  [26:26]        0x0 Ingress Shared Buffering Enable Flag (ISE)
  [27:27]        0x0 Egress Shared Buffering Enable Flag (ESE)
  [28:28]        0x0 Pending Packets (PP)
```
