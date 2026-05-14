---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# TLP Overview

## SPECIFICATIONS

- USB4 Specification, section 5.1: Transport Layer Packets
- USB4 Specification, section 5.1.3: Transport Layer Packet Types

## LINUX KERNEL

- [`drivers/thunderbolt/tb_msgs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_msgs.h): Packet header structures
- [`include/linux/thunderbolt.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h): Packet type enum
- [`'\<tb_cfg_pkg_type\>':'include/linux/thunderbolt.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L30): Control packet PDF values
- [`'\<tb_cfg_header\>':'drivers/thunderbolt/tb_msgs.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_msgs.h#L43): Packet header with route string

### [`enum tb_cfg_pkg_type`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L30)

[`enum tb_cfg_pkg_type`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/thunderbolt.h#L30) defines the PDF (Protocol Defined Field) values for control packets (HopID = 0):

```c
enum tb_cfg_pkg_type {
	TB_CFG_PKG_READ = 1,
	TB_CFG_PKG_WRITE = 2,
	TB_CFG_PKG_ERROR = 3,
	TB_CFG_PKG_NOTIFY_ACK = 4,
	TB_CFG_PKG_EVENT = 5,
	TB_CFG_PKG_XDOMAIN_REQ = 6,
	TB_CFG_PKG_XDOMAIN_RESP = 7,
	TB_CFG_PKG_OVERRIDE = 8,
	TB_CFG_PKG_RESET = 9,
	TB_CFG_PKG_ICM_EVENT = 10,
	TB_CFG_PKG_ICM_CMD = 11,
	TB_CFG_PKG_ICM_RESP = 12,
};
```

## DETAILS

### TLP Header

All the TLPs have a 1DW header:

```
Bit  31          28 27       27 26     23 22               16 15            8 7            0
     |-------------|-----------|---------|-------------------|----------------|------------|
DW0  |     PDF     |  SuppID*  |  Rsvd   |       HopID       |     Length     |     HEC    |
     |   (31:28)   |   (27)    |(26:23)  |     (22:16)       |    (15:8)      |    (7:0)   |
     |-------------|-----------|---------|-------------------|----------------|------------|
DW1  |                               First payload DW (31:0)                               |
     |-------------------------------------------------------------------------------------|
DW2  |                                          .                                          |
     |                                          .                                          |
     |                                          .                                          |
     |-------------------------------------------------------------------------------------|
DWn  |                               Last payload DW (31:0)                                |
     |-------------------------------------------------------------------------------------|

* SuppID = Supplemental ID
```

### Payload and Length

The Length is the number of bytes in the payload (and only the payload). This does NOT include the size of the header.

Note that the USB4 requires packets to be DW-aligned. However, even though the total packet has to be DW-aligned, the Length always reflect the actual size of the payload data. The underlying transmitting mechanism automatically pads the packet into DW-aligned size. Even so, the Length doesn't change during this padding process. It always reflects the size of the actual payload.

### HEC: header error correction

This is the error correction information for the header, and only the header itself. The underlying protocol packets are protected by error correction mechanism in that protocol, so no need to protect them.

This provides single bit correction for the header, and is capable of detecting multi-bit errors.

### HopID: routing information and TLP type

The HopID serves 2 purposes. First, it determines the routing information of that packet. But more importantly, because of the way HopIDs are reserved for different traffic, the HopID also implicitly determines the type of TLP.

For example, because the HopID 0 is always reserved for the control traffic, a packet with HopID 0 must be a Control packet. Simliar rules apply to the Link Management packet (HopID 1 to 7) and Protocol packets (HopID above 8).

### PDF: TLP subtype

The PDF value further differentiate the packets under the same traffic type. For example, there are different sub-types of the Control packets. They each has a different PDF value.

The interpretation of this field is determined by the TLP type. This is one of the reason it is called the "Protocol-defined" field, in the sense that its meaning depends on the protocol this packet is carrying.

### Supplemental ID

This is the flag identifying that it packet type is not what its HopID suggests, but rather it's a flow control packet regarding the Path that HopID is referring to.
