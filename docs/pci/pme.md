---
topics: pcie
tags:
    - "pcie"
    - "power-management"
---

# Power Management Event (PME)

## Registers

### In Power Management Capability

```
Capabilities: [80] Power Management version 3
        Flags: PMEClk- DSI- D1- D2- AuxCurrent=0mA PME(D0-,D1-,D2-,D3hot-,D3cold-)
        Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=0 PME-
```

1. `PME`: the states where a function is capable of sending PME.
2. `Status`: this is the `PMCSR` register:
    1. `PME-Enable`: whether or not the function is allowed to send PME.
    2. `PME`: has any PME from the function occured?

## Transition back to `D0`

A function can't simply switch to `D0` by itself. Instead, it has to send PME upstream to ask software for doing that. This also includes writing to its `PMCSR` to change the device D-state. 

However, before the the PME can be sent, the Link must be put back to `L0`. If the Link is under `L1`, then going through `Recovery` would be enough. If Link is in `L2` or `L3`, the main power has to be applied first. Because the Link is not usable at this point, there have to be other ways to request that power back.

## Request main power from Link

If `Vaux` is available, the device could signal the `WAKE#` pin. Then system can then power back the device so that Link could train, and once Link is ready the function finally send PME to ask software set it back to `D0`.

## References

### Videos

### Links

- [[PATCH -v7 4/4] PCIe, PM, Add PCIe runtime D3cold support](https://lore.kernel.org/all/1340418231-10134-5-git-send-email-ying.huang@intel.com/)
- [1.2. Native PCI Power Management](https://docs.kernel.org/power/pci.html#native-pci-power-management)
- [PCIe device states, Qualcomm Linux Interfaces Guide](https://docs.qualcomm.com/bundle/publicresource/topics/80-70018-8/pcie.html#pcie-device-states)
- [2.15 Power Management, KeyStone Architecture Peripheral Component Interconnect Express (PCIe)](https://www.ti.com/lit/ug/sprugs6d/sprugs6d.pdf): for D-state state machine diagram.
- [`include/uapi/linux/pci_regs.h`](https://elixir.bootlin.com/linux/v6.16/source/include/uapi/linux/pci_regs.h#L238): for register definitions.
- [Power Management Control And Status Register, 13th Generation Intel® Core™ Processor Datasheet, Volume 2 of 2](https://edc.intel.com/content/www/us/en/design/publications/13th-generation-core-processor-datasheet-volume-2-of-2/power-management-control-and-status-register-pmectrlstatus-offset-84/): for register definitions.
- [3.7. Power Management, F-Tile Avalon® Streaming Intel® FPGA IP for PCI Express* User Guide](https://www.intel.com/content/www/us/en/docs/programmable/683140/21-4-4-0-0/power-management-37666.html): an example how D-states are related to the L-states.
- [2.15.3 Relationship Between Device and Link Power States](https://www.ti.com/lit/ug/sprugs6d/sprugs6d.pdf): another example how D-states are related to the L-states.

### Specification

- 5.3.3 Power Management Event Mechanisms
- 6.1.5 PME Support
