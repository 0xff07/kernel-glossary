---
topics: pcie
tags:
    - "pcie"
    - "power-management"
---

# D3Cold

Configuration space is not accessible in `D3Cold`. Either there's a `WAKE#` or beacon to signal the need of main power, or there needs to be other platform-specific ways to power up the function again to `D0` (usually through ACPI support.)

## References

### Links

- [[PATCH -v7 4/4] PCIe, PM, Add PCIe runtime D3cold support](https://lore.kernel.org/all/1340418231-10134-5-git-send-email-ying.huang@intel.com/)
- [1.2. Native PCI Power Management](https://docs.kernel.org/power/pci.html#native-pci-power-management)
- [Firmware requirements for D3cold](https://learn.microsoft.com/en-us/windows-hardware/drivers/bringup/firmware-requirements-for-d3cold)

### Specification

- 5.3.1.4.2 D3cold State
