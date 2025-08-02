---
topics: pcie
tags:
    - "pcie"
    - "pcie-ltssm"
---

# Scenario: link speed change from L0

##  Registers

### In PCI Express Capability Structure

```
Capabilities: [c0] Express (v2) Endpoint, MSI 00
         ...
         LnkCap: Port #0, Speed 16GT/s, Width x4, ASPM L1, Exit Latency L1 <8us
                 ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp+
         LnkCtl: ASPM L1 Enabled; RCB 64 bytes, Disabled- CommClk+
                 ExtSynch+ ClockPM- AutWidDis- BWInt- AutBWInt-
         LnkSta: Speed 16GT/s, Width x4
                 TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt-
         ...
         LnkCap2: Supported Link Speeds: 2.5-16GT/s, Crosslink- Retimer+ 2Retimers+ DRS-
         LnkCtl2: Target Link Speed: 16GT/s, EnterCompliance- SpeedDis-
                  Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
                  Compliance Preset/De-emphasis: -6dB de-emphasis, 0dB preshoot
         LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete+ EqualizationPhase1+
                  EqualizationPhase2+ EqualizationPhase3+ LinkEqualizationRequest-
                  Retimer- 2Retimers- CrosslinkRes: unsupported
```

1. Link Capability Register
    - `BwNot`: Link Bandwidth Notification Capibility Structure. Notify host on link speed change.

2. Link Control Register
    - `BWInt` and `AutBWInt`: Link Badwidth Management Interrupt Enable and the Link Autonomous Bandwidth Interrupt Enable. For enabling bandwidth change due to Autonomous reasons (e.g. for power management reasons) or for actual bandwidth issues.

3. Link Status Register
    - `BWMgmt` and `ABWMgmt`: Link Bandwidth Management Status and Link Autonomous Bandwidth Status. 

4. Link Control Register 2
    - `SpeedDis`: Hardware Autonomous Speed Disable

### Invoke speed change from software

Software can do speed change by setting the Link Control register:

1. `Target Link Speed`: set this in the `LnkCtl2`.
5. `Retrain Link`: set bit in the Link Control register from a downstream port to force speed change. Only meaningful for a upsteram device.

## Stages of speed change

### Entering recovery

1. `Recovery.RcvrLock`: exchange TS1 in the original speed to achieve symbol lock.

    1. Set the `speed_change` bit to 1 to indicate intention for speed change.
    2. Also advertise the target speed by setting it at the Rate ID field.

    The other side ack by sending back TS1, also with `speed_change` set to 1. If both sides are fine, they enter the `Recover.RcvrCfg` and start exchanging TS2.

2. `Recovery.RcvrCfg`: upstream device send TS2 with both `speed_change` and `autonomous_change` to `1`, while downstream device send back TS2 with `speed_change` bit set to `1`.

3. `Recovery.Speed`: both side send EIOS and enter electric idle and do speed up.

### After speed changed (if no equalization needed)

After that, enter `Recovery.RecvrLock` again, by sending `EIEOS`.

Now they enter the `Recovery.RecvrLock` again, but with new speed from both side, and repeat the process again, but without setting the `speed_change` and `autonomous_change` bits:

1. `Recovery.RecvrLock`
2. `Recovery.RcvrCfg`
3. `Recovery.Idle`

And finally enter L0

### After speed change (if equalization needed)

Equalization is needed when first entering a Gen3 or Gen4 speed for the first time. The general feeling is that link equalization is needed whenever the link enters a 128b/130b-encoding speed for the first time, although for Gen5 it is allowed not to do Link Equalization when first entering this speed.

The difference is in that, after re-entering the `Recovery.RecvrLock` with a new speed, instead of going straight to `Recovery.RecvrCfg`, go for a detour to `Recovery.equalization` first, so it becomes:

1. `Recovery.RecvrLock`: EC (Equalization Control) will be set here, to suggest that link equalization comes next.
2. `Recovery.Equalization`
3. `Recovery.RecvrLock`: new!
4. `Recovery.RcvrCfg`
5. `Recovery.Idle`

And finally enter L0.

## References

### Videos

### Links

- p23, [Troubleshooting PCI Express® Link Training and Protocol Issues](https://pcisig.com/sites/default/files/files/02_01_Troubleshooting_PCI_Express_Link_Training_and_Protocol_Issues_FROZEN.pdf): for the speed change TS1 and TS2 sequence.
- [PCI Express* 3.0 Technology: PHY Implementation Considerations for Intel Platforms](https://www.intel.de/content/dam/doc/guide/pci-express3-phy-implementation-considerations-idf2009-presentation.pdf)
- [Link Status (LSTS) – Offset 52](https://edc.intel.com/content/www/it/it/design/publications/12th-generation-core-processor-datasheet-volume-2-of-2/link-status-lsts-offset-52_2/)
