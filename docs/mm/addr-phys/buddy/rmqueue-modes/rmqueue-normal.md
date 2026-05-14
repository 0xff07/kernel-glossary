---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# RMQUEUE_NORMAL

[`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) is the entry state of [`enum rmqueue_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461). It calls [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) on the requested migratetype, which walks [`zone->free_area[order..NR_PAGE_ORDERS-1]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138), takes the first non-empty [`free_list[migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138), and splits the buddy down via [`expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727) with the same migratetype so the leftover halves go back onto the same freelist. The pageblock's recorded migratetype already matches the requested migratetype (otherwise the buddy would have been on a different freelist), so [`pageblock_flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is not touched and no migratetype-boundary crossing occurs.

```
    enum rmqueue_mode state machine (entry state highlighted)
    ─────────────────────────────────────────────────────────

         rmqueue_buddy / rmqueue_bulk
         ─────────────────────────────
                    │
                    │  enum rmqueue_mode rmqm = RMQUEUE_NORMAL
                    │
                    ▼
         ┌────────────────────────┐
         │     RMQUEUE_NORMAL     │ ◀──── *mode reset to NORMAL
         │                        │       after every successful CLAIM
         │  __rmqueue_smallest(   │
         │     zone, order,       │
         │     migratetype)       │
         └───────────┬────────────┘
                     │
              page found?
              ┌──────┴──────┐
              │             │
             YES            NO ─── fallthrough ──▶  RMQUEUE_CMA
              │
              ▼
         return page
         (*mode unchanged: stays NORMAL)


    __rmqueue_smallest internal scan (per-order ascent)
    ───────────────────────────────────────────────────

       order:   r       r+1       r+2     ...   MAX
                ┌───┐   ┌───┐    ┌───┐         ┌───┐
                │   │   │   │    │ X │         │   │  zone->free_area[].free_list[migratetype]
                └───┘   └───┘    └───┘         └───┘
                  ▲
                  │   loop: take first page of free_list[migratetype]
                  │   if empty, ++current_order and try again
                  │   when found at current_order = high:
                  │       __del_page_from_free_list(page, zone, high, migratetype)
                  │       expand(zone, page, low=order, high, migratetype)
                  │       (split high..low+1 buddies back onto same migratetype list)

       Result: a page of exactly 'order' pages, taken from the smallest
       buddy at order >= 'order' that is on the requested migratetype's
       freelist.  No foreign migratetype pages are touched, so pageblocks
       remain pure with respect to mobility grouping.
```

## SUMMARY

[`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) is the first case label of the switch inside [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) and the value that callers initialize their local [`enum rmqueue_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461) variable to. It maps to a single call to [`__rmqueue_smallest(zone, order, migratetype)`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914), which scans the per-order freelists for the requested migratetype and returns a page from the smallest buddy that satisfies the request. A failure here causes [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) to fall through to [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463). A success leaves `*mode` at [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) so that the next iteration of [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) starts again from the same case label.

[`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) is also the value that [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) writes back to `*mode` on success. After [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) calls [`set_pageblock_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L556) and either [`__move_freepages_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1967) or [`expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727) on the converted pageblock, [`free_list[migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) holds the leftover free pages from that pageblock, so the next [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) iteration is expected to satisfy its request from [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) on that list.

## SPECIFICATIONS

(none; [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) is a Linux kernel internal enum value)

## LINUX KERNEL

### Enum definition

- [`'\<enum rmqueue_mode\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461): four-value enum used by [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) to remember where the last allocation found pay dirt, with [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) as the entry / reset value

### Upstream call chain

- [`'\<get_page_from_freelist\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c): zone-walk caller that selects a candidate zone and calls [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3393)
- [`'\<rmqueue\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3393): single-zone entry that dispatches to [`rmqueue_pcplist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) for [`pcp_allowed_order()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) requests or to [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) for higher orders
- [`'\<__rmqueue_pcplist\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3319): pulls a page from the per-CPU list and on cache miss calls [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) to refill it

### Producers (call sites that initialize `*mode = RMQUEUE_NORMAL`)

- [`'\<rmqueue_buddy\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222): per-call buddy-allocator entry that declares `enum rmqueue_mode rmqm = RMQUEUE_NORMAL;` immediately before invoking [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473); `rmqm` is discarded after the single call
- [`'\<rmqueue_bulk\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542): refills the per-CPU pageset by looping [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) under a single zone-lock acquisition; the local [`rmqm`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2552) survives across iterations, so a successful [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) on iteration N keeps iteration N+1 starting at [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462)

### Consumer (the dispatch)

- [`'\<__rmqueue\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473): switch on `*mode` with `fallthrough` between cases; the [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) case calls [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) and returns the page on success without writing to `*mode`

### Allocation primitive (the case body)

- [`'\<__rmqueue_smallest\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914): per-order ascent through [`zone->free_area[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) on the requested migratetype; the entire body of the [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) case label

### Buddy-split helpers (used by the allocation primitive)

- [`'\<get_page_from_free_area\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L912): peeks the head of [`area->free_list[migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) via [`list_first_entry_or_null()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/list.h)
- [`'\<page_del_and_expand\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1755): wrapper around [`__del_page_from_free_list()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L883) and [`expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727) that also calls [`account_freepages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L724)
- [`'\<__del_page_from_free_list\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L883): unlinks a page from [`free_area->free_list[migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) and decrements [`free_area->nr_free`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L140)
- [`'\<expand\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727): subdivides a high-order buddy into successively smaller buddies, returning the leftover halves to [`zone->free_area[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) on the same migratetype

### Concurrency

- [`zone->lock`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): zone-wide spinlock acquired by [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) and [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) before invoking [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473); held across the entire bulk loop so that the freelists are stable while `*mode` advances
- [`ALLOC_TRYLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h): bit checked at the top of [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) and [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542); when set, the lock acquisition uses [`spin_trylock_irqsave()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock.h) and the call returns failure if the lock is contended

### Migratetype and freelists

- [`'\<struct free_area\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138): array of per-migratetype free lists plus a count of free pages at this order; one such struct per (zone, order) pair
- [`'\<enum migratetype\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L64): includes [`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L65), [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L66), [`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L67), [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L69), and (with `CONFIG_CMA`) [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L82); the `migratetype` argument that [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) takes is one of these
- [`'\<NR_PAGE_ORDERS\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): upper bound of the per-order scan inside [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914)

### Related types

- [`'\<struct per_cpu_pages\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L744): per-CPU pageset whose cache miss triggers [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542); sticky [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) only matters across calls to [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) that share an `rmqm` variable, and the bulk loop is the only such caller
- [`'\<struct zone\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): contains [`free_area[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138), the [`lock`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) field, and per-zone statistics

### Tracepoints

- [`'\<trace_mm_page_alloc_zone_locked\>':'include/trace/events/kmem.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/trace/events/kmem.h#L238): emitted by [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) on every successful native-migratetype allocation; this is the tracepoint that fires on the [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) path

## KERNEL DOCUMENTATION

- [`Documentation/mm/page_allocation.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/page_allocation.rst): high-level overview of the page allocator and the buddy system
- [`Documentation/mm/physical_memory.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/physical_memory.rst): describes [`struct zone`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) and the per-zone [`free_area[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) array that [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) walks
- [`Documentation/mm/balance.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/balance.rst): explains the watermark and reserve concepts that [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) consults before invoking [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3393)

## OTHER SOURCES

- [LWN: The Linux page allocator](https://lwn.net/Articles/224254/)
- [LKML: mm: page_alloc: speed up fallbacks in rmqueue_bulk() (Johannes Weiner, Apr 2025)](https://lore.kernel.org/lkml/20250407180154.63348-1-hannes@cmpxchg.org/)

## DETAILS

### Mode initialization at the call sites

[`enum rmqueue_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461) is declared inside [`mm/page_alloc.c`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) just above [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473):

```c
enum rmqueue_mode {
	RMQUEUE_NORMAL,
	RMQUEUE_CMA,
	RMQUEUE_CLAIM,
	RMQUEUE_STEAL,
};
```

The two callers that drive [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) declare a stack-local variable of this type, initialized to [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462), and pass its address into the callee.

[`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) is the per-call entry point used for orders that bypass the per-CPU pageset:

```c
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
```

[`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) is the bulk refiller invoked from [`__rmqueue_pcplist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3319) to top up a per-CPU pageset with `count` pages in one shot:

```c
static int rmqueue_bulk(struct zone *zone, unsigned int order,
			unsigned long count, struct list_head *list,
			int migratetype, unsigned int alloc_flags)
{
	enum rmqueue_mode rmqm = RMQUEUE_NORMAL;
	unsigned long flags;
	int i;

	if (unlikely(alloc_flags & ALLOC_TRYLOCK)) {
		if (!spin_trylock_irqsave(&zone->lock, flags))
			return 0;
	} else {
		spin_lock_irqsave(&zone->lock, flags);
	}
	for (i = 0; i < count; ++i) {
		struct page *page = __rmqueue(zone, order, migratetype,
					      alloc_flags, &rmqm);
		if (unlikely(page == NULL))
			break;
```

The `rmqm` lives across all `count` iterations of the loop because the zone lock is held the entire time. According to the comment in [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) "The fallback logic is expensive and rmqueue_bulk() calls in a loop with the zone->lock held, meaning the freelists are not subject to any outside changes", every loop iteration after a successful [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) will try [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) again, since `rmqm` was not advanced. The first iteration that fails [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) advances `rmqm` past it, and subsequent iterations skip the doomed [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) call entirely.

### The case body

The [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) arm of the switch in [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) is the smallest of the four:

```c
switch (*mode) {
case RMQUEUE_NORMAL:
	page = __rmqueue_smallest(zone, order, migratetype);
	if (page)
		return page;
	fallthrough;
case RMQUEUE_CMA:
	...
```

[`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) is invoked with the migratetype passed in by the caller (which is itself derived from [`gfp_migratetype(gfp_mask)`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h) inside [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c)). On success, the function returns the page directly without modifying `*mode`, so `*mode` stays at [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462). On failure, the `fallthrough` keyword (a [C23 attribute](https://en.cppreference.com/w/c/language/attributes) emulated by the kernel macro) drops control into the next case.

### __rmqueue_smallest body

[`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) ascends the per-order [`zone->free_area[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) array starting at the requested order:

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

The loop touches only [`free_list[migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) at every order and never inspects the lists of other migratetypes, so no foreign pages are moved across freelists and no [`pageblock_flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) entry is mutated. [`get_page_from_free_area()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L912) is a single [`list_first_entry_or_null()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/list.h) call:

```c
static inline struct page *get_page_from_free_area(struct free_area *area,
					    int migratetype)
{
	return list_first_entry_or_null(&area->free_list[migratetype],
					struct page, buddy_list);
}
```

[`page_del_and_expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1755) wraps the unlink and split of the buddy:

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

When `high > low`, [`expand()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1727) repeatedly halves the buddy and pushes the unused half back onto [`free_area[high].free_list[migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138), one push per descent step:

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

		/*
		 * Mark as guard pages (or page), that will allow to
		 * merge back to allocator when buddy will be freed.
		 * Corresponding page table entries will not be touched,
		 * pages will stay not present in virtual address space
		 */
		if (set_page_guard(zone, &page[size], high))
			continue;

		__add_to_free_list(&page[size], zone, high, migratetype, false);
		set_buddy_order(&page[size], high);
		nr_added += size;
	}

	return nr_added;
}
```

The leftover halves stay on the same migratetype, so the pageblock containing the allocation remains pure with respect to that migratetype.

### Behavior under sticky mode

The case-fall-through structure of [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) interacts with the sticky `*mode` value in three observable ways for [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462). A success on [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) leaves `*mode` unchanged, so the next iteration in [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) re-enters at the same case label. A failure on [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) does not write to `*mode`; instead the [`switch`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2503) reaches a later case that may write `*mode = RMQUEUE_CMA` (in [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) line 2513) or `*mode = RMQUEUE_STEAL` (line 2530), which changes the entry point on the next iteration. A success on [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) explicitly resets `*mode = RMQUEUE_NORMAL` because the claim has populated [`free_list[migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) with the leftover pages from the converted pageblock, so the next iteration is expected to find a page on that list via [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914).

The relevant fragment of [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) shows all three behaviors in close succession:

```c
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
```

According to the in-tree comment "Replenished preferred freelist, back to normal mode", a successful claim leaves new entries on [`free_list[migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138), and retrying [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) on the next call will find one without needing to re-scan the foreign freelists.

### Caller descent into RMQUEUE_NORMAL

The full descent from a userspace allocation to [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) goes through [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c). It calls [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3393), which dispatches to either [`rmqueue_pcplist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) for orders satisfied by [`pcp_allowed_order(order)`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) (the per-CPU pageset path) or [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) for higher orders:

```c
struct page *rmqueue(struct zone *preferred_zone,
			struct zone *zone, unsigned int order,
			gfp_t gfp_flags, unsigned int alloc_flags,
			int migratetype)
{
	struct page *page;

	if (likely(pcp_allowed_order(order))) {
		page = rmqueue_pcplist(preferred_zone, zone, order,
				       migratetype, alloc_flags);
		if (likely(page))
			goto out;
	}

	page = rmqueue_buddy(preferred_zone, zone, order, alloc_flags,
							migratetype);

out:
	/* Separate test+clear to avoid unnecessary atomics */
	if ((alloc_flags & ALLOC_KSWAPD) &&
	    unlikely(test_bit(ZONE_BOOSTED_WATERMARK, &zone->flags))) {
		clear_bit(ZONE_BOOSTED_WATERMARK, &zone->flags);
		wakeup_kswapd(zone, 0, 0, zone_idx(zone));
	}

	VM_BUG_ON_PAGE(page && bad_range(zone, page), page);
	return page;
}
```

The pcplist path eventually calls [`__rmqueue_pcplist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3319), which on cache miss invokes [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) to refill the per-CPU lists:

```c
static inline
struct page *__rmqueue_pcplist(struct zone *zone, unsigned int order,
			int migratetype,
			unsigned int alloc_flags,
			struct per_cpu_pages *pcp,
			struct list_head *list)
{
	struct page *page;

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

		page = list_first_entry(list, struct page, pcp_list);
		list_del(&page->pcp_list);
		pcp->count -= 1 << order;
	} while (check_new_pages(page, order));

	return page;
}
```

Both [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) and [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) start their interaction with [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) at [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462). The difference is that [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) calls [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) once per allocation and the local `rmqm` is discarded, while [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) calls it `count` times in a row and the local `rmqm` accumulates state across the loop. The performance benefit of remembering [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) is therefore concentrated in the bulk path; in the single-page buddy path the optimization is irrelevant because the variable goes out of scope after one call.

### RMQUEUE_NORMAL as the reset state

[`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) takes from [`free_list[migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) and never touches any other freelist or [`pageblock_flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) entry. [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2463) takes from [`free_list[MIGRATE_CMA]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) instead. [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) calls [`__move_freepages_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1967) and [`set_pageblock_migratetype()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L556) to migrate every free page in a foreign pageblock to [`free_list[migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) and update the pageblock's recorded migratetype. [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2465) takes a single page from a foreign-migratetype freelist without modifying [`pageblock_flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), so the page-to-pageblock-migratetype mismatch persists until the page is freed. Of these four operations, only [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2464) leaves new entries on [`free_list[migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138), so it is the only one that resets `*mode` back to [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462); the other three leave [`free_list[migratetype]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L138) empty as it was, so [`*mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) advances or stays at the case label that just succeeded, and the next bulk iteration short-circuits to that case.

### History

[`enum rmqueue_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2461) was introduced by commit 2c847f27c37d ("mm: page_alloc: speed up fallbacks in `rmqueue_bulk()`") in April 2025 by Johannes Weiner. The patch was a fix for a 56.4% throughput regression on `vm-scalability::lru-file-mmap-read` reported by the kernel test robot, and an unrelated worst-case zone-lock hold-time regression of 280 ms reported by Carlos Song on NXP. According to the cover letter, "Modify `__rmqueue()` to remember the last successful fallback mode, and restart directly from there on the next `rmqueue_bulk()` iteration." Before this patch, [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) was a straight-line function that always started at [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1914) (the equivalent of [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462)) on every call, even when the caller was a tight bulk loop that had just observed an empty native freelist on the previous iteration. The new sticky `*mode` shifts the behavior so that the [`RMQUEUE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2462) case is skipped in subsequent iterations of [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) when the freelist is known to be empty, with a measured `vm-scalability` throughput recovery from 11.1 M to 33.6 M (+31.6% beyond the pre-regression baseline).
