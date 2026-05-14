---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Adapter Config Space (USB3)

## SPECIFICATIONS

- USB4 Specification, section 8.2.2.8: USB3 Adapter Configuration Capability

## LINUX KERNEL

- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): USB3 adapter register defines
- [`drivers/thunderbolt/tunnel.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tunnel.c): USB3 tunnel creation and bandwidth management

### USB3 Adapter CS Defines

From [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L481):

```c
#define ADP_USB3_CS_0                   0x00
#define ADP_USB3_CS_0_V                 BIT(30)  /* Valid */
#define ADP_USB3_CS_0_PE                BIT(31)  /* Path Enable */
#define ADP_USB3_CS_1                   0x01
#define ADP_USB3_CS_1_CUBW_MASK         GENMASK(11, 0)  /* Consumed Upstream BW */
#define ADP_USB3_CS_1_CDBW_MASK         GENMASK(23, 12) /* Consumed Downstream BW */
#define ADP_USB3_CS_1_CDBW_SHIFT        12
#define ADP_USB3_CS_1_HCA               BIT(31)  /* Host Controller Ack */
#define ADP_USB3_CS_2                   0x02
#define ADP_USB3_CS_2_AUBW_MASK         GENMASK(11, 0)  /* Allocated Upstream BW */
#define ADP_USB3_CS_2_ADBW_MASK         GENMASK(23, 12) /* Allocated Downstream BW */
#define ADP_USB3_CS_2_ADBW_SHIFT        12
#define ADP_USB3_CS_2_CMR               BIT(31)  /* CM Request */
#define ADP_USB3_CS_3                   0x03
#define ADP_USB3_CS_3_SCALE_MASK        GENMASK(5, 0)
#define ADP_USB3_CS_4                   0x04
#define ADP_USB3_CS_4_MSLR_MASK         GENMASK(18, 12) /* Max Supported Link Rate */
#define ADP_USB3_CS_4_MSLR_SHIFT        12
#define ADP_USB3_CS_4_MSLR_20G          0x1
```

## REGISTERS

### USB3 Adapter Capability Structure

```
0x000a 0x00000410 0b00000000 00000000 00000100 00010000 .... ADP_USB3_GX_CS_0
  [00:07]       0x10 Next Capability Pointer
  [08:15]        0x4 Capability ID
  [30:30]        0x0 Valid (V)
  [31:31]        0x0 Path Enable (PE)
0x000b 0x00000000 0b00000000 00000000 00000000 00000000 .... ADP_USB3_GX_CS_1
  [00:11]        0x0 Consumed Upstream Bandwidth
  [12:23]        0x0 Consumed Downstream Bandwidth
  [31:31]        0x0 Host Controller Ack (HCA)
0x000c 0x00000000 0b00000000 00000000 00000000 00000000 .... ADP_USB3_GX_CS_2
  [00:11]        0x0 Allocated Upstream Bandwidth
  [12:23]        0x0 Allocated Downstream Bandwidth
  [31:31]        0x0 Connection Manager Request (CMR)
0x000d 0x00000000 0b00000000 00000000 00000000 00000000 .... ADP_USB3_GX_CS_3
  [00:05]        0x0 Scale
0x000e 0x00001500 0b00000000 00000000 00010101 00000000 .... ADP_USB3_GX_CS_4
  [00:06]        0x0 Actual Link Rate
  [07:07]        0x0 USB3 Link Valid (ULV)
  [08:11]        0x5 Port Link State (PLS) → RxDetect state
  [12:18]        0x1 Maximum Supported Link Rate
```
