---
topics: pcie
tags:
    - "pcie"
    - "power-management"
---

# ASPM L1

Note that for `L1` state, other than directed by software through the Configuration Space, some hardware is capable of negotiating with each other and enter `L1` state autonomously, without letting software know. This capabiliy is called the *Active-state power management*, or ASPM. 

To distinguish those 2 cases (software-directed vs. autonomous), sometimes the software-directed `L1` state is called PCI-PM L1, and the autonomously entered L1 state by the ASPM feature is called ASPM L1.

## Registers

### In PCI Express Capability Structure

```
Capabilities: [40] Express (v2) Root Port (Slot+), MSI 00
        LnkCap: Port #11, Speed 16GT/s, Width x4, ASPM L0s L1, Exit Latency L0s <4us, L1 <64us
                ClockPM- Surprise- LLActRep+ BwNot+ ASPMOptComp+
        LnkCtl: ASPM L1 Enabled; RCB 64 bytes, Disabled- CommClk+
                ExtSynch- ClockPM- AutWidDis- BWInt+ AutBWInt+
```

In the Link Capibility Register and the Link Control Register:

1. `LnkCap`: `ASPM` field shows that which states a function is capable of entering through ASPM. In this case both ASPM `L0s` and ASPM `L1` are supported.
    The `Exit Latency` indicate the longest possible latency for a Link to resume from ASPM L-states. 
2. `LnkCtl`: `ASPM` field shows which ASPM states are currently enable. For example, here even though the `LnkCap` suggests that the function is both ASPM `L0s` and ASPM `L1` capable, only ASPM `L1` is currently enabled.

## Enter L1 by ASPM

Initiate by downstream device, by sending `PM_Active_State_Request_L1` to and then receive `PM_Request_ACK` DLLP. If both sides agrees with each other, they send EIOS and enter electrical idle.

## Exiting ASPM L1

Sending EIEOS and TS1.

## Topology matters

If ASPM L1 exit is initiated by an endpoint, components on the path all the way to the Root Complex have to be woken up as well. If ASPM L1 exit initiated by upstream component, all the downstream links that are in ASPM L1 will also be woken up, but not the ones in PCI-PM directed L1.

## References

### Links

- [ASPM on Linux, Linux Wireless documentation](https://wireless.docs.kernel.org/en/latest/en/users/documentation/aspm.html)
- [Firmware Test Suite - aspm test](https://wiki.ubuntu.com/FirmwareTestSuite/Reference/aspm)
- [`drivers/pci/pcie/aspm.c`](https://elixir.bootlin.com/linux/v6.16/source/drivers/pci/pcie/aspm.c)
- p34, [Troubleshooting PCI Express® Link Training and Protocol Issues](https://pcisig.com/sites/default/files/files/02_01_Troubleshooting_PCI_Express_Link_Training_and_Protocol_Issues_FROZEN.pdf): *ASPM `L1` the downstream component indicates the desire to enter the `L1` state, and sends `PM_Active_State_Request_L1` to the root, the root acknowledges with `PM_Request_ACK DLLP`*
- p39, [Troubleshooting PCI Express® Link Training and Protocol Issues](https://pcisig.com/sites/default/files/files/02_01_Troubleshooting_PCI_Express_Link_Training_and_Protocol_Issues_FROZEN.pdf): for conditions and Ordered Sets required to enter L1 states.
- [2.15.2 Link State Power Management, KeyStone Architecture Peripheral Component Interconnect Express (PCIe)](https://www.ti.com/lit/ug/sprugs6d/sprugs6d.pdf): for the L-states involved in ASPM
- [The Importance of Efficient SSD Power Management, PHISON Blog](https://phisonblog.com/the-importance-of-efficient-ssd-power-management-2/): for general ASPM description.

### Specification

- 4.2.6.7 L1
- 5.4.1.2 L1 ASPM State
