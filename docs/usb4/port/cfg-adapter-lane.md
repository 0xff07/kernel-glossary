---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Adapter Config Space (Lane)

## SPECIFICATIONS

- USB4 Specification, section 8.2.2.2: TMU Adapter Configuration Capability
- USB4 Specification, section 8.2.2.3: Lane Adapter Configuration Capability
- USB4 Specification, section 8.2.2.4: USB4 Port Capability

## LINUX KERNEL

- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): Lane adapter register defines
- [`drivers/thunderbolt/lc.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/lc.c): Link controller operations
- [`drivers/thunderbolt/usb4.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c): USB4 port operations
- [`'\<usb4_port_sb_read\>':'drivers/thunderbolt/usb4.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L1354): Read from sideband register via PORT_CS_1

### Lane Adapter CS Defines

From [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L339):

```c
#define LANE_ADP_CS_0                           0x00
#define LANE_ADP_CS_0_SUPPORTED_SPEED_MASK      GENMASK(19, 16)
#define LANE_ADP_CS_0_SUPPORTED_WIDTH_MASK      GENMASK(25, 20)
#define LANE_ADP_CS_0_SUPPORTED_WIDTH_DUAL      0x2
#define LANE_ADP_CS_0_CL0S_SUPPORT              BIT(26)
#define LANE_ADP_CS_0_CL1_SUPPORT               BIT(27)
#define LANE_ADP_CS_0_CL2_SUPPORT               BIT(28)
#define LANE_ADP_CS_1                           0x01
#define LANE_ADP_CS_1_TARGET_SPEED_MASK         GENMASK(3, 0)
#define LANE_ADP_CS_1_TARGET_SPEED_GEN3         0xc
#define LANE_ADP_CS_1_TARGET_WIDTH_MASK         GENMASK(5, 4)
#define LANE_ADP_CS_1_TARGET_WIDTH_SINGLE       0x1
#define LANE_ADP_CS_1_TARGET_WIDTH_DUAL         0x3
#define LANE_ADP_CS_1_CL0S_ENABLE               BIT(10)
#define LANE_ADP_CS_1_CL1_ENABLE                BIT(11)
#define LANE_ADP_CS_1_CL2_ENABLE                BIT(12)
#define LANE_ADP_CS_1_LD                        BIT(14)
#define LANE_ADP_CS_1_LB                        BIT(15)
#define LANE_ADP_CS_1_CURRENT_SPEED_MASK        GENMASK(19, 16)
#define LANE_ADP_CS_1_CURRENT_SPEED_GEN2        0x8
#define LANE_ADP_CS_1_CURRENT_SPEED_GEN3        0x4
#define LANE_ADP_CS_1_CURRENT_SPEED_GEN4        0x2
#define LANE_ADP_CS_1_CURRENT_WIDTH_MASK        GENMASK(25, 20)
#define LANE_ADP_CS_1_PMS                       BIT(30)
```

### USB4 Port CS Defines

From [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L374):

```c
#define PORT_CS_1                   0x01
#define PORT_CS_1_LENGTH_SHIFT      8
#define PORT_CS_1_TARGET_MASK       GENMASK(18, 16)
#define PORT_CS_1_TARGET_SHIFT      16
#define PORT_CS_1_RETIMER_INDEX_SHIFT 20
#define PORT_CS_1_WNR_WRITE         BIT(24)
#define PORT_CS_1_NR                BIT(25)
#define PORT_CS_1_RC                BIT(26)
#define PORT_CS_1_PND               BIT(31)
#define PORT_CS_18                  0x12
#define PORT_CS_18_BE               BIT(8)
#define PORT_CS_18_TCM              BIT(9)
#define PORT_CS_18_CPS              BIT(10)
#define PORT_CS_19                  0x13
#define PORT_CS_19_DPR              BIT(0)
#define PORT_CS_19_PC               BIT(3)
```

## OTHER SOURCES

- [USB4 Port Operations](https://community.cadence.com/cadence_blogs_8/b/fv/posts/usb4-port-operations)
- [[PATCH] thunderbolt: Fix PCIe device enumeration with delayed rescan](https://lore.kernel.org/all/20260121052744.233517-1-acelan.kao@canonical.com/)

## REGISTERS

Each Lane has its own adapter config space. Notably, the Lane 0 Adapter has some additional functionalities and has even more Capability Structures.

### TMU Adapter Capability Structure (Required)

```
0x000a 0x0000039c 0b00000000 00000000 00000011 10011100 .... TMU_ADP_CS_0
  [00:07]       0x9c Next Capability Pointer
  [08:15]        0x3 Capability ID
0x000b 0x00240000 0b00000000 00100100 00000000 00000000 .$.. TMU_ADP_CS_1
  [00:31]   0x240000 TxTimeToWire
0x000c 0x002e0000 0b00000000 00101110 00000000 00000000 .... TMU_ADP_CS_2
  [00:31]   0x2e0000 RxTimeToWire
0x000d 0x00000000 0b00000000 00000000 00000000 00000000 .... TMU_ADP_CS_3
  [29:29]        0x0 EnableUniDirectionalMode (UDM)
  [30:30]        0x0 Inter-Domain Time Responder (IDTR)
  [31:31]        0x0 Inter-Domain Time Initiator (IDTI)
0x000e 0x00000000 0b00000000 00000000 00000000 00000000 .... TMU_ADP_CS_4
  [00:15]        0x0 RX TSNOS Counter
  [16:31]        0x0 TX TSNOS Counter
0x000f 0x00000000 0b00000000 00000000 00000000 00000000 .... TMU_ADP_CS_5
  [00:15]        0x0 RX Packet Counter
  [16:31]        0x0 TX Packet Counter
0x0010 0x00100000 0b00000000 00010000 00000000 00000000 .... TMU_ADP_CS_6
  [01:01]        0x0 Disable Time Sync (DTS)
0x0011 0x00000000 0b00000000 00000000 00000000 00000000 .... TMU_ADP_CS_7
  [00:09]        0x0 Lost TSNOS Counter
  [10:19]        0x0 Lost Packet Counter
  [20:29]        0x0 Bad Packet Counter
```

As big as it is, only `UMD`, `IDTR`, `IDTI` are writable.

### Lane Adapter Capability Structure (Required)

#### LANE_ADP_CS_0

```
0x0036 0x1c3c013e 0b00011100 00111100 00000001 00111110 .<.> LANE_ADP_CS_0
  [00:07]       0x3e Next Capability Pointer
  [08:15]        0x1 Capability ID
  [16:19]        0xc Supported Link Speeds
  [20:21]        0x3 Supported Link Widths (SLW)
  [22:23]        0x0 Gen 4 Asymmetric Support (G4AS)
  [26:26]        0x1 CL0s Support
  [27:27]        0x1 CL1 Support
  [28:28]        0x1 CL2 Support
```

The registers in [`LANE_ADP_CS_0`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L339) are all read-only. They are fields describing the capabilities of this Lane.


#### LANE_ADP_CS_1

```
0x0037 0x5c18001c 0b01011100 00011000 00000000 00011100 \... LANE_ADP_CS_1
  [00:03]        0xc Target Link Speed → Router shall attempt Gen 3 speed
  [04:05]        0x1 Target Link Width → Establish two Single-Lane Links
  [06:07]        0x0 Target Asymmetric Link → Establish Symmetric Link
  [10:10]        0x0 CL0s Enable
  [11:11]        0x0 CL1 Enable
  [12:12]        0x0 CL2 Enable
  [14:14]        0x0 Lane Disable (LD)
  [15:15]        0x0 Lane Bonding (LB)
  [16:19]        0x8 Current Link Speed → Gen 2
  [20:25]        0x1 Negotiated Link Width → Single-Lane Link (x1)
  [26:29]        0x7 Adapter State → CLd
  [30:30]        0x1 PM Secondary (PMS)
```

This allows the CM enables said capabilities in [`LANE_ADP_CS_0`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L339). This also contains status bits the CM can retrieve.

For example, CM could enable lane bonding or disable a lane by those registers.

#### LANE_ADP_CS_2

```
0x0038 0x00000000 0b00000000 00000000 00000000 00000000 .... LANE_ADP_CS_2
  [00:06]        0x0 Logical Layer Errors
  [16:22]        0x0 Logical Layer Errors Enable
```

The last DW allows the Lane adapter to generater event according to bit masks set.

### USB4 Port Adapter Capability Structure (Lane 0 Required)

This structure also has a `Data[n]` region ([`PORT_CS_2`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L383) to `PORT_CS_17`) that the CM can use for extra operations. More importantly, this is also the way for the CM to access the Sideband Registers on the PHY on the port or retimers.

```
0x00ae 0x00000410 0b00000000 00000000 00000100 00010000 .... PORT_CS_18
  [00:07]       0x10 Cable USB4 Version
  [08:08]        0x0 Bonding Enabled (BE)
  [09:09]        0x0 TBT3-Compatible Mode (TCM)
  [10:10]        0x1 CLx Protocol Support (CPS)
  [11:11]        0x0 RS-FEC Enabled (Gen 2) (RE2)
  [12:12]        0x0 RS-FEC Enabled (Gen 3) (RE3)
  [13:13]        0x0 Router Detected (RD)
  [16:16]        0x0 Wake on Connect Status
  [17:17]        0x0 Wake on Disconnect Status
  [18:18]        0x0 Wake on USB4 Wake Status
  [19:19]        0x0 Wake on Inter-Domain Status
  [20:20]        0x0 Cable Gen 3 Support (CG3)
  [21:21]        0x0 Cable Gen 4 Support (CG4)
  [22:22]        0x0 Cable Asymmetric Support (CSA)
  [23:23]        0x0 Cable CLx Support (CSC)
  [24:24]        0x0 AsymmetricTransitionInProgress (TIP)
0x00af 0x00000006 0b00000000 00000000 00000000 00000110 .... PORT_CS_19
  [00:00]        0x0 Downstream Port Reset (DPR)
  [01:01]        0x1 Request RS-FEC Gen 2 (RS2)
  [02:02]        0x1 Request RS-FEC Gen 3 (RS3)
  [03:03]        0x0 USB4 Port is Configured (PC)
  [04:04]        0x0 USB4 Port is Inter-Domain (PID)
  [16:16]        0x0 Enable Wake on Connect
  [17:17]        0x0 Enable Wake on Disconnect
  [18:18]        0x0 Enable Wake on USB4 Wake
  [19:19]        0x0 Enable Wake on Inter-Domain
  [24:24]        0x0 StartAsymmetricFlow
  [30:30]        0x0 Initiate Gen 4 Link Recovery (ILR)
  [31:31]        0x0 Enable Gen 4 Link Recovery (ELR)
```

## DETAILS

### PORT_CS_0

This is the tandard capability header.

```
0x009c 0x00000636 0b00000000 00000000 00000110 00110110 ...6 PORT_CS_0
  [00:07]       0x36 Next Capability Pointer
  [08:15]        0x6 Capability ID
```

### PORT_CS_1 + PORT_CS_2-PORT_CS_17

```
0x009d 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_1
  [00:07]        0x0 Address
  [08:15]        0x0 Length
  [16:18]        0x0 Target
  [20:23]        0x0 Re-timer Index
  [24:24]        0x0 WnR
  [25:25]        0x0 No Response (NR)
  [26:26]        0x0 Result Code (RC)
  [31:31]        0x0 Pending (PND)
0x009e 0x00000000 0b00000000 00000000 00000000 00000000 .... PORT_CS_2 (to PORT_CS_17)
  [00:31]        0x0 Data[n]
```

These registers are the interface where the CM interact with the CM. The CM write the `Address`, `Length`, `Target` to interact with Sideband Registers on the PHY, either on the port itself, or on the retimers, and fill necessary parameters in the `Data[0]` to `Data[15]` registers, depending on the type of interaction.

### PORT_CS_18

```
0x00ae 0x00000410 0b00000000 00000000 00000100 00010000 .... PORT_CS_18
  [00:07]       0x10 Cable USB4 Version
  [08:08]        0x0 Bonding Enabled (BE)
  [09:09]        0x0 TBT3-Compatible Mode (TCM)
  [10:10]        0x1 CLx Protocol Support (CPS)
  [11:11]        0x0 RS-FEC Enabled (Gen 2) (RE2)
  [12:12]        0x0 RS-FEC Enabled (Gen 3) (RE3)
  [13:13]        0x0 Router Detected (RD)
  [16:16]        0x0 Wake on Connect Status
  [17:17]        0x0 Wake on Disconnect Status
  [18:18]        0x0 Wake on USB4 Wake Status
  [19:19]        0x0 Wake on Inter-Domain Status
  [20:20]        0x0 Cable Gen 3 Support (CG3)
  [21:21]        0x0 Cable Gen 4 Support (CG4)
  [22:22]        0x0 Cable Asymmetric Support (CSA)
  [23:23]        0x0 Cable CLx Support (CSC)
  [24:24]        0x0 AsymmetricTransitionInProgress (TIP)
```

Those fields are also read-only. They are even more status bits, for example if a Router is detected, if any wake event occurs etc.

### PORT_CS_19

This is the register to allow the CM to enable the functions inidicated by the [`PORT_CS_19`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L393)

```
0x00af 0x00000006 0b00000000 00000000 00000000 00000110 .... PORT_CS_19
  [00:00]        0x0 Downstream Port Reset (DPR)
  [01:01]        0x1 Request RS-FEC Gen 2 (RS2)
  [02:02]        0x1 Request RS-FEC Gen 3 (RS3)
  [03:03]        0x0 USB4 Port is Configured (PC)
  [04:04]        0x0 USB4 Port is Inter-Domain (PID)
  [16:16]        0x0 Enable Wake on Connect
  [17:17]        0x0 Enable Wake on Disconnect
  [18:18]        0x0 Enable Wake on USB4 Wake
  [19:19]        0x0 Enable Wake on Inter-Domain
  [24:24]        0x0 StartAsymmetricFlow
  [30:30]        0x0 Initiate Gen 4 Link Recovery (ILR)
  [31:31]        0x0 Enable Gen 4 Link Recovery (ELR)
```

The `DPR` can reset the downstream port.

## Appendix: ASCII Diagrams for structures

### TMU Adapter Capability Structure

```
+---------------------------------------------------------------------------------+
|               Reserved [31:16]       | Capability ID [15:08] | Next Ptr [07:00] |
+---------------------------------------------------------------------------------+
|                                TxTimeToWire [31:00]                             |
+---------------------------------------------------------------------------------+
|                                RxTimeToWire [31:00]                             |
+---------------------------------------------------------------------------------+
|IDTI [31] | IDTR [30] | UDM [29] |                     Reserved [28:00]          |
+---------------------------------------------------------------------------------+
|          TX TSNOS Counter [31:16]    |         RX TSNOS Counter [15:00]         |
+---------------------------------------------------------------------------------+
|         TX Packet Counter [31:16]    |         RX Packet Counter [15:00]        |
+---------------------------------------------------------------------------------+
|                           Reserved [31:02]                | DTS [01] | Rsvd[00] |
+---------------------------------------------------------------------------------+
|  Rsvd | Bad Packet Counter | Lost Packet Counter |      Lost TSNOS Counter      |
|[31:30]|      [29:20]       |       [19:10]       |             [9:0]            |
+---------------------------------------------------------------------------------+
```

### Lane Adapter Capability Structures

```
+-----------------------------------------------------------------------------------------------+
|Rsvd   | G4AS  | SLW   |CL2 | CL1|CL0s| Supported Link Speeds|  Capability ID  | Next Cap Ptr  |
|[31:29]|[23:22]|[21:20]|[28]|[27]|[26]|       [19:16]        |     [15:08]     |    [07:00]    |
+-----------------------------------------------------------------------------------------------+
|RS|PS|Adapter     |Negotiated Link|Current [20:|LB|LD|RS|CL2|CL1|CL0s| RS  | TAs |TWidth|TSpeed|
|  |  |State[29:26]|Width [25:20]  |LinkSpd  19]|  |  |  |   |   |    |[9:8]|[7:6]|[5:4] |[3:0] |
+-----------------------------------------------------------------------------------------------+
| Reserved [31:23] | Logical Err Enable [22:16] | Reserved [15:07] | Logical Errors  [06:00]    |
+-----------------------------------------------------------------------------------------------+
```

### Observed Lane Adapter Registers (dmesg)

The following dmesg excerpts come from a Dell laptop with an AMD USB4 host router (vendor 0x438, device 0x20e) running Linux 6.19.0-rc6 with `thunderbolt.dyndbg=+pt` enabled on the kernel command line. A Dell Thunderbolt 4 Dock (Intel JHL8540, vendor 0x8087, device 0xb26, 19 ports) is connected to domain 1 at route 0x2 (depth 1, through lane adapter port 2). Two OWC Envoy Express Thunderbolt 3 NVMe enclosures (Intel JHL6540, 8086:15c0) are connected downstream of the dock at routes 0x502 (through dock port 5) and 0x702 (through dock port 7). The packet traces shown here are from the hot-plug event at t=28s when the second OWC enclosure connects through dock port 7. These traces show the actual LANE_ADP_CS_0 and LANE_ADP_CS_1 register values as read from the wire via USB4 control packets.

```
    Domain 1 (NHI 0000:c7:00.6)

    Host Router (AMD 438:20e, route 0, depth 0, max port number 7)
      Ports: 1 NHI, 2-3 Lane, 4 USB3 Down, 5 PCIe Down, 6-7 DP IN
        │
        │ port 2+3 Link (dual-lane bond, 40 Gb/s)
        │
    Dell TB4 Dock (Intel 8087:b26, route 0x2, depth 1, max port number 19)
      Ports: 1-8 Lane, 9 PCIe Up, 10-12 PCIe Down,
             13-14 DP OUT, 15 Inactive, 16 USB3 Up, 17-19 USB3 Down
        │
        ├── port 5+6 Link ── OWC Envoy Express (8086:15c0, route 0x502, depth 2)
        │                     NVMe enclosure (TB3, max port number 5)
        │
        └── port 7+8 Link ── OWC Envoy Express (8086:15c0, route 0x702, depth 2)
                              NVMe enclosure (TB3, max port number 5)
```

#### LANE_ADP_CS_0 and LANE_ADP_CS_1 Read (Dock Port 7)

When the CM detects a hot-plug event on dock adapter 7, it reads the Lane Adapter Configuration Space to determine the link state. This read targets LANE_ADP_CS_0 (at capability offset 0x36) and LANE_ADP_CS_1 (offset 0x37):

```
[   28.067442] tb_tx Read Request Domain 1 Route 2 Adapter 7 / Lane
               0x02/---- 0x02384036 .8@6
                 [00:12]       0x36 Address
                 [13:18]        0x2 Read Size
                 [19:24]        0x7 Adapter Num
                 [25:26]        0x1 Configuration Space (CS) → Adapter Configuration Space
[   28.067478] tb_rx Read Response Domain 1 Route 2 Adapter 7 / Lane
               0x03/0036 0x003c013e .<.> LANE_ADP_CS_0
                 [00:07]       0x3e Next Capability Pointer
                 [08:15]        0x1 Capability ID
                 [16:19]        0xc Supported Link Speeds
                 [20:21]        0x3 Supported Link Widths (SLW)
                 [22:23]        0x0 Gen 4 Asymmetric Support (G4AS)
                 [26:26]        0x0 CL0s Support
                 [27:27]        0x0 CL1 Support
                 [28:28]        0x0 CL2 Support
               0x04/0037 0x4814001c H... LANE_ADP_CS_1
                 [00:03]        0xc Target Link Speed → Router shall attempt Gen 3 speed
                 [04:05]        0x1 Target Link Width → Establish two Single-Lane Links
                 [06:07]        0x0 Target Asymmetric Link → Establish Symmetric Link
                 [10:10]        0x0 CL0s Enable
                 [11:11]        0x0 CL1 Enable
                 [12:12]        0x0 CL2 Enable
                 [14:14]        0x0 Lane Disable (LD)
                 [15:15]        0x0 Lane Bonding (LB)
                 [16:19]        0x4 Current Link Speed → Gen 3
                 [20:25]        0x1 Negotiated Link Width → Single-Lane Link (x1)
                 [26:29]        0x2 Adapter State → CL0
                 [30:30]        0x1 PM Secondary (PMS)
```

LANE_ADP_CS_0 (Capability ID = 1 confirms this is the Lane Adapter Capability) reports: Supported Link Speeds = 0xc (Gen 2 and Gen 3 are supported), Supported Link Widths = 0x3 (both single-lane and dual-lane), and no CLx support (CL0s/CL1/CL2 all 0). The lack of CLx support on dock port 7 is notable because the dock's upstream port 1 does support CLx, showing that CLx capability is per-port. LANE_ADP_CS_1 shows the operational state: Target Link Speed is Gen 3, Target Link Width is single-lane (two separate links, not bonded), Current Link Speed is Gen 3 (20 Gb/s), Negotiated Width is x1 (single lane), and Adapter State = CL0 (the link is active and up). PMS = 1 (PM Secondary) means this is the secondary lane adapter in a dual-link pair.

#### Adapter State Values

The Adapter State field [29:26] in LANE_ADP_CS_1 indicates the current link state. The dmesg logs translate these numeric values:

```
[   28.067506] [393] thunderbolt 0000:c7:00.6: 2:7: is connected, link is up (state: 2)
```

State 2 corresponds to CL0 (active, link up). The driver reads PORT_CS_18[3:0] to obtain this value. Other state values observed in the dmesg include state 7 (CLd, unplugged):

```
[    5.392845] [12] thunderbolt 0000:c7:00.6: 2:3: is unplugged (state: 7)
[    6.161952] [12] thunderbolt 0000:c7:00.6: 2:7: is unplugged (state: 7)
```

State 7 (CLd) means the lane adapter has detected a disconnect and the link is down. The CM uses this to skip scanning ports that have nothing connected.

#### ADP_CS_4 Read (Buffer Credits and Plugged Status)

The CM also reads ADP_CS_4 to check buffer credits, the Plugged bit, and the Lock bit:

```
[   28.067603] tb_rx Read Response Domain 1 Route 2 Adapter 7 / Lane
               0x03/0004 0xc3c00000 .... ADP_CS_4
                 [00:09]        0x0 Non-Flow Controlled Buffers
                 [20:29]       0x3c Total Buffers
                 [30:30]        0x1 Plugged
                 [31:31]        0x1 Lock (LCK)
```

ADP_CS_4 at offset 0x4 of the Adapter Configuration Space reports: Total Buffers = 0x3c (60 credits), which matches the Intel JHL8540 dock's per-port credit allocation. Plugged = 1 confirms a device is physically connected. Lock = 1 means the adapter is in use and its configuration is locked. The CM subsequently writes ADP_CS_4 with Lock = 0 to unlock the adapter before reconfiguring it.

#### Lane Bonding Transitions (Credit Redistribution)

When the CM enables lane bonding, it redistributes credits between the dual-link pair. This is visible in the kernel logs for the dock's downstream ports 7 and 8 (connecting to the second OWC NVMe enclosure at route 702):

```
[   28.421872] [393] thunderbolt 0000:c7:00.6: 2:7: total credits changed 60 -> 120
[   28.422002] [393] thunderbolt 0000:c7:00.6: 2:8: total credits changed 60 -> 0
[   28.422132] [393] thunderbolt 0000:c7:00.6: 702:1: total credits changed 60 -> 120
[   28.422266] [393] thunderbolt 0000:c7:00.6: 702:2: total credits changed 60 -> 0
[   28.422537] [393] thunderbolt 0000:c7:00.6: 702: link width set to symmetric, dual lanes
```

Port 7 (Lane 0, the primary lane adapter) absorbs all credits from port 8 (Lane 1, the secondary): port 7 goes from 60 to 120 credits, port 8 goes from 60 to 0. The same redistribution happens on the device side (route 702): port 1 gets 120 credits, port 2 gets 0. After credit redistribution, tb_switch_set_link_width() enables dual-lane bonding by setting the LB (Lane Bonding) bit in LANE_ADP_CS_1 on both sides. This doubles the effective bandwidth from 20 Gb/s to 40 Gb/s.

#### USB4 Port Capability (PORT_CS_18/19) Read

The CM reads PORT_CS_18 and PORT_CS_19 from the USB4 Port Capability to check CLx protocol support, cable capabilities, and bonding status:

```
[   28.418732] tb_rx Read Response Domain 1 Route 702 Adapter 1 / Lane
               0x03/0101 0x01001401 .... CAP_STRUCT
                 [00:03]        0x1 USB4 Ports
                 [08:15]       0x14 Common Region Length
                 [16:27]      0x100 USB4 Port Region Length
```

The USB4 Port Capability structure header at Router CS offset 0x101 shows 1 USB4 port, a Common Region of 0x14 (20 DWORDs), and a USB4 Port Region of 0x100 (256 DWORDs). This capability provides the PORT_CS_18 and PORT_CS_19 registers used for CLx control, wake events, and sideband register access.
