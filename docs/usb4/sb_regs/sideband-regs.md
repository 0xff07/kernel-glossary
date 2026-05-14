---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Sideband Registers

Sideband registers are registers on the PHYs. Each PHY on a USB4 port or a USB4 retimer has a set of those register. Under the hood they use the sideband bus on the type-c connector to communicate.

## SPECIFICATIONS

- USB4 Specification, section 4.1.1.3.3: SB Register Definitions
- USB4 Specification, section 8.2.2.4: USB4 Port Capability
- USB4 Specification, section 8.3.2: Port Operations
- USB4 Specification, Table 8-64: List of Port Operations

## LINUX KERNEL

- [`drivers/thunderbolt/sb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h): Sideband register address defines and opcode enum
- [`drivers/thunderbolt/usb4.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c): Sideband read/write implementation
- [`'\<usb4_port_sb_read\>':'drivers/thunderbolt/usb4.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L1354): Read sideband register via PORT_CS_1
- [`'\<usb4_sb_target\>':'drivers/thunderbolt/tb.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L1374): Target selection (Router, Partner, Retimer)

### SB Register Address Defines

From [`drivers/thunderbolt/sb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h#L13):

```c
#define USB4_SB_VENDOR_ID        0x00
#define USB4_SB_PRODUCT_ID       0x01
#define USB4_SB_FW_VERSION       0x02
#define USB4_SB_DEBUG_CONF       0x05
#define USB4_SB_DEBUG            0x06
#define USB4_SB_LRD_TUNING       0x07
#define USB4_SB_OPCODE           0x08
#define USB4_SB_METADATA         0x09
#define USB4_SB_LINK_CONF        0x0c
#define USB4_SB_GEN23_TXFFE      0x0d
#define USB4_SB_GEN4_TXFFE       0x0e
#define USB4_SB_VERSION          0x0f
#define USB4_SB_DATA             0x12
```

### [`enum usb4_sb_opcode`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h#L21)

[`enum usb4_sb_opcode`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h#L21) defines Port Operation opcodes written to the SB Opcode register (0x08):

```c
enum usb4_sb_opcode {
	USB4_SB_OPCODE_ERR = 0x20525245,              /* "ERR " */
	USB4_SB_OPCODE_ONS = 0x444d4321,              /* "!CMD" */
	USB4_SB_OPCODE_ROUTER_OFFLINE = 0x4e45534c,   /* "LSEN" */
	USB4_SB_OPCODE_ENUMERATE_RETIMERS = 0x4d554e45,/* "ENUM" */
	USB4_SB_OPCODE_SET_INBOUND_SBTX = 0x5055534c, /* "LSUP" */
	USB4_SB_OPCODE_UNSET_INBOUND_SBTX = 0x50555355,/* "USUP" */
	USB4_SB_OPCODE_QUERY_LAST_RETIMER = 0x5453414c,/* "LAST" */
	USB4_SB_OPCODE_NVM_AUTH_WRITE = 0x48545541,    /* "AUTH" */
	USB4_SB_OPCODE_NVM_READ = 0x52524641,          /* "AFRR" */
	USB4_SB_OPCODE_READ_LANE_MARGINING_CAP = 0x50434452, /* "RDCP" */
	USB4_SB_OPCODE_RUN_HW_LANE_MARGINING = 0x474d4852,   /* "RHMG" */
	USB4_SB_OPCODE_RUN_SW_LANE_MARGINING = 0x474d5352,   /* "RSMG" */
	USB4_SB_OPCODE_READ_SW_MARGIN_ERR = 0x57534452,      /* "RDSW" */
};
```

### [`usb4_port_sb_read()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L1354)

[`usb4_port_sb_read()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L1354) reads a sideband register by writing the target, address, and length to [`PORT_CS_1`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L374), setting the PND (Pending) bit, then polling until it clears:

```c
int usb4_port_sb_read(struct tb_port *port, enum usb4_sb_target target,
		      u8 index, u8 reg, void *buf, u8 size)
{
	size_t dwords = DIV_ROUND_UP(size, 4);
	int ret;
	u32 val;

	val = reg;
	val |= size << PORT_CS_1_LENGTH_SHIFT;
	val |= (target << PORT_CS_1_TARGET_SHIFT) & PORT_CS_1_TARGET_MASK;
	if (target == USB4_SB_TARGET_RETIMER)
		val |= (index << PORT_CS_1_RETIMER_INDEX_SHIFT);
	val |= PORT_CS_1_PND;

	ret = tb_port_write(port, &val, TB_CFG_PORT,
			    port->cap_usb4 + PORT_CS_1, 1);
	if (ret)
		return ret;

	ret = usb4_port_wait_for_bit(port, port->cap_usb4 + PORT_CS_1,
				     PORT_CS_1_PND, 0, 500,
				     USB4_PORT_SB_DELAY);
	if (ret)
		return ret;

	/* ... check NR and RC bits, then read data ... */
	return buf ? usb4_port_read_data(port, buf, dwords) : 0;
}
```

## REGISTERS

### The SB Registers Space

Sideband Registers is a group of registers:

```
+-------------------+
|     Vendor ID     | 0x00 (4 byte)
+-------------------+
|     Product ID    | 0x01 (4 byte)
+-------------------+
|     (Reserved)    | 0x02-0x04
+-------------------+
|     Debug Config  | 0x05 (4 byte)
+-------------------+
|     Debug Data    | 0x06 (4 byte)
+-------------------+
|     LRD Tuning    | 0x07 (4 byte)
+-------------------+
|     Opcode        | 0x08 (4 byte)
+-------------------+
|     Metadata      | 0x09 (4 byte)
+-------------------+
|     (Undefined)   | 0x0A-0x0B
+-------------------+
|     Link Config   | 0x0C (3 byte)
+-------------------+
|     Gen2/3 TxFFE  | 0x0D (4 byte)
+-------------------+
|     Gen4 TxFFE    | 0x0E
+-------------------+
|     SBCh Version  | 0c0F (4 byte)
+-------------------+
| (Vendor Specific) | 0x10-0x11
+-------------------+
|                   |
|        Data       | 0x12 (64 bytes)
|                   |
+-------------------+
```

Some of the registers are status to be read, and some of them allow doing Port Operations, like what CM can ask the Router to perform certain Router Operation through the ROuter Config Space.

### 0x0C Link Configuration

```
+-------------------------------------------------------------------------------------------------------------+
|  (Reserved)  | Req Asym Rx  | Req Asym Tx | Asym Sup 3Rx | Asym Sup 3Tx | Gen4 Sup    | TBT3-Comp| SBCh Sup |
|     [23]     |      [22]    |      [21]   |      [20]    |      [19]    |   [18]      |   [17]   |   [16]   |
+-------------------------------------------------------------------------------------------------------------+
| RS-FEC Req G3| RS-FEC Req G2| Gen3 Sup    | Bonding Sup  |           Reserved         | EnReq L1 | EnReq L0 |
|      [15]    |      [14]    |     [13]    |     [12]     |            [11:10]         |    [9]   |    [8]   |
+-------------------------------------------------------------------------------------------------------------+
|                           (Reserved)                     | Asym Dec Rx  | Asym Dec Tx | EnDec L1 | EnDec L0 |
|                             [7:4]                        |      [3]     |     [2]     |    [1]   |    [0]   |
+-------------------------------------------------------------------------------------------------------------+
```

### 0x0D TxFFE

```
+--------------------------------------------------------------------------------------------------+
| Lane 1 Tx Active | Lane 1 Req Done | Lane 1 Rsvd      |            Lane 1 Setting                |
|      [31]        |      [30]       |     [29:28]      |                [27:24]                   |
+--------------------------------------------------------------------------------------------------+
| Lane 0 Tx Active | Lane 0 Req Done | Lane 0 Rsvd      |             Lane 0 Setting               |
|      [23]        |      [22]       |     [21:20]      |                [19:16]                   |
+--------------------------------------------------------------------------------------------------+
| Lane1 New Req    | Lane 1 Clk Sw   | Lane 1 Rx Active | Lane 1 Rx Lock   |      Lane 1 Request   |
|      [15]        |      [14]       |      [13]        |      [12]        |          [11:8]       |
+--------------------------------------------------------------------------------------------------+
| Lane 0 New Req   | Lane 0 Clk Sw   | Lane 0 Rx Active | Lane 0 Rx Lock   |      Lane 0 Request   |
|       [7]        |       [6]       |       [5]        |       [4]        |           [3:0]       |
+--------------------------------------------------------------------------------------------------+
```

### PORT_CS_1

Most of the parameters are specified in the [`PORT_CS_1`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L374) register, and placing data into the Data registers. The information in the [`PORT_CS_1`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L374) includes:

1. `Address`: address of the SB register
2. `Length`: number of bytes to read or write
3. `Target`: accessing regsters on this Router (`0`), Link Partner (`1`), or retimer (`2`)
4. `Re-Timer Index`: when targeting a retimer, this contains the Retimer Index to the target retimer.

See *8.2.2.4 USB4 Port Capability* for more information.

```
$ tbdump --route 0 --adapter 1 --nregs 17 PORT_CS_1 -vv
0x009d 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_1
  [00:07]        0x0 Address
  [08:15]        0x0 Length
  [16:18]        0x0 Target
  [20:23]        0x0 Re-timer Index
  [24:24]        0x0 WnR
  [25:25]        0x0 No Response (NR)
  [26:26]        0x0 Result Code (RC)
  [31:31]        0x0 Pending (PND)
```

### PORT_CS_2 to PORT_CS_17

These are register for passing parameters and retrieve results between the CM and the SB Registers. There are 256 bytes.

```
0x009e 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_2
  [00:31]        0x0 Data[0]
0x009f 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_3
  [00:31]        0x0 Data[1]
0x00a0 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_4
  [00:31]        0x0 Data[2]
0x00a1 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_5
  [00:31]        0x0 Data[3]
0x00a2 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_6
  [00:31]        0x0 Data[4]
0x00a3 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_7
  [00:31]        0x0 Data[5]
0x00a4 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_8
  [00:31]        0x0 Data[6]
0x00a5 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_9
  [00:31]        0x0 Data[7]
0x00a6 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_10
  [00:31]        0x0 Data[8]
0x00a7 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_11
  [00:31]        0x0 Data[9]
0x00a8 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_12
  [00:31]        0x0 Data[10]
0x00a9 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_13
  [00:31]        0x0 Data[11]
0x00aa 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_14
  [00:31]        0x0 Data[12]
0x00ab 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_15
  [00:31]        0x0 Data[13]
0x00ac 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_16
  [00:31]        0x0 Data[14]
0x00ad 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_17
  [00:31]        0x0 Data[15]
```

## DETAILS

### Accessing the Sideband Registers

There's no Config Space for PHYs. Access to the sideband registers are done indirectly through the USB4 Port Capability structure in the Config Space of the the Lane 0 Adapter.

### Perform Port Operations on PHY

```
+-------------------+
|                   | 0x00
+-------------------+
|                   | 0x01
+-------------------+
|                   | 0x02-0x04
+-------------------+
|                   | 0x05
+-------------------+
|                   | 0x06
+-------------------+
|                   | 0x07
+-------------------+
|     Opcode        | 0x08
+-------------------+
|     Metadata      | 0x09
+-------------------+
|                   | 0x0A-0x0B
+-------------------+
|                   | 0x0C
+-------------------+
|                   | 0x0D
+-------------------+
|                   | 0x0E
+-------------------+
|                   | 0c0F
+-------------------+
|                   | 0x10-0x11
+-------------------+
|                   |
|        Data       | 0x12 (64 bytes)
|                   |
+-------------------+
```

Writing to the `Opcode`, `Metadata`, and `Data` SB Registers allows the CM to request operations on the PHY where those SB registers belong to. For example, re-enumerate the retimers, lane margining, fault injections etc.

The CM set the `Opcode`, `Metadata`, and `Data` and issue the transaction. The Router can report the result meta data and result data in the `Metadata` and the `Data` rspectively.

### 0x0C Link Configuration

This register is about link management. This shows variaous capabilies, for example, Lane Bonding support, Sideband channel support, TBT3 compatible (necessary for a USB4 dock).

Enabling Request Lane 0 and Enabling Request Lane 1 are part of the Link Training process.

### 0x0D TxFFE

This register is for the Feed Forward Equalization between the Link Partners during Link Training. This stores the equalization parameters.
