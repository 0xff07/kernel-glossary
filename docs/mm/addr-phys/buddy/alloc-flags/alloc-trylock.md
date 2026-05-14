---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# ALLOC_TRYLOCK

[`ALLOC_TRYLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1293) is the page allocator's "do not block on `zone->lock`" bit. It is set by exactly one producer, [`alloc_frozen_pages_nolock_noprof()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L7622), the entry point used by callers (notably BPF helpers) that need to allocate pages from contexts where waiting on a spinlock is unsafe (NMI, hard IRQ on PREEMPT_RT, or any context that holds an [`raw_spinlock_t`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock_types_raw.h)). When the bit is set, [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3231) and [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2550) call [`spin_trylock_irqsave(&zone->lock, flags)`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock.h) and return NULL immediately on failure rather than blocking; [`cond_accept_memory()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L7554) short-circuits because [`try_to_accept_memory_one()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) needs a lock; and [`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L5014) skips [`should_fail_alloc_page()`](https://elixir.bootlin.com/linux/v6.19/source/mm/fail_page_alloc.c) (the fault-injection probe) because it calls [`get_random_u32()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/prandom.h) and [`printk()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/printk.h) which both want locks.

```
        ALLOC_* bit field (12 bits live in v6.19)
        ─────────────────────────────────────────────────────────────
        bit:  11    10    9     8     7     6     5     4     3     2     1     0
              ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
              │  K  │  T  │ HA  │ NF  │ CMA │ CP  │ MR  │ NB  │ OOM │ NW  │  WMARK    │
              └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                      ▲
                      │
                  ALLOC_TRYLOCK = 0x400

        Legend: K  = ALLOC_KSWAPD          NF  = ALLOC_NOFRAGMENT     NB  = ALLOC_NON_BLOCK
                T  = ALLOC_TRYLOCK         CP  = ALLOC_CPUSET         OOM = ALLOC_OOM
                HA = ALLOC_HIGHATOMIC      MR  = ALLOC_MIN_RESERVE    NW  = ALLOC_NO_WATERMARKS
```

```
        Lock-free allocation path with ALLOC_TRYLOCK
        ────────────────────────────────────────────────────────────────

         BPF helper or other lock-sensitive context calls
         alloc_pages_nolock() (which wraps alloc_frozen_pages_nolock_noprof):

         alloc_frozen_pages_nolock_noprof:
             /* Forbid wait-on-spinlock contexts on PREEMPT_RT */
             if (CONFIG_PREEMPT_RT && (in_nmi() || in_hardirq()))
                     return NULL;
             if (!pcp_allowed_order(order))
                     return NULL;
             if (deferred_pages_enabled())
                     return NULL;

             alloc_gfp = __GFP_NOWARN | __GFP_ZERO | __GFP_NOMEMALLOC
                       | __GFP_COMP | gfp_flags;
             unsigned int alloc_flags = ALLOC_TRYLOCK;        <── set
             struct alloc_context ac = { };

             prepare_alloc_pages(alloc_gfp, order, nid, NULL,
                                 &ac, &alloc_gfp, &alloc_flags);
             /* prepare_alloc_pages skips should_fail_alloc_page when
                ALLOC_TRYLOCK is set (next page) */

             page = get_page_from_freelist(alloc_gfp, order,
                                            alloc_flags, &ac);
             /* No __alloc_pages_slowpath fallback. */

         get_page_from_freelist iterates the zonelist:
             cond_accept_memory(zone, order, alloc_flags):
                 if (alloc_flags & ALLOC_TRYLOCK)
                         return false;          <── short-circuit
                 ...
             ...
             page = rmqueue(zonelist_zone(...), zone, order,
                            gfp_mask, alloc_flags, ac->migratetype);

         rmqueue:
             if (pcp_allowed_order(order)) {
                     page = rmqueue_pcplist(...)   /* uses local_lock */
                     if (page) goto out;
             }
             page = rmqueue_buddy(...)

         rmqueue_buddy:
             if (unlikely(alloc_flags & ALLOC_TRYLOCK)) {
                     if (!spin_trylock_irqsave(&zone->lock, flags))
                             return NULL;       <── give up
             } else {
                     spin_lock_irqsave(&zone->lock, flags);
             }
             ...

         rmqueue_bulk (called via rmqueue_pcplist refill):
             if (unlikely(alloc_flags & ALLOC_TRYLOCK)) {
                     if (!spin_trylock_irqsave(&zone->lock, flags))
                             return 0;          <── give up
             } else {
                     spin_lock_irqsave(&zone->lock, flags);
             }
             ...
```

## SUMMARY

[`ALLOC_TRYLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1293) (= `0x400`) is bit 10 of [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c). It exists to support [`alloc_pages_nolock()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h) (which expands to [`alloc_frozen_pages_nolock_noprof()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L7622)), a variant of the page allocator that tolerates being called from contexts that cannot block on a spinlock, including NMI, hard IRQ on PREEMPT_RT, and any context that holds a [`raw_spinlock_t`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock_types_raw.h). The producer hard-codes the GFP mask to a "non-blocking, non-reserve, zeroing, no-warn" shape ([`__GFP_NOWARN | __GFP_ZERO | __GFP_NOMEMALLOC | __GFP_COMP`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) plus the caller's [`gfp_flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) and explicitly excludes [`__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) and [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) on the grounds that direct reclaim cannot run and waking kswapd from arbitrary contexts is unsafe.

The flag is read in five places. [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3231) and [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2550) wrap their [`zone->lock`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) acquisition in a [`spin_trylock_irqsave()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock.h) and return failure immediately if the lock is contended. [`cond_accept_memory()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L7554) skips its work because accepting unaccepted memory takes a lock. [`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L5014) skips the [`should_fail_alloc_page()`](https://elixir.bootlin.com/linux/v6.19/source/mm/fail_page_alloc.c) fault-injection probe because that probe calls [`get_random_u32()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/prandom.h) (which holds a lock) and [`printk()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/printk.h) (which is unsafe in arbitrary contexts).

The producer also imposes layered upfront refusals before even calling [`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4986). On [`CONFIG_PREEMPT_RT`](https://elixir.bootlin.com/linux/v6.19/source/init/Kconfig.preempt) the function returns NULL from [`in_nmi()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/preempt.h) or [`in_hardirq()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/preempt.h) because [`spin_trylock()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock.h) on RT is implemented via [`raw_spin_lock()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock.h) which is unsafe in those contexts. The function also refuses orders above [`pcp_allowed_order()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) (because the slow buddy walk is too expensive without a fallback) and refuses when [`deferred_pages_enabled()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) is true (because [`_deferred_grow_zone()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) takes a lock).

There is no slowpath. [`alloc_frozen_pages_nolock_noprof()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L7622) calls [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) once and returns whatever it produced (NULL on failure). The lock-free contract excludes direct reclaim, OOM, and the entire [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) machinery; failure simply means "could not satisfy from existing freelists without taking the slow lock", and the caller is expected to handle it.

## SPECIFICATIONS

(none; [`ALLOC_TRYLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1293) is a Linux kernel internal flag)

## LINUX KERNEL

### Flag definition

- [`'\<ALLOC_TRYLOCK\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1293): bit `0x400`. Comment: "Only use spin_trylock in allocation path".

### Producers

- [`'\<alloc_frozen_pages_nolock_noprof\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L7622): the only producer. Sets `alloc_flags = ALLOC_TRYLOCK` as a fresh local before calling [`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4986).
- [`'\<alloc_pages_nolock\>':'include/linux/gfp.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h): the public API macro that callers (BPF helpers, etc.) use; expands to [`alloc_frozen_pages_nolock_noprof()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L7622) plus refcount setup.

### Consumers

- [`'\<rmqueue_buddy\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3231): wraps the [`zone->lock`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) acquisition in [`spin_trylock_irqsave()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock.h). Returns NULL on contention.
- [`'\<rmqueue_bulk\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2550): same trylock wrapper used when the per-CPU pageset needs a refill. Returns 0 on contention.
- [`'\<cond_accept_memory\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L7554): early `return false` because [`try_to_accept_memory_one()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) takes a lock.
- [`'\<prepare_alloc_pages\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L5014): skips [`should_fail_alloc_page()`](https://elixir.bootlin.com/linux/v6.19/source/mm/fail_page_alloc.c) when [`ALLOC_TRYLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1293) is set, because the fault-injection probe calls [`get_random_u32()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/prandom.h) and [`printk()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/printk.h).

### GFP mapping

- The flag is decided by the producer entry-point and not derived from `gfp_mask`. The producer hard-codes the [`gfp_t`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/types.h) shape: `__GFP_NOWARN | __GFP_ZERO | __GFP_NOMEMALLOC | __GFP_COMP | gfp_flags` (with `gfp_flags` restricted to [`__GFP_ACCOUNT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) by [`VM_WARN_ON_ONCE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmdebug.h)).
- [`__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) and [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) are deliberately omitted from the GFP mask. The first because direct reclaim cannot run (no slowpath); the second because [`wakeup_kswapd()`](https://elixir.bootlin.com/linux/v6.19/source/mm/vmscan.c) is unsafe from arbitrary contexts.
- [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) is set so [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) returns `0` (no reserve grant). The lock-free path is not allowed to deplete reserves.

### Concurrency

- [`'\<spin_trylock_irqsave\>':'include/linux/spinlock.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock.h): the lock primitive used by both consumer sites.
- [`'\<zone\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): owns [`zone->lock`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), the per-zone spinlock that the buddy allocator holds.
- [`'\<in_nmi\>':'include/linux/preempt.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/preempt.h): predicate that the producer checks under [`CONFIG_PREEMPT_RT`](https://elixir.bootlin.com/linux/v6.19/source/init/Kconfig.preempt) to refuse calling at all.
- [`'\<in_hardirq\>':'include/linux/preempt.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/preempt.h): paired predicate.

### Related types and helpers

- [`'\<pcp_allowed_order\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c): predicate that returns true for orders the per-CPU pageset can satisfy. The producer refuses higher orders.
- [`'\<deferred_pages_enabled\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c): predicate that returns true when [`_deferred_grow_zone()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) might still expand the zone. The producer refuses in that case because growing takes a lock.
- [`'\<__free_frozen_pages\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c): the matching free path. The producer passes [`FPI_TRYLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) when freeing on the memcg-charge failure path so the free side also uses trylock.
- [`'\<gfpflags_allow_spinning\>':'include/linux/gfp.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h): predicate consulted elsewhere (e.g. by allocator slow paths) to detect whether a `gfp_t` is "lock-free safe". The producer comment explicitly notes that omitting [`__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) and [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) is the condition for `gfpflags_allow_spinning()` to return true.

## KERNEL DOCUMENTATION

- [`Documentation/bpf/`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/bpf): the BPF subsystem documentation. BPF programs running in NMI/IRQ context are the primary consumer of [`alloc_pages_nolock()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h).
- [`Documentation/locking/locktypes.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/locking/locktypes.rst): the locktypes hierarchy that explains why [`raw_spinlock_t`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock_types_raw.h)/[`spinlock_t`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock_types.h) interactions matter under [`CONFIG_PREEMPT_RT`](https://elixir.bootlin.com/linux/v6.19/source/init/Kconfig.preempt).

## OTHER SOURCES

## DETAILS

### Producer: alloc_frozen_pages_nolock_noprof

The producer is short and explanatory.

```c
struct page *alloc_frozen_pages_nolock_noprof(gfp_t gfp_flags, int nid, unsigned int order)
{
	/*
	 * Do not specify __GFP_DIRECT_RECLAIM, since direct claim is not allowed.
	 * Do not specify __GFP_KSWAPD_RECLAIM either, since wake up of kswapd
	 * is not safe in arbitrary context.
	 *
	 * These two are the conditions for gfpflags_allow_spinning() being true.
	 *
	 * Specify __GFP_NOWARN since failing alloc_pages_nolock() is not a reason
	 * to warn. Also warn would trigger printk() which is unsafe from
	 * various contexts. We cannot use printk_deferred_enter() to mitigate,
	 * since the running context is unknown.
	 *
	 * Specify __GFP_ZERO to make sure that call to kmsan_alloc_page() below
	 * is safe in any context. Also zeroing the page is mandatory for
	 * BPF use cases.
	 *
	 * Though __GFP_NOMEMALLOC is not checked in the code path below,
	 * specify it here to highlight that alloc_pages_nolock()
	 * doesn't want to deplete reserves.
	 */
	gfp_t alloc_gfp = __GFP_NOWARN | __GFP_ZERO | __GFP_NOMEMALLOC | __GFP_COMP
			| gfp_flags;
	unsigned int alloc_flags = ALLOC_TRYLOCK;
	struct alloc_context ac = { };

	VM_WARN_ON_ONCE(gfp_flags & ~__GFP_ACCOUNT);
	/*
	 * In PREEMPT_RT spin_trylock() will call raw_spin_lock() which is
	 * unsafe in NMI. If spin_trylock() is called from hard IRQ the current
	 * task may be waiting for one rt_spin_lock, but rt_spin_trylock() will
	 * mark the task as the owner of another rt_spin_lock which will
	 * confuse PI logic, so return immediately if called form hard IRQ or
	 * NMI.
	 *
	 * Note, irqs_disabled() case is ok. This function can be called
	 * from raw_spin_lock_irqsave region.
	 */
	if (IS_ENABLED(CONFIG_PREEMPT_RT) && (in_nmi() || in_hardirq()))
		return NULL;
	if (!pcp_allowed_order(order))
		return NULL;

	/* Bailout, since _deferred_grow_zone() needs to take a lock */
	if (deferred_pages_enabled())
		return NULL;

	if (nid == NUMA_NO_NODE)
		nid = numa_node_id();

	prepare_alloc_pages(alloc_gfp, order, nid, NULL, &ac,
			    &alloc_gfp, &alloc_flags);

	/*
	 * Best effort allocation from percpu free list.
	 * If it's empty attempt to spin_trylock zone->lock.
	 */
	page = get_page_from_freelist(alloc_gfp, order, alloc_flags, &ac);

	/* Unlike regular alloc_pages() there is no __alloc_pages_slowpath(). */

	if (memcg_kmem_online() && page && (gfp_flags & __GFP_ACCOUNT) &&
	    unlikely(__memcg_kmem_charge_page(page, alloc_gfp, order) != 0)) {
		__free_frozen_pages(page, order, FPI_TRYLOCK);
		page = NULL;
	}
	trace_mm_page_alloc(page, order, alloc_gfp, ac.migratetype);
	kmsan_alloc_page(page, order, alloc_gfp);
	return page;
}
```

Five upfront refusals run before the allocation proper. [`VM_WARN_ON_ONCE(gfp_flags & ~__GFP_ACCOUNT)`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmdebug.h) traps callers passing GFP bits other than [`__GFP_ACCOUNT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), since the public API only accepts that one bit and any other GFP bit is a contract violation. [`CONFIG_PREEMPT_RT && (in_nmi() || in_hardirq())`](https://elixir.bootlin.com/linux/v6.19/source/init/Kconfig.preempt) refuses NMI and hard-IRQ callers under RT, because RT's [`spin_trylock()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock.h) walks down to [`raw_spin_lock()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock.h) which is itself unsafe in NMI, and in hard IRQ the priority-inheritance logic gets confused if the task already holds an RT spinlock; the softer `irqs_disabled()` case (IRQs off but the task is not in hardirq) is fine. [`!pcp_allowed_order(order)`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) refuses orders the per-CPU pageset cannot serve, because higher orders go through [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) which walks [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473), and doing that without the slowpath fallback is impractical. [`deferred_pages_enabled()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) refuses callers during boot before every page is initialised, because [`_deferred_grow_zone()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) may have to take a lock to expand the zone and the trylock path will not risk that. The fifth refusal is implicit and happens later via the [`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4986) `should_fail_alloc_page()` skip described in the next section.

The GFP bits the producer adds are also informative. [`__GFP_NOWARN`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) suppresses the warning because failure is expected here, [`__GFP_ZERO`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) zeroes the page before returning (the comment explains that [`kmsan_alloc_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/kmsan.h) on the success path needs zeroed pages, and BPF callers need them too), [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) is decorative and signals intent (the producer's `alloc_flags = ALLOC_TRYLOCK` does not include [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262), and the function never reaches [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) which would call [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549)), and [`__GFP_COMP`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) produces compound pages for `order > 0`.

### Consumer: prepare_alloc_pages should_fail skip

[`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L5012) skips fault injection when [`ALLOC_TRYLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1293) is set.

```c
	might_alloc(gfp_mask);

	/*
	 * Don't invoke should_fail logic, since it may call
	 * get_random_u32() and printk() which need to spin_lock.
	 */
	if (!(*alloc_flags & ALLOC_TRYLOCK) &&
	    should_fail_alloc_page(gfp_mask, order))
		return false;
```

[`should_fail_alloc_page()`](https://elixir.bootlin.com/linux/v6.19/source/mm/fail_page_alloc.c) is the fault-injection probe used by `failslab` and similar testing infrastructure. It internally calls [`get_random_u32()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/prandom.h) (which may take a lock) and [`printk()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/printk.h) (which is unsafe from arbitrary contexts). Skipping the probe for [`ALLOC_TRYLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1293) callers preserves the lock-free contract at the cost of disabling fault injection on this path.

### Consumer: rmqueue_buddy

[`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3225) wraps the [`zone->lock`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) acquisition.

```c
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
			...
		}
		spin_unlock_irqrestore(&zone->lock, flags);
	} while (check_new_pages(page, order));
```

The trylock branch returns NULL outright on contention. There is no retry, no spin loop, and no fallback. Contention on [`zone->lock`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) translates directly into allocation failure that the caller has to absorb.

### Consumer: rmqueue_bulk

[`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) is called from [`rmqueue_pcplist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) when the per-CPU pageset is empty and needs a refill.

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
		/* ... linked list bookkeeping ... */
	}
	spin_unlock_irqrestore(&zone->lock, flags);

	return i;
}
```

Same trylock pattern. Returning `0` here (no pages refilled) bubbles up to [`rmqueue_pcplist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) which returns NULL, which bubbles up to [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) which falls through to [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222), which itself does the trylock and returns NULL.

The two trylock sites (PCP refill + buddy fallback) cover the two ways a [`zone->lock`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) acquisition can happen in the allocator. Order-0 allocations typically go through PCP and only reach [`rmqueue_bulk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2542) on refill; higher orders within `pcp_allowed_order()` may go straight to [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) when PCP cannot serve them.

### Consumer: cond_accept_memory short-circuit

[`cond_accept_memory()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L7544) is part of the unaccepted-memory mechanism for confidential VMs (TDX, SEV-SNP). It runs inside [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) before the watermark check.

```c
static bool cond_accept_memory(struct zone *zone, unsigned int order,
			       int alloc_flags)
{
	long to_accept, wmark;
	bool ret = false;

	if (list_empty(&zone->unaccepted_pages))
		return false;

	/* Bailout, since try_to_accept_memory_one() needs to take a lock */
	if (alloc_flags & ALLOC_TRYLOCK)
		return false;
```

[`try_to_accept_memory_one()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) holds a lock to safely move pages from the [`zone->unaccepted_pages`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) list to the regular freelists. The trylock path refuses to do that. A lock-free caller cannot be allowed to drive memory acceptance because the operation is too costly to do under a trylock and too important to be dropped silently.

An [`ALLOC_TRYLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1293) caller therefore never causes unaccepted memory to be accepted. If the regular freelists cannot satisfy the request and there is unaccepted memory available, the [`ALLOC_TRYLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1293) request fails, while the next non-trylock allocation accepts the memory and replenishes.

### Lock-free path skips the slowpath

[`alloc_frozen_pages_nolock_noprof()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L7679) explicitly does not call [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c). The comment in the source reads "Unlike regular alloc_pages() there is no `__alloc_pages_slowpath()`."

The slowpath does direct reclaim, direct compaction, OOM, and other operations that require sleeping or holding locks. None of these are compatible with the lock-free contract. Even the slowpath retry loop's [`wake_all_kswapds()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4453) call would touch wait queues that may not be safe to touch from NMI.

The lock-free path therefore exists as a strict subset of the regular allocator, providing PCP plus a single trylock attempt at [`zone->lock`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h). If that subset cannot satisfy the request, the request fails.

### Memcg charge failure path

The producer also handles the rare case where the allocation succeeded but [`__memcg_kmem_charge_page()`](https://elixir.bootlin.com/linux/v6.19/source/mm/memcontrol.c) fails.

```c
	if (memcg_kmem_online() && page && (gfp_flags & __GFP_ACCOUNT) &&
	    unlikely(__memcg_kmem_charge_page(page, alloc_gfp, order) != 0)) {
		__free_frozen_pages(page, order, FPI_TRYLOCK);
		page = NULL;
	}
```

The free path is invoked with [`FPI_TRYLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) so that [`__free_frozen_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) also uses the trylock variant when returning the page. This preserves the lock-free contract on the failure path.

### Interaction with PREEMPT_RT raw_spinlock_t

The comment on the upfront refusal explains the [`CONFIG_PREEMPT_RT`](https://elixir.bootlin.com/linux/v6.19/source/init/Kconfig.preempt) constraint.

```c
	/*
	 * In PREEMPT_RT spin_trylock() will call raw_spin_lock() which is
	 * unsafe in NMI. If spin_trylock() is called from hard IRQ the current
	 * task may be waiting for one rt_spin_lock, but rt_spin_trylock() will
	 * mark the task as the owner of another rt_spin_lock which will
	 * confuse PI logic, so return immediately if called form hard IRQ or
	 * NMI.
	 *
	 * Note, irqs_disabled() case is ok. This function can be called
	 * from raw_spin_lock_irqsave region.
	 */
	if (IS_ENABLED(CONFIG_PREEMPT_RT) && (in_nmi() || in_hardirq()))
		return NULL;
```

Under [`CONFIG_PREEMPT_RT`](https://elixir.bootlin.com/linux/v6.19/source/init/Kconfig.preempt), [`spin_trylock()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock.h) on a [`spinlock_t`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock_types.h) (which is what [`zone->lock`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is) becomes [`rt_spin_trylock()`](https://elixir.bootlin.com/linux/v6.19/source/kernel/locking/rtmutex.c), implemented on top of an rt-mutex. The rt-mutex code path includes [`raw_spin_lock()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock.h) on internal mutex state, and that raw lock is itself unsafe in NMI. Separately, if the current task is already blocked waiting for one rt-mutex somewhere up the stack and the trylock succeeds, the task is now recorded as the owner of two rt-mutexes simultaneously, and the priority-inheritance logic does not handle this case and would attribute the wrong priority boost. Either issue is enough to make the operation unsafe in hard IRQ or NMI.

The fix is the upfront refusal that does not even attempt the allocation from these contexts. The caller (typically a BPF tracepoint handler) has to handle the NULL return.

A non-RT kernel does not have these issues because [`spin_trylock()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/spinlock.h) on a regular spinlock is implemented via a single atomic operation that is safe everywhere. The producer therefore allows hard-IRQ and NMI callers on non-RT.
