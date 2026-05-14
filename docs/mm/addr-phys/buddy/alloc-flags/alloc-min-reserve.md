---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# ALLOC_MIN_RESERVE

[`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) is the page allocator's "high-priority caller" bit. It is the [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) twin of [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), since a [`BUILD_BUG_ON`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4488) in [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479) pins the two values to the same bit position so the GFP-side [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) can be OR'd straight into the ALLOC-side word without a per-bit translation. The flag tells [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3600) to cut the per-zone min watermark in half so the caller can dip into the bottom 50% of the reserve. It is also the gate that lets [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4509) set [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) for high-order non-blocking allocations, and the trigger that lets [`zone_watermark_fast()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3705) ignore [`zone->watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L884) on order-0 allocations.

```
        ALLOC_* bit field (12 bits live in v6.19)
        ─────────────────────────────────────────────────────────────
        bit:  11    10    9     8     7     6     5     4     3     2     1     0
              ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
              │  K  │  T  │ HA  │ NF  │ CMA │ CP  │ MR  │ NB  │ OOM │ NW  │  WMARK    │
              └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                                            ▲
                                            │
                                ALLOC_MIN_RESERVE = 0x20    (bit-aligned with __GFP_HIGH)

        Legend: K  = ALLOC_KSWAPD          NF  = ALLOC_NOFRAGMENT     NB  = ALLOC_NON_BLOCK
                T  = ALLOC_TRYLOCK         CP  = ALLOC_CPUSET         OOM = ALLOC_OOM
                HA = ALLOC_HIGHATOMIC      MR  = ALLOC_MIN_RESERVE    NW  = ALLOC_NO_WATERMARKS
```

```
        Watermark fraction ladder for ALLOC_MIN_RESERVE
        ────────────────────────────────────────────────────────────────

        zone->_watermark[WMARK_MIN] = M    (configured by min_free_kbytes)

                                                   __zone_watermark_ok min:

        no ALLOC_RESERVES bits                                M
                                                              ▲
                                                              │ enforced
                                                              │ floor
        ALLOC_MIN_RESERVE alone (__GFP_HIGH)                  M/2
        (`alloc_flags & ALLOC_MIN_RESERVE` -> min -= min/2)

        ALLOC_MIN_RESERVE | ALLOC_NON_BLOCK                   M/2 - M/8
        (also `if (alloc_flags & ALLOC_NON_BLOCK)             = 3M/8
            min -= min/4` of the reduced value)               (62.5% off)

        ALLOC_MIN_RESERVE | ALLOC_OOM                         (M/2) / 2
                                                              = M/4
                                                              (75% off)

        ALLOC_MIN_RESERVE | ALLOC_NON_BLOCK | ALLOC_OOM       (3M/8) / 2
                                                              = 3M/16
                                                              (~81% off)

        ALLOC_NO_WATERMARKS                                   bypass
                                                              (separate flag)
```

```
        Producer paths -> ALLOC_MIN_RESERVE
        ────────────────────────────────────────────────────────────────

          gfp_to_alloc_flags(gfp_mask, order):
              alloc_flags |= (gfp_mask & (__GFP_HIGH | __GFP_KSWAPD_RECLAIM))
                                            │
                  BUILD_BUG_ON(__GFP_HIGH != (__force gfp_t) ALLOC_MIN_RESERVE)
                                            │
                                            ▼
              if (!__GFP_DIRECT_RECLAIM && !__GFP_NOMEMALLOC) {
                  alloc_flags |= ALLOC_NON_BLOCK
                  if (order > 0 && (alloc_flags & ALLOC_MIN_RESERVE))
                      alloc_flags |= ALLOC_HIGHATOMIC
              }

              if (alloc_flags & ALLOC_MIN_RESERVE)
                  alloc_flags &= ~ALLOC_CPUSET     /* GFP_ATOMIC etc. */

              else if (rt_or_dl_task(current) && in_task())
                  alloc_flags |= ALLOC_MIN_RESERVE   /* RT in process ctx */

          __alloc_pages_slowpath() NOFAIL fallback (separate path):
              page = __alloc_pages_cpuset_fallback(
                  gfp_mask, order, ALLOC_MIN_RESERVE, ac);
              /* explicit, last-chance grant (NOT ALLOC_NO_WATERMARKS) */
```

## SUMMARY

[`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) (= `0x20`) is bit 5 of [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) and the [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) representation of [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h). [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4498) uses the bit-equality (asserted by [`BUILD_BUG_ON(__GFP_HIGH != (__force gfp_t) ALLOC_MIN_RESERVE)`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4488)) to copy the GFP bit into the [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) word with a single mask-AND-OR. A second producer site at [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4520) grants the bit to [`rt_or_dl_task()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/rt.h#L33) callers running [`in_task()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/preempt.h) on the direct-reclaim path so RT/DL tasks get the same reserve access as [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) callers without having to set the GFP bit.

The flag is part of the [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) bundle. [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3600) cuts the min watermark by 50% when [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) is set, and an inner check inside the same `if` block applies an additional 25% cut when [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) is also set, leaving the threshold at `3M/8` (62.5% off). The orthogonal [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) cut runs after this block and halves whatever remains.

[`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) plays three additional roles inside [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479). It gates [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) for non-blocking high-order allocations (`order > 0 && ALLOC_MIN_RESERVE`). It lets the same function clear [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) on non-blocking [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) requests (typically [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)), so cpuset isolation cannot starve the request. And it tells [`zone_watermark_fast()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3705) to ignore [`zone->watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L884) for order-0 allocations whose watermark index is [`WMARK_MIN`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L709), preserving the boost mechanism's role of waking kswapd without penalising urgent callers.

[`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4972) uses [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) as a deliberate substitute for [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) in its [`__GFP_NOFAIL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) escape hatch. According to the comment, granting full reserves to a NOFAIL caller could deplete the pool, so the path settles for the 50% cut instead.

## SPECIFICATIONS

(none; [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) is a Linux kernel internal flag)

## LINUX KERNEL

### Flag definition

- [`'\<ALLOC_MIN_RESERVE\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282): bit `0x20`. Comment: "`__GFP_HIGH` set. Allow access to 50% of the min watermark."
- [`'\<ALLOC_RESERVES\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297): the bundle macro that `__zone_watermark_*` test as a single condition. Members: [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278), [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282), [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292), [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273).
- [`'\<__GFP_HIGH\>':'include/linux/gfp_types.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): the GFP-side twin. The compile-time invariant [`__GFP_HIGH == ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4488) is enforced by [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479).

### Producers

- [`'\<gfp_to_alloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479): the primary producer. Line 4498 ORs `gfp_mask & __GFP_HIGH` straight into [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) using the bit-equality. Line 4520 grants [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) to [`rt_or_dl_task()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/rt.h#L33) callers in process context.
- [`'\<__alloc_pages_slowpath\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4972): the [`__GFP_NOFAIL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) escape hatch sets [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) explicitly when calling [`__alloc_pages_cpuset_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4034). The flag is hand-set rather than derived from `gfp_mask`, deliberately bypassing the producer chain.

### Consumers

- [`'\<__zone_watermark_ok\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3600): cuts `min` by 50% when [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) is set. The same `if` block additionally cuts the result by 25% when [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) is set.
- [`'\<zone_watermark_fast\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3705): order-0 [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) callers retry the watermark check ignoring [`zone->watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L884), so the boost mechanism never blocks them.
- [`'\<gfp_to_alloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4509): tests [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) to decide whether to set [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) (gated also on `order > 0`).
- [`'\<gfp_to_alloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4517): clears [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) for non-blocking [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) callers. Comment: "Ignore cpuset mems for non-blocking `__GFP_HIGH` (probably `GFP_ATOMIC`) rather than fail."
- [`'\<__zone_watermark_unusable_free\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567): the bundle test `!(alloc_flags & ALLOC_RESERVES)` includes [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282), so an [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) caller is allowed to count [`z->nr_free_highatomic`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) towards usable free pages.

### GFP mapping

- [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): the GFP-side bit copied verbatim into [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282). [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) (= [`__GFP_HIGH | __GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) is the most common caller path.
- [`__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): when absent, [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4505) also sets [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278); the resulting `MIN_RESERVE | NON_BLOCK` combination is the typical [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) shape that hits the 62.5% cut.
- [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): suppresses the [`ALLOC_NON_BLOCK | ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) escalation but NOT [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) itself; an explicit [`__GFP_HIGH | __GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) caller still gets the 50% min cut.
- [`__GFP_NOFAIL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): the dedicated NOFAIL-after-OOM path in [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4972) sets [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) explicitly, ignoring `gfp_mask`.

### Watermarks and reserves

- [`'\<wmark_pages\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1077): yields `mark = z->_watermark[w] + z->watermark_boost`. [`zone_watermark_fast()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3705) discards the boost component for [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) order-0 callers.
- [`'\<zone\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): the [`_watermark[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L883) array (indexed by [`enum zone_watermarks`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L708)) and [`watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L884) field are the targets of the cuts.
- [`'\<boost_watermark\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c): the function that updates [`watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L884) under [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294). [`zone_watermark_fast()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3705) ignores its result for high-priority callers.

### ALLOC_RESERVES bundle and watermark fractions

- [`'\<__zone_watermark_ok\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585): the only function that reads the bundle composition, with cuts that compound from [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) (50%) to [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) (additional 25%) to [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) (50% of the remainder).

### Related types and helpers

- [`'\<rt_or_dl_task\>':'include/linux/sched/rt.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/rt.h#L33): the predicate that grants [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) to RT/DL callers without [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h).
- [`'\<in_task\>':'include/linux/preempt.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/preempt.h): conjunct in the RT/DL grant; the bit is granted only in process context, not from softirq/hardirq.
- [`'\<__alloc_pages_cpuset_fallback\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4034): the helper that the [`__GFP_NOFAIL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) escape uses, called with a hand-set [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282).

## KERNEL DOCUMENTATION

- [`Documentation/admin-guide/sysctl/vm.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/admin-guide/sysctl/vm.rst): describes `min_free_kbytes` (the source of `_watermark[WMARK_MIN]`) and `watermark_boost_factor` (the boost that [`zone_watermark_fast()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3705) skips for [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) callers).
- [`Documentation/mm/physical_memory.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/physical_memory.rst): the per-zone watermark model.

## OTHER SOURCES

## DETAILS

### Bit equality with __GFP_HIGH

[`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479) opens with two compile-time invariants.

```c
static inline unsigned int
gfp_to_alloc_flags(gfp_t gfp_mask, unsigned int order)
{
	unsigned int alloc_flags = ALLOC_WMARK_MIN | ALLOC_CPUSET;

	/*
	 * __GFP_HIGH is assumed to be the same as ALLOC_MIN_RESERVE
	 * and __GFP_KSWAPD_RECLAIM is assumed to be the same as ALLOC_KSWAPD
	 * to save two branches.
	 */
	BUILD_BUG_ON(__GFP_HIGH != (__force gfp_t) ALLOC_MIN_RESERVE);
	BUILD_BUG_ON(__GFP_KSWAPD_RECLAIM != (__force gfp_t) ALLOC_KSWAPD);
```

The "save two branches" comment refers to the next line.

```c
	alloc_flags |= (__force int)
		(gfp_mask & (__GFP_HIGH | __GFP_KSWAPD_RECLAIM));
```

Without the bit-equality invariants, this single OR would have to be split into two `if (gfp_mask & __GFP_HIGH) alloc_flags |= ALLOC_MIN_RESERVE;` style branches plus the analogous pair for [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294). The bit-equality keeps the fast path branchless. The same equivalence is what allows [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843) to merge `reserve_flags` directly into [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) without per-bit translation.

### Producer: gfp_to_alloc_flags

The full producer is short enough to read in one block.

```c
static inline unsigned int
gfp_to_alloc_flags(gfp_t gfp_mask, unsigned int order)
{
	unsigned int alloc_flags = ALLOC_WMARK_MIN | ALLOC_CPUSET;

	BUILD_BUG_ON(__GFP_HIGH != (__force gfp_t) ALLOC_MIN_RESERVE);
	BUILD_BUG_ON(__GFP_KSWAPD_RECLAIM != (__force gfp_t) ALLOC_KSWAPD);

	/*
	 * The caller may dip into page reserves a bit more if the caller
	 * cannot run direct reclaim, or if the caller has realtime scheduling
	 * policy or is asking for __GFP_HIGH memory.  GFP_ATOMIC requests will
	 * set both ALLOC_NON_BLOCK and ALLOC_MIN_RESERVE(__GFP_HIGH).
	 */
	alloc_flags |= (__force int)
		(gfp_mask & (__GFP_HIGH | __GFP_KSWAPD_RECLAIM));

	if (!(gfp_mask & __GFP_DIRECT_RECLAIM)) {
		/*
		 * Not worth trying to allocate harder for __GFP_NOMEMALLOC even
		 * if it can't schedule.
		 */
		if (!(gfp_mask & __GFP_NOMEMALLOC)) {
			alloc_flags |= ALLOC_NON_BLOCK;

			if (order > 0 && (alloc_flags & ALLOC_MIN_RESERVE))
				alloc_flags |= ALLOC_HIGHATOMIC;
		}

		/*
		 * Ignore cpuset mems for non-blocking __GFP_HIGH (probably
		 * GFP_ATOMIC) rather than fail, see the comment for
		 * cpuset_current_node_allowed().
		 */
		if (alloc_flags & ALLOC_MIN_RESERVE)
			alloc_flags &= ~ALLOC_CPUSET;
	} else if (unlikely(rt_or_dl_task(current)) && in_task())
		alloc_flags |= ALLOC_MIN_RESERVE;

	alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, alloc_flags);

	if (defrag_mode)
		alloc_flags |= ALLOC_NOFRAGMENT;

	return alloc_flags;
}
```

Three of the four [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) interactions on this page live inside this function. The bit is set in two complementary cases (the GFP copy and the RT/DL grant) and read in two cases (the [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) gate and the [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) clear). The two branches of the `if (!(gfp_mask & __GFP_DIRECT_RECLAIM))` block are mutually exclusive. A non-direct-reclaim caller falls into the first branch and never reaches the RT/DL grant, while a direct-reclaim-capable RT/DL caller falls into the second branch and gets [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) added to whatever the GFP copy already produced (which is `0` for RT/DL kernel callers without [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)).

### The ALLOC_HIGHATOMIC gate

[`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) is set only when both `order > 0` and [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) is set.

```c
		if (!(gfp_mask & __GFP_NOMEMALLOC)) {
			alloc_flags |= ALLOC_NON_BLOCK;

			if (order > 0 && (alloc_flags & ALLOC_MIN_RESERVE))
				alloc_flags |= ALLOC_HIGHATOMIC;
		}
```

The combined precondition selects high-order non-blocking [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) callers ([`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) for `order > 0`, plus [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) opt-out). These are exactly the callers whose failure mode is an interrupt-time crash, so the [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) reserve is opened up on their behalf. [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) acts as the "I am important" gate; the order check picks the subset for which an external reserve pool actually exists.

### The ALLOC_CPUSET clear

The same producer also strips [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) from the non-blocking [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) shape.

```c
		/*
		 * Ignore cpuset mems for non-blocking __GFP_HIGH (probably
		 * GFP_ATOMIC) rather than fail, see the comment for
		 * cpuset_current_node_allowed().
		 */
		if (alloc_flags & ALLOC_MIN_RESERVE)
			alloc_flags &= ~ALLOC_CPUSET;
```

A [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) caller in interrupt context cannot wait, cannot block, and is often the kernel itself. Insisting on cpuset isolation for these callers risks failing the allocation when there are pages on neighbouring nodes; the [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) check is used here as a proxy for "this caller is too important to fail for a cpuset-policy reason."

### The RT/DL grant

The else-branch handles direct-reclaim-capable RT/DL tasks.

```c
	} else if (unlikely(rt_or_dl_task(current)) && in_task())
		alloc_flags |= ALLOC_MIN_RESERVE;
```

[`rt_or_dl_task()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/rt.h#L33) checks `p->prio` for RT or DL priority class.

```c
static inline bool rt_or_dl_task(struct task_struct *p)
{
	return rt_or_dl_prio(p->prio);
}
```

A real-time or deadline task running in process context (`in_task()`) gets the same 50% min cut as a [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) caller, on the grounds that an RT/DL task missing its deadline because of memory pressure is a worse outcome than dipping into reserves. The grant is conditional on direct-reclaim capability so it does not stack with the non-blocking branch above; an RT [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) caller takes the first branch and gets the bit through the [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) copy, not through this RT/DL one.

### Watermark cut: __zone_watermark_ok

[`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585) handles [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) inside the [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) bundle.

```c
	if (unlikely(alloc_flags & ALLOC_RESERVES)) {
		/*
		 * __GFP_HIGH allows access to 50% of the min reserve as well
		 * as OOM.
		 */
		if (alloc_flags & ALLOC_MIN_RESERVE) {
			min -= min / 2;

			/*
			 * Non-blocking allocations (e.g. GFP_ATOMIC) can
			 * access more reserves than just __GFP_HIGH. Other
			 * non-blocking allocations requests such as GFP_NOWAIT
			 * or (GFP_KERNEL & ~__GFP_DIRECT_RECLAIM) do not get
			 * access to the min reserve.
			 */
			if (alloc_flags & ALLOC_NON_BLOCK)
				min -= min / 4;
		}

		/*
		 * OOM victims can try even harder than the normal reserve
		 * users on the grounds that it's definitely going to be in
		 * the exit path shortly and free memory. Any allocation it
		 * makes during the free path will be small and short-lived.
		 */
		if (alloc_flags & ALLOC_OOM)
			min -= min / 2;
	}
```

Starting from `M = wmark_pages(zone, alloc_flags & ALLOC_WMARK_MASK)`, bare [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) yields `M -> M/2` (50% off), [`ALLOC_MIN_RESERVE | ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) yields `M/2 -> M/2 - (M/2)/4 = 3M/8` (62.5% off, the [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) shape), and [`ALLOC_MIN_RESERVE | ALLOC_NON_BLOCK | ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) yields `3M/8 -> 3M/16` (~81% off).

Note the inner [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) cut is gated on [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) being set. A bare [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) (e.g. [`GFP_NOWAIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) gets no cut at all from the bundle path. According to the comment in the function, "GFP_NOWAIT or (GFP_KERNEL & ~__GFP_DIRECT_RECLAIM) do not get access to the min reserve."

### Boost ignore: zone_watermark_fast

[`zone_watermark_fast()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3670) is the fast-path watermark probe called from [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3791). After the order-0 fast check and the regular [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585) call, it has one more retry for [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) callers.

```c
	if (__zone_watermark_ok(z, order, mark, highest_zoneidx, alloc_flags,
					free_pages))
		return true;

	/*
	 * Ignore watermark boosting for __GFP_HIGH order-0 allocations
	 * when checking the min watermark. The min watermark is the
	 * point where boosting is ignored so that kswapd is woken up
	 * when below the low watermark.
	 */
	if (unlikely(!order && (alloc_flags & ALLOC_MIN_RESERVE) && z->watermark_boost
		&& ((alloc_flags & ALLOC_WMARK_MASK) == WMARK_MIN))) {
		mark = z->_watermark[WMARK_MIN];
		return __zone_watermark_ok(z, order, mark, highest_zoneidx,
					alloc_flags, free_pages);
	}

	return false;
}
```

The condition fires only for order-0 allocations with [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) set, when [`zone->watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L884) is non-zero, and when the requested watermark is [`WMARK_MIN`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L709). The probe then recomputes `mark` from [`z->_watermark[WMARK_MIN]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L883) directly (without adding [`watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L884)) and re-runs [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585). The [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) cut still applies inside [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585), so the effective threshold for an [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) order-0 caller hitting a boosted zone is `_watermark[WMARK_MIN] / 2`.

The rationale is that the boost mechanism's purpose is to wake kswapd more aggressively under fragmentation pressure; if the boost itself blocked an urgent caller, the system would defeat its own goal.

### NOFAIL escape: __alloc_pages_slowpath

[`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4948) reaches a final NOFAIL escape after every other path has failed.

```c
	if (unlikely(nofail)) {
		/*
		 * Lacking direct_reclaim we can't do anything to reclaim memory,
		 * we disregard these unreasonable nofail requests and still
		 * return NULL
		 */
		if (!can_direct_reclaim)
			goto fail;

		/*
		 * Help non-failing allocations by giving some access to memory
		 * reserves normally used for high priority non-blocking
		 * allocations but do not use ALLOC_NO_WATERMARKS because this
		 * could deplete whole memory reserves which would just make
		 * the situation worse.
		 */
		page = __alloc_pages_cpuset_fallback(gfp_mask, order, ALLOC_MIN_RESERVE, ac);
		if (page)
			goto got_pg;

		cond_resched();
		goto retry;
	}
```

The hand-set [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) is deliberately the weaker grant. According to the comment, a NOFAIL caller is asking the allocator to spin until it succeeds, so granting the full bypass would let one bad caller drain the reserves. The 50% min cut keeps some headroom for parallel reclaim-driving paths that might come through with [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262).

### Interaction with ALLOC_NON_BLOCK and ALLOC_OOM

[`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) sits at the top of an "importance ladder" inside [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297). Alone, it is the bare [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) grant and takes 50% off the watermark. Adding [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) (the [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) shape) takes the cut to 62.5%; without [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282), [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) gets no cut at all from the bundle path, since the inner `if (alloc_flags & ALLOC_NON_BLOCK)` only runs inside the outer [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) `if`. Adding [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) takes whatever remained and halves it again. [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) is also a bundle member but does not contribute to the watermark cut directly; its effect runs through [`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567) (suppressing the `nr_free_highatomic` subtraction) and the per-order check at [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3662) (accepting [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h)).

The dependency of [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) on [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) for the watermark cut is enforced by the producer too. [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4505) sets [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) only when [`__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) is absent (and not [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)), which is the same shape that typically also carries [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) for [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h). A non-blocking caller without [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) (e.g. [`GFP_NOWAIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) gets only [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278), which is in the bundle but does not by itself trigger any cut, and is documented on its own page.
