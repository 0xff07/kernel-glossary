---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# RMQUEUE_CMA

[`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) is the second case label of [`enum rmqueue_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461). It diverts a movable allocation from the normal [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66) freelist to the [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) freelist via [`__rmqueue_cma_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1953), which is itself a thin wrapper that calls [`__rmqueue_smallest(zone, order, MIGRATE_CMA)`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914). The mode is gated by [`alloc_flags & ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286), which [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776) sets only when the gfp migratetype is [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66).

```
    enum rmqueue_mode state machine (RMQUEUE_CMA highlighted)
    ─────────────────────────────────────────────────────────

       RMQUEUE_NORMAL  (fall-through on __rmqueue_smallest miss)
              │
              ▼
       ┌─────────────────────────┐
       │      RMQUEUE_CMA        │  ◀── *mode set here on success
       │                         │      (sticky across rmqueue_bulk
       │  if (alloc_flags &      │       iterations until CMA also
       │       ALLOC_CMA) {      │       runs out)
       │     page = __rmqueue_   │
       │       cma_fallback();   │
       │     if (page) {         │
       │       *mode = RMQUEUE_  │
       │              CMA;       │
       │       return page;      │
       │     }                   │
       │  }                      │
       └───────────┬─────────────┘
                   │
            page found?
            ┌──────┴──────┐
            │             │
           YES            NO ─ fallthrough ──▶ RMQUEUE_CLAIM
            │
            ▼
       return page  (*mode = RMQUEUE_CMA)


    Pre-switch CMA balancing (top of __rmqueue, runs every call)
    ───────────────────────────────────────────────────────────

       if (CONFIG_CMA &&
           (alloc_flags & ALLOC_CMA) &&
           NR_FREE_CMA_PAGES > NR_FREE_PAGES / 2) {

           page = __rmqueue_cma_fallback(zone, order);
           if (page) return page;     // *mode is NOT touched here
       }

       This pre-check exists to keep the CMA pool from monopolizing
       free memory.  When more than half of the zone's free pages
       sit in CMA blocks, biased CMA satisfaction is preferred over
       MIGRATE_MOVABLE so that future unmovable allocations have
       headroom in non-CMA pageblocks.


    ALLOC_CMA gating (gfp_to_alloc_flags_cma)
    ─────────────────────────────────────────

       gfp_t   →   gfp_migratetype(gfp_mask)   →   alloc_flags

       __GFP_MOVABLE  → MIGRATE_MOVABLE  → ALLOC_CMA set
       __GFP_RECLAIMABLE → MIGRATE_RECLAIMABLE → ALLOC_CMA *not* set
       (default)      → MIGRATE_UNMOVABLE → ALLOC_CMA *not* set

       Only MIGRATE_MOVABLE allocations may visit RMQUEUE_CMA, because
       CMA's contiguous-memory contract requires that pages parked in
       CMA pageblocks be migratable on demand.
```

## SUMMARY

[`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) reaches the [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) reservoir, which sits alongside the regular migratetypes in [`zone->free_area[].free_list[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) but holds pages whose physical contiguity is reserved for CMA users. According to the comment on [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) "Only movable pages can be allocated from MIGRATE_CMA pageblocks and page allocator never implicitly change migration type of MIGRATE_CMA pageblock", a movable allocation may borrow these pages but they remain owned by CMA and cannot pollute non-CMA pageblocks.

[`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) is reached in two ways. The first is by fall-through from [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) when the requested-migratetype freelist is empty. The second is via the unconditional pre-switch check at the top of [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) that runs even when the [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66) freelist is healthy, deliberately preferring CMA when the CMA pool exceeds half of [`NR_FREE_PAGES`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h). The pre-check does not advance `*mode`, so it has no effect on sticky-mode bookkeeping. Only the in-switch case writes `*mode = RMQUEUE_CMA`, making subsequent [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) iterations skip the doomed [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) call that just failed.

## SPECIFICATIONS

(none; [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) is a Linux kernel internal enum value; CMA itself is documented in the kernel tree)

## LINUX KERNEL

### Enum value

- [`'\<enum rmqueue_mode\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461): four-value enum used by [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) to remember the last successful allocation mode; [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) is value 1

### GFP-to-alloc_flags mapping

- [`'\<gfp_to_alloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479): translates the gfp mask into [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h); calls [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776) before returning
- [`'\<gfp_to_alloc_flags_cma\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776): adds [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) when [`gfp_migratetype(gfp_mask) == MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h); the only place [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) is set in the entire allocator

### Producers (call sites that initialize `rmqm` and pass `ALLOC_CMA`)

- [`'\<rmqueue_buddy\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222): per-call entry that initializes `rmqm = RMQUEUE_NORMAL` and passes [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) (carrying [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286)) into [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473)
- [`'\<rmqueue_bulk\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542): bulk refiller; shared `rmqm` survives across `count` iterations, so a sticky [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) skips the empty native [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66) freelist on subsequent iterations

### Consumer (the dispatch and the pre-switch check)

- [`'\<__rmqueue\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473): runs the over-half pre-switch CMA check at the top, then enters the switch where the [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) case label lives. The pre-check leaves `*mode` untouched; the in-switch case writes `*mode = RMQUEUE_CMA` on success

### Allocation primitives (the case body)

- [`'\<__rmqueue_cma_fallback\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1953): one-line wrapper that calls [`__rmqueue_smallest(zone, order, MIGRATE_CMA)`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914); the only entry point to the [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) freelist (with `CONFIG_CMA` off it returns `NULL`)
- [`'\<__rmqueue_smallest\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914): per-order ascent on the requested-migratetype freelist; with [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) it operates on [`free_list[MIGRATE_CMA]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) and the leftover halves go back onto the same list

### Buddy-split helpers (used by the allocation primitive)

- [`'\<page_del_and_expand\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1755): wrapper around [`__del_page_from_free_list()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L883), [`expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727), and [`account_freepages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L724); invoked with `migratetype = MIGRATE_CMA` so leftovers stay on the CMA list
- [`'\<expand\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727): subdivides the buddy down to the requested order; the migratetype is preserved across all push-backs to keep [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) pageblocks pure
- [`'\<get_page_from_free_area\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L912): peeks the head of [`area->free_list[MIGRATE_CMA]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138)

### Per-zone counters consulted by the over-half pre-check

- [`'\<zone_page_state\>':'include/linux/vmstat.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/vmstat.h): reads atomic per-zone counters used by the over-half rule at the top of [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473)
- [`NR_FREE_CMA_PAGES`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): per-zone counter incremented when a [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) page is freed
- [`NR_FREE_PAGES`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): per-zone total free count across all migratetypes including CMA; the over-half rule fires when [`NR_FREE_CMA_PAGES > NR_FREE_PAGES / 2`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2483)

### Migratetype and freelists

- [`'\<enum migratetype\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L64): defines [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) when `CONFIG_CMA` is enabled, with the comment "MIGRATE_CMA migration type is designed to mimic the way ZONE_MOVABLE works"
- [`'\<struct free_area\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138): contains [`free_list[MIGRATE_TYPES]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138), one of which is the [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) list
- [`fallbacks[MIGRATE_PCPTYPES][MIGRATE_PCPTYPES - 1]`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1946): does NOT list [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) as a source or target, so [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) is unreachable from [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) and [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465)

### Concurrency

- [`zone->lock`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): held by [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) and [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) across the call to [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) and across the entire bulk loop, ensuring `*mode` advances against a stable freelist set

### Flag definition

- [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286): bit `0x80` defined in [`mm/internal.h`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h). Comment "allow allocations from CMA areas"

### Tracepoints

- [`'\<trace_mm_page_alloc_zone_locked\>':'include/trace/events/kmem.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/trace/events/kmem.h#L238): emitted by [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) inside [`__rmqueue_cma_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1953); the migratetype field shows [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) so traces can distinguish CMA satisfaction from native [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66) satisfaction

## KERNEL DOCUMENTATION

- [`Documentation/admin-guide/mm/cma_debugfs.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/admin-guide/mm/cma_debugfs.rst): describes the CMA debugfs interface; useful for reading the live size and used count of a CMA region
- [`Documentation/mm/page_allocation.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/page_allocation.rst): page allocator overview that places CMA in context with the other migratetypes
- [`Documentation/mm/physical_memory.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/physical_memory.rst): describes [`struct zone`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) and the per-zone counters [`NR_FREE_PAGES`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) and [`NR_FREE_CMA_PAGES`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) consulted by the over-half check

## OTHER SOURCES

- [LWN: A reworked contiguous memory allocator](https://lwn.net/Articles/447405/)
- [LWN: CMA and ARM](https://lwn.net/Articles/450286/)
- [LKML: mm: page_alloc: speed up fallbacks in rmqueue_bulk() (Johannes Weiner, Apr 2025)](https://lore.kernel.org/lkml/20250407180154.63348-1-hannes@cmpxchg.org/)

## DETAILS

### MIGRATE_CMA in the migratetype enum

CMA pageblocks are tracked as a distinct migratetype. The relevant fragment of [`include/linux/mmzone.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) makes the contract explicit:

```c
enum migratetype {
	MIGRATE_UNMOVABLE,
	MIGRATE_MOVABLE,
	MIGRATE_RECLAIMABLE,
	MIGRATE_PCPTYPES,	/* the number of types on the pcp lists */
	MIGRATE_HIGHATOMIC = MIGRATE_PCPTYPES,
#ifdef CONFIG_CMA
	/*
	 * MIGRATE_CMA migration type is designed to mimic the way
	 * ZONE_MOVABLE works.  Only movable pages can be allocated
	 * from MIGRATE_CMA pageblocks and page allocator never
	 * implicitly change migration type of MIGRATE_CMA pageblock.
	 *
	 * The way to use it is to change migratetype of a range of
	 * pageblocks to MIGRATE_CMA which can be done by
	 * __free_pageblock_cma() function.
	 */
	MIGRATE_CMA,
	__MIGRATE_TYPE_END = MIGRATE_CMA,
#else
	__MIGRATE_TYPE_END = MIGRATE_HIGHATOMIC,
#endif
#ifdef CONFIG_MEMORY_ISOLATION
	MIGRATE_ISOLATE,	/* can't allocate from here */
#endif
	MIGRATE_TYPES
};
```

The "page allocator never implicitly change migration type" rule has two consequences for [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463). [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) and [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) skip [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) because [`fallbacks[][]`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1946) is sized as `[MIGRATE_PCPTYPES][MIGRATE_PCPTYPES - 1]` (4-by-3, indexable only with [`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L65), [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66), and [`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L67)), so neither steal nor claim can ever target a [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) block. The only way to allocate from [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) is therefore through the dedicated [`__rmqueue_cma_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1953) call inside [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473), which goes through [`__rmqueue_smallest(zone, order, MIGRATE_CMA)`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) and operates only on [`free_list[MIGRATE_CMA]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138).

### __rmqueue_cma_fallback wrapper

[`__rmqueue_cma_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1953) is a one-line wrapper:

```c
#ifdef CONFIG_CMA
static __always_inline struct page *__rmqueue_cma_fallback(struct zone *zone,
					unsigned int order)
{
	return __rmqueue_smallest(zone, order, MIGRATE_CMA);
}
#else
static inline struct page *__rmqueue_cma_fallback(struct zone *zone,
					unsigned int order) { return NULL; }
#endif
```

The `#else` branch returns `NULL` unconditionally when `CONFIG_CMA` is not enabled. The dead-code branches inside [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) that test [`alloc_flags & ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) become free to elide when the bit is hard-coded to zero (the [`#else`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1290) branch in [`mm/internal.h`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) defines `ALLOC_NOFRAGMENT 0` analogously, and the same kind of compile-time elision applies).

### The pre-switch over-half rule

The first thing [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) does, before the switch on `*mode`, is consult the CMA balance:

```c
static __always_inline struct page *
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

	switch (*mode) {
	...
```

The over-half rule prevents the [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) reservoir from monopolizing the zone's free memory. According to the in-tree comment, "Balance movable allocations between regular and CMA areas by allocating from CMA when over half of the zone's free memory is in the CMA area." When more than half of [`NR_FREE_PAGES`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) sits in CMA blocks, draining CMA first preserves headroom in the non-CMA pageblocks for unmovable and reclaimable allocations that cannot use CMA. This pre-check does not write `*mode`, so it does not interact with the sticky-mode bookkeeping done by the in-switch case.

### The case body

The [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) arm of the switch does write `*mode` on success:

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
	...
```

The [`alloc_flags & ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) guard ensures that non-movable allocations skip [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) entirely. On success, `*mode = RMQUEUE_CMA` is sticky across [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) iterations. The next iteration enters [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) at `case RMQUEUE_CMA:` and skips the [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) call that just failed in iteration N. According to the cover letter for commit 2c847f27c37d "Since rmqueue_bulk() holds the zone->lock over the entire batch, the freelists are not subject to outside changes; when the search for a block to claim has already failed, there is no point in trying again for the next page." The same reasoning applies to the empty [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66) freelist that pushed the mode to [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) in the first place.

### ALLOC_CMA gating

[`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) is set by [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776), called from [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479). The function is a single conditional:

```c
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

[`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L67) and [`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L65) allocations therefore never see [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) and consequently can never reach [`__rmqueue_cma_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1953). CMA's contiguous-memory contract requires that pages currently parked in a CMA pageblock be migratable on demand when the CMA owner asks for the contiguous range back, and reclaimable or unmovable pages cannot satisfy that requirement.

The bit is defined in [`mm/internal.h`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h):

```c
#define ALLOC_CMA		 0x80 /* allow allocations from CMA areas */
#ifdef CONFIG_ZONE_DMA32
#define ALLOC_NOFRAGMENT	0x100 /* avoid mixing pageblock types */
#else
#define ALLOC_NOFRAGMENT	  0x0
#endif
```

### Two distinct paths into __rmqueue_cma_fallback

[`__rmqueue_cma_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1953) is invoked from two places inside [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473), and they have different semantics. The pre-switch check at the top runs unconditionally on every [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) entry and bypasses the switch entirely on success without writing `*mode`. The in-switch [`case RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2509) runs only when [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) has just failed (or when sticky mode jumped here directly), and on success writes `*mode = RMQUEUE_CMA`. The pre-switch check fires in healthy zones where the requested-migratetype freelist is fine but CMA happens to be over half full; the in-switch case fires in pressured zones where the requested freelist has run dry.

The asymmetric `*mode` handling falls out of the design. The pre-switch firing means the [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66) freelist is healthy, so the next [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) iteration should re-enter at [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) and succeed cheaply (no need to be sticky). The in-switch firing means the [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66) freelist was empty just now and is still empty, so the next iteration should skip it (sticky [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463)).

### __rmqueue_smallest with MIGRATE_CMA

The actual page extraction is done by [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914), the same function that backs [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462). The only difference is the `migratetype` argument, which is hard-coded to [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) by [`__rmqueue_cma_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1953):

```c
static __always_inline
struct page *__rmqueue_smallest(struct zone *zone, unsigned int order,
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

The leftover halves produced by [`expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727) inside [`page_del_and_expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1755) go back onto [`free_list[MIGRATE_CMA]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138), so [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) pageblocks remain pure. The tracepoint at the bottom records `migratetype = MIGRATE_CMA`, distinguishing CMA-satisfied movable allocations from native [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66) ones in trace output.

### Counters that drive the over-half rule

[`zone_page_state(zone, NR_FREE_CMA_PAGES)`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/vmstat.h) reads a per-zone counter that tracks pages currently sitting on [`free_list[MIGRATE_CMA]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138). [`zone_page_state(zone, NR_FREE_PAGES)`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/vmstat.h) reads the total free count across all migratetypes including CMA. Both counters are vmstat atomic counters, updated by [`__mod_zone_page_state()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/vmstat.h) on every freelist insertion and removal, so the over-half check is a pair of atomic loads.

The rule [`NR_FREE_CMA_PAGES > NR_FREE_PAGES / 2`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2483) means "more than half of all free pages happen to be in CMA blocks." This is the only place in [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) where the [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82) reservoir is preferred over the requested-migratetype freelist; everywhere else, [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) on the requested type is tried first.

### Caller descent

Every [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) firing originates from one of two callers of [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473). [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) handles single-page high-order requests:

```c
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

[`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) handles per-CPU pageset refills:

```c
for (i = 0; i < count; ++i) {
	struct page *page = __rmqueue(zone, order, migratetype,
				      alloc_flags, &rmqm);
	if (unlikely(page == NULL))
		break;
```

In [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) the local `rmqm` is discarded after one call, so a [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) success affects only that single allocation. In [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) the `rmqm` survives across all `count` iterations, so a single sticky [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) skip optimization can compound into a measurable cost reduction over a batch of dozens of pages.

### History

The sticky [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) state was introduced together with the rest of [`enum rmqueue_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461) by commit 2c847f27c37d ("mm: page_alloc: speed up fallbacks in `rmqueue_bulk()`") in April 2025. Before the patch, [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) called [`__rmqueue_cma_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1953) only as a one-shot fallback after [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) failed, with no memory across [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) iterations. The pre-switch over-half check is older (it predates [`enum rmqueue_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461)) and was not touched by the speed-up patch; it was added much earlier as a balancing heuristic to keep the CMA pool from monopolizing the zone, and survives unchanged at the top of [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) as the only `*mode`-independent CMA path.
