---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# ALLOC_HIGHATOMIC

[`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) is the page allocator's "high-order atomic" bit. It tells [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3237) to pull from the dedicated [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) freelist before walking the regular migrate-type fallback chain, and tells [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3952) to call [`reserve_highatomic_pageblock()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3427) immediately after a successful allocation so that future high-order atomic callers find a populated reserve. The flag is set only by [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4509) and only when both `order > 0` and [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) are already true (i.e. the high-order [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) shape). It is part of the [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) bundle and participates in the [`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567) bundle test, but contributes no per-bit watermark cut on its own.

```
        ALLOC_* bit field (12 bits live in v6.19)
        ─────────────────────────────────────────────────────────────
        bit:  11    10    9     8     7     6     5     4     3     2     1     0
              ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
              │  K  │  T  │ HA  │ NF  │ CMA │ CP  │ MR  │ NB  │ OOM │ NW  │  WMARK    │
              └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                                ▲
                                │
                  ALLOC_HIGHATOMIC = 0x200

        Legend: K  = ALLOC_KSWAPD          NF  = ALLOC_NOFRAGMENT     NB  = ALLOC_NON_BLOCK
                T  = ALLOC_TRYLOCK         CP  = ALLOC_CPUSET         OOM = ALLOC_OOM
                HA = ALLOC_HIGHATOMIC      MR  = ALLOC_MIN_RESERVE    NW  = ALLOC_NO_WATERMARKS
```

```
        MIGRATE_HIGHATOMIC pool lifecycle
        ────────────────────────────────────────────────────────────────

         (1) RESERVATION (first successful order>0 GFP_ATOMIC alloc)

             get_page_from_freelist:
                 page = rmqueue(zonelist_zone(...), zone, order,
                                gfp_mask, alloc_flags, ...)
                 if (page) {
                         prep_new_page(page, order, gfp_mask, alloc_flags);
                         if (unlikely(alloc_flags & ALLOC_HIGHATOMIC))
                                 reserve_highatomic_pageblock(page, order, zone);
                         return page;
                 }

             reserve_highatomic_pageblock:
                 max_managed = ALIGN(zone_managed_pages(zone)/100,
                                     pageblock_nr_pages)
                 (cap: ~1% of zone, minimum 1 pageblock)

                 if (zone->nr_reserved_highatomic >= max_managed) return;

                 spin_lock(&zone->lock);
                 if (order < pageblock_order)
                         move_freepages_block(zone, page, mt,
                                              MIGRATE_HIGHATOMIC)
                         zone->nr_reserved_highatomic += pageblock_nr_pages;
                 else
                         change_pageblock_range(page, order, MIGRATE_HIGHATOMIC)
                         zone->nr_reserved_highatomic += 1 << order;

         (2) CONSUMPTION (subsequent order>0 GFP_ATOMIC alloc)

             rmqueue_buddy:
                 spin_lock(&zone->lock);
                 if (alloc_flags & ALLOC_HIGHATOMIC)
                         page = __rmqueue_smallest(zone, order,
                                                   MIGRATE_HIGHATOMIC);
                 if (!page) {
                         page = __rmqueue(zone, order, migratetype, ...)
                         if (!page && (alloc_flags & (ALLOC_OOM|ALLOC_NON_BLOCK)))
                                 page = __rmqueue_smallest(zone, order,
                                                           MIGRATE_HIGHATOMIC);
                 }

         (3) DRAIN (regular allocator under pressure)

             __alloc_pages_direct_reclaim:
                 if (!page && drained == false) {
                         unreserve_highatomic_pageblock(ac, force=false);
                                                 /* preserve at least one
                                                    pageblock unless ... */
                         drained = true;
                         goto retry;
                 }

         (4) DRAIN (last-chance before OOM)

             should_reclaim_retry:
                 if (!ret)  /* no zone can satisfy even with full reclaim */
                         return unreserve_highatomic_pageblock(ac, force=true);

             unreserve_highatomic_pageblock(ac, force):
                 for each zone in zonelist:
                     for order in 0..NR_PAGE_ORDERS:
                         page = get_page_from_free_area(area,
                                                        MIGRATE_HIGHATOMIC)
                         move_freepages_block(zone, page,
                                              MIGRATE_HIGHATOMIC,
                                              ac->migratetype)
                         zone->nr_reserved_highatomic -= size
                         return true
```

## SUMMARY

[`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) (= `0x200`) is bit 9 of [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c). The producer is the conditional inside [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4509), and the bit is set only when the request is high-order, non-blocking, and high-priority (the [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) shape with `order > 0`). The producer rationale is that an interrupt-time high-order allocation that fails is harder to recover from than the more common atomic order-0 path, so the kernel maintains a private reserve of high-order pageblocks that only this caller class can drain.

The flag triggers two distinct behaviours. First, [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3237) tries [`__rmqueue_smallest(zone, order, MIGRATE_HIGHATOMIC)`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) before the regular migrate-type walker. The reserve is opened up first because the caller is the one for whom the reserve was created. Second, after [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) returns a page on this path, [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3952) calls [`reserve_highatomic_pageblock()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3427) which converts the surrounding pageblock(s) to [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) up to a per-zone cap of approximately 1% of [`zone_managed_pages()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h).

The pool drains via [`unreserve_highatomic_pageblock()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3479), which has two callers. [`__alloc_pages_direct_reclaim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4442) calls it with `force = false` after a failed reclaim attempt (preserving at least one pageblock per zone), and [`should_reclaim_retry()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4654) calls it with `force = true` immediately before [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4922) goes to OOM (drain everything). The drain converts [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) pageblocks back to the caller's [`ac->migratetype`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h), so the reserve is recycled into the regular allocator rather than being lost.

[`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3655) accepts a free [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) area as a valid candidate for the high-order check when either [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) or [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) is set; otherwise the per-order walk skips the reserve. [`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567) suppresses the `nr_free_highatomic` subtraction for any [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) member; [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) qualifies, so the reserve counts as available free memory for the threshold check.

## SPECIFICATIONS

(none; [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) is a Linux kernel internal flag)

## LINUX KERNEL

### Flag definition

- [`'\<ALLOC_HIGHATOMIC\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292): bit `0x200`. Comment: "Allows access to MIGRATE_HIGHATOMIC".
- [`'\<ALLOC_RESERVES\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297): bundle macro that includes [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292).

### Producers

- [`'\<gfp_to_alloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4509): the only producer. Condition: `order > 0 && (alloc_flags & ALLOC_MIN_RESERVE)` inside the `!__GFP_DIRECT_RECLAIM && !__GFP_NOMEMALLOC` branch.

### Consumers

- [`'\<rmqueue_buddy\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3237): pulls from [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) before the regular [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473). Also at line 3251 as the [`OOM|NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) fallback path.
- [`'\<get_page_from_freelist\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3952): after [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) success, calls [`reserve_highatomic_pageblock()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3427) when [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) is set.
- [`'\<__zone_watermark_ok\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3655): `(alloc_flags & (ALLOC_HIGHATOMIC|ALLOC_OOM))` accepts a free [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) area for the high-order check.
- [`'\<__zone_watermark_unusable_free\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567): bundle membership in [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) suppresses the `nr_free_highatomic` subtraction.

### GFP mapping

- [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): provides [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) which is the gating bit for [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292).
- [`__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): when present, the entire branch that sets [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) is skipped.
- [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): an explicit opt-out that suppresses [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) (and [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278)) so the caller does not draw from reserves.
- [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) at `order > 0`: the canonical producer combination. Yields [`ALLOC_NON_BLOCK | ALLOC_MIN_RESERVE | ALLOC_HIGHATOMIC | ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h).

### Migratetype and freelists

- [`'\<MIGRATE_HIGHATOMIC\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): the dedicated migratetype for the reserve pool.
- [`'\<reserve_highatomic_pageblock\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3427): grows the pool, capped at 1% of [`zone_managed_pages(zone)`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), aligned to [`pageblock_nr_pages`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), with a minimum of one pageblock.
- [`'\<unreserve_highatomic_pageblock\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3479): drains the pool from two callers, [`__alloc_pages_direct_reclaim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4442) (force=false, preserve one pageblock) and [`should_reclaim_retry()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4654) (force=true, drain everything before OOM).
- [`'\<__rmqueue_smallest\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914): the buddy-list walker called with hard-coded `migratetype = MIGRATE_HIGHATOMIC` from [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3237).
- [`'\<move_freepages_block\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c): converts an entire pageblock between migrate types under [`zone->lock`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h).
- [`'\<change_pageblock_range\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c): converts a range covering more than one pageblock (when `order >= pageblock_order`).

### ALLOC_RESERVES bundle and watermark fractions

- [`'\<__zone_watermark_ok\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585): bundle membership through [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297). [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) does NOT contribute its own per-bit cut. The high-order check at [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3655) accepts a [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) free area as a candidate.
- [`'\<__zone_watermark_unusable_free\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567): suppresses `unusable_free += z->nr_free_highatomic` for bundle members so the reserve counts as free.

### Related types and helpers

- [`'\<zone\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): owns [`nr_reserved_highatomic`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) (pages reserved as [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h)) and [`nr_free_highatomic`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) (pages currently free in the reserve).
- [`'\<__alloc_pages_direct_reclaim\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4442): drains the reserve once after each direct-reclaim attempt that fails to satisfy the request.
- [`'\<should_reclaim_retry\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4583): the pre-OOM gate. Force-drains the reserve when no zone can satisfy the request even with full reclaim.

## KERNEL DOCUMENTATION

- [`Documentation/admin-guide/sysctl/vm.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/admin-guide/sysctl/vm.rst): describes the watermark tunables that influence the reserve sizing through `min_free_kbytes`.
- [`Documentation/mm/physical_memory.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/physical_memory.rst): the per-zone watermark and migrate-type model.

## OTHER SOURCES

## DETAILS

### Producer

The producer condition is the inner `if` inside the [`!__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) branch of [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479).

```c
		if (!(gfp_mask & __GFP_NOMEMALLOC)) {
			alloc_flags |= ALLOC_NON_BLOCK;

			if (order > 0 && (alloc_flags & ALLOC_MIN_RESERVE))
				alloc_flags |= ALLOC_HIGHATOMIC;
		}
```

All three branch conditions must hold. The outer branch requires [`__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) absent (the caller cannot block). The inner-outer branch requires [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) absent (the caller has not opted out of reserves). The innermost branch requires `order > 0` and [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) set, which requires [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) (or the RT/DL grant, though RT/DL takes the else-branch and does not reach here).

[`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) at `order > 0` therefore produces [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292). Order-0 [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) does not. The kernel keeps no equivalent reserve for order-0 atomic allocations; they fall back to the regular freelists and rely on the [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) watermark cut and the [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3250) [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) last-chance pull triggered by [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278).

### Reservation: reserve_highatomic_pageblock

The reservation runs on the success path of [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3791).

```c
try_this_zone:
		page = rmqueue(zonelist_zone(ac->preferred_zoneref), zone, order,
				gfp_mask, alloc_flags, ac->migratetype);
		if (page) {
			prep_new_page(page, order, gfp_mask, alloc_flags);

			/*
			 * If this is a high-order atomic allocation then check
			 * if the pageblock should be reserved for the future
			 */
			if (unlikely(alloc_flags & ALLOC_HIGHATOMIC))
				reserve_highatomic_pageblock(page, order, zone);

			return page;
		}
```

The reservation function imposes a soft cap of approximately 1% of the zone.

```c
static void reserve_highatomic_pageblock(struct page *page, int order,
					 struct zone *zone)
{
	int mt;
	unsigned long max_managed, flags;

	/*
	 * The number reserved as: minimum is 1 pageblock, maximum is
	 * roughly 1% of a zone. But if 1% of a zone falls below a
	 * pageblock size, then don't reserve any pageblocks.
	 * Check is race-prone but harmless.
	 */
	if ((zone_managed_pages(zone) / 100) < pageblock_nr_pages)
		return;
	max_managed = ALIGN((zone_managed_pages(zone) / 100), pageblock_nr_pages);
	if (zone->nr_reserved_highatomic >= max_managed)
		return;

	spin_lock_irqsave(&zone->lock, flags);

	/* Recheck the nr_reserved_highatomic limit under the lock */
	if (zone->nr_reserved_highatomic >= max_managed)
		goto out_unlock;

	/* Yoink! */
	mt = get_pageblock_migratetype(page);
	/* Only reserve normal pageblocks (i.e., they can merge with others) */
	if (!migratetype_is_mergeable(mt))
		goto out_unlock;

	if (order < pageblock_order) {
		if (move_freepages_block(zone, page, mt, MIGRATE_HIGHATOMIC) == -1)
			goto out_unlock;
		zone->nr_reserved_highatomic += pageblock_nr_pages;
	} else {
		change_pageblock_range(page, order, MIGRATE_HIGHATOMIC);
		zone->nr_reserved_highatomic += 1 << order;
	}

out_unlock:
	spin_unlock_irqrestore(&zone->lock, flags);
}
```

For `order < pageblock_order` the entire surrounding pageblock is moved to [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) via [`move_freepages_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c), and [`zone->nr_reserved_highatomic`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) increases by [`pageblock_nr_pages`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h). For `order >= pageblock_order` the multi-pageblock range covered by the allocation is converted via [`change_pageblock_range()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c), and [`zone->nr_reserved_highatomic`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) increases by `1 << order`.

The `migratetype_is_mergeable(mt)` check excludes [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) and [`MIGRATE_ISOLATE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) blocks from being recoloured, since those have separate accounting that the [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) drain logic does not understand.

### Consumption: rmqueue_buddy

[`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) is the slow path inside [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) (used when the per-CPU pageset cannot satisfy the request). [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) appears at the top of the function.

```c
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
```

The first [`__rmqueue_smallest(zone, order, MIGRATE_HIGHATOMIC)`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) is the [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) consumer that offers a high-order [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) caller the dedicated reserve before the regular freelists. If the reserve does not have a free block of the requested order, the function falls through to the regular [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) walker.

The second [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) call is the [`ALLOC_NON_BLOCK | ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) last-chance fallback documented separately on the `ALLOC_NON_BLOCK` and `ALLOC_OOM` pages. It can also pull from [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), but only when the regular fallback has exhausted, with the goal of letting the dying caller use the reserve rather than fail.

### Watermark interaction: __zone_watermark_unusable_free

[`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3558) governs how `nr_free_highatomic` is treated.

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
```

For an [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) caller (or any other [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) member), the `if (likely(!(alloc_flags & ALLOC_RESERVES)))` is false, so [`z->nr_free_highatomic`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is NOT added to `unusable_free`. The watermark check therefore counts pages currently sitting in the [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) reserve as available, which matches the reality that [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3237) will draw on them.

A regular allocation (no [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) bit) sees `nr_free_highatomic` subtracted, so the high-atomic reserve is hidden from the threshold check. The reserve is therefore not consumed by accident by callers who do not qualify.

### Watermark interaction: __zone_watermark_ok high-order check

The per-order walk inside [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3640) also acknowledges the reserve.

```c
	for (o = order; o < NR_PAGE_ORDERS; o++) {
		struct free_area *area = &z->free_area[o];
		int mt;

		if (!area->nr_free)
			continue;

		for (mt = 0; mt < MIGRATE_PCPTYPES; mt++) {
			if (!free_area_empty(area, mt))
				return true;
		}

#ifdef CONFIG_CMA
		if ((alloc_flags & ALLOC_CMA) &&
		    !free_area_empty(area, MIGRATE_CMA)) {
			return true;
		}
#endif
		if ((alloc_flags & (ALLOC_HIGHATOMIC|ALLOC_OOM)) &&
		    !free_area_empty(area, MIGRATE_HIGHATOMIC)) {
			return true;
		}
	}
```

The standard PCP migrate-types are always considered; [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is gated on [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286); [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is gated on [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) or [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273). The pairing with [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) is what lets an OOM victim accept a [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) free area, since the victim does not carry [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) (which is set only for non-blocking high-order callers) but gets equivalent access through the [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) bit.

### Drain: unreserve_highatomic_pageblock

Two callers drain the reserve.

```c
/* mm/page_alloc.c, in __alloc_pages_direct_reclaim, line ~4442 */
if (!page && !drained) {
	unreserve_highatomic_pageblock(ac, false);
	drained = true;
	goto retry;
}
```

```c
/* mm/page_alloc.c, in should_reclaim_retry, line ~4654 */
out:
	/* Before OOM, exhaust highatomic_reserve */
	if (!ret)
		return unreserve_highatomic_pageblock(ac, true);

	return ret;
```

The two calls differ in `force`. In [`__alloc_pages_direct_reclaim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4442), the call is `force=false`, which preserves at least one pageblock per zone.

```c
		/*
		 * Preserve at least one pageblock unless memory pressure
		 * is really high.
		 */
		if (!force && zone->nr_reserved_highatomic <=
					pageblock_nr_pages)
			continue;
```

The intent is that direct reclaim under moderate pressure should not destroy the reserve outright. The drain happens once per direct-reclaim attempt (the `drained = true` flag prevents looping).

In [`should_reclaim_retry()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4654), the call is `force=true` and runs only when no zone in the zonelist can satisfy the request even after counting all reclaimable pages. At that point the allocator is about to invoke OOM, so destroying the reserve is the correct choice.

The drain converts pageblocks back to the caller's [`ac->migratetype`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h).

```c
			/*
			 * Convert to ac->migratetype and avoid the normal
			 * pageblock stealing heuristics. Minimally, the caller
			 * is doing the work and needs the pages. More
			 * importantly, if the block was always converted to
			 * MIGRATE_UNMOVABLE or another type then the number
			 * of pageblocks that cannot be completely freed
			 * may increase.
			 */
			if (order < pageblock_order)
				ret = move_freepages_block(zone, page,
							   MIGRATE_HIGHATOMIC,
							   ac->migratetype);
			else {
				move_to_free_list(page, zone, order,
						  MIGRATE_HIGHATOMIC,
						  ac->migratetype);
				change_pageblock_range(page, order,
						       ac->migratetype);
				ret = 1;
			}
```

The choice of [`ac->migratetype`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) (rather than the original migrate type the block came from) is deliberate. The caller that triggered the drain is the one most likely to consume the freed pages, so giving them the right migrate type avoids a second migrate-type fallback round.

### Watermark boost coupling

[`reserve_highatomic_pageblock()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3427) does not directly touch [`zone->watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), but the related [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307) (which is a different path for normal pageblock stealing) does, and the two interact when [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) falls through the [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) mode. That coupling is documented on the `ALLOC_KSWAPD` page; for [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) specifically, the only watermark interaction is the bundle membership in [`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567) and the high-order acceptance in [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3655).

### Reserve cap at 1% of zone

The comment in [`reserve_highatomic_pageblock()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3434) reads "minimum is 1 pageblock, maximum is roughly 1% of a zone." The 1% target balances two failure modes. A too-small reserve fails to absorb realistic high-order interrupt-time bursts, leaving the [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) caller to crash the kernel; a too-large reserve starves the regular allocator of contiguous pageblocks, increasing fragmentation and reducing huge-page success.

The one-pageblock floor matters on small zones. When 1% falls below [`pageblock_nr_pages`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), the function returns early and reserves nothing, deferring entirely to the regular [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) machinery. The cap applies per zone, so a multi-zone NUMA system has the cap multiplied by the number of populated zones.
