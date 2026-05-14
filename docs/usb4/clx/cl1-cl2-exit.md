---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# CL1/CL2 Exit (without re-timers)

## SPECIFICATIONS

- USB4 Specification, section 4.2.1.6.5.2: Gen 2 and Gen 3 Exit flow from CL1 or CL2 state (No Re-timers on the Link)

## LINUX KERNEL

- [`drivers/thunderbolt/clx.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/clx.c): CLx management
- [`drivers/thunderbolt/tb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h): [`LANE_ADP_CS_1`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L348) status fields reflect current adapter state

The CL1/CL2 exit flow is handled in hardware by the lane adapters. The CM monitors exit by reading the adapter state field in [`LANE_ADP_CS_1`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb_regs.h#L348) (bits 29:26).

## DETAILS

### LFPS, Idle, and SLOS1

```
==============================
Router A        Router B
==============================
 |                  |
 |==== LFPS ======> |
 |<=== LFPS ======  |
 |                  |
 |---- Idle --------|
 |                  |
 |==== SLOS1 =====> |
 |<=== SLOS1 =====  |
```

Note that the `SLOS1` will be sent at the speed where they got suspended. The `TxFFE` registers on both sides are also recovered, so there's no need to redo Link Equalization.

### LOCK1, LOCK2, TS1, TS2

After seeing the `SLOS1` order set, both Routers enter `LOCK1` substate of the `Training` state (Phase 5 of the link traing). Then follows link training process.

Note that the link parameters negotiated in Phase 1 to Phase 4 are recovered as-is during resume.
