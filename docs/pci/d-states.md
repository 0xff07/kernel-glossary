---
topics: pcie
tags:
    - "pcie"
    - "power-management"
---

# D-States

See references.

## Registers

### In Power Management Capability

```
Capabilities: [80] Power Management version 3
        Flags: PMEClk- DSI- D1- D2- AuxCurrent=0mA PME(D0-,D1-,D2-,D3hot-,D3cold-)
        Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=0 PME-
```

1. `D1` and `D2`: whether or not the function support `D1` and `D2`. 
2. `PME`: the states where a function is capable of sending PME.
3. `Status`: this is the `PMCSR` register:
    1. `D0`: the device is in `D0` state.
    2. `NoSoftRst`: when resuming from `D3Hot`, the function can goes back to `D0` (initialized), instead of having to go to `D0` (uninitialized) first (i.e. no need to do soft reset when transitioning from `D3Hot` to `D0`).
    3. `PME-Enable`: whether or not the function is allowed to send PME.
    4. `DSel` and `DScale`: TODO.
    5. `PME`: has any PME from the function occured?

## References

### Videos

### Links

- [1.2. Native PCI Power Management](https://docs.kernel.org/power/pci.html#native-pci-power-management)
- [Device power states](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/device-power-states)
- [PCIe device states, Qualcomm Linux Interfaces Guide](https://docs.qualcomm.com/bundle/publicresource/topics/80-70018-8/pcie.html#pcie-device-states)
- 2.15.1 Device Power Management, [KeyStone Architecture Peripheral Component Interconnect Express (PCIe)](https://www.ti.com/lit/ug/sprugs6d/sprugs6d.pdf): for D-state transition diagram.
- 2.15.3 Relationship Between Device and Link Power States, [KeyStone Architecture Peripheral Component Interconnect Express (PCIe)](https://www.ti.com/lit/ug/sprugs6d/sprugs6d.pdf): an example how D-states are related to the L-states.
- [Power Management Control And Status Register, 13th Generation Intel® Core™ Processor Datasheet, Volume 2 of 2](https://edc.intel.com/content/www/us/en/design/publications/13th-generation-core-processor-datasheet-volume-2-of-2/power-management-control-and-status-register-pmectrlstatus-offset-84/): for `PMCSR` definitions.
- [`include/uapi/linux/pci_regs.h`](https://elixir.bootlin.com/linux/v6.16/source/include/uapi/linux/pci_regs.h#L238): for macros and register definition in Linux.
- [3.7. Power Management, F-Tile Avalon® Streaming Intel® FPGA IP for PCI Express* User Guide](https://www.intel.com/content/www/us/en/docs/programmable/683140/21-4-4-0-0/power-management-37666.html): an example how D-states are related to the L-states.

### Specification

- 5.3.1 Device Power Management States (D-States) of a Function
