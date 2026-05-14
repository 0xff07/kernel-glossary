---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# Link Training Phase 5, LOCK1

## SPECIFICATIONS

## LINUX KERNEL

- [`drivers/thunderbolt/sb_regs.h`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h): [`USB4_SB_GEN23_TXFFE`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/sb_regs.h#L44) (TxFFE register at 0x0D) used during link equalization

Phase 5 link equalization is handled in hardware by the lane adapters and PHY. The negotiated TxFFE parameters are readable through the sideband registers.

## DETAILS

### Overview

```
              Phase 5 – Link Equalization (TxFFE Negotiation)

   ┌─────────────┐                                       ┌─────────────┐
   │ Transmitter │                                       │   Receiver  │
   └──────┬──────┘                                       └──────┬──────┘
          │                                                     │
          │ 1. Tx starts with Tx Active=1, ReqDone=0            │
          │---------------------------------------------------->│
          │                                                     │
          │ 2. Tx reads Rx Status & TxFFE Request               │
          │<----------------------------------------------------│
          │                                                     │
          │ 3. If Rx Locked=1 → negotiation complete            │
          │    Else if NewReq=0 → retry (poll)                  │
          │    Else → process new TxFFE request                 │
          │                                                     │
          │ 4. Tx loads requested TxFFE Preset (0..15 for Gen2/3│
          │    or 0..49 for Gen4)                               │
          │    Updates Tx Status: TxFFE Setting + ReqDone=1     │
          │---------------------------------------------------->│
          │                                                     │
          │ 5. Tx reads Rx Status again                         │
          │<----------------------------------------------------│
          │                                                     │
          │ 6. If NewReq=1 → Rx still tuning → retry            │
          │    Else → go back to Step 2 or finish if locked     │
          │                                                     │

```

### TxFFE Negotiation

```

 Data Path (High-speed lanes)                    Control Path (Sideband)
────────────────────────────────────────────     ─────────────────────────────────────

 Router A (Tx)                Router B (Rx)      Router A (Tx)         Router B (Rx)
 ─────────────────────────    ───────────────    ─────────────────     ───────────────


   SLOS1 ───────────── PAM2 Ordered Set ─────────────▶ SLOS1
         (FFE setting embedded in symbols)

                                                   Initial Condition:

                                                   Tx's TxFFE Control Reg
                                                   - Tx Active = 1
                                                   - Request Done = 0
                                                   Tx polling Rx's TxFFE

                                                                       Rx's TxFFE Control Reg
                                                                       - Rx Locked = 0
                                                                       - Rx Active = 0
                                                                       - New Request = 0


                                                                        STEP 1: Rx Read
                                                   ─────── Rx Read  ───────────▶
                                                   Tx responds:
                                                   - Tx Active = 1
                                                                        Rx sees:
                                                                        - Tx Active = 1
                                                                        Raise new request:
                                                                        - Rx Active = 1
                                                                        - Rx Lock = 0
                                                                        - New Request = 1
                                                                        - PresetIdx = <N>

                                                   STEP 2: Tx (polling)
                                                   "Rx Status & TxFFE Request"
                                                   ◀─────── Tx Read  ───────────
                                                                        Rx responds:
                                                                        - Rx Lock = 0
                                                                        - New Request = 1
                                                                        - PresetIdx = <N>

                                                   Tx sees new request:
                                                   - Rx Lock = 0
                                                   - New Request = 1
                                                   Applies requested preset:
                                                   - TxFFE Setting Reg = <N>
                                                   - Request Done = 1

                                                                         STEP 3: ACK
                                                                         Rx Reads
                                                   ──────── RT Read ───────────▶
                                                   Tx Responds
                                                   - Request Done = 1
                                                                         Rx sees RequestDone
                                                                         - New Request = 0

   TS1  ───────────── PAM2 Ordered Set ─────────────▶  TS1
         (FFE retuned, continues with new preset)

                                                      Step 4:
                                                      Rx checks eye opening
                                                      If not locked → issue new Req (New Request: 0 -> 1)
                                                      If locked → stop requests (Rx Lock: 0 -> 1)

   TS2  ───────────── PAM2 Ordered Set ─────────────▶  TS2
         (FFE fixed, Equalization complete)
```
