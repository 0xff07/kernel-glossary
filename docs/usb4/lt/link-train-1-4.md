---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Link Training Phase 1, Part 5 (UFP VDOs)

## SPECIFICATIONS

## LINUX KERNEL

Link Training Phase 1 is handled by the USB Type-C port controller and PD stack, outside the thunderbolt driver. The thunderbolt driver begins its work after Phase 1 completes.

## OTHER INFORMATION

- [USB Power Delivery](https://www.usb.org/sites/default/files/D2T2-1%20-%20USB%20Power%20Delivery.pdf): for UFP VDO1 and UFP VDO2

## DETAILS

```
 Source (DFP)                           Sink (UFP, USB4 Device)
─────────────────────────────────────────────────────
   |                                            |
   | --- Structured VDM: Discover Identity ---> |
   |                                            |
   | <--- Structured VDM: Response (ACK) -------|
   |          ID Header VDO +                   |
   |               Cert VDO +                   |
   |            Product VDO +                   |
   |               UFP VDO1 +                   |
   |               UFP VDO2                     |
```

### UFP VDO1 (capability related)

1. Device capability: [`USB2`](https://elixir.bootlin.com/linux/v6.19/source/include/dt-bindings/usb/pd.h#L367), `USB3.2` etc. Support for USB4 is also marked here ([`USB4`](https://elixir.bootlin.com/linux/v6.19/source/include/dt-bindings/usb/pd.h#L366)).
2. Alternate mode bitmask: if TBT is supported, it'll also be masked here.
3. USB highest speed: see slides.

### UFP VDO2 (power related)

Max power when operating under USB4 and USB 3.2 (charging and without charging). Hard-coded by the designer.

