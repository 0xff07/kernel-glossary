---
topics: usb4
tags:
    - "usb4"
    - "verification-needed"
---

# CL1/CL2 Exit (with re-timers)

## SPECIFICATIONS

- USB4 Specification, section 4.2.1.6: Low Power States (CL0s, CL1, and CL2)
- USB4 Specification, section 4.2.1.6.5: Exit from State
- USB4 Specification, section C.1.2.2: Example: Exit from CL2 (or CL1) State

## LINUX KERNEL

- [`drivers/thunderbolt/retimer.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/retimer.c): Retimer enumeration and management
- [`drivers/thunderbolt/usb4.c`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c): Sideband operations for retimer access
- [`'\<usb4_port_sb_read\>':'drivers/thunderbolt/usb4.c'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/usb4.c#L1354): Read retimer sideband registers via PORT_CS_1
- [`'\<usb4_sb_target\>':'drivers/thunderbolt/tb.h'`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L1374): Sideband target enum (ROUTER, PARTNER, RETIMER)

### [`enum usb4_sb_target`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L1374)

[`enum usb4_sb_target`](https://elixir.bootlin.com/linux/v6.19/source/drivers/thunderbolt/tb.h#L1374) selects the target for sideband register accesses. `USB4_SB_TARGET_RETIMER` is used to access retimer registers during CLx exit recovery:

```c
enum usb4_sb_target {
	USB4_SB_TARGET_ROUTER,
	USB4_SB_TARGET_PARTNER,
	USB4_SB_TARGET_RETIMER,
};
```

## DETAILS

### LFPS, Idle, and SLOS1

```
===================================================================================================================
Adapter A        Retimer1-4          Retimer2-3          Retimer3-2          Retimer4-1          Adapter B
===================================================================================================================
 |                  |                   |                   |                   |                   |
 |==== LFPS ======> |==== LFPS =======> |==== LFPS =======> |==== LFPS =======> |==== LFPS =======> | Detect
 |<=== LFPS ======  |<=== LFPS =======  |<=== LFPS =======  |<=== LFPS =======  |<=== LFPS =======  | Mirror
 |                  |                   |                   |                   |                   |
 |---- Idle --------|---- Idle ---------|---- Idle ---------|---- Idle ---------|---- Idle ---------| Idle
 |      SLOS1       |                   |                   |                   |       SLOS1       |

```

The difference is that in order for `SLOS1` to pass, the Re-timers need to recover first, and the `SLOS1` can pass.

### Router and Re-timer behaviors

#### Re-timer emits `CL_WAKE1.x`

The `SLOS1` triggers chain reaction where the Re-timers start to emit `CL_WAKE1.x` ordered sets.

```
===================================================================================================================
Adapter A        Retimer1-4          Retimer2-3          Retimer3-2          Retimer4-1          Adapter B
===================================================================================================================
 |                  |                   |                   |                   |                   |
 |                  |-- CL_WAKE1.4(L) → |                   |                   |                   |
 | ◀-- CL_WAKE1.1(L)|                   |                   |                   |                   |
 |                  |                   |-- CL_WAKE1.3(L) → |                   |                   |
 |                  |  ◀-- CL_WAKE1.2(L)|                   |                   |                   |
 |                  |                   |                   |-- CL_WAKE1.2(L) → |                   |
 |                  |                   |  ◀-- CL_WAKE1.3(L)|                   |                   |
 |                  |                   |                   |                   |-- CL_WAKE1.1(L) → |
 |                  |                   |                   | <-- CL_WAKE1.4(L) |                   |
 |                  |                   |                   |                   |                   |
 |                  |                   |                   |                   |  ◀-- CL_WAKE2.1(L)|
 |                  |                   |                   |                   |                   |
```

#### Routers bounce back with `CL_WAKE2.x`

On receive of 3 conesecutive `CL_WAKE1.x` order set, the Router starts to send the `CL_WAKE2.x` ordered set.

```
===================================================================================================================
Adapter A        Retimer1-4          Retimer2-3          Retimer3-2          Retimer4-1          Adapter B
===================================================================================================================
 |                  |                   |                   |                   |                   |
 |<-CL_WAKE1.1(1-4)-|                   |                   |                   |                   |
 |                  |                   |                   |                   |                   |
 |-- CL_WAKE2.1 --> |                   |                   |                   |--CL_WAKE1.1(4-1)->|
 |                  |                   |                   |                   |                   |
 |                  |                   |                   |                   | <--- CL_WAKE2.1-- |
 |                  |                   |                   |                   |                   |
```

#### Re-timers become transparent after receiving 3 `CL_WAKE2.x`

Once a Re-timer `x` receive 3 back-to-back `CL_WAKE2.x`, it no longer emits the `CL_WAKE1.x` ordered set. Rather, it pass through whatever traffic from that side.

#### Re-timers pass `CL_WAKE2.y` not targeting itself

If a Re-timer receives 2 back-to-back `CL_WAKE2.y` where `y` is not its own index, it will still let it through:

```
===================================================================================================================
Adapter A        Retimer1-4          Retimer2-3          Retimer3-2          Retimer4-1          Adapter B
===================================================================================================================
|----CL_WAKE2.2--->|                   |                   |                   |<---CL_WAKE2.2---- |
|                  |----CL_WAKE2.2---> |                   |<---CL_WAKE2.2---- |                   |
|                  |                   |                   |                   |                   |
```

The wording that the specification uses is "sending the last `CL_WAKE2` symbols received and 2 locally-generated symbols", but essentially this allows the `CL_WAKE2` targeting other Re-timers pass through this Re-timer.

#### The Goal

For Re-timers, the end goal of this is each Re-timer receive 3 `CL_WAKE2.x`, where `x` is its Re-timer index. The Re-timers needs those packages from the Router to recover the clocks. Both directions must achieve this.

### Retimer clock resume

Re-timer rely on ordered sets (`CL_WAKE2.x`)from Rounters to synchronize the clock.

#### `1` in the `1-4` and `4-1`

If a re-timer `x` receives certain amout of `CL_WAKE2.x` from one side (where `x` is the Re-timer number seen from that side), it allows traffic "pass through" from that side. In hypethetical example, this happens first to the `-1` side of the Re-timer `1-4` and Retimer `4-1`:

```
===================================================================================================================
Adapter A        Retimer1-4          Retimer2-3          Retimer3-2          Retimer4-1          Adapter B
===================================================================================================================
|--- CL_WAKE2.1 -->|                   |                   |                   | ◀-- CL_WAKE2.1 -- |
|                  |<                  |                   |                  >|                   |
|                  |<                  |                   |                  >|                   |
|                  |<                  |                   |                  >|                   |
```

At the same time, Re-timer `2-3` and Re-timer `3-2` are also emitting `CL_WAKE2.2` to Adapter A and Adapter B respectively. Once the Re-timer `1-4` and Re-timer `4-1` allow the pass-through of ordered sets from their `-1` side, those `CL_WAKE2.2` can now reach the `-2` side of the Re-timer `2-3` and Re-timer `3-2`:

This allows the `CL_WAKE_1.2` from the Re-timer `2-3` and Re-timer `3-2` pass through the `1-4` and `4-1`. When `CL_WAKE_1.2` reaches the Routers, the Router start to emit the `CL_WAKE2.2` as responses to `CL_WAKE_1.1`:

```
===================================================================================================================
Adapter A        Retimer1-4          Retimer2-3          Retimer3-2          Retimer4-1          Adapter B
===================================================================================================================
|                  |<---CL_WAKE1.2---- |                   | ----CL_WAKE1.2--->|                   |
|<---CL_WAKE1.2--- |<                  |                   |                  >| ---CL_WAKE1.2---->|
|----CL_WAKE2.2--->|<                  |                   |                  >|<---CL_WAKE2.2---- |
|                  |<                  |                   |                  >|                   |
|                  |<                  |                   |                  >|                   |
|                  |<                  |                   |                  >|                   |
```

When Re-timer `1-4` and Re-timer `4-1` receives enough of `CL_WAKE2.y` where `y` is a Re-tiemer index other than itself, it'll let that `CL_WAKE2.y` leak through, which brings this process to the Re-timers one layer deeper.

#### `2` in the `2-3` and `3-2`

Now this follows the simialar patterns. The Retimer `2-3` and `3-2` receives `CL_WAKES2.2`. This unblocks the `CL_WAKE1.3` ordered sets and allows it to reach the routers:

```
===================================================================================================================
Adapter A        Retimer1-4          Retimer2-3          Retimer3-2          Retimer4-1          Adapter B
===================================================================================================================
|  ---CL_WAKE2.2-->|<                  |                   |                  >|<--CL_WAKE2.2---  |
|                  |<                  |                   |                  >|                  |
|                  |<  --CL_WAKE2.2--->|                   |<--CL_WAKE2.2---  >|                  |
|                  |<                  |<                 >|                  >|                  |
|                  |<                  |<                 >|                  >|                  |
```

Once the `CL_WAKE2.3` reach the Router, the Routers bounce back with a `CL_WAKE2.3`:


```
===================================================================================================================
Adapter A        Retimer1-4          Retimer2-3          Retimer3-2          Retimer4-1          Adapter B
===================================================================================================================
|                  |<                  |<                 >|                  >|                   |
|                  |<                  |<--CL_WAKE1.3--   >|                  >|                   |
|                  |<                  |<  ---CL_WAKE1.3-->|                  >|                   |
|                  |<---CL_WAKE1.3---- |<                 >| ----CL_WAKE1.3--->|                   |
|<---CL_WAKE1.3--- |<                  |<                 >|                  >| ---CL_WAKE1.3---->|
|----CL_WAKE2.3--->|<                  |<                 >|                  >|<---CL_WAKE2.3---- |
```

#### `3` in the `2-3` and `3-2`

Now that the `-2` side of the Re-timer `2-3` and the Re-timer `3-2` are unlocked, `CL_WAKE2.3` can now reach the the `-3` side of Re-timer `2-3` and Re-timer `3-2`. They will wake up the `3` side of Re-timer `2-3` and Re-timer `3-2`:

```
Time ↓
===================================================================================================================
Adapter A        Retimer1-4          Retimer2-3          Retimer3-2          Retimer4-1          Adapter B
===================================================================================================================
|                  |<                  |<                 >|                  >|                  |
|--CL_WAKE2.3(A)-->|<                  |<                 >|                  >|<--CL_WAKE2.3(B)--|
|                  |<                  |<                 >|                  >|                  |
|                  |<--CL_WAKE2.3(A)-->|<                 >|<--CL_WAKE2.3(B)- >|                  |
|                  |<                  |<                 >|                  >|                  |
|                  |<                  |<--CL_WAKE2.3(B)-->|                  >|                  |
|                  |<                 >|<                 >|                  >|                  |
|                  |<                 >|<                 >|                  >|                  |
|                  |<                 >|<  CL_WAKE2.3(A)-->|<                 >|                  |
|                  |<                 >|<                 >|<                 >|                  |
|                  |<                 >|<                 >|<                 >|                  |
```

This allows the `CL_WAKE1.4` sent from Re-timer `4-1` reaches Adapter A, and the bounced back `CL_WAKE2.4` will be able to reache Re-timer `1-4`. Same for Re-timer `4-1`. (This process is not drawn.)

#### `4` in the `1-4` and `4-1`

Finally, the `-4` side of the Re-timer `1-4` and Re-timer `4-1` are woken up. This finishes process:

```
Time ↓
===================================================================================================================
Adapter A        Retimer1-4          Retimer2-3          Retimer3-2          Retimer4-1          Adapter B
===================================================================================================================
|                  |<                 >|<                 >|<                 >|                  |
|--CL_WAKE2.4(A)-->|<                 >|<                 >|<                 >|<--CL_WAKE2.4(B)--|
|                  |<                 >|<                 >|<                 >|                  |
|                  |<   CL_WAKE2.4(A)->|<                 >|<--CL_WAKE2.4(B)  >|                  |
|                  |<                 >|<                 >|>                 >|                  |
|                  |<                 >|<-CL_WAKE2.4(B)-- >|<                 >|                  |
|                  |<                 >|< --CL_WAKE2.4(A)->|<                 >|                  |
|                  |<                 >|<                 >|<                 >|                  |
|                  |<-CL_WAKE2.4(B)   >|<                 >|<  CL_WAKE2.4(A)-->|                  |
|                 >|<                 >|<                 >|<                 >|<                 |
|                 >|<                 >|<                 >|<                 >|<                 |
```

### Final stages

Once a a Router receives 7 back-to-back `CL_WAKE2` ordered set, it starts to send `SLOS1` instead. Now that traffic in both direction in the Re-timers are pass and their clocks are synced with the Routers, they become transparent. The `SLOS1` can pass through.

The rest of the re-training process then follows the case where there's no re-timer (`LOCK1`, `LOCK2`, `TS1`, `TS2`).
