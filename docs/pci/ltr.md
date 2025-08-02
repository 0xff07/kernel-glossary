---
topics: pcie
tags:
    - "pcie"
    - "power-management"
---

# Latency Tolerance Reporting (LTR)

Defines the time interval that a function can tolerate without being served by the root. After this period of time, the function might be entering lower power management state. A function reports this so that the root can prioritize its services.

## Registers

### In Latency Tolerance Reporting Capability Structure

```
Capabilities: [1b8 v1] Latency Tolerance Reporting
        Max snoop latency: 15728640ns
        Max no snoop latency: 15728640ns
```

### In L1 PM Substates Capability Structure

```
Capabilities: [900 v1] L1 PM Substates
        L1SubCap: PCI-PM_L1.2+ PCI-PM_L1.1+ ASPM_L1.2+ ASPM_L1.1- L1_PM_Substates+
                  PortCommonModeRestoreTime=32us PortTPowerOnTime=10us
        L1SubCtl1: PCI-PM_L1.2+ PCI-PM_L1.1+ ASPM_L1.2+ ASPM_L1.1-
                   T_CommonMode=0us LTR1.2_Threshold=116736ns
        L1SubCtl2: T_PwrOn=50us
```

## References

### Specification

- 6.18 Latency Tolerance Reporting (LTR) Mechanism
