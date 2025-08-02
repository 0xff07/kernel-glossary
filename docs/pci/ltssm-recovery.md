---
topics: pcie
tags:
    - "pcie"
    - "pcie-ltssm"
---

# Recovery

The general pattern for a `Recovery` would look like:

1. Enter `Recovery` by sending `TS1`: also fill the fields in the `TS1` to indicate the reason and details to enter `Recovery` (e.g. `speed_change` ot not? `autonomous_change` or not?)
2. `Recovery.RecvrLock`: achieve bit lock and symbol lock by exchanging TS1.
3. `Recovery.RecvrCfg`: the actual stage where meaningful communication happens. Negotiate what both sides would like to do by exchanging TS2.
4. Once both the agreement is reached, switch to the next state and starting doing that.

## References

### Links

- p28, [Troubleshooting PCI Express® Link Training and Protocol Issues](https://pcisig.com/sites/default/files/files/02_01_Troubleshooting_PCI_Express_Link_Training_and_Protocol_Issues_FROZEN.pdf): the Recovery state machine.
- [PCI Express* 3.0 Technology: PHY Implementation Considerations for Intel Platforms](https://www.intel.de/content/dam/doc/guide/pci-express3-phy-implementation-considerations-idf2009-presentation.pdf): as old as it might look, this is one of the very few sources that actually mentioned 128b/130b encoding and the role of 128b/130b TS1/TS2 in speed change.

### Specification

- 4.2.6.4 Recovery
