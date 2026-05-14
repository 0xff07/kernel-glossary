---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# ALLOC_KSWAPD

[`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) is the page allocator's "wake kswapd to refill the freelists" bit. It is the [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) twin of [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), since a [`BUILD_BUG_ON`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4489) in [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479) pins the two values to the same bit position so the GFP-side bit can be OR'd straight into the ALLOC-side word without translation. The flag has three reader sites. [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4764) wakes every kswapd in the zonelist before the first attempt and again on every retry. [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3412) does an additional kswapd wakeup when a successful allocation drops the zone below its boosted watermark. [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2330) calls [`boost_watermark()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2188) and sets [`ZONE_BOOSTED_WATERMARK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) (which is what [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3412) reacts to) when the kernel claims a foreign pageblock as part of a fragmenting allocation. The slowpath's reserve-grant rebuild explicitly preserves the bit when overwriting [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) so that kswapd wake-ups continue across reserve-driven retries.

```
        ALLOC_* bit field (12 bits live in v6.19)
        ─────────────────────────────────────────────────────────────
        bit:  11    10    9     8     7     6     5     4     3     2     1     0
              ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
              │  K  │  T  │ HA  │ NF  │ CMA │ CP  │ MR  │ NB  │ OOM │ NW  │  WMARK    │
              └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                ▲
                │
                ALLOC_KSWAPD = 0x800   (bit-aligned with __GFP_KSWAPD_RECLAIM)

        Legend: K  = ALLOC_KSWAPD          NF  = ALLOC_NOFRAGMENT     NB  = ALLOC_NON_BLOCK
                T  = ALLOC_TRYLOCK         CP  = ALLOC_CPUSET         OOM = ALLOC_OOM
                HA = ALLOC_HIGHATOMIC      MR  = ALLOC_MIN_RESERVE    NW  = ALLOC_NO_WATERMARKS
```

```
        kswapd wake-up paths driven by ALLOC_KSWAPD
        ────────────────────────────────────────────────────────────────

         (1) Slowpath entry (every retry):

             __alloc_pages_slowpath:
                 alloc_flags = gfp_to_alloc_flags(gfp_mask, order)

                 if (alloc_flags & ALLOC_KSWAPD)
                         wake_all_kswapds(order, gfp_mask, ac);   <── wake

                 page = get_page_from_freelist(...)
                 ...
             retry:
                 if (alloc_flags & ALLOC_KSWAPD)
                         wake_all_kswapds(order, gfp_mask, ac);   <── wake again

                 reserve_flags = __gfp_pfmemalloc_flags(gfp_mask);
                 if (reserve_flags)
                         alloc_flags = gfp_to_alloc_flags_cma(...) |
                                       (alloc_flags & ALLOC_KSWAPD);
                                                            <── preserve

         (2) Successful allocation that crossed the boost threshold:

             rmqueue(..., alloc_flags, ...):
                 page = rmqueue_pcplist(...) or rmqueue_buddy(...)

                 if ((alloc_flags & ALLOC_KSWAPD) &&
                     test_bit(ZONE_BOOSTED_WATERMARK, &zone->flags)) {
                         clear_bit(ZONE_BOOSTED_WATERMARK, &zone->flags);
                         wakeup_kswapd(zone, 0, 0, zone_idx(zone));   <── wake
                 }

         (3) Fragmenting claim that boosted the watermark:

             try_to_claim_block(zone, page, current_order, order,
                                start_type, block_type, alloc_flags):
                 if (current_order >= pageblock_order)
                         /* whole-pageblock claim, no boost needed */
                         return page;

                 if (boost_watermark(zone) && (alloc_flags & ALLOC_KSWAPD))
                         set_bit(ZONE_BOOSTED_WATERMARK, &zone->flags);

         (4) wake_all_kswapds (the actual wake routine):

             reclaim_order = (defrag_mode) ? max(order, pageblock_order) : order;
             for each zone in zonelist:
                 if (!managed_zone(zone)) continue;
                 if (last_pgdat == zone->zone_pgdat) continue;  /* dedup */
                 wakeup_kswapd(zone, gfp_mask, reclaim_order, highest_zoneidx);
                 last_pgdat = zone->zone_pgdat;
```

## SUMMARY

[`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) (= `0x800`) is bit 11 of [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c). [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4498) sets the bit by ANDing `gfp_mask & __GFP_KSWAPD_RECLAIM` and ORing the result straight into [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) (a single instruction thanks to the [`BUILD_BUG_ON`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4489) bit-equivalence). [`alloc_flags_nofragment()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3748) does the same on the fast path. [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) is part of [`GFP_KERNEL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) and [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), so the great majority of allocations carry [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) automatically.

The slowpath wakes kswapd at two well-defined points. The first is right after the slowpath entry, before the first [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4768) attempt. If the fast path failed, kswapd should already be running, and the wake-up ensures it does not sleep. The second is at the top of the `retry:` loop, fired on every iteration to keep kswapd reclaiming for callers that are spinning around the slowpath waiting for memory. The wake call eventually reaches [`wakeup_kswapd()`](https://elixir.bootlin.com/linux/v6.19/source/mm/vmscan.c) on every populated NUMA node in the zonelist (deduplicated by pgdat).

The reserve-grant rebuild at [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843) explicitly preserves [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) when overwriting [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) with the reserve grant via `alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, reserve_flags) | (alloc_flags & ALLOC_KSWAPD)`. The reserve grant on its own would not include [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) (because [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) does not produce that bit), so the explicit OR is the only way the wakeups continue.

A second, indirect wake-up path comes from the watermark-boost mechanism. When [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307) takes a foreign pageblock to satisfy a fragmenting allocation, it calls [`boost_watermark()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2188) which raises [`zone->watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L884), and (when [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) is set) marks the zone with [`ZONE_BOOSTED_WATERMARK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h). The next [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3393) call on that zone reads the bit, clears it, and wakes kswapd to recover the inflated headroom. Every fragmenting allocation therefore gets a brief watermark inflation that kswapd is expected to absorb.

## SPECIFICATIONS

(none; [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) is a Linux kernel internal flag)

## LINUX KERNEL

### Flag definition

- [`'\<ALLOC_KSWAPD\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294): bit `0x800`. Comment: "allow waking of kswapd, `__GFP_KSWAPD_RECLAIM` set".
- [`'\<__GFP_KSWAPD_RECLAIM\>':'include/linux/gfp_types.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): the GFP-side twin. The compile-time invariant `__GFP_KSWAPD_RECLAIM == ALLOC_KSWAPD` is enforced by [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4489).

### Producers

- [`'\<gfp_to_alloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4498): the slowpath producer, expressed as the single OR `alloc_flags |= (gfp_mask & (__GFP_HIGH | __GFP_KSWAPD_RECLAIM))`.
- [`'\<alloc_flags_nofragment\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3748): the fast-path producer, using the same bit-equivalence trick as `alloc_flags = (gfp_mask & __GFP_KSWAPD_RECLAIM)`.
- [`'\<__alloc_pages_slowpath\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4846): the reserve-grant rebuild explicitly preserves [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) by `(alloc_flags & ALLOC_KSWAPD)` OR after overwriting [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c).

### Consumers

- [`'\<__alloc_pages_slowpath\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4764): the entry-point wake-up. `if (alloc_flags & ALLOC_KSWAPD) wake_all_kswapds(order, gfp_mask, ac);`
- [`'\<__alloc_pages_slowpath\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4840): the retry-loop wake-up. Identical predicate, fires every retry iteration so kswapd does not accidentally go to sleep while the allocator loops.
- [`'\<rmqueue\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3412): the post-allocation boost-cleanup. Tests `(alloc_flags & ALLOC_KSWAPD) && ZONE_BOOSTED_WATERMARK`, clears the bit, calls [`wakeup_kswapd(zone, 0, 0, zone_idx(zone))`](https://elixir.bootlin.com/linux/v6.19/source/mm/vmscan.c).
- [`'\<try_to_claim_block\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2330): the boost-set site. `if (boost_watermark(zone) && (alloc_flags & ALLOC_KSWAPD)) set_bit(ZONE_BOOSTED_WATERMARK, &zone->flags);`

### GFP mapping

- [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): the canonical source. Carried by [`GFP_KERNEL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), [`GFP_NOWAIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), [`GFP_USER`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), most [`GFP_*`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) shapes.
- [`__GFP_NORETRY`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): orthogonal; does not affect [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) directly. The slowpath retry loop where the second wake-up sits is exited early by [`__GFP_NORETRY`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), so a [`__GFP_NORETRY`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) caller wakes kswapd at most twice.

### Concurrency

- [`'\<wake_all_kswapds\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4453): walks the zonelist, deduplicates by pgdat, calls [`wakeup_kswapd()`](https://elixir.bootlin.com/linux/v6.19/source/mm/vmscan.c) per pgdat.
- [`'\<wakeup_kswapd\>':'mm/vmscan.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/vmscan.c): the per-pgdat wake helper. Sets [`pgdat->kswapd_highest_zoneidx`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) and [`pgdat->kswapd_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), then signals [`pgdat->kswapd_wait`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h).
- [`'\<kswapd\>':'mm/vmscan.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/vmscan.c): the per-pgdat reclaim thread. Sleeps on [`kswapd_wait`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) and runs [`balance_pgdat()`](https://elixir.bootlin.com/linux/v6.19/source/mm/vmscan.c) when woken.
- [`'\<boost_watermark\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2188): the watermark-boost helper. Updates [`zone->watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L884) using `watermark_boost_factor`. Returns true on success.
- [`'\<try_to_claim_block\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307): the foreign-pageblock claim path. The only caller of [`boost_watermark()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2188).

### Watermarks and reserves

- [`'\<zone\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): owns [`watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L884) and the [`flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) field that holds [`ZONE_BOOSTED_WATERMARK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h).
- [`'\<wmark_pages\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1077): adds [`watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L884) to the per-zone threshold; the boost is what kswapd is expected to bring back to baseline.
- [`'\<watermark_boost_factor\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c): the sysctl that scales the boost amount.

### Defrag mode interaction

- [`'\<defrag_mode\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L307): when set, [`wake_all_kswapds()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4462) raises `reclaim_order` to `max(order, pageblock_order)` so kswapd reclaims at pageblock granularity, matching the [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) policy.

## KERNEL DOCUMENTATION

- [`Documentation/admin-guide/sysctl/vm.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/admin-guide/sysctl/vm.rst): describes `watermark_boost_factor` (the sysctl that controls [`boost_watermark()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2188)) and the kswapd-related tunables.
- [`Documentation/mm/physical_memory.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/physical_memory.rst): the per-zone watermark and kswapd model.

## OTHER SOURCES

## DETAILS

### Bit equivalence with __GFP_KSWAPD_RECLAIM

[`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479) opens with two compile-time invariants.

```c
	BUILD_BUG_ON(__GFP_HIGH != (__force gfp_t) ALLOC_MIN_RESERVE);
	BUILD_BUG_ON(__GFP_KSWAPD_RECLAIM != (__force gfp_t) ALLOC_KSWAPD);
```

The "save two branches" comment refers to the next line.

```c
	alloc_flags |= (__force int)
		(gfp_mask & (__GFP_HIGH | __GFP_KSWAPD_RECLAIM));
```

Without the bit-equivalence, this would have to be split into two `if (...) alloc_flags |= ...;` statements. The same equivalence is what makes the slowpath rebuild's `(alloc_flags & ALLOC_KSWAPD)` cheap to extract.

### Producer: alloc_flags_nofragment

[`alloc_flags_nofragment()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3739) is the fast-path producer. The first line of the function is.

```c
	alloc_flags = (__force int) (gfp_mask & __GFP_KSWAPD_RECLAIM);
```

A direct copy of the GFP bit into [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) using the bit-equivalence. The function then conditionally adds [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) and returns. The caller ([`__alloc_frozen_pages_noprof()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L5234) for the singleton path or [`alloc_pages_bulk_noprof()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L5127) for the bulk path) ORs the result into its [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) before calling [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c).

### Slowpath wake-ups

[`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4760) has the first kswapd wake right after computing [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c).

```c
	alloc_flags = gfp_to_alloc_flags(gfp_mask, order);

	/* ... preferred-zoneref recompute ... */

	if (alloc_flags & ALLOC_KSWAPD)
		wake_all_kswapds(order, gfp_mask, ac);

	/*
	 * The adjusted alloc_flags might result in immediate success, so try
	 * that first
	 */
	page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
	if (page)
		goto got_pg;
```

The wake happens before the first slowpath [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) attempt. Even if that attempt succeeds (rare; the fast path already tried), kswapd has been woken and will refill the freelists for subsequent allocations.

The second wake-up sits inside the `retry:` label.

```c
retry:
	/*
	 * Deal with possible cpuset update races or zonelist updates to avoid
	 * infinite retries.
	 */
	if (check_retry_cpuset(cpuset_mems_cookie, ac) ||
	    check_retry_zonelist(zonelist_iter_cookie))
		goto restart;

	/* Ensure kswapd doesn't accidentally go to sleep as long as we loop */
	if (alloc_flags & ALLOC_KSWAPD)
		wake_all_kswapds(order, gfp_mask, ac);

	reserve_flags = __gfp_pfmemalloc_flags(gfp_mask);
	if (reserve_flags)
		alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, reserve_flags) |
					  (alloc_flags & ALLOC_KSWAPD);
```

According to the comment "Ensure kswapd doesn't accidentally go to sleep as long as we loop", an allocator that is spinning around the slowpath waiting for memory needs kswapd to keep reclaiming. Without the per-iteration wake, kswapd could finish a balance pass and go back to sleep, leaving the allocator stuck.

The reserve-grant rebuild on the next two lines preserves [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) by carrying it through the `(alloc_flags & ALLOC_KSWAPD)` OR. The new [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) value is `reserve_flags [| ALLOC_CMA] [| ALLOC_KSWAPD]`. If the bit had been dropped by the rebuild, the next iteration's `if (alloc_flags & ALLOC_KSWAPD)` check would be false and kswapd would not be woken, even though the caller is still looping.

### wake_all_kswapds: the actual wake routine

[`wake_all_kswapds()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4453) walks the zonelist.

```c
static void wake_all_kswapds(unsigned int order, gfp_t gfp_mask,
			     const struct alloc_context *ac)
{
	struct zoneref *z;
	struct zone *zone;
	pg_data_t *last_pgdat = NULL;
	enum zone_type highest_zoneidx = ac->highest_zoneidx;
	unsigned int reclaim_order;

	if (defrag_mode)
		reclaim_order = max(order, pageblock_order);
	else
		reclaim_order = order;

	for_each_zone_zonelist_nodemask(zone, z, ac->zonelist, highest_zoneidx,
					ac->nodemask) {
		if (!managed_zone(zone))
			continue;
		if (last_pgdat == zone->zone_pgdat)
			continue;
		wakeup_kswapd(zone, gfp_mask, reclaim_order, highest_zoneidx);
		last_pgdat = zone->zone_pgdat;
	}
}
```

`reclaim_order` is `max(order, pageblock_order)` when [`defrag_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L307) is set, which tells kswapd to reclaim at pageblock granularity, matching the [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) policy that prefers entire pageblocks over fragments. The `last_pgdat` deduplication skips zones that share a pgdat with an already-woken zone, since there is one kswapd thread per pgdat and calling [`wakeup_kswapd()`](https://elixir.bootlin.com/linux/v6.19/source/mm/vmscan.c) repeatedly on zones of the same pgdat would just touch the same wait-queue.

### Watermark boost: try_to_claim_block

The second [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) consumer chain runs through [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307). When the kernel claims a foreign pageblock to satisfy a fragmenting allocation, the function tries to compensate by inflating the zone's watermark.

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

	/*
	 * Boost watermarks to increase reclaim pressure to reduce the
	 * likelihood of future fallbacks. Wake kswapd now as the node
	 * may be balanced overall and kswapd will not wake naturally.
	 */
	if (boost_watermark(zone) && (alloc_flags & ALLOC_KSWAPD))
		set_bit(ZONE_BOOSTED_WATERMARK, &zone->flags);
```

The boost happens only for sub-pageblock claims. The `if (current_order >= pageblock_order) return page;` early return covers the whole-pageblock case. A sub-pageblock claim is a fragmenting event. The foreign pageblock is split, with part going to the requesting migratetype and part remaining in the original. Future allocations of either type are now more likely to claim more pageblocks, so kswapd is woken to free pages and reduce the future need.

[`boost_watermark()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2188) is the routine that updates [`zone->watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L884).

```c
static inline bool boost_watermark(struct zone *zone)
{
	unsigned long max_boost;

	if (!watermark_boost_factor)
		return false;
	/*
	 * Don't bother in zones that are unlikely to produce results.
	 * On small machines, including kdump capture kernels running
	 * in a small area, boosting the watermark can cause an out of
	 * memory situation immediately.
	 */
	if ((pageblock_nr_pages * 4) > zone_managed_pages(zone))
		return false;

	max_boost = mult_frac(zone->_watermark[WMARK_HIGH],
			watermark_boost_factor, 10000);

	/*
	 * high watermark may be uninitialised if fragmentation occurs
	 * very early in boot so do not boost. We do not fall
	 * through and boost by pageblock_nr_pages as failing
	 * allocations that early means that reclaim is not going
	 * to help and it may even be impossible to reclaim the
	 * boosted watermark resulting in a hang.
	 */
	if (!max_boost)
		return false;

	max_boost = max(pageblock_nr_pages, max_boost);

	zone->watermark_boost = min(zone->watermark_boost + pageblock_nr_pages,
		max_boost);

	return true;
}
```

The function returns true when the boost actually moved (the `watermark_boost_factor` is non-zero, the zone is large enough, and the high watermark is initialised). [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2330) only sets [`ZONE_BOOSTED_WATERMARK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) when both the boost succeeded and [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) is set. The [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) check matters because the wake-up is what makes the boost recoverable, and a caller that opted out of kswapd wakeups should also not inflate the watermark.

### rmqueue: post-allocation boost cleanup

[`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3393) reads [`ZONE_BOOSTED_WATERMARK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) at the end of every successful allocation.

```c
out:
	/* Separate test+clear to avoid unnecessary atomics */
	if ((alloc_flags & ALLOC_KSWAPD) &&
	    unlikely(test_bit(ZONE_BOOSTED_WATERMARK, &zone->flags))) {
		clear_bit(ZONE_BOOSTED_WATERMARK, &zone->flags);
		wakeup_kswapd(zone, 0, 0, zone_idx(zone));
	}
```

There are two phases in the usage pattern. A non-atomic [`test_bit()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/bitops.h) probe runs first, and an atomic [`clear_bit()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/bitops.h) only follows when the probe was true. According to the comment "Separate test+clear to avoid unnecessary atomics", the clear is racy (a second allocator could re-set the bit between the probe and the clear), but the consequences are benign (one extra wake-up).

The wake call uses `gfp_mask = 0` and `order = 0`, which signals to [`wakeup_kswapd()`](https://elixir.bootlin.com/linux/v6.19/source/mm/vmscan.c) that the wake is for general reclaim, not for a specific high-order request.

### Watermark boost flow

```
            try_to_claim_block (sub-pageblock claim)
                            │
                            ▼
                  boost_watermark(zone):
                  zone->watermark_boost += pageblock_nr_pages
                  (capped at watermark_boost_factor / 10000 of WMARK_HIGH)
                            │
                            ▼
                  if (alloc_flags & ALLOC_KSWAPD)
                          set_bit(ZONE_BOOSTED_WATERMARK, &zone->flags)
                            │
                            ▼ (later allocation on same zone)
            rmqueue:
                  ... allocation succeeds ...
                  if ((alloc_flags & ALLOC_KSWAPD) &&
                      ZONE_BOOSTED_WATERMARK is set) {
                          clear_bit(...)
                          wakeup_kswapd(zone, 0, 0, zone_idx(zone))
                  }
                            │
                            ▼
            kswapd reclaims, unwinds zone->watermark_boost over time
            (background; the bit just signals "wake me to start").
```

The mechanism is asymmetric. The boost is added quickly (on a single fragmenting claim) and is unwound slowly (kswapd does background reclaim that gradually reduces [`watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L884) toward zero). [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) is the linchpin, because without it the boost is added but never signalled, so kswapd would not start the unwinding promptly.

### __GFP_KSWAPD_RECLAIM membership in GFP_ATOMIC

[`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) is `__GFP_HIGH | __GFP_KSWAPD_RECLAIM`. An atomic allocation cannot block, so it cannot drive direct reclaim; the only reclaim available to it is kswapd's. Including [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) in [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) lets interrupt-time atomic allocations trigger kswapd wakeups through the slowpath wake at line 4764 (in practice [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) usually succeeds in the fast path and the slowpath is not reached). The atomic caller does not wait, but kswapd starts running and the next atomic caller has a better chance.

[`GFP_NOWAIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) (= [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) only) follows the same pattern, where the caller does not wait but kswapd is woken to refill for future requests.

### Allocation paths that lack ALLOC_KSWAPD

A caller that explicitly clears [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) gets neither the slowpath wake-ups nor the boost-cleanup wake-up. The [`__GFP_NOWARN | __GFP_NOMEMALLOC | __GFP_NORETRY`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) probes the kernel uses for cost estimation strip [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) so the probe does not generate background work, and the [`alloc_frozen_pages_nolock_noprof()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L7648) path that sets [`ALLOC_TRYLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1293) drops the bit too. Such callers do not wake kswapd and are responsible for not relying on kswapd's reclaim activity to satisfy their requests.
