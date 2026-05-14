---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Adapter Config Space (DP)

## SPECIFICATIONS

- USB4 Specification, section 8.2.2.6: DP Adapter Configuration Capabilities

## LINUX KERNEL

- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): DP adapter register defines
- [`drivers/thunderbolt/tunnel.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tunnel.c): DP tunnel creation and bandwidth allocation

### DP Adapter CS Defines

From [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L403):

```c
#define ADP_DP_CS_0                         0x00
#define ADP_DP_CS_0_VIDEO_HOPID_MASK        GENMASK(26, 16)
#define ADP_DP_CS_0_VIDEO_HOPID_SHIFT       16
#define ADP_DP_CS_0_AE                      BIT(30)  /* AUX Enable */
#define ADP_DP_CS_0_VE                      BIT(31)  /* Video Enable */
#define ADP_DP_CS_1_AUX_TX_HOPID_MASK       GENMASK(10, 0)
#define ADP_DP_CS_1_AUX_RX_HOPID_MASK       GENMASK(21, 11)
#define ADP_DP_CS_1_AUX_RX_HOPID_SHIFT      11
#define ADP_DP_CS_2                         0x02
#define ADP_DP_CS_2_NRD_MLC_MASK            GENMASK(2, 0)
#define ADP_DP_CS_2_HPD                     BIT(6)
#define ADP_DP_CS_2_NRD_MLR_MASK            GENMASK(9, 7)
#define ADP_DP_CS_2_CA                      BIT(10)  /* CM Ack */
#define ADP_DP_CS_2_GR_MASK                 GENMASK(12, 11) /* Granularity */
#define ADP_DP_CS_2_CMMS                    BIT(20)
#define ADP_DP_CS_2_ESTIMATED_BW_MASK       GENMASK(31, 24)
#define ADP_DP_CS_3                         0x03
#define ADP_DP_CS_3_HPDC                    BIT(9)   /* HPD Output Clear */
#define ADP_DP_CS_8                         0x08
#define ADP_DP_CS_8_DPME                    BIT(30)
#define ADP_DP_CS_8_DR                      BIT(31)
```

## REGISTERS

### DP IN Adapter Capability Structure

```
0x0039 0x0009048f 0b00000000 00001001 00000100 10001111 .... ADP_DP_CS_0
  [00:07]       0x8f Next Capability Pointer
  [08:15]        0x4 Capability ID
  [16:22]        0x9 Video HopID
  [30:30]        0x0 AUX Enable (AE)
  [31:31]        0x0 Video Enable (VE)
0x003a 0x00004008 0b00000000 00000000 01000000 00001000 ..@. ADP_DP_CS_1
  [00:06]        0x8 AUX Tx HopID
  [11:17]        0x8 AUX Rx HopID
0x003b 0x00000000 0b00000000 00000000 00000000 00000000 .... ADP_DP_CS_2
  [00:02]        0x0 NRD Max Lane Count (NRD MLC) → 1 Lane
  [03:03]        0x0 SW Link Init (SWLI)
  [06:06]        0x0 HPD Status
  [07:09]        0x0 NRD Max Link Rate (NRD MLR) → 1.62 Gbps/lane
  [10:10]        0x0 CM Ack (CA)
  [11:12]        0x0 Granularity (GR) → 0.25
  [13:15]        0x0 Group_ID
  [16:19]        0x0 CM_ID
  [20:20]        0x0 CM BW Allocation Mode Support (CMMS)
  [24:31]        0x0 Estimated BW
0x003c 0x01010005 0b00000001 00000001 00000000 00000101 .... ADP_DP_CS_3
  [09:09]        0x0 HPD Output Clear (HPDC)
  [10:10]        0x0 HPD Output Set (HPDS)
0x003d 0x15c0a334 0b00010101 11000000 10100011 00110100 ...4 DP_LOCAL_CAP
  [00:03]        0x4 Protocol Adapter Version → Version 1.0
  [04:07]        0x3 Maximal DPCD Rev → DPCD r1.4a
  [08:11]        0x3 8b10b Maximal Link Rate → 8.1 Gbps/lane
  [12:14]        0x2 Maximal Lane Count → 4 Lanes
  [15:15]        0x1 8b10b MST Capability
  [16:16]        0x0 Panel Replay Tunneling Optimization Support
  [17:17]        0x0 128b/132b Link Layer & 10Gbps/Lane Support
  [18:18]        0x0 20Gbps/Lane Support
  [19:19]        0x0 13.5Gbps/Lane Support
  [20:20]        0x0 ALPM Support
  [22:22]        0x1 8b10b TPS3 Capability
  [24:24]        0x1 8b10b TPS4 Capability
  [25:25]        0x0 8b10b FEC Not Supported
  [26:26]        0x1 Secondary Split Capability
  [27:27]        0x0 LTTPR Not Supported
  [28:28]        0x1 DP IN BW Allocation Mode Support
  [29:29]        0x0 DSC Not Supported
0x003e 0x00000000 0b00000000 00000000 00000000 00000000 .... DP_REMOTE_CAP
  [00:03]        0x0 Protocol Adapter Version
  [04:07]        0x0 Maximal DPCD Rev → DPCD r1.1
  [08:11]        0x0 8b10b Maximal Link Rate → 1.62 Gbps/lane
  [12:14]        0x0 Maximal Lane Count → 1 Lane
  [15:15]        0x0 8b10b MST Capability
  [16:16]        0x0 Panel Replay Tunneling Optimization Support
  [17:17]        0x0 128b/132b Link Layer & 10Gbps/Lane Support
  [18:18]        0x0 20Gbps/Lane Support
  [19:19]        0x0 13.5Gbps/Lane Support
  [20:20]        0x0 ALPM Support
  [22:22]        0x0 8b10b TPS3 Capability
  [24:24]        0x0 8b10b TPS4 Capability
  [25:25]        0x0 8b10b FEC Not Supported
  [26:26]        0x0 Secondary Split Capability
  [27:27]        0x0 LTTPR Not Supported
  [29:29]        0x0 DSC Not Supported
0x003f 0x00000000 0b00000000 00000000 00000000 00000000 .... DP_STATUS
  [00:02]        0x0 Lane Count
  [08:11]        0x0 Link Rate → 1.62 Gbps/lane
  [24:31]        0x0 Allocated BW
0x0040 0x00000000 0b00000000 00000000 00000000 00000000 .... DP_COMMON_CAP
  [00:03]        0x0 Protocol Adapter Version
  [04:07]        0x0 Maximal DPCD Rev → DPCD r1.1
  [08:11]        0x0 8b10b Maximal Link Rate → 1.62 Gbps/lane
  [12:14]        0x0 Maximal Lane Count → 1 Lane
  [15:15]        0x0 8b10b MST Capability
  [16:16]        0x0 Panel Replay Tunneling Optimization Support
  [17:17]        0x0 128b/132b Link Layer & 10Gbps/Lane Support
  [18:18]        0x0 20Gbps/Lane Support
  [19:19]        0x0 13.5Gbps/Lane Support
  [20:20]        0x0 ALPM Support
  [22:22]        0x0 8b10b TPS3 Capability
  [24:24]        0x0 8b10b TPS4 Capability
  [25:25]        0x0 8b10b FEC Not Supported
  [26:26]        0x0 Secondary Split Capability
  [27:27]        0x0 LTTPR Not Supported
  [29:29]        0x0 DSC Not Supported
  [31:31]        0x0 DPRX Capabilities Read Done
```

## DETAILS

### DP OUT Adapter Capability Structure

The only differences are:

1. DW2: there's another field called "Maximum Accumulation Cycles" caved out from the the reserved field.
2. DW3: the DPDS and the HPDC fields are no longer there.
