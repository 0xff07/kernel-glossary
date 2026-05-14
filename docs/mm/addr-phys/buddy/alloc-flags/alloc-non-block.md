---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# ALLOC_NON_BLOCK

[`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) is the page allocator's "this caller cannot block" bit. It is set by [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4507) for any caller that lacks [`__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) and has not opted out with [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h). The flag participates in the [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) bundle and gives non-blocking callers two practical concessions. They get an additional 25% cut on top of the [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) 50% cut inside [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3617) (yielding the [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) 62.5% reduction), and they get a last-chance pull from the [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) reserve in [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3250) when the regular [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) returns NULL. Bare [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) (e.g. [`GFP_NOWAIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) buys neither the 25% cut (gated on [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282)) nor any extra freelist behaviour beyond bundle membership, and the bundle membership itself only suppresses the `nr_free_highatomic` subtraction in [`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567).

```
        ALLOC_* bit field (12 bits live in v6.19)
        ─────────────────────────────────────────────────────────────
        bit:  11    10    9     8     7     6     5     4     3     2     1     0
              ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
              │  K  │  T  │ HA  │ NF  │ CMA │ CP  │ MR  │ NB  │ OOM │ NW  │  WMARK    │
              └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                                                  ▲
                                                  │
                                       ALLOC_NON_BLOCK = 0x10

        Legend: K  = ALLOC_KSWAPD          NF  = ALLOC_NOFRAGMENT     NB  = ALLOC_NON_BLOCK
                T  = ALLOC_TRYLOCK         CP  = ALLOC_CPUSET         OOM = ALLOC_OOM
                HA = ALLOC_HIGHATOMIC      MR  = ALLOC_MIN_RESERVE    NW  = ALLOC_NO_WATERMARKS
```

```
        Watermark fraction ladder for ALLOC_NON_BLOCK
        ────────────────────────────────────────────────────────────────

        zone->_watermark[WMARK_MIN] = M

        ALLOC_NON_BLOCK alone (GFP_NOWAIT)              M
        (in ALLOC_RESERVES but no per-bit cut)

        ALLOC_NON_BLOCK | ALLOC_MIN_RESERVE             M/2 - M/8  = 3M/8
        (this is the GFP_ATOMIC shape;                    (62.5% off)
         outer 50% cut from MIN_RESERVE,
         inner 25% cut from NON_BLOCK)

        same + ALLOC_OOM                                 3M/16
                                                          (~81% off)
```

```
        Producer + last-chance pull
        ────────────────────────────────────────────────────────────────

          gfp_to_alloc_flags(gfp_mask, order):
              if (!(gfp_mask & __GFP_DIRECT_RECLAIM)) {
                  if (!(gfp_mask & __GFP_NOMEMALLOC)) {
                      alloc_flags |= ALLOC_NON_BLOCK;        <── set
                      if (order > 0 && (alloc_flags & ALLOC_MIN_RESERVE))
                          alloc_flags |= ALLOC_HIGHATOMIC;
                  }
                  ...
              }

          rmqueue_buddy(zone, order, alloc_flags, migratetype):
              spin_lock_irqsave(&zone->lock, flags);
              page = NULL;
              if (alloc_flags & ALLOC_HIGHATOMIC)
                      page = __rmqueue_smallest(zone, order, MIGRATE_HIGHATOMIC);
              if (!page) {
                      page = __rmqueue(zone, order, migratetype, ...);
                      if (!page && (alloc_flags & (ALLOC_OOM|ALLOC_NON_BLOCK)))
                              page = __rmqueue_smallest(zone, order,
                                                        MIGRATE_HIGHATOMIC);
                                                  <── last-chance for
                                                      non-blocking callers
              }

          __zone_watermark_unusable_free(z, order, alloc_flags):
              long unusable_free = (1 << order) - 1;
              if (likely(!(alloc_flags & ALLOC_RESERVES)))
                      unusable_free += READ_ONCE(z->nr_free_highatomic);
              /* ALLOC_NON_BLOCK is in ALLOC_RESERVES, so the high-atomic
                 reserve counts as usable for the threshold check. */
```

## SUMMARY

[`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) (= `0x10`) is bit 4 of [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c). The producer is [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4507), which sets the bit whenever `!(gfp_mask & __GFP_DIRECT_RECLAIM) && !(gfp_mask & __GFP_NOMEMALLOC)`. That covers [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) (= [`__GFP_HIGH | __GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)), [`GFP_NOWAIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) (= [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) only), interrupt-context allocations, and any caller that strips [`__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) by hand.

The watermark cut is asymmetric. Inside [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3617), [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) only contributes when [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) is also set. In that combined case ([`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)), the outer [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) cuts `min` to `M/2`, and the inner [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) cut takes another 25%, leaving `3M/8`. A bare [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) caller (typical [`GFP_NOWAIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) is in the [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) bundle for the purposes of [`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567), but the watermark threshold itself is not cut.

The [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3250) consumer is the more concrete benefit. After the regular migrate-type fallback chain in [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) has returned NULL, the function pulls from [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) for callers that hold either [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) or [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278). The rationale (per the in-tree comment) is that failing an order-0 [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) now is worse than failing a future high-order atomic, so the reserve is opened up for the non-blocking caller.

[`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) has no separate clearing site. It enters the slowpath via [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479) on the first try and is OR-rewritten on retry only when [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) returns a non-zero `reserve_flags`, which overwrites [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) with the reserve grant plus carry-over of [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) only.

## SPECIFICATIONS

(none; [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) is a Linux kernel internal flag)

## LINUX KERNEL

### Flag definition

- [`'\<ALLOC_NON_BLOCK\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278): bit `0x10`. Comment: "Caller cannot block. Allow access to 25% of the min watermark or 62.5% if `__GFP_HIGH` is set."
- [`'\<ALLOC_RESERVES\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297): the bundle macro. Members: [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278), [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282), [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292), [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273).

### Producers

- [`'\<gfp_to_alloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4507): the only call site that sets [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278). Triggered when `!(gfp_mask & __GFP_DIRECT_RECLAIM) && !(gfp_mask & __GFP_NOMEMALLOC)`.

### Consumers

- [`'\<__zone_watermark_ok\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3617): cuts `min` by an additional 25% when [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) is set; gated on the outer [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) `if`.
- [`'\<rmqueue_buddy\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3250): pulls from [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) via [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) when [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) returns NULL and either [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) or [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) is set.
- [`'\<__zone_watermark_unusable_free\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567): bundle membership in [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) skips the `unusable_free += z->nr_free_highatomic` subtraction.

### GFP mapping

- [`__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): when present, [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) is NOT set. This is the gating bit.
- [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): when set, suppresses [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) (and [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) by transitivity). The producer comment is "Not worth trying to allocate harder for `__GFP_NOMEMALLOC` even if it can't schedule."
- [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): orthogonal. Sets [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282), which is the gate for the inner [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) watermark cut.
- [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): the canonical [`ALLOC_NON_BLOCK | ALLOC_MIN_RESERVE | ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) shape.
- [`GFP_NOWAIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): [`ALLOC_NON_BLOCK | ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) only; bundle member but no watermark cut.

### ALLOC_RESERVES bundle and watermark fractions

- [`'\<__zone_watermark_ok\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585): the only function whose threshold arithmetic depends on the bundle composition. The inner [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) `if` lives inside the outer [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) `if`, so a bare [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) caller takes no per-bit cut. Combined with [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) (which is outside the [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) `if`), the OOM cut still applies on top.

### Migratetype and freelists

- [`'\<__rmqueue\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473): the regular migrate-type fallback walker. Returns NULL when none of [`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h)/[`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h)/[`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h)/[`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) (when [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286)) hold a free block of the requested order.
- [`'\<__rmqueue_smallest\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914): the targeted single-migratetype walker that [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3250) calls with `migratetype = MIGRATE_HIGHATOMIC` for non-blocking callers.
- [`'\<MIGRATE_HIGHATOMIC\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): the reserve pool that [`reserve_highatomic_pageblock()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3427) populates and that [`unreserve_highatomic_pageblock()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3479) drains under pressure.

## KERNEL DOCUMENTATION

- [`Documentation/admin-guide/sysctl/vm.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/admin-guide/sysctl/vm.rst): describes the `min_free_kbytes` watermark from which the [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) cut is calculated.
- [`Documentation/mm/physical_memory.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/physical_memory.rst): the per-zone watermark model.

## OTHER SOURCES

## DETAILS

### Producer site

[`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479) is the only producer. The relevant block.

```c
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
```

The outer test gates the entire non-blocking branch on the absence of [`__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h). Inside, the [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) opt-out is honoured early so a caller that explicitly does not want reserve access is excluded. The remainder sets [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) unconditionally and conditionally upgrades to [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) when the caller is also high-priority and high-order. The [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) clear is unrelated to [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) (it is gated on [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282)) but lives in the same block.

[`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) = [`__GFP_HIGH | __GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) yields [`ALLOC_NON_BLOCK | ALLOC_MIN_RESERVE | ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) (plus [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) for `order > 0`). [`GFP_NOWAIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) = [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) only yields [`ALLOC_NON_BLOCK | ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h). Hard interrupt context with any non-blocking GFP yields the same shape as one of the above (plus or minus [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) for [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)). The fourth shape is hand-stripped [`__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), which is rare and shows up in some lockdep-sensitive paths.

### Consumer: __zone_watermark_ok

The arithmetic is in the [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) block.

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

The structural detail to note is that the inner `if (alloc_flags & ALLOC_NON_BLOCK)` lives inside the outer `if (alloc_flags & ALLOC_MIN_RESERVE)`. A bare [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) caller (like [`GFP_NOWAIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) reaches the outer [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) gate, falls past the inner [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) `if` (no cut), and either falls past the [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) `if` too (no cut) or takes the OOM-only 50% cut. The "62.5% off" headline number applies only to the [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) shape.

Starting from `M = wmark_pages(zone, alloc_flags & ALLOC_WMARK_MASK)`, [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) alone yields `M` with no cut, [`ALLOC_NON_BLOCK | ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) ([`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) yields `M -> M/2 -> M/2 - (M/2)/4 = 3M/8` (62.5% off), [`ALLOC_NON_BLOCK | ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) yields `M -> M/2` (50% off, OOM cut only), and [`ALLOC_NON_BLOCK | ALLOC_MIN_RESERVE | ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) yields `3M/8 -> 3M/16` (about 81% off).

### Consumer: rmqueue_buddy last-chance pull

[`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) is the slow buddy-list path inside [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c). Its body in full.

```c
page *rmqueue_buddy(struct zone *preferred_zone, struct zone *zone,
			   unsigned int order, unsigned int alloc_flags,
			   int migratetype)
{
	struct page *page;
	unsigned long flags;

	do {
		page = NULL;
		if (unlikely(alloc_flags & ALLOC_TRYLOCK)) {
			if (!spin_trylock_irqsave(&zone->lock, flags))
				return NULL;
		} else {
			spin_lock_irqsave(&zone->lock, flags);
		}
		if (alloc_flags & ALLOC_HIGHATOMIC)
			page = __rmqueue_smallest(zone, order, MIGRATE_HIGHATOMIC);
		if (!page) {
			enum rmqueue_mode rmqm = RMQUEUE_NORMAL;

			page = __rmqueue(zone, order, migratetype, alloc_flags, &rmqm);

			/*
			 * If the allocation fails, allow OOM handling and
			 * order-0 (atomic) allocs access to HIGHATOMIC
			 * reserves as failing now is worse than failing a
			 * high-order atomic allocation in the future.
			 */
			if (!page && (alloc_flags & (ALLOC_OOM|ALLOC_NON_BLOCK)))
				page = __rmqueue_smallest(zone, order, MIGRATE_HIGHATOMIC);

			if (!page) {
				spin_unlock_irqrestore(&zone->lock, flags);
				return NULL;
			}
		}
		spin_unlock_irqrestore(&zone->lock, flags);
	} while (check_new_pages(page, order));

	__count_zid_vm_events(PGALLOC, page_zonenum(page), 1 << order);
	zone_statistics(preferred_zone, zone, 1);

	return page;
}
```

The order of operations matters. A high-order [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) caller hits the [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) branch first and gets the dedicated reserve immediately. A non-blocking caller without [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) (typically [`GFP_NOWAIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) or [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) at order 0) falls through the regular [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) walker; only when it returns NULL does the [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) test trigger the [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) reach.

[`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) is the buddy-list walker that scans `zone->free_area[order..NR_PAGE_ORDERS]` for a free block of the given migrate type.

```c
page *__rmqueue_smallest(struct zone *zone, unsigned int order,
						int migratetype)
{
	unsigned int current_order;
	struct free_area *area;
	struct page *page;

	/* Find a page of the appropriate size in the preferred list */
	for (current_order = order; current_order < NR_PAGE_ORDERS; ++current_order) {
		area = &(zone->free_area[current_order]);
		page = get_page_from_free_area(area, migratetype);
		if (!page)
			continue;

		page_del_and_expand(zone, page, order, current_order,
				    migratetype);
		trace_mm_page_alloc_zone_locked(page, order, migratetype,
				pcp_allowed_order(order) &&
				migratetype < MIGRATE_PCPTYPES);
		return page;
	}

	return NULL;
}
```

The migrate-type argument is hard-coded to [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) by the [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3250) call site, so the walker only inspects the high-atomic free lists. The [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) pool itself is populated by [`reserve_highatomic_pageblock()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3427), called from [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3791) immediately after a successful [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) allocation.

### Consumer: __zone_watermark_unusable_free

[`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) is part of the [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) bundle, so it influences the unusable-free computation indirectly.

```c
static inline long __zone_watermark_unusable_free(struct zone *z,
				unsigned int order, unsigned int alloc_flags)
{
	long unusable_free = (1 << order) - 1;

	/*
	 * If the caller does not have rights to reserves below the min
	 * watermark then subtract the free pages reserved for highatomic.
	 */
	if (likely(!(alloc_flags & ALLOC_RESERVES)))
		unusable_free += READ_ONCE(z->nr_free_highatomic);

#ifdef CONFIG_CMA
	/* If allocation can't use CMA areas don't use free CMA pages */
	if (!(alloc_flags & ALLOC_CMA))
		unusable_free += zone_page_state(z, NR_FREE_CMA_PAGES);
#endif

	return unusable_free;
}
```

For an [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) caller, the `if (likely(!(alloc_flags & ALLOC_RESERVES)))` condition is false, so [`z->nr_free_highatomic`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is NOT subtracted. The high-atomic reserve counts as available free memory for the watermark check, which is consistent with the fact that [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3250) will let the same caller pull from it if [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) returns NULL.

### GFP_NOWAIT joins the bundle without a watermark cut

The placement of [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) in [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) without a per-bit cut is deliberate. The bundle test gates the unusable-free subtraction and the high-order [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) acceptance in [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3662) for [`ALLOC_HIGHATOMIC | ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) callers. A bare [`GFP_NOWAIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) caller does not need the watermark cut because it is supposed to be a soft probe (the caller deals with allocation failure cleanly), but it still benefits from the bundle membership because allowing the high-atomic reserve to count as free improves its chance of passing the watermark.

According to the function comment, "GFP_NOWAIT or (GFP_KERNEL & ~__GFP_DIRECT_RECLAIM) do not get access to the min reserve", which makes the split between "in the bundle" and "gets a cut" explicit. [`GFP_NOWAIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) is supposed to fail cheaply when memory is tight, and only [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) callers (i.e. [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) earn the deeper reach.

### Interaction with ALLOC_HIGHATOMIC

[`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4509) sets [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) only when both `order > 0` and [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) is already set inside the same producer, which is itself inside the [`!__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) branch that sets [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278). The combined precondition is therefore "high-order, non-blocking, [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), no [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)" (i.e. [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) at `order > 0`).

When that combination holds, [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) carries [`ALLOC_NON_BLOCK | ALLOC_MIN_RESERVE | ALLOC_HIGHATOMIC | ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) into [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3791). Inside [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3231), the [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) branch fires first and pulls from the reserve directly, so the [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) last-chance branch only matters for the order-0 case (where [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) was not set).

### Slowpath retry behaviour

[`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843) recomputes [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) on retry. The recomputation overwrites the old value.

```c
	reserve_flags = __gfp_pfmemalloc_flags(gfp_mask);
	if (reserve_flags)
		alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, reserve_flags) |
					  (alloc_flags & ALLOC_KSWAPD);
```

If [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) returns [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) or [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273), [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) is rewritten to that value (plus optional [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) and the carried-over [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294)), and [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) is dropped. A caller that started with [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) and reached the OOM path therefore loses its 25% inner cut on the OOM retry, and the new [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) sees only [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273), which gives a 50% min cut on its own.

The drop is consistent with the producer-side intent. The [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) early return shows that the kernel does not want to "try harder" for non-blocking callers in slow paths. By the time the slowpath reaches OOM, the caller is already past the cheap-fail boundary, and the deeper reserve grant supersedes the non-blocking discount.
