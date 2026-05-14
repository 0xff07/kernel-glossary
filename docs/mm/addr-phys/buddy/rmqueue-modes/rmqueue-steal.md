---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# RMQUEUE_STEAL

[`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) takes a single page from a foreign-migratetype freelist without changing the migratetype recorded for the surrounding pageblock. The work is done by [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437), which scans foreign-migratetype freelists from the requested order upward and takes the smallest sufficiently-large buddy. Because the pageblock's migratetype stays at its original (foreign) value, the page returned to the caller now sits in a pageblock whose recorded migratetype does not match the use the page is being put to. When the caller eventually calls [`__free_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) on it, the free path consults [`get_pageblock_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) and deposits the page on the original (foreign) freelist, so the mismatch is self-healing on free. The mode is gated by [`!(alloc_flags & ALLOC_NOFRAGMENT)`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288): when the caller insists on no fragmentation, the steal path is skipped entirely.

```
    enum rmqueue_mode state machine (RMQUEUE_STEAL highlighted)
    ──────────────────────────────────────────────────────────

      RMQUEUE_NORMAL    RMQUEUE_CMA    RMQUEUE_CLAIM
              │              │              │
              └──────────────┴──────────────┘   (fall-through chain)
                             │
                             ▼
       ┌────────────────────────────────────┐
       │           RMQUEUE_STEAL            │
       │                                    │
       │  if (!(alloc_flags &               │
       │        ALLOC_NOFRAGMENT)) {        │
       │    page = __rmqueue_steal();       │
       │    if (page) {                     │
       │      *mode = RMQUEUE_STEAL;        │
       │      return page;                  │
       │    }                               │
       │  }                                 │
       └─────────────────┬──────────────────┘
                         │
                  page found?
                  ┌──────┴──────┐
                  │             │
                 YES            NO
                  │             │
                  ▼             ▼
          return page      return NULL
       (*mode = STEAL)     (allocator slowpath kicks in:
                            reclaim, compaction, retry)


    __rmqueue_steal scan (bottom-up, smallest buddy first)
    ──────────────────────────────────────────────────────

      order:  r       r+1     r+2     ...     MAX
              ┌──┐    ┌──┐    ┌──┐            ┌──┐
              │  │    │ X│    │  │            │  │  free_area[].free_list[FALLBACK]
              └──┘    └──┘    └──┘            └──┘
                       ▲
                       │ for current_order = order to NR_PAGE_ORDERS-1:
                       │     fallback_mt = find_suitable_fallback(area,
                       │                                          current_order,
                       │                                          start_migratetype,
                       │                                          claimable=false)
                       │     if fallback_mt == -1: continue
                       │     page = get_page_from_free_area(area, fallback_mt)
                       │     page_del_and_expand(zone, page, order,
                       │                         current_order, fallback_mt)
                       │     return page
                       │
                       │ Note: leftovers from expand() go back to the
                       │ FALLBACK migratetype's freelist, NOT to
                       │ start_migratetype's.  The block stays mixed.


    Sticky mode behavior across rmqueue_bulk iterations
    ──────────────────────────────────────────────────

      iteration N:    enter at NORMAL → fail → CMA fail → CLAIM fail
                      → STEAL succeed → *mode = RMQUEUE_STEAL
                      
      iteration N+1:  enter at STEAL directly (skip NORMAL/CMA/CLAIM)
                      → STEAL succeed → *mode = RMQUEUE_STEAL
                      
      iteration N+k:  STEAL fails → return NULL → rmqueue_bulk loop breaks
                      → caller (alloc_pages_bulk_noprof or
                                 __rmqueue_pcplist) handles the short
                                 batch
```

## SUMMARY

[`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) and [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) differ along two axes. [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) scans from [`MAX_PAGE_ORDER`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) downward and calls [`set_pageblock_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L556) on the pageblock it harvests; [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) scans from `order` upward and never touches [`pageblock_flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h). [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) on success resets `*mode = RMQUEUE_NORMAL` because the claim has populated the requested-migratetype freelist; [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) on success writes `*mode = RMQUEUE_STEAL` because the requested-migratetype freelist is still empty and the next [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) iteration should resume from the steal scan rather than re-walking the empty native and CMA lists.

[`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) exists because some allocations are not eligible for the claim path. [`should_try_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2230) returns `false` for small movable allocations, on the grounds that compaction can later relocate the misplaced pages. For those requests, the choice is between stealing a single page (without converting the pageblock) or failing the allocation. According to commit c2f6ea38fc1b ("mm: page_alloc: don't steal single pages from biggest buddy"), the codified rule is to take "either... the whole block, or fall back to the smallest buddy"; the first half is [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464), the second is [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465).

## SPECIFICATIONS

(none; [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) is a Linux kernel internal enum value)

## LINUX KERNEL

### Enum value

- [`'\<enum rmqueue_mode\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461): four-value enum used by [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) to remember the last successful allocation mode; [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) is value 3 (the highest)

### GFP-to-alloc_flags mapping

- [`'\<gfp_to_alloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479): sets [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) when [`defrag_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) is enabled; with the flag set, [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) skips the [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) call inside the [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) case

### Producers (call sites that initialize `rmqm` and supply `alloc_flags`)

- [`'\<rmqueue_buddy\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222): per-call buddy-allocator entry; the local `rmqm` is discarded after one call so the [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) write to `*mode` has no effect across calls
- [`'\<rmqueue_bulk\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542): bulk refiller; the `rmqm` here survives across `count` iterations, so a `*mode = RMQUEUE_STEAL` write makes every subsequent iteration in the same bulk call enter [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) at the steal case label and skip the three earlier cases

### Consumer (the dispatch)

- [`'\<__rmqueue\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473): the [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) case checks [`!(alloc_flags & ALLOC_NOFRAGMENT)`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288), calls [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437), and on success writes `*mode = RMQUEUE_STEAL`. This is the only mode whose case body is gated by an `alloc_flags` check

### Steal implementation (the case body)

- [`'\<__rmqueue_steal\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437): bottom-up scan from `order` to [`NR_PAGE_ORDERS - 1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), taking the first foreign-type buddy and splitting it via [`page_del_and_expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1755) with `migratetype = fallback_mt` so leftovers stay on the foreign freelist

### Fallback selection

- [`'\<find_suitable_fallback\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2278): called with `claimable=false`, returns either a fallback migratetype or `-1` (never `-2`); the `should_try_claim_block` gate is bypassed entirely

### Buddy-split helpers (used by the steal implementation)

- [`'\<get_page_from_free_area\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L912): peeks the head of [`area->free_list[fallback_mt]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138)
- [`'\<page_del_and_expand\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1755): unlinks the buddy via [`__del_page_from_free_list()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L883) and splits it via [`expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727); leftovers go back onto the foreign migratetype's freelist
- [`'\<__del_page_from_free_list\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L883): unlinks a page from [`free_area->free_list[fallback_mt]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) and decrements [`free_area->nr_free`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L140)
- [`'\<expand\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727): subdivides the chosen buddy and pushes the leftover halves onto [`free_area[].free_list[fallback_mt]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) (the foreign list, NOT the requested migratetype's list)

### Concurrency

- [`zone->lock`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): held by [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) and [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) across the entire steal scan, so the [`fallbacks[][]`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1946) freelists are stable while [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) walks them; this is what makes the sticky `*mode = RMQUEUE_STEAL` skip optimization safe across [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) iterations

### Migratetype and fallback table

- [`'\<enum migratetype\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L64): includes the migratetypes that can be steal-targets; [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) and [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L69) are excluded by the [`fallbacks[][]`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1946) table layout
- [`fallbacks[MIGRATE_PCPTYPES][MIGRATE_PCPTYPES - 1]`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1946): preference order is [`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L65) -> { [`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L67), [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66) }, [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66) -> { [`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L67), [`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L65) }, [`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L67) -> { [`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L65), [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66) }
- [`'\<struct free_area\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138): per-(zone, order) array whose [`free_list[fallback_mt]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) is the source of every stolen page

### Flag definition

- [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288): bit `0x100` (zero when [`CONFIG_ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig) is off). When set, gates the entire [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) case body off

### Tracepoints

- [`'\<trace_mm_page_alloc_extfrag\>':'include/trace/events/kmem.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/trace/events/kmem.h#L269): emitted by [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) on success; the `current_order` field equals the buddy order taken (which equals `order` when the freelist had a buddy at exactly the request size, larger when [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) had to ascend orders to find one). Distinguishable from a [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) emission of the same event by checking that `current_order == order` (steal) vs. `current_order > order` or `>= pageblock_order` (claim)

## KERNEL DOCUMENTATION

- [`Documentation/mm/page_allocation.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/page_allocation.rst): page allocator overview that motivates pageblock-grouped fallback and identifies single-page poaching as a fragmentation source
- [`Documentation/mm/balance.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/balance.rst): describes how watermarks and reclaim relate to fallback pressure
- [`Documentation/mm/physical_memory.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/physical_memory.rst): describes [`struct zone`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) and the per-zone [`free_area[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) array

## OTHER SOURCES

- [LKML: mm: page_alloc: don't steal single pages from biggest buddy (Johannes Weiner, Feb 2025)](https://lore.kernel.org/lkml/20250225001023.1494422-2-hannes@cmpxchg.org/)
- [LKML: mm: page_alloc: speed up fallbacks in rmqueue_bulk() (Johannes Weiner, Apr 2025)](https://lore.kernel.org/lkml/20250407180154.63348-1-hannes@cmpxchg.org/)
- [LWN: Fixing page-cluster sizing](https://lwn.net/Articles/745014/)

## DETAILS

### The case body

The [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) arm of the switch in [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) is the only one with an outer guard on `alloc_flags`:

```c
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
```

The [`!(alloc_flags & ALLOC_NOFRAGMENT)`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) guard means an [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) request never even attempts the steal path. The fall-through from [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) reaches this case, sees the guard, and falls off the end of the switch to `return NULL`. The slowpath in [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) reiterates the zonelist with [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) cleared (after invoking reclaim and compaction), at which point [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) becomes available.

On success, the case writes `*mode = RMQUEUE_STEAL` so the next [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) iteration enters [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) at this case label. The earlier cases ([`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462), [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463), [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464)) are skipped, since under the zone-lock invariant of [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) those freelists must still be empty. According to the cover letter for commit 2c847f27c37d "Since rmqueue_bulk() holds the zone->lock over the entire batch, the freelists are not subject to outside changes; when the search for a block to claim has already failed, there is no point in trying again for the next page." Skipping the three earlier cases is what the sticky mode does for performance: each skipped case avoids one [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) per-order ascent, one [`__rmqueue_cma_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1953) call, and one full [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) top-down scan that already failed once.

### __rmqueue_steal body

[`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) ascends the per-order [`zone->free_area[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) array from the requested order upward, taking the first available buddy:

```c
static __always_inline struct page *
__rmqueue_steal(struct zone *zone, int order, int start_migratetype)
{
	struct free_area *area;
	int current_order;
	struct page *page;
	int fallback_mt;

	for (current_order = order; current_order < NR_PAGE_ORDERS; current_order++) {
		area = &(zone->free_area[current_order]);
		fallback_mt = find_suitable_fallback(area, current_order,
						     start_migratetype, false);
		if (fallback_mt == -1)
			continue;

		page = get_page_from_free_area(area, fallback_mt);
		page_del_and_expand(zone, page, order, current_order, fallback_mt);
		trace_mm_page_alloc_extfrag(page, order, current_order,
					    start_migratetype, fallback_mt);
		return page;
	}

	return NULL;
}
```

The scan direction is bottom-up, from the request order toward [`NR_PAGE_ORDERS - 1`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h). This is the opposite direction from [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382), which scans top-down from [`MAX_PAGE_ORDER`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h). According to commit c2f6ea38fc1b "we just need to temporarily steal unmovable or reclaimable pages that are closest to the request size", the bottom-up direction is chosen to take the smallest possible buddy from a foreign block, leaving the larger buddies in that block intact for a future [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) attempt.

The migratetype passed to [`page_del_and_expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1755) is `fallback_mt`, not `start_migratetype`. [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) instead passes `start_type` to its tail call into [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) after [`set_pageblock_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L556) has flipped the pageblock. Here, [`expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727) deposits the leftover halves on [`free_area[].free_list[fallback_mt]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138), the foreign migratetype's list, so the pageblock retains its old type and the freelists stay consistent with the [`pageblock_flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) array (the c0cd6f557b90 hygiene rule). The single page handed to the caller now sits in a pageblock whose recorded migratetype does not match the use the page is being put to. When the caller eventually frees it, [`get_pageblock_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) returns `fallback_mt` and the page goes back onto the foreign freelist.

### find_suitable_fallback with claimable=false

[`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) calls [`find_suitable_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2278) with `claimable=false`, which short-circuits the [`should_try_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2230) gate:

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

With `claimable=false`, the function returns either a valid `fallback_mt` or `-1` (no fallback freelist has anything at this order). The caller never sees `-2`, so the steal scan does not abort early. This matches the steal mode's "take whatever you can get" semantics, in contrast to the claim mode's "only take it if it'd be worthwhile to convert the whole block."

### The fallbacks preference table

[`find_suitable_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2278) walks the [`fallbacks[][]`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1946) table in declaration order, returning the first foreign migratetype with a non-empty freelist. The table is a small static array:

```c
/*
 * This array describes the order lists are fallen back to when
 * the free lists for the desirable migrate type are depleted
 *
 * The other migratetypes do not have fallbacks.
 */
static int fallbacks[MIGRATE_PCPTYPES][MIGRATE_PCPTYPES - 1] = {
	[MIGRATE_UNMOVABLE]   = { MIGRATE_RECLAIMABLE, MIGRATE_MOVABLE   },
	[MIGRATE_MOVABLE]     = { MIGRATE_RECLAIMABLE, MIGRATE_UNMOVABLE },
	[MIGRATE_RECLAIMABLE] = { MIGRATE_UNMOVABLE,   MIGRATE_MOVABLE   },
};
```

A request for [`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L65) prefers stealing from [`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L67) over stealing from [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66), because polluting a reclaimable block with unmovable pages costs less than polluting a movable block with unmovable pages (movable blocks support compaction; polluting them blocks that). The dimensions `MIGRATE_PCPTYPES` (4) and `MIGRATE_PCPTYPES - 1` (3) mean only the three "basic" migratetypes are indexable. [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L69) and [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) are excluded as both targets and sources of fallback.

### The expand call deposits leftovers on the foreign list

[`page_del_and_expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1755) is the same wrapper used by [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914), and it forwards the `migratetype` argument unchanged to [`__del_page_from_free_list()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L883) and [`expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727):

```c
static __always_inline void page_del_and_expand(struct zone *zone,
						struct page *page, int low,
						int high, int migratetype)
{
	int nr_pages = 1 << high;

	__del_page_from_free_list(page, zone, high, migratetype);
	nr_pages -= expand(zone, page, low, high, migratetype);
	account_freepages(zone, -nr_pages, migratetype);
}
```

In the [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) call site, the `migratetype` argument is `fallback_mt` (the foreign type). [`expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727) walks the buddy tree halving the buddy down to the request order, and at each step pushes the leftover half back onto [`free_area[high].free_list[fallback_mt]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138):

```c
static inline unsigned int expand(struct zone *zone, struct page *page, int low,
				  int high, int migratetype)
{
	unsigned int size = 1 << high;
	unsigned int nr_added = 0;

	while (high > low) {
		high--;
		size >>= 1;
		VM_BUG_ON_PAGE(bad_range(zone, &page[size]), &page[size]);

		if (set_page_guard(zone, &page[size], high))
			continue;

		__add_to_free_list(&page[size], zone, high, migratetype, false);
		set_buddy_order(&page[size], high);
		nr_added += size;
	}

	return nr_added;
}
```

The [`__add_to_free_list()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) call uses `migratetype = fallback_mt`, so the leftover halves return to the source freelist. Only the single page at `size = 1 << order` (the requested-size head) is handed to the caller. The pageblock's recorded migratetype (in [`pageblock_flags[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h)) stays at `fallback_mt`.

### __rmqueue_steal vs __rmqueue_smallest

[`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) and [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) share the bottom-up scan structure, but differ in two ways. First, [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) scans only [`free_list[migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) (the requested type), while [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) consults [`find_suitable_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2278) on each iteration to choose a foreign list. Second, [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) returns leftovers to the requested-type list (no migration mismatch), while [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) returns leftovers to the foreign list (preserving the mismatch in the pageblock).

The two functions also differ in their tracepoint emission. [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) emits `trace_mm_page_alloc_zone_locked` (the generic "page came off a freelist" event). [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) emits `trace_mm_page_alloc_extfrag` (the "external fragmentation event" tracepoint) which records both the start and the fallback migratetypes, so consumers can quantify pollution events.

### Caller descent and the final return

Both [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) and [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) initialize `rmqm = RMQUEUE_NORMAL`, so [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) is reached only by full fall-through on the first call. In [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222), the `rmqm` write is discarded after the call returns; in [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542), the write makes subsequent iterations enter directly at [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) until the steal path itself fails:

```c
for (i = 0; i < count; ++i) {
	struct page *page = __rmqueue(zone, order, migratetype,
				      alloc_flags, &rmqm);
	if (unlikely(page == NULL))
		break;
	...
	list_add_tail(&page->pcp_list, list);
}
spin_unlock_irqrestore(&zone->lock, flags);

return i;
```

When [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) returns `NULL`, the loop breaks early and [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) returns the actual count refilled (which may be less than the requested `count`). [`__rmqueue_pcplist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3319), the caller, recomputes `pcp->count += alloced << order` and then checks whether the per-CPU list is still empty:

```c
do {
	if (list_empty(list)) {
		int batch = nr_pcp_alloc(pcp, zone, order);
		int alloced;

		alloced = rmqueue_bulk(zone, order,
				batch, list,
				migratetype, alloc_flags);

		pcp->count += alloced << order;
		if (unlikely(list_empty(list)))
			return NULL;
	}
```

If the bulk refill returned zero pages, the `list_empty()` check returns true and [`__rmqueue_pcplist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3319) returns `NULL`. The failure propagates up through [`rmqueue_pcplist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) and [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3393) to [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c), which moves on to the next zone in the zonelist or invokes [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) for reclaim, compaction, and OOM.

### Sticky mode benefit measurement

The cover letter for commit 2c847f27c37d quantifies the benefit of the sticky mode on `vm-scalability::lru-file-mmap-read`. According to the test report:

```
                                vanilla baseline
                                       ↓
   25525364 ±  3%     -56.4%   11135467 (regression introduced by c2f6ea38fc)
                       -57.8%   10779336 (after acc4d5ff0b merge tag)
                       +31.6%   33581409 (with this patch applied)
```

The "+31.6%" is calculated relative to the regressed baseline. According to the same report on Carlos Song's worst-case zone-lock hold-time measurements:

```
   2dd482ba627d (before freelist hygiene):    1ms
   c0cd6f557b90  (after freelist hygiene):   90ms
  next-20250319    (steal smallest buddy):  280ms
     this patch                          :    8ms
```

The 280 ms peak corresponds to a `rmqueue_bulk()` batch where every iteration walked the full claim and steal scans without remembering the previous failure. With sticky mode, the first iteration pays the full cost and sets `*mode = RMQUEUE_STEAL`; subsequent iterations skip directly to the case that just succeeded, reducing the total scan work by a factor of `count`.

### History

[`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) was extracted from `__rmqueue_fallback()` by commit 2c847f27c37d ("mm: page_alloc: speed up fallbacks in `rmqueue_bulk()`") in April 2025. The semantic split between "claim a block" and "steal a page" was established earlier by commit c2f6ea38fc1b ("mm: page_alloc: don't steal single pages from biggest buddy") in February 2025. According to the c2f6ea38fc1b commit message, "The fallback code searches for the biggest buddy first in an attempt to steal the whole block and encourage type grouping down the line. The approach used to be this: Non-movable requests will split the largest buddy and steal the remainder. This splits up contiguity, but it allows subsequent requests of this type to fall back into adjacent space. Movable requests go and look for the smallest buddy instead. The thinking is that movable requests can be compacted, so grouping is less important than retaining contiguity. c0cd6f557b90 ('mm: page_alloc: fix freelist movement during block conversion') enforces freelist type hygiene, which restricts stealing to either claiming the whole block or just taking the requested chunk; no additional pages or buddy remainders can be stolen any more. The patch mishandled when to switch to finding the smallest buddy in that new reality. As a result, it may steal the exact request size, but from the biggest buddy. This causes fracturing for no good reason. Fix this by committing to the new behavior: either steal the whole block, or fall back to the smallest buddy."

The performance and fragmentation tradeoffs from the `mmtest::thpchallenge` benchmark are recorded in the c2f6ea38fc1b commit message. According to the data, the patched kernel produces fewer single-page pollution events for the `[movable from unmovable]` combination (drops from 4841 to 868) at the cost of more for the `[movable from reclaimable]` combination (rises from 5278 to 12568). According to the same commit message, "The numbers for 'from movable' (the least preferred fallback option, and most detrimental to compactability) are down across the board."

The 2c847f27c37d patch then introduced [`enum rmqueue_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461) and made [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) a sticky-mode state machine, formalizing the four-way distinction between native, CMA, claim, and steal. [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) is the only one of the four whose case body is gated by an [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) check; an [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) request that has fallen through [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462), [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463), and [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) returns `NULL` from [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) and the slowpath retries the zonelist with [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) cleared rather than allowing the steal that would mix migratetypes within a pageblock.
