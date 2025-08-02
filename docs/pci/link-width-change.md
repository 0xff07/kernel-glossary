---
topics: pcie
tags:
    - "pcie"
    - "pcie-ltssm"
---

# Scenario: link width change from L0

This could be due to reliability issues, or to reduce power consumption.

## Registers

### In PCIe Capility Structure

1. `AutWidDis`: Hardware Autonomous Width Disable

## Prerequisites

In the `Configuration.Complete` of the Link Training process, both side of the Link set the 6th bit in the Rate ID of the `TS2` to indicate if they are up-configuring capable.

Note that the 6th bit in the Rate ID field serves more like a flag. The meaning of that flag changes according to what state the Link is in.

## Process

### Entering Recovery from L0

From `L0`, enter Recovery state by sending TS1. The 6th bit of Rate ID now means if we are requesting Recovery due to a `speed_change`, which is not. So this bit is set to `0`. Also set the `aunonomous_change` bit in the `TS1` accordingly.

### Recovery

The first 2 stages are somewhat standard in all the `Recovery` processes. However, in the last stage, the downstream device sends PAD-PAD to send both side back to Configuration:

1. `Recovery.RcvrLock`: with `speed_change` set to `0` 
2. `Recovery.RcvrCfg`
3. `Recovery.Speed`: in stead of entering logical idle, send `TS1` with PAD-PAD Link Number and Lane Number.

### Configuration

This is now fall to the sandard Configuration stages: upstream guess a Link Number, downstream replies, upstream guess a Lane Number scheme, downstream replies.

After entering the `Configuration.Idle`, they are ready to enter `L0`.

## References

### Videos

### Links
