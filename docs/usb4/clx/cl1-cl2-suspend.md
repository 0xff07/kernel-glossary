---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# CL1/CL2 Entry

## LINUX KERNEL

- [`drivers/thunderbolt/clx.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/clx.c): CLx enable/disable logic
- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): Lane adapter CLx register defines
- [`'\<tb_switch_clx_enable\>':'drivers/thunderbolt/clx.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/clx.c#L321): Enables CLx on upstream port of a router

### [`tb_switch_clx_enable()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/clx.c#L321)

[`tb_switch_clx_enable()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/clx.c#L321) enables CL1 and/or CL2 on both sides of the upstream link. It checks that both the upstream and downstream ports support the requested CLx states by reading [`LANE_ADP_CS_0`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L339), then writes the enable bits in [`LANE_ADP_CS_1`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L348):

```c
int tb_switch_clx_enable(struct tb_switch *sw, unsigned int clx)
{
	/* ... validation ... */
	up = tb_upstream_port(sw);
	down = tb_switch_downstream_port(sw);

	up_clx_support = tb_port_clx_supported(up, clx);
	down_clx_support = tb_port_clx_supported(down, clx);

	if (!up_clx_support || !down_clx_support)
		return -EOPNOTSUPP;

	ret = tb_port_clx_enable(up, clx);
	if (ret)
		return ret;

	ret = tb_port_clx_enable(down, clx);
	if (ret) {
		tb_port_clx_disable(up, clx);
		return ret;
	}

	sw->clx |= clx;
	return 0;
}
```

### CLx Lane Adapter Register Defines

From [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L339):

```c
/* LANE_ADP_CS_0: capabilities (read-only) */
#define LANE_ADP_CS_0_CL0S_SUPPORT   BIT(26)
#define LANE_ADP_CS_0_CL1_SUPPORT    BIT(27)
#define LANE_ADP_CS_0_CL2_SUPPORT    BIT(28)

/* LANE_ADP_CS_1: enables and status */
#define LANE_ADP_CS_1_CL0S_ENABLE    BIT(10)
#define LANE_ADP_CS_1_CL1_ENABLE     BIT(11)
#define LANE_ADP_CS_1_CL2_ENABLE     BIT(12)
```

## OTHER SOURCES

- [[PATCH] thunderbolt: Fix PCIe device enumeration with delayed rescan](https://lore.kernel.org/all/20260121052744.233517-1-acelan.kao@canonical.com/)

## REGISTERS

### In [`LANE_ADP_CS_0`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L339)

This shows if a Lane is capable of `CL1` or `CL2`

```diff
0x0036 0x1c3c013e 0b00011100 00111100 00000001 00111110 .<.> LANE_ADP_CS_0
  [00:07]       0x3e Next Capability Pointer
  [08:15]        0x1 Capability ID
  [16:19]        0xc Supported Link Speeds
  [20:21]        0x3 Supported Link Widths (SLW)
  [22:23]        0x0 Gen 4 Asymmetric Support (G4AS)
  [26:26]        0x1 CL0s Support
+ [27:27]        0x1 CL1 Support
+ [28:28]        0x1 CL2 Support
```

### [`LANE_ADP_CS_1`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L348)

This shows if the `CL1` and `CL2` suspend are activated.

```
0x0037 0x5c18001c 0b01011100 00011000 00000000 00011100 \... LANE_ADP_CS_1
  [00:03]        0xc Target Link Speed → Router shall attempt Gen 3 speed
  [04:05]        0x1 Target Link Width → Establish two Single-Lane Links
  [06:07]        0x0 Target Asymmetric Link → Establish Symmetric Link
  [10:10]        0x0 CL0s Enable
+ [11:11]        0x0 CL1 Enable
+ [12:12]        0x0 CL2 Enable
  [14:14]        0x0 Lane Disable (LD)
  [15:15]        0x0 Lane Bonding (LB)
  [16:19]        0x8 Current Link Speed → Gen 2
  [20:25]        0x1 Negotiated Link Width → Single-Lane Link (x1)
+ [26:29]        0x7 Adapter State → CLd
  [30:30]        0x1 PM Secondary (PMS)
```

## DETAILS

### CL1/CL2 Entry Flow

```
┌───────────────────────────────┐                               ┌───────────────────────────────┐
│           DFP Router          │                               │           UFP Router          │
│        (Host Side)            │                               │     (Device / Peripheral)     │
├───────────────────────────────┤                               ├───────────────────────────────┤
│  High-Speed Lanes             │                               │  High-Speed Lanes             │
│     Lane0_TX± ────────────────┼─ CL_OFF[3] ────── CLx_REQ[1]─▶│     Lane0_TX±                 │
│     Lane0_RX± ◀───────────────┼─────── CLy_ACK/CL_NAK[2]──────│───  Lane0_RX±                 │
│                               │                               │                               │
│     Lane1_TX± ────────────────┼─ CL_OFF[3] ────── CLx_REQ[1]─▶│     Lane1_TX±                 │
│     Lane1_RX± ◀───────────────┼─────── CLy_ACK/CL_NAK[2]──────│───  Lane1_RX±                 │
│                               │                               │                               │
│  Sideband Signals             │                               │  Sideband Signals             │
│     SBTX  ────────────────────┼──────────────────────────────▶│     SBTX                      │
│     SBRX  ◀───────────────────┼───────────────────────────────│───  SBRX                      │
│      │                        │                               │      │                        │
│      ▼                        │                               │      ▼                        │
│  ┌─────────────────────────┐  │                               │  ┌─────────────────────────┐  │
│  │ Sideband Register Space │  │                               │  │ Sideband Register Space │  │
│  │                         │  │                               │  │                         │  │
│  │                         │  │                               │  │                         │  │
│  └─────────────────────────┘  │                               │  └─────────────────────────┘  │
│                               │                               │                               │
│  CC / Power                   │                               │  CC / Power                   │
│     CC1/CC2 ──────────────────┼──────────────────────────────▶│     CC1/CC2                   │
│     VBUS/VCONN/GND ───────────┼──────────────────────────────▶│     VBUS/VCONN/GND            │
└───────────────────────────────┘                               └───────────────────────────────┘
```

### Initiate: `CLx_REQ`

Router A initiate entering of CL1 or CL2 by sending `CL1_REQ` or `CL2_REQ`

```
┌───────────────────────────────┐                               ┌───────────────────────────────┐
│  High-Speed Lanes             │                               │  High-Speed Lanes             │
│     Lane0_TX± ────────────────┼────────────────── CLx_REQ ───▶│     Lane0_TX±                 │
│     Lane0_RX± ◀───────────────┼───────────────────────────────│───  Lane0_RX±                 │
│                               │                               │                               │
│     Lane1_TX± ────────────────┼────────────────── CLx_REQ ───▶│     Lane1_TX±                 │
│     Lane1_RX± ◀───────────────┼───────────────────────────────│───  Lane1_RX±                 │
└───────────────────────────────┘                               └───────────────────────────────┘
```

### Case 1: Reject

UFP can simply reject entering of lower state at all, by sending `CL_NACK`.

```
┌───────────────────────────────┐                               ┌───────────────────────────────┐
│  High-Speed Lanes             │                               │  High-Speed Lanes             │
│     Lane0_TX± ────────────────┼──────────────────────────────▶│     Lane0_TX±                 │
│     Lane0_RX± ◀───────────────┼───────────────────CL_NAK[2.2]─│───  Lane0_RX±                 │
│                               │                               │                               │
│     Lane1_TX± ────────────────┼──────────────────────────────▶│     Lane1_TX±                 │
│     Lane1_RX± ◀───────────────┼───────────────────CL_NAK[2.2]─│───  Lane1_RX±                 │
│                               │                               │                               │
└───────────────────────────────┘                               └───────────────────────────────┘
```

### Case 2: UFP accepts as-is

The UFP can accept it as is, by sending `CLy_ACK` ordered set, where `y` is equal to `x`, the low power state that the DFP just proposed.

```
┌───────────────────────────────┐                               ┌───────────────────────────────┐
│  High-Speed Lanes             │                               │  High-Speed Lanes             │
│     Lane0_TX± ────────────────┼──────────────────────────────▶│     Lane0_TX±                 │
│     Lane0_RX± ◀───────────────┼── CLy_ACK ────────────────────│───  Lane0_RX±                 │
│                               │                               │                               │
│     Lane1_TX± ────────────────┼──────────────────────────────▶│     Lane1_TX±                 │
│     Lane1_RX± ◀───────────────┼─ CLy_ACK ─────────────────────│───  Lane1_RX±                 │
│                               │                               │                               │
└───────────────────────────────┘                               └───────────────────────────────┘
```

### Case 3: UFP proposes a shallower sate


Alternatively, UFP acn a a shallower power state instead, by sending ordered set `CLy_ACK`, where `y < x`. Note that if the UFP support `CL0s`, it could also send `CL0s_ACK` and enters `CL0s` instead

```
┌───────────────────────────────┐                               ┌───────────────────────────────┐
│  High-Speed Lanes             │                               │  High-Speed Lanes             │
│     Lane0_TX± ────────────────┼──────────────────────────────▶│     Lane0_TX±                 │
│     Lane0_RX± ◀───────────────┼──────────────────── CLy_ACK ──│───  Lane0_RX±                 │
│                               │                               │                               │
│     Lane1_TX± ────────────────┼──────────────────────────────▶│     Lane1_TX±                 │
│     Lane1_RX± ◀───────────────┼─────────────────── CLy_ACK ───│───  Lane1_RX±                 │
│                               │                               │                               │
└───────────────────────────────┘                               └───────────────────────────────┘
```

### Final Step: `CL_OFF`

If this is not rejected, the final step of entering the low power state is to send `CL_OFF` ordered sets:

```
┌───────────────────────────────┐                               ┌───────────────────────────────┐
│  High-Speed Lanes             │                               │  High-Speed Lanes             │
│     Lane0_TX± ────────────────┼─ CL_OFF[3] ──────────────────▶│     Lane0_TX±                 │
│     Lane0_RX± ◀───────────────┼───────────────────────────────│───  Lane0_RX±                 │
│                               │                               │                               │
│     Lane1_TX± ────────────────┼─ CL_OFF[3] ──────────────────▶│     Lane1_TX±                 │
│     Lane1_RX± ◀───────────────┼───────────────────────────────│───  Lane1_RX±                 │
│                               │                               │                               │
└───────────────────────────────┘                               └───────────────────────────────┘
```

### Observed CLx Capability Checks (dmesg)

The following dmesg excerpts come from a Dell laptop with an AMD USB4 host router (vendor 0x438, device 0x20e) running Linux 6.19.0-rc6 with `thunderbolt.dyndbg=+pt` enabled on the kernel command line. A Dell Thunderbolt 4 Dock (Intel JHL8540, vendor 0x8087, device 0xb26, 19 ports) is connected to domain 1 at route 0x2 (depth 1, through lane adapter port 2). Two OWC Envoy Express Thunderbolt 3 NVMe enclosures (Intel JHL6540, 8086:15c0) are connected downstream of the dock at routes 0x502 (through dock port 5) and 0x702 (through dock port 7). CL states allow the USB4 link to enter low-power modes between packet bursts, reducing power consumption. However, CL state support requires both ends of the link to support it. These dmesg excerpts show how the kernel checks CLx capabilities and what happens when one side does not support CLx.

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

#### CLx Support Check on the Depth-1 Link

When the Dell dock first connects at route 0x2, the CM checks CLx support on both sides of the host-to-dock link by calling tb_port_clx_supported() which reads LANE_ADP_CS_0 bits [26:28] (CL0s/CL1/CL2 Support):

```
[    4.641532] [12] thunderbolt 0000:c7:00.6: 2:1: CLx: CL0s/CL1 supported
[    4.641540] [12] thunderbolt 0000:c7:00.6: 0:2: CLx: CL0s/CL1 not supported
```

The dock's upstream port (2:1, meaning route 2, adapter 1) supports CL0s and CL1. However, the host router's downstream port (0:2, meaning route 0, adapter 2) does not support CLx. Since tb_switch_clx_enable() requires both sides (upstream and downstream) to support CLx, CL states are effectively unavailable for this link. The CLx support bits are read from LANE_ADP_CS_0: CL0s Support [26], CL1 Support [27], CL2 Support [28].

#### Packet Trace Showing CLx Bits in LANE_ADP_CS_0

The CLx support bits are visible in the USB4 packet traces. When reading LANE_ADP_CS_0 from dock port 7 during the second hot-plug event at t=28s, the CLx bits are all zero:

```
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
```

Dock port 7 (a downstream-facing lane adapter on the Intel JHL8540) reports CL0s = 0, CL1 = 0, CL2 = 0, meaning this particular port does not support any CLx states. This contrasts with the dock's upstream port 1, which does support CL0s and CL1. CLx capability varies per port on a router, not just per device. The raw register value 0x003c013e has bits [28:26] all clear, matching the decoded output.

#### Impact on TMU Mode Selection

The CLx check directly influences the TMU mode selection. When neither CL0s nor CL1 is available on the link, the CM falls back to bi-directional HiFi TMU mode (the highest-accuracy but highest-power mode):

```
[    4.641545] [12] thunderbolt 0000:c7:00.6: 2: TMU: mode change off -> bi-directional, HiFi requested
[    4.644275] [12] thunderbolt 0000:c7:00.6: 2: TMU: mode set to: bi-directional, HiFi
```

Because the host port 0:2 does not support CLx, tb_enable_tmu() selects bi-directional HiFi mode. If CL1 had been supported on both sides, the CM would have chosen uni-directional LowRes or enhanced uni-directional MedRes mode instead, saving power. The TMU mode determines the time synchronization accuracy and power overhead on the link.

#### Repeated CLx Check After Downstream Device Setup

The CLx check is performed again later (after the downstream OWC NVMe enclosure at route 502 is fully configured) because tb_enable_clx() is called as part of the post-scan sequence:

```
[    6.161826] [12] thunderbolt 0000:c7:00.6: 2:1: CLx: CL0s/CL1 supported
[    6.161834] [12] thunderbolt 0000:c7:00.6: 0:2: CLx: CL0s/CL1 not supported
```

The same result is obtained: the dock's upstream port supports CLx but the host's downstream port does not. This second check occurs because CL states are only enabled on the depth-1 link (between the host and the first-level device), and the CM rechecks after every topology change that might affect bandwidth or power state decisions.

#### CLx Current Mode

The dock also reports its current CLx mode after initial setup:

```
[    3.887999] [12] thunderbolt 0000:c7:00.6: 2: CLx: current mode: disabled
```

CLx is disabled on the dock because tb_switch_clx_enable() was not called (the host port does not support it). The "current mode: disabled" is read from the sw->clx field, which remains 0 when no CL states are enabled. On a system where both ends support CLx, this would show "CL0s" or "CL1" after enable.
