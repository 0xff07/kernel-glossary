---
topics: pcie
tags:
    - "pcie"
    - "pcie-ltssm"
---

# `Configuration`

In this stage, the upstream component will try to figure out how many Links it's going to make out of all the trained Lanes in the Polling state, and group those Lanes into Links.

## Links and Lanes

This is achieved by exchanging the Lane Number and Link Number field by `TS1`.

The Link Number is negotiated first, by exchanging `TS1`, and then the Lane Number in each Link is determined again by exchanging `TS1`.

The Lanes are grouped by Link Numbers provided by the upstream component. Lanes belongs to the same Link Shares the same Link Number.

Each Lane within the same Link also has a unique Lane Number. Note that the Lane Number is unique only within a Link.

## `Configuration.Lanewidth`

The states in `Configuration.LaneWidth` are steps to determine how to partition existing Lanes into Links:

1. The upstream component first proposes a non-PAD Link Number, and
2. The downstream component could response with the same Link Number if it agrees with the proposed Link Number, or
3. Correct the upstream component by sending back `TS1` with proper Link Number.

## `Configuration.Lanenum`

After both side agree with the Link Number, the Lane Numbers of each Lane within that Link are determined.

The process is somewhat similar to that of the Link Number:

1. The upstream component first proposes a set of Lane Numbers, starting from `0` (let's say `0` to `n-1`), for Lanes within that Link, by populating the fields in the `TS1` sent to each Lane.
2. The downstream component can either agree with that Lane-Numbering scheme, by sending back `TS1`s with the same Lane-Number scheme, or
3. Correct the upstream component by filling the correct Lane Number in `TS1`. THe scheme proposed by downstream component also has to be consecutive numbers begin with 0.

Note that upstream component always expect consecutive Lane Numbers starting from 0 from the downstream component. This is helpful for detecting wrong bifurcation.

For example, if after sending `[0, 7]`, but instead of a re-ordering of `[0, 7]` upstream component receives `[0, 3]` and `[0, 3]` in Lane Numbering, this might be a suggestion that the downstream are 2 x4 Links, instead of a single x8 Link. 

Also note that downstream component could could propose using fewer Lanes than upstream component has proposed.

## References

### Links

- Configuration, [PCIe Deep Dive, Part 4: LTSSM](https://scolton.blogspot.com/2024/01/pcie-deep-dive-part-4-ltssm.html)

### Specification

- 4.2.6.3 Configuration
