---
topics: pcie
tags:
    - "pcie"
    - "power-management"
---

# L0s state

`L0s` is ASPM-specific. Note that `L0s` state is direction specific. It is possible on the Tx direction the Link is in L0s, and the Rx direction is not.

## Entering L0s state

Send EIOS and stay in Electrical Idle for certain period of time.

## Exit L0s state

Send EIEOS first, and then certain number of FTS ordered set, and send an SOS. How many FTS is need has already described in the TS1 and TS2 (in the `N_FTS`) during the Link Training.

Note that if the receiver is not able to achieve bit lock and symbol lock in time, both directions in the Link will go to Recovery.

## References

### Videos

### Links

- [2.15.2 Link State Power Management, KeyStone Architecture Peripheral Component Interconnect Express (PCIe)](https://www.ti.com/lit/ug/sprugs6d/sprugs6d.pdf): for the ASPM L-state diagram.
- [The Importance of Efficient SSD Power Management, PHISON Blog](https://phisonblog.com/the-importance-of-efficient-ssd-power-management-2/): for Link State diagram.

### Specification

- 4.2.6.6 L0s
- 5.4.1.1 L0s ASPM State
- 5.4.1.3 ASPM Configuration
