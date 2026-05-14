---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# RMQUEUE_CLAIM

[`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) is the third case label of [`enum rmqueue_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461). It tries to convert an entire foreign pageblock to the requested migratetype before taking a page out of it, so that subsequent allocations of the same migratetype find native pages on the freelist. The work is delegated to [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382), which scans foreign-migratetype freelists from [`MAX_PAGE_ORDER`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) down toward `min_order` and calls [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307) on each candidate. A successful claim resets `*mode = RMQUEUE_NORMAL` because the requested freelist has just been replenished.

```
    enum rmqueue_mode state machine (RMQUEUE_CLAIM highlighted)
    ──────────────────────────────────────────────────────────

       RMQUEUE_NORMAL     RMQUEUE_CMA
              │                │
              └────────┬───────┘   (fall-through chain)
                       ▼
       ┌──────────────────────────────┐
       │       RMQUEUE_CLAIM          │
       │                              │
       │  page = __rmqueue_claim();   │
       │  if (page) {                 │
       │    *mode = RMQUEUE_NORMAL; ◀─┼──── reset, freelist replenished
       │    return page;              │
       │  }                           │
       └──────────────┬───────────────┘
                      │
               page found?
               ┌──────┴──────┐
               │             │
              YES            NO ─ fallthrough ─▶ RMQUEUE_STEAL
               │
               ▼
          return page  (*mode = RMQUEUE_NORMAL)


    __rmqueue_claim scan (top-down, large to small)
    ───────────────────────────────────────────────

      order:  MAX  MAX-1   ...   pageblock_order   ...   min_order
              ┌──┐  ┌──┐         ┌──┐                    ┌──┐
              │X │  │  │         │ X│                    │  │  free_area[].free_list[FALLBACK]
              └──┘  └──┘         └──┘                    └──┘
                ▲
                │ for current_order = MAX_PAGE_ORDER down to min_order:
                │     fallback_mt = find_suitable_fallback(area, current_order,
                │                                          start_migratetype, true)
                │     if fallback_mt == -2: break  (claim heuristic refuses)
                │     if fallback_mt == -1: continue (no fallback free at this order)
                │     try_to_claim_block(zone, page, current_order, order, ...)


    try_to_claim_block decision tree
    ────────────────────────────────

       current_order >= pageblock_order ?
       │
       ├── YES ─▶ del_page_from_free_list(page, current_order, block_type)
       │         change_pageblock_range(page, current_order, start_type)
       │         expand(zone, page, order, current_order, start_type)
       │         account_freepages(zone, nr_added, start_type)
       │         return page  (whole pageblock(s) claimed in one shot)
       │
       └── NO ─▶ boost_watermark(zone)            (raise WMARK_HIGH)
                set_bit(ZONE_BOOSTED_WATERMARK)   (kicks kswapd later)
                prep_move_freepages_block(...)    (count free + movable)
                if free + alike >= pageblock/2:
                  __move_freepages_block(...)     (move all free pages)
                  set_pageblock_migratetype(...)  (flip the block's type)
                  return __rmqueue_smallest(zone, order, start_type)
                else:
                  return NULL                      (block was too mixed)


    ALLOC_NOFRAGMENT raises min_order
    ─────────────────────────────────

       if (order < pageblock_order && alloc_flags & ALLOC_NOFRAGMENT)
           min_order = pageblock_order;

       Effect: when ALLOC_NOFRAGMENT is set, only whole-pageblock claims
       are attempted; sub-pageblock claims (which would fragment the
       block by partially polluting it) are refused.  The slowpath
       reiterates the zonelist without ALLOC_NOFRAGMENT if nothing else
       works.
```

## SUMMARY

[`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) mutates the migratetype of one or more pageblocks. It calls [`set_pageblock_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L556) to flip the [`pageblock_flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) entry, and either [`__move_freepages_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1967) (sub-pageblock branch) or [`expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727) on a >= [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h) buddy (whole-block branch) to relocate every currently-free page in those pageblocks from the foreign-migratetype freelist to the requested-migratetype freelist. The returned page comes from a tail call to [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) on the now-converted freelist; the leftover pages from the converted pageblock stay on it for subsequent allocations to consume.

Mutating the pageblock migratetype changes the routing of all future [`__free_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) calls for any page within that pageblock. The free path looks up the pageblock's recorded migratetype via [`get_pageblock_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) and deposits the page on the matching freelist, so any in-use pages within the converted pageblock will, when they are eventually freed, land on the new migratetype's freelist rather than their original one. The conversion therefore persists until another [`set_pageblock_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L556) call (typically from another [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464), from compaction, or from CMA range setup) flips it again. The freelist-hygiene rule from commit c0cd6f557b90 ("mm: page_alloc: fix freelist movement during block conversion") relies on this invariant: a free page's migratetype on its freelist is always the migratetype recorded for its pageblock.

The mode is gated by the [`should_try_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2230) heuristic, which permits claims only for orders large enough that the ratio of converted pages to wasted scans stays favorable, or for migratetypes ([`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L65), [`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L67)) where polluting a movable block would cause permanent fragmentation. The interaction with [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) raises the minimum claim order to [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h), so an [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) request only ever claims whole pageblocks.

## SPECIFICATIONS

(none; [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) is a Linux kernel internal enum value)

## LINUX KERNEL

### Enum value

- [`'\<enum rmqueue_mode\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461): four-value enum used by [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) to remember the last successful allocation mode; [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) is value 2

### GFP-to-alloc_flags mapping

- [`'\<gfp_to_alloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479): sets [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) when [`defrag_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) is enabled, raising the claim minimum order to [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h); also sets [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) from [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)

### Producers (call sites that initialize `rmqm` and supply `alloc_flags`)

- [`'\<rmqueue_buddy\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222): per-call buddy-allocator entry that initializes `rmqm = RMQUEUE_NORMAL` and passes [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) through to [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473)
- [`'\<rmqueue_bulk\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542): bulk refiller; the local `rmqm` survives across `count` iterations, but a successful [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) resets it back to [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) so the next iteration re-enters the switch at the [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) call on [`free_list[migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138), which the just-completed claim has populated with leftover pages

### Consumer (the dispatch)

- [`'\<__rmqueue\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473): the [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) case calls [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) and on success writes `*mode = RMQUEUE_NORMAL`

### Claim implementation (the case body)

- [`'\<__rmqueue_claim\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382): top-down scan from [`MAX_PAGE_ORDER`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) toward `min_order`, calling [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307) on each candidate; raises `min_order` to [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h) when [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) is set
- [`'\<try_to_claim_block\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307): two-phase claim, taking the whole block immediately for orders >= [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h) and otherwise running the "free + alike >= half pageblock" heuristic
- [`'\<__rmqueue_smallest\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914): tail call from the sub-pageblock branch of [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307) once the block has been re-typed; this is what actually returns the page

### Heuristics / fallback selection

- [`'\<should_try_claim_block\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2230): heuristic gate consulted by [`find_suitable_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2278) when called with `claimable=true`; returns `true` for orders >= [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h)/2, for non-movable allocations, or when [`page_group_by_mobility_disabled`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) is set
- [`'\<find_suitable_fallback\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2278): walks the [`fallbacks[][]`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1946) preference list to find a foreign migratetype with a free page at the requested order; tri-valued return distinguishes "no fallback at this order" (`-1`) from "claim heuristic refuses" (`-2`)

### Block conversion primitives (whole-block branch)

- [`'\<change_pageblock_range\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L944): flips the migratetype of every pageblock spanned by a high-order page in one walk; used on the >= [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h) branch of [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307)
- [`'\<set_pageblock_migratetype\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L556): writes the new migratetype bits into the [`pageblock_flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) array, completing the conversion
- [`'\<expand\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727): subdivides the high-order buddy and pushes the leftover halves onto the new (claimed) migratetype freelist

### Block conversion primitives (sub-pageblock branch)

- [`'\<prep_move_freepages_block\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2000): pre-walk that counts free pages and movable-by-LRU/movable-ops pages within the pageblock; aborts if the block straddles a zone boundary
- [`'\<__move_freepages_block\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1967): walks every PFN of the candidate pageblock and moves each [`PageBuddy`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/page-flags.h)-marked head from `old_mt`'s freelist to `new_mt`'s freelist via [`move_to_free_list()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c)

### Watermarks and reserves

- [`'\<boost_watermark\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2188): bumps [`zone->watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) by [`pageblock_nr_pages`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h) on every sub-pageblock claim, capped by [`watermark_boost_factor`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c)
- [`ZONE_BOOSTED_WATERMARK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): per-zone flag bit set inside [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307) after a successful sub-pageblock claim; cleared by [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3393) on the subsequent allocation, which also wakes [`kswapd`](https://elixir.bootlin.com/linux/v6.19/source/mm/vmscan.c)
- [`zone->watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): per-zone surcharge added on top of the static [`WMARK_*`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) array; raised by [`boost_watermark()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2188) and decayed by [`balance_pgdat()`](https://elixir.bootlin.com/linux/v6.19/source/mm/vmscan.c)
- [`'\<wakeup_kswapd\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c): woken from [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3393) when [`ZONE_BOOSTED_WATERMARK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) was set, to enforce the elevated watermark via reclaim

### Concurrency

- [`zone->lock`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): held across [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382), [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307), [`__move_freepages_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1967), and [`set_pageblock_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L556); ensures the freelist surgery and the pageblock-flags update appear atomic to other allocators

### Migratetype and fallback table

- [`'\<enum migratetype\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L64): defines the migratetypes that can be claim-targets
- [`fallbacks[MIGRATE_PCPTYPES][MIGRATE_PCPTYPES - 1]`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1946): static preference table consulted by [`find_suitable_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2278); only [`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L65), [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66), and [`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L67) are indexable, so [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) and [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L69) are never claim targets
- [`'\<struct free_area\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138): per-(zone, order) array whose [`free_list[fallback_mt]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) is the source of the candidate buddy at each scan iteration

### Flag definition

- [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288): bit `0x100` (zero when [`CONFIG_ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig) is off). Comment "avoid mixing pageblock types"; raises `min_order` to [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h) inside [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382)
- [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294): bit `0x800`. Comment "allow waking of kswapd, __GFP_KSWAPD_RECLAIM set"; gates the [`ZONE_BOOSTED_WATERMARK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) bit-set inside [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307)

### Tracepoints

- [`'\<trace_mm_page_alloc_extfrag\>':'include/trace/events/kmem.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/trace/events/kmem.h#L269): emitted by [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) on success; records the start migratetype, the fallback migratetype, the request order, and the buddy order
- [`'\<trace_mm_page_alloc_zone_locked\>':'include/trace/events/kmem.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/trace/events/kmem.h#L238): emitted by the tail call to [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) inside [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307) for the sub-pageblock case

## KERNEL DOCUMENTATION

- [`Documentation/mm/page_allocation.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/page_allocation.rst): page allocator overview that motivates pageblock-grouped fallback as an anti-fragmentation strategy
- [`Documentation/mm/balance.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/balance.rst): explains watermark management and how kswapd interacts with watermark boosting
- [`Documentation/mm/physical_memory.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/physical_memory.rst): describes [`struct zone`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) and the [`pageblock_flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) array that [`set_pageblock_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L556) writes to

## OTHER SOURCES

- [LWN: Avoiding fragmentation with page mobility](https://lwn.net/Articles/224254/)
- [LWN: Linux page allocator and fragmentation](https://lwn.net/Articles/745014/)
- [LKML: mm: page_alloc: speed up fallbacks in rmqueue_bulk() (Johannes Weiner, Apr 2025)](https://lore.kernel.org/lkml/20250407180154.63348-1-hannes@cmpxchg.org/)
- [LKML: mm: page_alloc: don't steal single pages from biggest buddy (Johannes Weiner, Feb 2025)](https://lore.kernel.org/lkml/20250225001023.1494422-2-hannes@cmpxchg.org/)

## DETAILS

### The case body

The [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) arm of the switch in [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) consists of one call and one mode reset:

```c
case RMQUEUE_CLAIM:
	page = __rmqueue_claim(zone, order, migratetype, alloc_flags);
	if (page) {
		/* Replenished preferred freelist, back to normal mode. */
		*mode = RMQUEUE_NORMAL;
		return page;
	}
	fallthrough;
case RMQUEUE_STEAL:
	...
```

According to the in-tree comment "Replenished preferred freelist, back to normal mode", the reset to [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) follows from the post-state of a successful claim. The claim has converted at least one foreign pageblock into the requested migratetype, with the remainder of the converted block deposited on [`free_list[start_migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) by either [`__move_freepages_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1967) (sub-pageblock case) or [`expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727) (>= pageblock_order case). The next [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) iteration enters [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) at [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) and finds a page on [`free_list[start_migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) without re-running the claim scan.

### __rmqueue_claim top-down scan

[`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) walks free_areas from the largest order downward and applies [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307) at each step:

```c
static __always_inline struct page *
__rmqueue_claim(struct zone *zone, int order, int start_migratetype,
						unsigned int alloc_flags)
{
	struct free_area *area;
	int current_order;
	int min_order = order;
	struct page *page;
	int fallback_mt;

	/*
	 * Do not steal pages from freelists belonging to other pageblocks
	 * i.e. orders < pageblock_order. If there are no local zones free,
	 * the zonelists will be reiterated without ALLOC_NOFRAGMENT.
	 */
	if (order < pageblock_order && alloc_flags & ALLOC_NOFRAGMENT)
		min_order = pageblock_order;

	/*
	 * Find the largest available free page in the other list. This roughly
	 * approximates finding the pageblock with the most free pages, which
	 * would be too costly to do exactly.
	 */
	for (current_order = MAX_PAGE_ORDER; current_order >= min_order;
				--current_order) {
		area = &(zone->free_area[current_order]);
		fallback_mt = find_suitable_fallback(area, current_order,
						     start_migratetype, true);

		/* No block in that order */
		if (fallback_mt == -1)
			continue;

		/* Advanced into orders too low to claim, abort */
		if (fallback_mt == -2)
			break;

		page = get_page_from_free_area(area, fallback_mt);
		page = try_to_claim_block(zone, page, current_order, order,
					  start_migratetype, fallback_mt,
					  alloc_flags);
		if (page) {
			trace_mm_page_alloc_extfrag(page, order, current_order,
						    start_migratetype, fallback_mt);
			return page;
		}
	}

	return NULL;
}
```

According to the function's comment "Find the largest available free page in the other list. This roughly approximates finding the pageblock with the most free pages, which would be too costly to do exactly." A buddy at order N occupies `1 << N` contiguous pages within a single pageblock, so a buddy on the freelist at a high order tells the scanner that at least that many pages in some pageblock are free, without the scanner having to count free pages in any pageblock. Converting that pageblock then transfers `1 << N` (and possibly more) free pages to the requested-migratetype freelist in one [`__move_freepages_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1967) call.

### find_suitable_fallback with claimable=true

[`find_suitable_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2278) is the function that turns the "is there anything to claim at this order?" question into a tri-valued answer:

```c
int find_suitable_fallback(struct free_area *area, unsigned int order,
			   int migratetype, bool claimable)
{
	int i;

	if (claimable && !should_try_claim_block(order, migratetype))
		return -2;

	if (area->nr_free == 0)
		return -1;

	for (i = 0; i < MIGRATE_PCPTYPES - 1 ; i++) {
		int fallback_mt = fallbacks[migratetype][i];

		if (!free_area_empty(area, fallback_mt))
			return fallback_mt;
	}

	return -1;
}
```

The `claimable=true` argument used by [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) introduces the distinction between "no fallback available" (`-1`) and "claim heuristic refuses" (`-2`). The caller treats the two outcomes differently: `-1` means continue scanning at lower orders (a smaller buddy might still be claimable), while `-2` means break out of the entire scan because [`should_try_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2230) has decided that no order in the rest of the loop will be allowed to claim either.

### should_try_claim_block heuristic

[`should_try_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2230) is the gate that decides which (order, start_migratetype) pairs are worth a claim attempt:

```c
static bool should_try_claim_block(unsigned int order, int start_mt)
{
	/*
	 * Leaving this order check is intended, although there is
	 * relaxed order check in next check. The reason is that
	 * we can actually claim the whole pageblock if this condition met,
	 * but, below check doesn't guarantee it and that is just heuristic
	 * so could be changed anytime.
	 */
	if (order >= pageblock_order)
		return true;

	/*
	 * Above a certain threshold, always try to claim, as it's likely there
	 * will be more free pages in the pageblock.
	 */
	if (order >= pageblock_order / 2)
		return true;

	/*
	 * Unmovable/reclaimable allocations would cause permanent
	 * fragmentations if they fell back to allocating from a movable block
	 * (polluting it), so we try to claim the whole block regardless of the
	 * allocation size. Later movable allocations can always steal from this
	 * block, which is less problematic.
	 */
	if (start_mt == MIGRATE_RECLAIMABLE || start_mt == MIGRATE_UNMOVABLE)
		return true;

	if (page_group_by_mobility_disabled)
		return true;

	/*
	 * Movable pages won't cause permanent fragmentation, so when you alloc
	 * small pages, we just need to temporarily steal unmovable or
	 * reclaimable pages that are closest to the request size. After a
	 * while, memory compaction may occur to form large contiguous pages,
	 * and the next movable allocation may not need to steal.
	 */
	return false;
}
```

The function returns `true` (claim is worthwhile) in four cases. A request whose order is at least [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h) is allowed because the request alone would consume the whole block. A request at >= half [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h) is allowed because the buddy at that size implies the rest of the pageblock is likely free too. Any [`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L65) or [`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L67) request is allowed regardless of size, because polluting a movable block with non-movable pages would break compaction permanently. The boot-time fallback when [`page_group_by_mobility_disabled`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) is set treats every request as claimable. A small movable request returns `false` and falls through to [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) on the next case label, because compaction will eventually clean up the pollution.

### try_to_claim_block: the >= pageblock_order branch

When [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) finds a buddy whose order is at least [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h), the [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307) implementation takes a fast path:

```c
static struct page *
try_to_claim_block(struct zone *zone, struct page *page,
		   int current_order, int order, int start_type,
		   int block_type, unsigned int alloc_flags)
{
	int free_pages, movable_pages, alike_pages;
	unsigned long start_pfn;

	/* Take ownership for orders >= pageblock_order */
	if (current_order >= pageblock_order) {
		unsigned int nr_added;

		del_page_from_free_list(page, zone, current_order, block_type);
		change_pageblock_range(page, current_order, start_type);
		nr_added = expand(zone, page, order, current_order, start_type);
		account_freepages(zone, nr_added, start_type);
		return page;
	}
```

The buddy is removed from the foreign-type freelist by [`del_page_from_free_list()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c). [`change_pageblock_range()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L944) iterates over every pageblock spanned by the buddy and calls [`set_pageblock_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L556) on each, since a buddy of order > [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h) covers multiple pageblocks:

```c
static void change_pageblock_range(struct page *pageblock_page,
				   int start_order, int migratetype)
{
	int nr_pageblocks = 1 << (start_order - pageblock_order);

	while (nr_pageblocks--) {
		set_pageblock_migratetype(pageblock_page, migratetype);
		pageblock_page += pageblock_nr_pages;
	}
}
```

[`expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727) then halves the buddy down to the requested order, depositing the leftover halves onto [`free_area[].free_list[start_type]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) (the now-native list). The whole-block branch never fails because the entire pageblock is already free by virtue of the buddy being on the freelist at order >= [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h).

### try_to_claim_block: the partial-block branch

When the buddy is smaller than [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h), the rest of [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307) runs the "free + alike >= half pageblock" heuristic, which counts free buddies and migration-compatible in-use pages in the candidate pageblock and proceeds with the conversion only when the count crosses half the pageblock:

```c
	/*
	 * Boost watermarks to increase reclaim pressure to reduce the
	 * likelihood of future fallbacks. Wake kswapd now as the node
	 * may be balanced overall and kswapd will not wake naturally.
	 */
	if (boost_watermark(zone) && (alloc_flags & ALLOC_KSWAPD))
		set_bit(ZONE_BOOSTED_WATERMARK, &zone->flags);

	/* moving whole block can fail due to zone boundary conditions */
	if (!prep_move_freepages_block(zone, page, &start_pfn, &free_pages,
				       &movable_pages))
		return NULL;

	/*
	 * Determine how many pages are compatible with our allocation.
	 * For movable allocation, it's the number of movable pages which
	 * we just obtained. For other types it's a bit more tricky.
	 */
	if (start_type == MIGRATE_MOVABLE) {
		alike_pages = movable_pages;
	} else {
		/*
		 * If we are falling back a RECLAIMABLE or UNMOVABLE allocation
		 * to MOVABLE pageblock, consider all non-movable pages as
		 * compatible. If it's UNMOVABLE falling back to RECLAIMABLE or
		 * vice versa, be conservative since we can't distinguish the
		 * exact migratetype of non-movable pages.
		 */
		if (block_type == MIGRATE_MOVABLE)
			alike_pages = pageblock_nr_pages
						- (free_pages + movable_pages);
		else
			alike_pages = 0;
	}
	/*
	 * If a sufficient number of pages in the block are either free or of
	 * compatible migratability as our allocation, claim the whole block.
	 */
	if (free_pages + alike_pages >= (1 << (pageblock_order-1)) ||
			page_group_by_mobility_disabled) {
		__move_freepages_block(zone, start_pfn, block_type, start_type);
		set_pageblock_migratetype(pfn_to_page(start_pfn), start_type);
		return __rmqueue_smallest(zone, order, start_type);
	}

	return NULL;
}
```

The first action is the call to [`boost_watermark()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2188), which raises the zone's watermark and arms the [`ZONE_BOOSTED_WATERMARK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) flag. The flag is consumed by [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3393) on the way out of the allocation, which clears the bit and wakes [`kswapd`](https://elixir.bootlin.com/linux/v6.19/source/mm/vmscan.c) so that the elevated watermark gets enforced by reclaim:

```c
static inline bool boost_watermark(struct zone *zone)
{
	unsigned long max_boost;

	if (!watermark_boost_factor)
		return false;
	...
	max_boost = mult_frac(zone->_watermark[WMARK_HIGH],
			watermark_boost_factor, 10000);
	...
	max_boost = max(pageblock_nr_pages, max_boost);

	zone->watermark_boost = min(zone->watermark_boost + pageblock_nr_pages,
		max_boost);

	return true;
}
```

The watermark boost matters because a sub-pageblock claim implies that the requested-migratetype freelist had run dry, and the kernel responds by raising the bar that future allocations have to clear, which in turn forces [`kswapd`](https://elixir.bootlin.com/linux/v6.19/source/mm/vmscan.c) to free more pages and reduce the chance of another fallback in the near future.

[`prep_move_freepages_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2000) walks the pageblock and counts free buddy heads and movable-by-LRU/movable-ops pages:

```c
static bool prep_move_freepages_block(struct zone *zone, struct page *page,
				      unsigned long *start_pfn,
				      int *num_free, int *num_movable)
{
	unsigned long pfn, start, end;

	pfn = page_to_pfn(page);
	start = pageblock_start_pfn(pfn);
	end = pageblock_end_pfn(pfn);

	/*
	 * The caller only has the lock for @zone, don't touch ranges
	 * that straddle into other zones. While we could move part of
	 * the range that's inside the zone, this call is usually
	 * accompanied by other operations such as migratetype updates
	 * which also should be locked.
	 */
	if (!zone_spans_pfn(zone, start))
		return false;
	if (!zone_spans_pfn(zone, end - 1))
		return false;

	*start_pfn = start;

	if (num_free) {
		*num_free = 0;
		*num_movable = 0;
		for (pfn = start; pfn < end;) {
			page = pfn_to_page(pfn);
			if (PageBuddy(page)) {
				int nr = 1 << buddy_order(page);

				*num_free += nr;
				pfn += nr;
				continue;
			}
			/*
			 * We assume that pages that could be isolated for
			 * migration are movable. But we don't actually try
			 * isolating, as that would be expensive.
			 */
			if (PageLRU(page) || page_has_movable_ops(page))
				(*num_movable)++;
			pfn++;
		}
	}

	return true;
}
```

The `alike_pages` calculation in [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307) interprets the counts according to the migratetype combination involved. A movable-into-X claim trusts the LRU/movable-ops count directly. A non-movable-into-movable claim treats every non-movable, non-free page as compatible. A non-movable-into-non-movable claim refuses to assume anything and counts only the actually-free pages as alike, which is the conservative interpretation.

The "free + alike >= half pageblock" check then decides whether the block is worth claiming or whether it would just leave the block too mixed for grouping to be useful. On success, [`__move_freepages_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1967) moves every free buddy onto the new freelist and [`set_pageblock_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L556) flips the pageblock's recorded migratetype. The tail call to [`__rmqueue_smallest(zone, order, start_type)`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) is what actually returns the page, now from the native freelist that was just populated.

### __move_freepages_block walks the pageblock

[`__move_freepages_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1967) does the actual freelist surgery for the partial-block case:

```c
static int __move_freepages_block(struct zone *zone, unsigned long start_pfn,
				  int old_mt, int new_mt)
{
	struct page *page;
	unsigned long pfn, end_pfn;
	unsigned int order;
	int pages_moved = 0;

	VM_WARN_ON(start_pfn & (pageblock_nr_pages - 1));
	end_pfn = pageblock_end_pfn(start_pfn);

	for (pfn = start_pfn; pfn < end_pfn;) {
		page = pfn_to_page(pfn);
		if (!PageBuddy(page)) {
			pfn++;
			continue;
		}

		/* Make sure we are not inadvertently changing nodes */
		VM_BUG_ON_PAGE(page_to_nid(page) != zone_to_nid(zone), page);
		VM_BUG_ON_PAGE(page_zone(page) != zone, page);

		order = buddy_order(page);

		move_to_free_list(page, zone, order, old_mt, new_mt);

		pfn += 1 << order;
		pages_moved += 1 << order;
	}

	return pages_moved;
}
```

For every PFN in the pageblock, [`PageBuddy(page)`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/page-flags.h) tells whether this PFN is a buddy head. If so, [`move_to_free_list()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) unlinks it from `old_mt`'s list at order `order` and re-links it onto `new_mt`'s list at the same order. Pages that are not buddy heads are skipped because they are either in-use pages (which will become whatever migratetype the pageblock gets, but their freelist position is irrelevant) or non-head buddy bodies (which will be moved together with their head).

### ALLOC_NOFRAGMENT and min_order

[`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) raises `min_order` to [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h) when [`alloc_flags & ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) is set. According to the function's comment "Do not steal pages from freelists belonging to other pageblocks i.e. orders < pageblock_order. If there are no local zones free, the zonelists will be reiterated without ALLOC_NOFRAGMENT." This restriction means an [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) allocation only ever performs whole-pageblock claims, never sub-pageblock ones. The purpose is to avoid the partial-conversion case that produces mixed pageblocks; if no whole-pageblock claim is possible, the slowpath above [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) reiterates the zonelist with [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) cleared and tries [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) again with the relaxed rule.

[`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479) sets [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) when [`defrag_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) is enabled:

```c
alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, alloc_flags);

if (defrag_mode)
	alloc_flags |= ALLOC_NOFRAGMENT;

return alloc_flags;
```

[`defrag_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) is a boot-time tunable that biases the allocator toward keeping pageblocks pure with respect to migratetype. With [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) set, [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) raises `min_order` to [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h) and so only converts pageblocks that are already entirely free (the buddy at order >= [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h) covers the whole block, so no in-use foreign-typed pages remain after the [`set_pageblock_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L556) flip). [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) is skipped entirely (the gating is in [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473), see below). Together these two restrictions mean an [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) request never produces a pageblock with a recorded migratetype that disagrees with any of its in-use pages.

### Watermark boosting and ZONE_BOOSTED_WATERMARK

[`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3393) is the only consumer of [`ZONE_BOOSTED_WATERMARK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h):

```c
out:
	/* Separate test+clear to avoid unnecessary atomics */
	if ((alloc_flags & ALLOC_KSWAPD) &&
	    unlikely(test_bit(ZONE_BOOSTED_WATERMARK, &zone->flags))) {
		clear_bit(ZONE_BOOSTED_WATERMARK, &zone->flags);
		wakeup_kswapd(zone, 0, 0, zone_idx(zone));
	}
```

The bit is set inside [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307) by `set_bit(ZONE_BOOSTED_WATERMARK, &zone->flags)` after a successful [`boost_watermark()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2188), and only if [`alloc_flags & ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) is set on the request (because waking kswapd makes no sense if the caller has [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) cleared). The two-step test+clear pattern in [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3393) avoids the atomic [`clear_bit()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/bitops/atomic.h) on the common path where the bit is not set; only when [`test_bit()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/bitops/non-atomic.h) returns true does the slower atomic clear-and-wake fire.

### Caller descent

The full descent from a userspace allocation to [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) follows the same path as for [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) and [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463), just deeper into the switch. The caller-side initialization is identical:

```c
if (!page) {
	enum rmqueue_mode rmqm = RMQUEUE_NORMAL;

	page = __rmqueue(zone, order, migratetype, alloc_flags, &rmqm);
```

The mode does not start at [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464); it gets there only by fall-through from earlier cases that found their freelists empty. Inside [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542), the variable `rmqm` may be advanced by an earlier iteration to [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) or [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465), but [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) is the one mode that explicitly resets to [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) on success, since it has just produced a fresh supply of native pages.

### Tracepoint emission

[`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) emits `trace_mm_page_alloc_extfrag` on every successful claim. The recorded fields are `page` (the page returned), `order` (the requested allocation order), `current_order` (the order of the buddy that was claimed, always at least `order`), `start_migratetype` (the requested migratetype, i.e. the destination), and `fallback_mt` (the foreign migratetype that was converted, i.e. the source). The trace event also fires inside [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437), so trace consumers must compare `current_order` against [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pageblock-flags.h) to distinguish whole-block claims from sub-pageblock claims, and against `order` to distinguish claims from steals (a steal returns the buddy at exactly `current_order`, while a claim may have converted a much larger buddy).

### History

[`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) is the first half of what used to be a single `__rmqueue_fallback()` function. Commit 2c847f27c37d ("mm: page_alloc: speed up fallbacks in `rmqueue_bulk()`") split the old function into [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) (whole-or-mostly-whole pageblock conversion) and [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) (single-page poaching), then introduced [`enum rmqueue_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461) so that [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) iterations could remember which of the two had already been tried (and either failed, or succeeded with consequences for subsequent iterations).

The earlier commit c2f6ea38fc1b ("mm: page_alloc: don't steal single pages from biggest buddy") introduced the conceptual split that motivated the function split. Before c2f6ea38fc1b, the fallback path searched top-down for the biggest buddy, then either claimed the whole block or stole just the requested chunk from it. According to the c2f6ea38fc1b commit message "Fix this by committing to the new behavior: either steal the whole block, or fall back to the smallest buddy. Remove single-page stealing from `steal_suitable_fallback()`. Rename it to `try_to_steal_block()` to make the intentions clear. If this fails, always fall back to the smallest buddy." This earlier fix established that "claim whole block from biggest buddy" and "steal one page from smallest buddy" are two distinct modes, and the 2c847f27c37d patch merely formalized that distinction in the [`enum rmqueue_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461) state machine.

The function name went through one rename in the process. c2f6ea38fc1b renamed `steal_suitable_fallback()` to `try_to_steal_block()`. 2c847f27c37d then renamed `try_to_steal_block()` to [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307) to align with the new "claim" terminology used by [`enum rmqueue_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461) and [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382). The terminology distinction is that "claim" means converting a whole or mostly-whole pageblock, while "steal" means taking a single page without converting the block; these are now separate functions calling different leaf operations.
