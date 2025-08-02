---
topics: pcie
tags:
    - "pcie"
    - "pcie-ltssm"
---

# Polling

The goal is to find out lanes that are at lease capable of exchanging training sequences, by repeatedly exchanging `TS1` and then `TS2`, and see response from the other side.

The `Polling` state only makes sure each Lane is functional, but doesn't group them into Links. At the end of `Polling` state, Lanes still haven't been bonded into Links. Grouping Lanes into Links happens dynamically in the `Configuration` state of the LTSSM (which is the next state after `Polling` if things goes normally).

## `Polling.Active`

`Polling` begins from each Lane sending 1024 `TS1`s, in the meanwhile receiving `TS1` from the other side of a Lane, basically trying to confirm that the link partner has recovered the clock and then achieve symbol lock.

Once all Lanes receives enough of `TS1`, or start seeing the link partner sending `TS2`, the LTSSM is ready to enter the Polling.Configuration state.

### `Polling.Configuration`

in `Polling.Configuration` state, both sides of the Lanes start sending `TS2` instead. Again, once certain amount of `TS2`s are successfully sent and received, the link training moves on to the `Polling` state.

## References

### Videos

- [PCIE Protocol - LTSSM Detailed discussion (00:08:27)](https://youtu.be/XN85E0h_1SE?si=wMavYNstoirObp73&t=507)

### Links

- [PCIe Deep Dive, Part 4: LTSSM](https://scolton.blogspot.com/2024/01/pcie-deep-dive-part-4-ltssm.html)

### Specification

- 4.2.4.13 Lane vs. Link Training
- 4.2.6.2 Polling

