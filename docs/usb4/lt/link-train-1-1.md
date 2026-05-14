---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Link Training Phase 1, Part 1

## SPECIFICATIONS

## LINUX KERNEL

Link Training Phase 1 is handled by the USB Type-C port controller and PD stack, outside the thunderbolt driver. The thunderbolt driver begins its work after Phase 1 completes.

## OTHER INFORMATION

- [What is the USB Type-C Signal Plan? How does orientation independence happen?](https://youtu.be/jZcB7JT9Bqw)
- [USB Type-C Essentials: An Introduction to USB Type-C Technology](https://youtu.be/V1OiQoyjDOo)
- [Universal Serial Bus Type-C Cable and Connector Specification](https://www.usb.org/sites/default/files/USB%20Type-C%20Spec%20R2.0%20-%20August%202019.pdf)

## DETAILS

In this phase, the Type-C connectors tries to check the physical capability of both ports and the capable, as well as the plug orientation, the upstream/downstream ports, and identify the capability of the cable (e.g. whether it's electronically marked).

### Plug-orientation problem

```
┌───────────────────────────────────────────────────────────────────────┐
│ A1   A2    A3    A4    A5    A6    A7    A8    A9    A10   A11   A12  │
│ GND  TX1+  TX1-  VBUS  CC1   D+    D-    SBU1  VBUS  RX2-  RX2+  GND  │
 ======================================================================
│ GND  RX1+  RX1-  VBUS  SBU2  D-    D+    CC2   VBUS  TX2-  TX2+  GND  │
│ B12  B11   B10   B9    B8    B7    B6    B5    B4    B3    B2    B1   │
└───────────────────────────────────────────────────────────────────────┘
```

The USB-C port controllers communicate via the PD protocol through the CC channel. Although there are [`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49) (`A5`) and [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50) (`B5`), when plugged, only one of them will become the CC channel. The other one will become the `VCON` for power supply. Determining which is which is one of the gaol here.

### Key resistors: `Rp`, `Rd`, `Ra`

```
Host (Source) side                                Device (Sink) side
────────────────────                              ──────────────────

 PU ─ Rp ──► CC1 ──────────── Cable ───────────► CC1 ── Rd (5.1kΩ) ── GND
 PU ─ Rp ──► CC2 ─── Ra (≈1kΩ in plug) ────────► GND

```

#### On DFP: pull-up resistor `Rp`

A downstream-facing port always has pull-up resistor on its `A5` and `B5`. There's only 2 possible values for each pull-up voltage:

1. For 3.3V, it could be either 12k or 4.7k
2. For 5.0V, it could be either 22k or 10k.

This resistor is also used by cable designer to advertise the `VBUS` power:

1. 12k (for 3.3V PU) or 22k (for 5.0V PU): the host uses `5V@1.5A` `VCONN` power.
1. 4.7k (for 3.3V PU) or 10k (for 5.0V PU): the host uses `5V@3.0A` `VCONN` power.

#### On UFP: pull-down resistor `Rd`

A upstream-facing port always has pull-up resistor on its `A5` and `B5`. This value is constant 5.1k Ohm.

#### On eletronically-marked cable `Ra`

The USB4 requires an eletronically-marked cable to work. Cables are marked by a 1K resistor on their `VCONN`. This load makes it possible for the type-C port controller to detect electronically-marked cable simply by checking the voltage.

#### The detection mechanism

When Host-Cable-Device are connected, the `Rp`-`Rd` pair on one of the `CC` channels and the `Rp`-`Ra` pair on the other channel essentially become 2 independent voltage dividers:

```
Host (Source) side                                Device (Sink) side
────────────────────                              ──────────────────

 PU ─ Rp ──► CC1 ──────────── Cable ───────────► CC1 ── Rd (5.1kΩ) ── GND
 PU ─ Rp ──► CC2 ─── Ra (≈1kΩ in plug) ────────► GND

 PU = 3.3 V/5.0 V
```

Or this:

```
Host (Source) side                                Device (Sink) side
────────────────────                              ──────────────────

 PU ─ Rp ──► CC1 ─── Ra (≈1kΩ in plug) ────────► GND
 PU ─ Rp ──► CC2 ──────────── Cable ───────────► CC2 ── Rd (5.1kΩ) ── GND

 PU = 3.3 V/5.0 V
```

The way the values `Rp`, `Rd`, `Ra` are design in the spec makes it easy to determine the following questions by the voltage at the downstream of `Rp` on both channels, or the `Vsense`:

1. Plug orientation: i.e. among [`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49) an [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50) which one is `CC` and which one is `VCON`. This is achieved by comparing `Vsense` on both channel.
2. Identify electronically-marked cable: by load inducred by `Rd`.
3. Advertise what `VBUS` power to use: by the value of `Rp`, reflected on the value of `Vsense`.

### Q1: Plug Orientation

The port controller decides orientation by the comparing voltage values immediately after the `Rp` on [`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49) and [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50). The one with the larger voltage value is the `CC` channel, and the other one is the `VCON`

#### Case: A-side up

```
Host (Source) side                                Device (Sink) side
────────────────────                              ──────────────────

 PU ─ Rp ──► CC1 ──────────── Cable ───────────► CC1 ── Rd (5.1kΩ) ── GND
 PU ─ Rp ──► CC2 ─── Ra (≈1kΩ in plug) ────────► GND

```

The `Vsense` on [`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49) and [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50) are:

1. On [`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49): 0.4V / 0.94V / 1.7V (depends on `Rp` value: 56k / 22k / 10k)
2. On [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50): ~0.1 V (the `Ra` signature)

By the `Vsense` values, the host sees `Rd` on [`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49) and determines this is the `CC` channel to use (A-side up). By this logic the [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50) must be the `VCONN`, so the Host then supplies `VCONN` on [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50) and enables `VBUS`.

#### Case: B-side up

```
Host (Source) side                                Device (Sink) side
────────────────────                              ──────────────────

 PU ─ Rp ──► CC1 ─── Ra (≈1kΩ in plug) ────────► GND
 PU ─ Rp ──► CC2 ──────────── Cable ───────────► CC2 ── Rd (5.1kΩ) ── GND

```

The `Vsense` are:

1. [`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49): ~0.1V (`Ra` signature)
2. [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50): 0.4 / 0.94 / 1.7 V (depends on `Rp` value)

Host sees `Rd` on [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50) and determines this is the B-side up orientation, then it takes [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50) as the `CC` channel, and supplies `VCONN` on [`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49) and enables `VBUS`.

### Q2: eMarked cable

The actual value `Vsense` on the `VCONN` is also an indicator whether a cable is electronically marked. If the `Ra` were not there, the `Vsense` on `VCONN` would be `GND` instead of `1/(Rp + 1k)`. This is how an electronically marked is detected.

### Q3: VBUS Power

The value of `Rp` on the DFP, hence the `Vsense`, is used by the cable designer to advertise the the `VBUS` power. This again is detected by the `Vsense` this `Rp` induced. See the following example.

### Example: 5V Pull-Up voltage

```
Host (DFP / Source)                             Cable plug                          Device (UFP / Sink)
────────────────────────────────────────────────────────────────────────────────────────────────────────────

   PU (5.0V)
     │
     ├─ Rp = 22k  ──►── CC1  ─────────── Ra ≈ 1.0k (in plug) ────────► GND
     │
     └─ Rp = 22k  ──►── CC2  ─────────────── Cable conductor ───────────►  CC2 ── Rd = 5.1k ── GND
                     (Rp options: 22k / 10k)
```

#### The Vsense values

On the [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50) channel, according to the `Rp` value, the `Vsense` could be:

- With `Rp` = 22 kΩ, `Vsense` ≈ 0.941 V
- With `Rp` = 10 kΩ, `Vsense` ≈ 1.689 V

On the other hand, for the [`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49)'s `Vsense` (that sees Ra ≈ 1 kΩ):

- With `Rp` = 22 kΩ: `Vsense` ≈ 0.217 V
- With `Rp` = 10 kΩ: `Vsense` ≈ 0.455 V

#### Q1: The Plug Orientation

In whatever cases, the `Vsense` in [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50) is always smaller than that of [`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49) (a.k.a. Host sees `Rd` on [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50)), so [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50) is the `CC` channel, the orientation is B-side up (CC2 active).

#### Q2: Identify eMarked Cable

The other `CC` ([`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49)) shows non-zero voltage (due to `Ra` is not `0`), so the host may enable `VCONN` on [`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49) to power cable e-marker.

#### Q3: The VCONN Power

This is according to the `Vsense` range on the `CC` channel.

- With `Rp` = 22 kΩ, `Vsense` = 0.941 V  -> 1.5 A
- With `Rp` = 10 kΩ, `Vsense` = 1.689 V  -> 3.0 A

### Example: 3.3V Pull-up voltage

```
Host (DFP / Source)                     Cable plug                    Device (UFP / Sink)
─────────────────────────────────────────────────────────────────────────────────────────────

   VBUS (3.3V)
     │
     ├─ Rp (12k / 4.7k) ──► CC1 ─────── Ra ≈ 1.0k (in plug) ───────►  GND
     │
     └─ Rp (12k / 4.7k) ──► CC2 ───────  Cable conductor  ─────────►  CC2 ── Rd = 5.1k ── GND

```

#### The Vsense values

`Vsense` on the [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50):

- With `Rp` = 12k,  `Vsense` = 0.984 V
- With `Rp` = 4.7k, `Vsense` = 1.717 V

`Vsense` on the [`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49):

- With `Rp` = 12k,  `Vsense` = 0.253 V
- With `Rp` = 4.7k, `Vsense` = 0.579 V

#### Q1: The Plug Orientation

Comparing the `Vsense` in both [`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49) and [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50), the `Vsense` in [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50) is always smaller than that of [`CC1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L49) (a.k.a. Host sees `Rd` on [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50)), so [`CC2`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mfd/stm32-timers.h#L50) is the `CC` channel, the orientation is B-side up.

#### Q2: eMarked Cable

The `Vsense` in `VCONN` is non-zero, so it is electronically marked.

#### Q3: The VBUS Power

The `Rp` value can be speculated by the `Vsense` on the `CC` channel. This determines the intended power:

- With `Rp` = 12k,  `Vsense` = 0.253 V
- With `Rp` = 4.7k, `Vsense` = 0.579 V
