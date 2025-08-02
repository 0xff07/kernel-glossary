---
topics: pcie
tags:
    - "pcie"
    - "pcie-ltssm"
---

# L2/L3 Ready

This is the transient state between `L0` and actually entering `L2` or `L3`.

## Entering L2/L3 Ready

This happens after software put all the downstream components into `D3Hot`, and all the Links further downsteam have been in `L1` state.

System software broadcasts `PME_Turn_Off` downstream to quiese PME first. This put the Link briefly into `L0` state. The `PME_To_Ack` from downstream components are gathered and routed upstream.

Once all the downstream components ack, in each individual Links the components exchange `PM_L23_Ready` DLLP and `PM_Request_Ack` DLLP, then both side enters electrical idle.

## References

### Videos

### Links

- [2.15.2 Link State Power Management, KeyStone Architecture Peripheral Component Interconnect Express (PCIe)](https://www.ti.com/lit/ug/sprugs6d/sprugs6d.pdf): for the L-state diagram.
- [The Importance of Efficient SSD Power Management, PHISON Blog](https://phisonblog.com/the-importance-of-efficient-ssd-power-management-2/): for L-state diagram.
- Endpoint D3 Entry, [3.7. Power Management, F-Tile Avalon® Streaming Intel® FPGA IP for PCI Express* User Guide](https://www.intel.com/content/www/us/en/docs/programmable/683140/21-4-4-0-0/power-management-37666.html): an example for transitioning to L2 and L3.

### Specification

- 5.3.2 PM Software Control of the Link Power Management State
- 5.3.2.3 Entry into the L2/L3 Ready State
