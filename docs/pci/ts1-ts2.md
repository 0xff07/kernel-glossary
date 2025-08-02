---
topics: pcie
tags:
    - "pcie"
    - "pcie-ltssm"
---

# TS1 and TS2

Note that training after a reset starts in Gen 1 speed even if the function is capable of higher speed. Gen 1 speed uses 8b/10b encoding, so the Link Trainig process at this stage also uses the 8b/10b version of `TS1` and `TS2`, instead of the 128b/130b ones that the Gen 3 speed (and above) use. Only when the Link enters `L0` and change the speed through the `Recovery` can the Link starts using the 128b/130b version of `TS1` and `TS2`.

## Common fields

## In 8b/10b encoding

## In 128b/130b encoding

## References

### Links

- [PCIe Deep Dive, Part 4: LTSSM](https://scolton.blogspot.com/2024/01/pcie-deep-dive-part-4-ltssm.html)
- p26, [Troubleshooting PCI Express®
Link Training and Protocol Issues](https://pcisig.com/sites/default/files/files/02_01_Troubleshooting_PCI_Express_Link_Training_and_Protocol_Issues_FROZEN.pdf)

### Specification

- 4.2.4.1 Training Sequences
