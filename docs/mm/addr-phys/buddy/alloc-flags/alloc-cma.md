---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# ALLOC_CMA

[`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) is the page allocator's "CMA areas are eligible" bit. It tells [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) that the [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) freelists may be drawn from, tells [`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567) to count free CMA pages as usable, and tells the high-order check inside [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3651) to accept a free [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) area as a valid candidate. The flag is set by [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776), a thin helper that examines [`gfp_migratetype(gfp_mask)`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h#L20) and ORs in [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) when the resulting migrate type is [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h). Two non-allocator readers also take the bit. [`page_reporting_process_zone()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_reporting.c#L260) probes the watermark with [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) before reporting free pages to the host, and [`migrate_balanced_pgdat()`](https://elixir.bootlin.com/linux/v6.19/source/mm/migrate.c#L2614) probes a target node before NUMA balancing migrates pages onto it.

```
        ALLOC_* bit field (12 bits live in v6.19)
        ─────────────────────────────────────────────────────────────
        bit:  11    10    9     8     7     6     5     4     3     2     1     0
              ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
              │  K  │  T  │ HA  │ NF  │ CMA │ CP  │ MR  │ NB  │ OOM │ NW  │  WMARK    │
              └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                                ▲     ▲
                                │     │
                                │     ALLOC_CMA = 0x80
                                │
                                ALLOC_NOFRAGMENT = 0x100

        Legend: K  = ALLOC_KSWAPD          NF  = ALLOC_NOFRAGMENT     NB  = ALLOC_NON_BLOCK
                T  = ALLOC_TRYLOCK         CP  = ALLOC_CPUSET         OOM = ALLOC_OOM
                HA = ALLOC_HIGHATOMIC      MR  = ALLOC_MIN_RESERVE    NW  = ALLOC_NO_WATERMARKS
```

```
        CMA freelist fallback chain in __rmqueue
        ────────────────────────────────────────────────────────────────

         gfp_migratetype(gfp_mask):
             takes the __GFP_MOVABLE | __GFP_RECLAIMABLE bits, shifts
             them down to MIGRATE_MOVABLE / MIGRATE_RECLAIMABLE / etc.

         gfp_to_alloc_flags_cma(gfp_mask, alloc_flags):
             #ifdef CONFIG_CMA
                 if (gfp_migratetype(gfp_mask) == MIGRATE_MOVABLE)
                         alloc_flags |= ALLOC_CMA;        <── set
             #endif

         __rmqueue(zone, order, migratetype, alloc_flags, &mode):
             if (CONFIG_CMA && (alloc_flags & ALLOC_CMA) &&
                 NR_FREE_CMA_PAGES(zone) > NR_FREE_PAGES(zone) / 2) {
                     page = __rmqueue_cma_fallback(zone, order);
                     if (page) return page;            <── balance
             }

             switch (*mode) {
             case RMQUEUE_NORMAL:
                     page = __rmqueue_smallest(zone, order, migratetype);
                     if (page) return page;
                     fallthrough;
             case RMQUEUE_CMA:
                     if (alloc_flags & ALLOC_CMA) {
                             page = __rmqueue_cma_fallback(zone, order);
                             if (page) {
                                     *mode = RMQUEUE_CMA;
                                     return page;
                             }
                     }
                     fallthrough;
             case RMQUEUE_CLAIM:
                     page = __rmqueue_claim(zone, order, migratetype, ...);
                     if (page) { *mode = RMQUEUE_NORMAL; return page; }
                     fallthrough;
             case RMQUEUE_STEAL:
                     if (!(alloc_flags & ALLOC_NOFRAGMENT))
                             page = __rmqueue_steal(zone, order, migratetype);
             }

         __rmqueue_cma_fallback(zone, order):
             return __rmqueue_smallest(zone, order, MIGRATE_CMA);
```

```
        Watermark interaction
        ────────────────────────────────────────────────────────────────

         __zone_watermark_unusable_free(z, order, alloc_flags):
             #ifdef CONFIG_CMA
                 if (!(alloc_flags & ALLOC_CMA))
                         unusable_free += zone_page_state(z, NR_FREE_CMA_PAGES);
             #endif
             /* CMA pages count as free for ALLOC_CMA callers */

         __zone_watermark_ok(z, order, mark, ..., alloc_flags, free_pages):
             /* high-order walk */
             for (o = order; o < NR_PAGE_ORDERS; o++) {
                     ...
                     #ifdef CONFIG_CMA
                     if ((alloc_flags & ALLOC_CMA) &&
                         !free_area_empty(area, MIGRATE_CMA))
                             return true;
                     #endif
                     if ((alloc_flags & (ALLOC_HIGHATOMIC|ALLOC_OOM)) &&
                         !free_area_empty(area, MIGRATE_HIGHATOMIC))
                             return true;
             }
```

## SUMMARY

[`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) (= `0x80`) is bit 7 of [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c). It is gated on [`CONFIG_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig); the producer and the major consumer paths are wrapped in `#ifdef CONFIG_CMA`. The producer is [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776), called from three places. [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4523) calls it at the end of its slowpath rebuild, [`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L5018) calls it on the fast path, and [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843) calls it when merging [`reserve_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) into [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c). All three call sites apply the same condition `gfp_migratetype(gfp_mask) == MIGRATE_MOVABLE`.

The mapping from [`__GFP_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) to [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is established by [`gfp_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h#L20), which extracts the [`__GFP_MOVABLE | __GFP_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) bits and shifts them down. A bare [`__GFP_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) (e.g. [`GFP_HIGHUSER_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) yields [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h); a [`__GFP_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) caller (e.g. dentry/inode caches) yields [`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h); a non-movable kernel allocation yields [`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h). Only the first of those gets [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286).

[`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) has two distinct CMA paths. The first is a load-balancing pull at the very top. When more than half of the zone's free pages are sitting in CMA, the function pulls from [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) before even consulting the requested migrate type. This balances movable allocations between the regular and CMA areas so that the CMA areas do not stay full while the regular [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) freelist is depleted. The second is the regular fallback chain. After [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) on the requested migrate type returns NULL, the [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) case in the `switch` falls through to [`__rmqueue_cma_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1953) (which is just [`__rmqueue_smallest(zone, order, MIGRATE_CMA)`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914)) before continuing to [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) and [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c).

The watermark code reads the bit in two complementary ways. [`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567) treats [`NR_FREE_CMA_PAGES`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/vmstat.h) as unusable when [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) is NOT set, since a non-movable caller cannot use CMA pages and they should not count towards the watermark. [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3651) accepts a free [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) area as a candidate for the high-order check when [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) is set. A movable caller therefore sees more usable free memory and can target [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) blocks, while a non-movable caller sees less and cannot.

## SPECIFICATIONS

(none; [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) is a Linux kernel internal flag)

## LINUX KERNEL

### Flag definition

- [`'\<ALLOC_CMA\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286): bit `0x80`. Comment: "allow allocations from CMA areas".

### Producers

- [`'\<gfp_to_alloc_flags_cma\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776): the only producer. Three callers all funnel through this helper.
- [`'\<gfp_to_alloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4523): calls [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776) at the end of the slowpath rebuild.
- [`'\<prepare_alloc_pages\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L5018): calls [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776) on the fast path before the first [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) attempt.
- [`'\<__alloc_pages_slowpath\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843): merges `gfp_to_alloc_flags_cma(gfp_mask, reserve_flags)` when [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) returns non-zero.

### Consumers

- [`'\<__rmqueue\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2485): the load-balancing CMA pull at the top, gated on `NR_FREE_CMA_PAGES > NR_FREE_PAGES/2`.
- [`'\<__rmqueue\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2510): the [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) fallthrough case that falls back to [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) after the requested migrate type came up empty.
- [`'\<__zone_watermark_unusable_free\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3572): adds [`NR_FREE_CMA_PAGES`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/vmstat.h) to `unusable_free` when [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) is NOT set.
- [`'\<__zone_watermark_ok\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3651): high-order check accepts [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) free area when [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) is set.
- [`'\<page_reporting_process_zone\>':'mm/page_reporting.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_reporting.c#L276): a non-allocator caller. Probes `zone_watermark_ok(zone, 0, watermark, 0, ALLOC_CMA)` to decide whether to report free pages to the host hypervisor.
- [`'\<migrate_balanced_pgdat\>':'mm/migrate.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/migrate.c#L2632): the NUMA balancing migration target check. Calls `zone_watermark_ok(zone, 0, ..., ZONE_MOVABLE, ALLOC_CMA)` to decide whether moving pages onto the target pgdat keeps it balanced.
- [`'\<__compaction_suitable\>':'mm/compaction.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/compaction.c#L2381): the direct-compaction watermark check. Always passes [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) on the grounds that "pages in CMA pageblocks are considered" for compaction.
- [`'\<compaction_suit_allocation_order\>':'mm/compaction.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/compaction.c#L2498): cross-checks `!(alloc_flags & ALLOC_CMA)` to decide whether unmovable allocations have enough non-CMA free memory to consider compaction useful.
- [`'\<__isolate_free_page\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3147): probes `zone_watermark_ok(zone, 0, watermark, 0, ALLOC_CMA)` before isolating a free page for hot-remove or compaction.

### GFP mapping

- [`__GFP_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): the GFP bit that produces [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) through [`gfp_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h#L20). Triggers [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286).
- [`__GFP_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): yields [`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h). Does NOT trigger [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286).
- [`__GFP_MOVABLE | __GFP_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) (forbidden combination): [`gfp_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h#L20) [`VM_WARN_ON()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmdebug.h)s and the result is undefined; the [`BUILD_BUG_ON`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/build_bug.h) at the same line shows the bit-shifted combination would map to [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) which is not a callable migratetype.
- [`page_group_by_mobility_disabled`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) (boot-time `mobility_disabled` heuristic): [`gfp_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h#L20) returns [`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) unconditionally, suppressing [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286).
- [`GFP_HIGHUSER_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): the canonical user-space movable allocation. Yields [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) and therefore [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286).

### Migratetype and freelists

- [`'\<MIGRATE_CMA\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): the migratetype reserved for CMA areas (gated on [`CONFIG_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig)).
- [`'\<gfp_migratetype\>':'include/linux/gfp.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h#L20): the GFP-to-migratetype translator. The compile-time invariants in this function pin [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) to [`__GFP_MOVABLE >> GFP_MOVABLE_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h).
- [`'\<__rmqueue_cma_fallback\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1953): single-line wrapper `__rmqueue_smallest(zone, order, MIGRATE_CMA)`.
- [`'\<__rmqueue_smallest\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914): the buddy-list walker that scans `zone->free_area[order..NR_PAGE_ORDERS]` for the named migratetype.
- [`'\<NR_FREE_CMA_PAGES\>':'include/linux/vmstat.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/vmstat.h): the per-zone vmstat counter read by [`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3572) and [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2486).

### Related types and helpers

- [`'\<struct cma\>':'include/linux/cma.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cma.h): the CMA descriptor that owns the actual contiguous range. Allocated by [`cma_init_reserved_areas()`](https://elixir.bootlin.com/linux/v6.19/source/mm/cma.c) and consumed by [`cma_alloc()`](https://elixir.bootlin.com/linux/v6.19/source/mm/cma.c) and the page allocator's [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) freelists.

## KERNEL DOCUMENTATION

- [`Documentation/admin-guide/mm/cma_debugfs.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/admin-guide/mm/cma_debugfs.rst): the debugfs interface for inspecting CMA areas, their populated counts, and the allocation/free counters.
- [`Documentation/core-api/dma-api-howto.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/core-api/dma-api-howto.rst): the DMA API howto, including [`dma_alloc_coherent()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/dma-mapping.h) backed by [`cma_alloc()`](https://elixir.bootlin.com/linux/v6.19/source/mm/cma.c).
- [`Documentation/mm/physical_memory.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/physical_memory.rst): per-zone watermark and migrate-type model.

## OTHER SOURCES

## DETAILS

### Producer: gfp_to_alloc_flags_cma

The producer is one short helper in [`mm/page_alloc.c`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776).

```c
/* Must be called after current_gfp_context() which can change gfp_mask */

static inline unsigned int gfp_to_alloc_flags_cma(gfp_t gfp_mask,
						  unsigned int alloc_flags)
{
#ifdef CONFIG_CMA
	if (gfp_migratetype(gfp_mask) == MIGRATE_MOVABLE)
		alloc_flags |= ALLOC_CMA;
#endif
	return alloc_flags;
}
```

The `must be called after current_gfp_context()` comment matters because [`current_gfp_context()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/mm.h) applies scoped masks (`PF_MEMALLOC_PIN` clears [`__GFP_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), `PF_MEMALLOC_NOFS` clears [`__GFP_FS`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), and so on). A caller that runs under [`memalloc_pin_save()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/mm.h) ends up with [`__GFP_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) cleared, which means [`gfp_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h#L20) returns something other than [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) and [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) is not set. That is exactly what [`PF_MEMALLOC_PIN`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) is meant to do; it prevents the caller from receiving CMA pages so a long-term pin (e.g. [`get_user_pages_longterm()`](https://elixir.bootlin.com/linux/v6.19/source/mm/gup.c)) does not block CMA's contiguous reservation contract.

[`gfp_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h#L20) maps the GFP movable/reclaimable bits to the migratetype.

```c
static inline int gfp_migratetype(const gfp_t gfp_flags)
{
	VM_WARN_ON((gfp_flags & GFP_MOVABLE_MASK) == GFP_MOVABLE_MASK);
	BUILD_BUG_ON((1UL << GFP_MOVABLE_SHIFT) != ___GFP_MOVABLE);
	BUILD_BUG_ON((___GFP_MOVABLE >> GFP_MOVABLE_SHIFT) != MIGRATE_MOVABLE);
	BUILD_BUG_ON((___GFP_RECLAIMABLE >> GFP_MOVABLE_SHIFT) != MIGRATE_RECLAIMABLE);
	BUILD_BUG_ON(((___GFP_MOVABLE | ___GFP_RECLAIMABLE) >>
		      GFP_MOVABLE_SHIFT) != MIGRATE_HIGHATOMIC);

	if (unlikely(page_group_by_mobility_disabled))
		return MIGRATE_UNMOVABLE;

	/* Group based on mobility */
	return (__force unsigned long)(gfp_flags & GFP_MOVABLE_MASK) >> GFP_MOVABLE_SHIFT;
}
```

The [`BUILD_BUG_ON`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/build_bug.h) chain pins the bit positions so the shift operation is correct. The key fact for this page is that only [`__GFP_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) (alone, not combined with [`__GFP_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) yields [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) and triggers [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286).

### Three call sites for the producer

The three call sites all yield the same result for a given `gfp_mask`, but cover different points in the allocator's lifecycle.

#### Fast path (prepare_alloc_pages)

```c
	*alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, *alloc_flags);
```

This runs once per allocation, before the first [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) attempt. Sets [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) for the entire allocation lifetime if the request is movable.

#### Slowpath (gfp_to_alloc_flags)

```c
	alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, alloc_flags);
```

The slowpath rebuild also calls the helper at the end. This is technically redundant for callers that came through [`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4986), but the slowpath is also called for retries after [`check_retry_cpuset()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4660) returns true, and the rebuild has to start from scratch.

#### Slowpath reserve merge (__alloc_pages_slowpath retry)

```c
	reserve_flags = __gfp_pfmemalloc_flags(gfp_mask);
	if (reserve_flags)
		alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, reserve_flags) |
					  (alloc_flags & ALLOC_KSWAPD);
```

When a caller earns [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) or [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273), [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) is rewritten. The rewrite goes through [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776) so that a movable caller that has reached the reserve path keeps [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) on top of the new reserve grant. The reserve grant strengthens the request rather than replacing it, and the caller's original migratetype intent is preserved.

### Consumer: __rmqueue load-balancing pull

[`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) starts with a CMA-balancing check before touching the regular freelists.

```c
page *
__rmqueue(struct zone *zone, unsigned int order, int migratetype,
	  unsigned int alloc_flags, enum rmqueue_mode *mode)
{
	struct page *page;

	if (IS_ENABLED(CONFIG_CMA)) {
		/*
		 * Balance movable allocations between regular and CMA areas by
		 * allocating from CMA when over half of the zone's free memory
		 * is in the CMA area.
		 */
		if (alloc_flags & ALLOC_CMA &&
		    zone_page_state(zone, NR_FREE_CMA_PAGES) >
		    zone_page_state(zone, NR_FREE_PAGES) / 2) {
			page = __rmqueue_cma_fallback(zone, order);
			if (page)
				return page;
		}
	}
```

The intent is to keep the CMA area utilised so that contiguous-memory consumers (the original purpose of CMA) can still find blocks when they ask for them. Without the balancing pull, movable allocations would prefer the regular freelists, leaving CMA full and forcing the contiguous-memory consumers to migrate or compact when they need a range.

The threshold `NR_FREE_CMA_PAGES > NR_FREE_PAGES / 2` is conservative. The balancing only kicks in when more than half of the zone's free pages are in CMA.

### Consumer: __rmqueue fallback chain

After the load-balance check, [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) walks the migrate-type fallback chain via a switch that supports four modes.

```c
	switch (*mode) {
	case RMQUEUE_NORMAL:
		page = __rmqueue_smallest(zone, order, migratetype);
		if (page)
			return page;
		fallthrough;
	case RMQUEUE_CMA:
		if (alloc_flags & ALLOC_CMA) {
			page = __rmqueue_cma_fallback(zone, order);
			if (page) {
				*mode = RMQUEUE_CMA;
				return page;
			}
		}
		fallthrough;
	case RMQUEUE_CLAIM:
		page = __rmqueue_claim(zone, order, migratetype, alloc_flags);
		if (page) {
			/* Replenished preferred freelist, back to normal mode. */
			*mode = RMQUEUE_NORMAL;
			return page;
		}
		fallthrough;
	case RMQUEUE_STEAL:
		if (!(alloc_flags & ALLOC_NOFRAGMENT)) {
			page = __rmqueue_steal(zone, order, migratetype);
			if (page) {
				*mode = RMQUEUE_STEAL;
				return page;
			}
		}
	}
	return NULL;
}
```

The order of preference walks from the requested migratetype to CMA, then to claiming a whole pageblock, and finally to stealing pages from another pageblock. The CMA branch is gated on [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) so that non-movable callers never reach the [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) freelists. The fallthrough behaviour matters because when the requested migratetype is empty, the CMA fallback is tried before claiming or stealing a pageblock, and the CMA pull is cheaper than either of those.

[`__rmqueue_cma_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1953) is the smallest function in the file.

```c
page *__rmqueue_cma_fallback(struct zone *zone,
					unsigned int order)
{
	return __rmqueue_smallest(zone, order, MIGRATE_CMA);
}
```

A direct delegation to [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) with `migratetype = MIGRATE_CMA` hard-coded.

### Consumer: __zone_watermark_unusable_free

The watermark decision treats CMA pages asymmetrically.

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

A non-movable caller adds [`NR_FREE_CMA_PAGES`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/vmstat.h) to `unusable_free` (so the threshold check sees less free memory). A movable caller does not, so CMA pages count toward the usable pool. This is consistent with [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) being willing to pull from CMA only for [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) callers.

### Consumer: __zone_watermark_ok high-order check

The per-order check inside [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3640) considers [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) only when [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) is set.

```c
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
```

The PCP migrate-types ([`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), [`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h)) are always considered. [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is gated, and [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is gated on [`ALLOC_HIGHATOMIC | ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) (documented on those flag pages).

### Non-allocator readers

Two callers outside the page allocator read [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) when probing watermarks.

[`page_reporting_process_zone()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_reporting.c#L260) (free-page reporting to the host hypervisor) checks the zone before walking the freelists.

```c
	/* Generate minimum watermark to be able to guarantee progress */
	watermark = low_wmark_pages(zone) +
		    (PAGE_REPORTING_CAPACITY << page_reporting_order);

	/*
	 * Cancel request if insufficient free memory or if we failed
	 * to allocate page reporting statistics for the zone.
	 */
	if (!zone_watermark_ok(zone, 0, watermark, 0, ALLOC_CMA))
		return err;
```

The watermark probe uses [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) so that [`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567) does not subtract CMA free pages. The reporting code temporarily isolates pages from any of the freelists (movable, reclaimable, CMA, etc.) and reports them to the host as candidates for reclamation; CMA pages count toward the reportable pool.

[`migrate_balanced_pgdat()`](https://elixir.bootlin.com/linux/v6.19/source/mm/migrate.c#L2614) (NUMA balancing) checks a target pgdat.

```c
		if (!zone_watermark_ok(zone, 0,
				       high_wmark_pages(zone) +
				       nr_migrate_pages,
				       ZONE_MOVABLE, ALLOC_CMA))
			continue;
		return true;
```

NUMA balancing migrates pages between nodes to follow the task that accesses them. The target zone must hold the high watermark plus the number of pages being migrated. [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) is passed because the migrated pages are movable and may legitimately end up in CMA.

[`__compaction_suitable()`](https://elixir.bootlin.com/linux/v6.19/source/mm/compaction.c#L2374) does the same for direct compaction.

```c
	/*
	 * ALLOC_CMA is used, as pages in CMA pageblocks are considered
	 * suitable migration targets, so satisfying the watermark there
	 * would also imply that the allocation could be done in CMA.
	 */
	return __zone_watermark_ok(zone, 0, watermark, highest_zoneidx,
				   ALLOC_CMA, free_pages);
```

According to the comment, compaction migrates pages and migration treats CMA pageblocks as valid destinations, so the watermark probe should reflect that.

### MIGRATE_RECLAIMABLE allocations skip ALLOC_CMA

[`gfp_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h#L20) returns [`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) for [`__GFP_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) callers (such as dentry and inode caches that the slab shrinker can reclaim). [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776) gates [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) on `gfp_migratetype(gfp_mask) == MIGRATE_MOVABLE`, so reclaimable allocations do NOT receive [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286).

CMA's contiguous-memory contract requires that pages in CMA pageblocks be migratable on demand. A reclaimable page is dropped by the shrinker, while a movable page is relocated by migration. CMA pageblocks need migration targets, so [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776) restricts [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) to [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h).

### Interaction with PF_MEMALLOC_PIN

Long-term pins (e.g. through [`get_user_pages_longterm()`](https://elixir.bootlin.com/linux/v6.19/source/mm/gup.c) or RDMA) cannot land on CMA pages because the pin would block CMA from migrating the page. The kernel handles this through [`memalloc_pin_save()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/mm.h), which sets [`PF_MEMALLOC_PIN`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) on [`current->flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h). [`current_gfp_context()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/mm.h) clears [`__GFP_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) when the flag is set, so a subsequent [`gfp_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h#L20) returns something other than [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) and [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776) does not set [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286). The "must be called after `current_gfp_context()`" comment on [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776) refers to this ordering, since the scoped masks must run first so that the pin context can withdraw [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) eligibility.
