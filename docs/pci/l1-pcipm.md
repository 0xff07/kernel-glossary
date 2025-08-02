---
topics: pcie
tags:
    - "pcie"
    - "power-management"
---

# PCI-PM L1

## Entering L1 state

Put the downstream component into `D1`, `D2`, or `D3Hot`. The software writes to `PMCSR` register to initiate change.

Under the hood, On transition to one of those D-states, the downstream component send a `PM_Enter_L1` DLLP upstream, and the upsteam component replies with a `PM_Request_ACK` DLLP to ack it. They must agree upon this because this is software-directed. Finally, they exchange `EIOS` to enter electrical idle.

### Exiting L1 state

Exchange `EIEOS` and go to the Recovery state. This can be initiated by either upstream component or downstream component.

In the Recovery state machine:

1. `Recovery.RecvrLock`: do not indicate `spped_change`
2. `Recovery.RcvrCfg`
3. `Recovery.Idle`

(Probably the easiest path for the Recovery state)

And finally to L0.

### Exiting L1 substates

TODO

## References

### Videos

- [Low Power Overview](https://youtu.be/IuOJZ6XT5wU)

### Links

- p34, [Troubleshooting PCI Express® Link Training and Protocol Issues](https://pcisig.com/sites/default/files/files/02_01_Troubleshooting_PCI_Express_Link_Training_and_Protocol_Issues_FROZEN.pdf): *PCIPM uses a config write from the RC to the `PMSCR` register on the device to initiate the transition to `L1`*
- 2.15.3 Relationship Between Device and Link Power States, [KeyStone Architecture Peripheral Component Interconnect Express (PCIe)](https://www.ti.com/lit/ug/sprugs6d/sprugs6d.pdf): for mapping between allowed L-states for different combination of D-states.
- 2.15.2 Link State Power Management, [KeyStone Architecture Peripheral Component Interconnect Express (PCIe)](https://www.ti.com/lit/ug/sprugs6d/sprugs6d.pdf): for the L-state diagram.
- [Troubleshooting PCI Express® Link Training and Protocol Issues](https://pcisig.com/sites/default/files/files/02_01_Troubleshooting_PCI_Express_Link_Training_and_Protocol_Issues_FROZEN.pdf): for conditions and Ordered Sets required to enter L1 states.
- [The Importance of Efficient SSD Power Management, PHISON Blog](https://phisonblog.com/the-importance-of-efficient-ssd-power-management-2/): for L-state diagram.
- p39, [Making the Most of PCIe® Low Power Features](https://pcisig.com/making-most-pcie%C2%AE-low-power-features): for what packets look like when entering L1. Although the example is L1.2, but the entrace and exit part are similar.

### Specification

- 4.2.6.7 L1
- 5.3.2 PM Software Control of the Link Power Management State
- 5.3.2.1 Entry into the L1 State
- 5.3.2.2 Exit from L1 State
