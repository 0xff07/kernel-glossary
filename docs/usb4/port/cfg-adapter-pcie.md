---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Adapter Config Space (PCIe)

## SPECIFICATIONS

- USB4 Specification, section 8.2.2.7: PCIe Adapter Configuration Capability

## LINUX KERNEL

- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): PCIe adapter register defines
- [`drivers/thunderbolt/tunnel.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tunnel.c): PCIe tunnel creation and activation

### PCIe Adapter CS Defines

From [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L475):

```c
#define ADP_PCIE_CS_0               0x00
#define ADP_PCIE_CS_0_PE            BIT(31)  /* Path Enable */
#define ADP_PCIE_CS_1               0x01
#define ADP_PCIE_CS_1_EE            BIT(0)   /* Extended Encapsulation */
```

## REGISTERS

### PCIe Adapter Capability Structure

```
0x0039 0x2186043a 0b00100001 10000110 00000100 00111010 !..: ADP_PCIE_CS_0
  [00:07]       0x3a Next Capability Pointer
  [08:15]        0x4 Capability ID
  [16:16]        0x0 Link
  [17:17]        0x1 TX EI
  [18:18]        0x1 RX EI
  [19:19]        0x0 RST
  [25:28]        0x0 LTSSM → Detect state
  [31:31]        0x0 Path Enable (PE)
```
