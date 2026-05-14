---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# ALLOC_NOFRAGMENT

[`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) is the page allocator's "avoid mixing pageblock types" bit. When set, [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2526) refuses to fall through to the [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) path (which would steal small fragments from foreign migrate-types), [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2396) raises its minimum order to [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) (so an entire pageblock is claimed rather than partial), and [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3807) skips remote-NUMA zones that would force fragmentation. Two helpers produce the bit. [`alloc_flags_nofragment()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3739) conditionally sets it on multi-node systems with [`CONFIG_ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/init/Kconfig) when the preferred zone is [`ZONE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), and [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4525) sets it unconditionally when the [`defrag_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L307) sysctl is enabled. The bit is dropped by [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3858) when scanning crosses to a remote NUMA node and again at line 3981 after a UMA-fragmented walk, and by [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4914) before going to OOM.

```
        ALLOC_* bit field (12 bits live in v6.19)
        ─────────────────────────────────────────────────────────────
        bit:  11    10    9     8     7     6     5     4     3     2     1     0
              ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
              │  K  │  T  │ HA  │ NF  │ CMA │ CP  │ MR  │ NB  │ OOM │ NW  │  WMARK    │
              └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                                ▲
                                │
                  ALLOC_NOFRAGMENT = 0x100   (= 0 when !CONFIG_ZONE_DMA32 && !defrag_mode)

        Legend: K  = ALLOC_KSWAPD          NF  = ALLOC_NOFRAGMENT     NB  = ALLOC_NON_BLOCK
                T  = ALLOC_TRYLOCK         CP  = ALLOC_CPUSET         OOM = ALLOC_OOM
                HA = ALLOC_HIGHATOMIC      MR  = ALLOC_MIN_RESERVE    NW  = ALLOC_NO_WATERMARKS
```

```
        Producer paths -> ALLOC_NOFRAGMENT
        ────────────────────────────────────────────────────────────────

         Path A (fast path, conditional, ZONE_DMA32 only):

         __alloc_frozen_pages_noprof:
             alloc_flags |= alloc_flags_nofragment(
                                zonelist_zone(ac.preferred_zoneref), gfp);

         alloc_flags_nofragment(zone, gfp_mask):
             alloc_flags = (gfp_mask & __GFP_KSWAPD_RECLAIM)
                            /* same bit position as ALLOC_KSWAPD */

             if (defrag_mode) {
                     alloc_flags |= ALLOC_NOFRAGMENT;       <── set
                     return alloc_flags;
             }

             #ifdef CONFIG_ZONE_DMA32
                 if (!zone) return alloc_flags;
                 if (zone_idx(zone) != ZONE_NORMAL) return alloc_flags;
                 BUILD_BUG_ON(ZONE_NORMAL - ZONE_DMA32 != 1);
                 if (nr_online_nodes > 1 && !populated_zone(--zone))
                         return alloc_flags;
                 alloc_flags |= ALLOC_NOFRAGMENT;          <── set
             #endif

         Path B (slowpath, unconditional via defrag_mode):

         gfp_to_alloc_flags(gfp_mask, order):
             ...
             if (defrag_mode)
                     alloc_flags |= ALLOC_NOFRAGMENT;      <── set

         Drop sites:

         get_page_from_freelist:
             /* If moving to a remote node, retry but allow
              * fragmenting fallbacks. Locality is more important
              * than fragmentation avoidance. */
             alloc_flags &= ~ALLOC_NOFRAGMENT;             <── clear
             goto retry;

             /* It's possible on a UMA machine to get through all zones
              * that are fragmented. If avoiding fragmentation, reset
              * and try again. */
             if (no_fallback && !defrag_mode) {
                     alloc_flags &= ~ALLOC_NOFRAGMENT;     <── clear
                     goto retry;
             }

         __alloc_pages_slowpath (before OOM):
             /* Reclaim/compaction failed to prevent the fallback */
             if (defrag_mode && (alloc_flags & ALLOC_NOFRAGMENT)) {
                     alloc_flags &= ~ALLOC_NOFRAGMENT;     <── clear
                     goto retry;
             }
```

```
        Consumer behaviour
        ────────────────────────────────────────────────────────────────

         get_page_from_freelist (per-zone scan):
             no_fallback = alloc_flags & ALLOC_NOFRAGMENT;
             ...
             if (no_fallback && !defrag_mode && nr_online_nodes > 1 &&
                 zone != zonelist_zone(ac->preferred_zoneref)) {
                     /* moving to remote node */
                     local_nid = zonelist_node_idx(ac->preferred_zoneref);
                     if (zone_to_nid(zone) != local_nid) {
                             alloc_flags &= ~ALLOC_NOFRAGMENT;
                             goto retry;       /* re-scan with fallback OK */
                     }
             }
             /* and at the bottom of the function: */
             if (no_fallback && !defrag_mode) {
                     alloc_flags &= ~ALLOC_NOFRAGMENT;
                     goto retry;
             }

         __rmqueue (RMQUEUE_STEAL case):
             case RMQUEUE_STEAL:
                 if (!(alloc_flags & ALLOC_NOFRAGMENT)) {
                         page = __rmqueue_steal(zone, order, migratetype);
                         if (page) ...
                 }
                 /* with ALLOC_NOFRAGMENT, refuse to steal smaller blocks */

         __rmqueue_claim:
             if (order < pageblock_order && alloc_flags & ALLOC_NOFRAGMENT)
                     min_order = pageblock_order;
             /* claim only whole pageblocks, not fragments */
```

## SUMMARY

[`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) (= `0x100`) is bit 8 of [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c). On systems built without [`CONFIG_ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/init/Kconfig), [`mm/internal.h`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1290) redefines the macro to `0`, so the bit physically does not exist there. The flag is meaningful on multi-node ZONE_DMA32 systems and on any system that has enabled the `defrag_mode` sysctl ([`/proc/sys/vm/defrag_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L6762)).

The producer logic in [`alloc_flags_nofragment()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3739) is layered. First, it always preserves [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) using the bit-position equivalence with [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294). Second, when [`defrag_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L307) is set, it unconditionally adds [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) and returns. Third, on [`CONFIG_ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/init/Kconfig) systems where the preferred zone is [`ZONE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), it checks whether [`ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is populated on at least one online node and adds the bit only if so. The local node's [`ZONE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is preferred over a remote node's [`ZONE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) when [`ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is available locally as a fallback for unmovable allocations.

The consumer side has two main effects. [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) gates the [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) (cross-migrate-type fragment-stealing) path on `!(alloc_flags & ALLOC_NOFRAGMENT)` so a [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) caller refuses to fragment a foreign pageblock. [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2396) raises its minimum order to [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) so claiming converts an entire pageblock rather than splitting one. [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3807) reads the bit at the top of the function and uses it for two retry-with-clear branches. One fires when the scan crosses to a remote NUMA node (local fragmentation is worse than remote allocation), and the other fires at the bottom of the function for the UMA case (all zones are fragmented).

The [`defrag_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L307) sysctl strengthens the contract. With it enabled, the bit stays set across the remote-node check (the `!defrag_mode` conjunct in `get_page_from_freelist`) and is only cleared in the very last pre-OOM retry of [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4914). The intent is that a system administrator who values fragmentation avoidance over latency can prevent the allocator from fragmenting pageblocks except as a last resort.

## SPECIFICATIONS

(none; [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) is a Linux kernel internal flag)

## LINUX KERNEL

### Flag definition

- [`'\<ALLOC_NOFRAGMENT\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288): bit `0x100` under [`CONFIG_ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/init/Kconfig). The non-DMA32 branch redefines it to `0`, making the bit a no-op on those configurations.
- [`'\<defrag_mode\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L307): the sysctl-controlled global integer that strengthens [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) into an unconditional contract.

### Producers

- [`'\<alloc_flags_nofragment\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3739): the per-allocation producer. Called from [`__alloc_frozen_pages_noprof()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L5234) on the fast path and [`alloc_pages_bulk_noprof()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L5127) on the bulk path.
- [`'\<gfp_to_alloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4525): the slowpath producer. `if (defrag_mode) alloc_flags |= ALLOC_NOFRAGMENT;` at the end of the rebuild.

### Consumers

- [`'\<__rmqueue\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2526): gates the [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) call. With [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288), the [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) case returns NULL.
- [`'\<__rmqueue_claim\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2396): raises `min_order` to [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) when [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) is set and `order < pageblock_order`.
- [`'\<get_page_from_freelist\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3807): reads the bit into the local `no_fallback` and uses it for the remote-node skip and the UMA-fragmented retry. Clears the bit at line 3858 (remote node) and 3981 (UMA fallback).
- [`'\<__alloc_pages_slowpath\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4914): pre-OOM clear. `if (defrag_mode && (alloc_flags & ALLOC_NOFRAGMENT)) { alloc_flags &= ~ALLOC_NOFRAGMENT; goto retry; }`.

### GFP mapping

- [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): not directly related, but [`alloc_flags_nofragment()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3739) preserves it via the bit-equivalence with [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294); the function returns the OR of the two bits to its caller.
- No GFP bit maps directly to [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288). The bit is purely internal, decided by zone topology and the `defrag_mode` sysctl.

### Migratetype and freelists

- [`'\<__rmqueue_steal\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437): the cross-migrate-type fragment stealer. Called by [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2526) only when [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) is NOT set.
- [`'\<__rmqueue_claim\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382): the whole-pageblock claim path. With [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288), the minimum order is [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h).
- [`'\<find_suitable_fallback\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c): scans for a foreign migrate-type to claim or steal from. The `claim_only` argument distinguishes the two cases.
- [`'\<try_to_claim_block\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307): the actual whole-pageblock conversion logic. Documented further on the `ALLOC_KSWAPD` page.

### Zone topology

- [`'\<zone\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): the producer logic walks `zone->zone_pgdat->node_zones[]` to check whether [`ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is populated.
- [`'\<populated_zone\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): predicate that returns true when a zone has any pages.
- [`'\<zone_idx\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): returns the zone's index in [`node_zones[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h).
- [`'\<nr_online_nodes\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c): the count of online NUMA nodes. The producer skips the [`ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) probe on UMA systems (`nr_online_nodes <= 1`).

## KERNEL DOCUMENTATION

- [`Documentation/admin-guide/sysctl/vm.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/admin-guide/sysctl/vm.rst): describes the `defrag_mode` sysctl that turns [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) into an unconditional contract.
- [`Documentation/mm/physical_memory.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/physical_memory.rst): the per-zone migrate-type model that fragmentation avoidance protects.

## OTHER SOURCES

## DETAILS

### Flag bit and architecture gating

[`mm/internal.h`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1287) wraps the bit in a configuration check.

```c
#ifdef CONFIG_ZONE_DMA32
#define ALLOC_NOFRAGMENT	0x100 /* avoid mixing pageblock types */
#else
#define ALLOC_NOFRAGMENT	  0x0
#endif
```

On configurations without [`CONFIG_ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/init/Kconfig), the macro evaluates to `0`. Every `alloc_flags |= ALLOC_NOFRAGMENT` becomes a no-op, every `alloc_flags & ALLOC_NOFRAGMENT` evaluates to `0`, and the consumer paths that branch on the bit fall through unconditionally to their fragmenting alternatives. The `defrag_mode` sysctl path goes through the same producer, so it is also no-op on these configurations.

The gating is correct. [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) was originally introduced to handle the [`ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) interaction where a multi-node system has [`ZONE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) preferred but can fall back to [`ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) for unmovable allocations. The sysctl-driven [`defrag_mode`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L307) extension reuses the same machinery, so it inherits the gating.

### Producer: alloc_flags_nofragment

[`alloc_flags_nofragment()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3739) is short and dense.

```c
static inline unsigned int
alloc_flags_nofragment(struct zone *zone, gfp_t gfp_mask)
{
	unsigned int alloc_flags;

	/*
	 * __GFP_KSWAPD_RECLAIM is assumed to be the same as ALLOC_KSWAPD
	 * to save a branch.
	 */
	alloc_flags = (__force int) (gfp_mask & __GFP_KSWAPD_RECLAIM);

	if (defrag_mode) {
		alloc_flags |= ALLOC_NOFRAGMENT;
		return alloc_flags;
	}

#ifdef CONFIG_ZONE_DMA32
	if (!zone)
		return alloc_flags;

	if (zone_idx(zone) != ZONE_NORMAL)
		return alloc_flags;

	/*
	 * If ZONE_DMA32 exists, assume it is the one after ZONE_NORMAL and
	 * the pointer is within zone->zone_pgdat->node_zones[]. Also assume
	 * on UMA that if Normal is populated then so is DMA32.
	 */
	BUILD_BUG_ON(ZONE_NORMAL - ZONE_DMA32 != 1);
	if (nr_online_nodes > 1 && !populated_zone(--zone))
		return alloc_flags;

	alloc_flags |= ALLOC_NOFRAGMENT;
#endif /* CONFIG_ZONE_DMA32 */
	return alloc_flags;
}
```

The function returns a partial `alloc_flags` value that the caller ORs into its existing word. The first layer preserves [`__GFP_KSWAPD_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), saving the caller a separate OR (the bit-position equivalence with [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) is enforced by [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4490)). The second layer is the `defrag_mode` short-circuit; when the sysctl is set, [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) is added unconditionally and the function returns early, skipping the zone topology check below because `defrag_mode` should apply universally rather than just to ZONE_DMA32 fallback scenarios. The third layer is the [`CONFIG_ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/init/Kconfig) heuristic, which adds the bit when the preferred zone is non-NULL and is [`ZONE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), and (on multi-node systems) when the [`ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) one slot before the preferred zone (reached via the `--zone` post-decrement of the [`node_zones[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) pointer) is populated. The [`BUILD_BUG_ON`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/build_bug.h) asserts that [`ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) sits exactly one slot before [`ZONE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) in the array, so the pointer-arithmetic shortcut is safe. On UMA systems the function assumes [`ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is populated whenever [`ZONE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is (the comment makes this assumption explicit), so no probe is needed.

The logic captures the original [`ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) intent. The local node's [`ZONE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) is preferred over a remote node's because the remote node has [`ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) available for a future unmovable allocation, and fragmenting the remote [`ZONE_NORMAL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) early would prevent that.

### Producer: gfp_to_alloc_flags (defrag_mode path)

The end of [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4521) duplicates the `defrag_mode` check inside the slowpath rebuild.

```c
	alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, alloc_flags);

	if (defrag_mode)
		alloc_flags |= ALLOC_NOFRAGMENT;

	return alloc_flags;
```

The slowpath does not call [`alloc_flags_nofragment()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3739) because by the time the slowpath rebuild runs, the topology-driven heuristic is no longer interesting (the first attempt already either succeeded or proved that the preferred zone could not satisfy the request). The `defrag_mode` part is still relevant, since it is a sysadmin-policy contract rather than a topology heuristic, and should hold across retries.

### Slowpath retreat from defrag_mode

[`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4910) has a special clear right before going to OOM.

```c
	/* Reclaim/compaction failed to prevent the fallback */
	if (defrag_mode && (alloc_flags & ALLOC_NOFRAGMENT)) {
		alloc_flags &= ~ALLOC_NOFRAGMENT;
		goto retry;
	}
```

When `defrag_mode` is set AND the allocator has reached the point where reclaim and compaction have failed AND [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) is still set, the slowpath drops the bit and retries. This is the last opportunity to satisfy the request before OOM, so the contract is relaxed.

The order of operations matters. The bit is dropped only after every other tactic has been tried (direct reclaim, direct compaction, retry loops). A `defrag_mode` administrator gets the maximum benefit (no fragmentation under any normal pressure) and the minimum cost (a single extra retry before OOM when the system genuinely cannot satisfy the request without fragmenting).

### Consumer: __rmqueue RMQUEUE_STEAL gate

[`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2519) has the gate at the bottom of its switch.

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
}
```

[`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437) is the cross-migrate-type fragment stealer. It walks the free areas looking for a block of the same order in any compatible foreign migratetype and pulls just enough pages out to satisfy the request, leaving the rest in the foreign migratetype. This causes type mixing, where a [`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) allocation lands inside what was a [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) pageblock, and the resulting fragment is impossible to migrate later because of the unmovable contents.

[`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) prohibits this. With the bit set, the [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) case returns NULL immediately, leaving the [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2473) caller (typically [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c)) to deal with the failure. The failure usually propagates back up, [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) moves on to the next zone, or the slowpath kicks in.

### Consumer: __rmqueue_claim min_order bump

[`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) is the whole-pageblock claim path that runs in the [`RMQUEUE_CLAIM`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) case (between [`RMQUEUE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) and [`RMQUEUE_STEAL`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c)). The relevant comment and code.

```c
	/*
	 * Do not steal pages from freelists belonging to other pageblocks
	 * i.e. orders < pageblock_order. If there are no local zones free,
	 * the zonelists will be reiterated without ALLOC_NOFRAGMENT.
	 */
	if (order < pageblock_order && alloc_flags & ALLOC_NOFRAGMENT)
		min_order = pageblock_order;
```

Without [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288), [`__rmqueue_claim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2382) walks orders from `MAX_PAGE_ORDER` down to the requested `order`, looking for a fallback-eligible block. With the bit set, the lower bound becomes [`pageblock_order`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), so the function only considers blocks that span at least one whole pageblock. [`try_to_claim_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2307) (the helper that actually does the conversion) will then convert the entire pageblock to the requested migratetype rather than splitting one.

This rules out partial-pageblock claims, which would create the same kind of type-mixing fragmentation as [`__rmqueue_steal()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L2437). The comment "the zonelists will be reiterated without ALLOC_NOFRAGMENT" refers to the [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3981) fallback that drops the bit and retries.

### Consumer: get_page_from_freelist

[`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3791) reads the bit at the top of the function.

```c
retry:
	/*
	 * Scan zonelist, looking for a zone with enough free.
	 * See also cpuset_current_node_allowed() comment in kernel/cgroup/cpuset.c.
	 */
	no_fallback = alloc_flags & ALLOC_NOFRAGMENT;
```

The `no_fallback` local is then used in two retry-with-clear branches.

#### First clear (remote NUMA node)

```c
		if (no_fallback && !defrag_mode && nr_online_nodes > 1 &&
		    zone != zonelist_zone(ac->preferred_zoneref)) {
			int local_nid;

			/*
			 * If moving to a remote node, retry but allow
			 * fragmenting fallbacks. Locality is more important
			 * than fragmentation avoidance.
			 */
			local_nid = zonelist_node_idx(ac->preferred_zoneref);
			if (zone_to_nid(zone) != local_nid) {
				alloc_flags &= ~ALLOC_NOFRAGMENT;
				goto retry;
			}
		}
```

The block fires when the scan has reached a zone that is not the preferred zone, on a different NUMA node, with the caller not requesting `defrag_mode`. The recovery drops [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) and restarts the entire scan. According to the comment, locality matters more than fragmentation avoidance. A page on the local node (even with fragmentation) is faster than a page on a remote node (without fragmentation), so the policy backs off.

The `!defrag_mode` conjunct is the linchpin. With the sysctl set, the policy is reversed and the scan continues across nodes without dropping the bit.

#### Second clear (UMA fallback)

```c
	/*
	 * It's possible on a UMA machine to get through all zones that are
	 * fragmented. If avoiding fragmentation, reset and try again.
	 */
	if (no_fallback && !defrag_mode) {
		alloc_flags &= ~ALLOC_NOFRAGMENT;
		goto retry;
	}

	return NULL;
```

This runs after the for-loop exits without finding a satisfying zone. On UMA the remote-node clear above never fires (only one node), so the bit would still be set; this clear handles the case where every zone refused because of [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288). Same `!defrag_mode` conjunct.

### Interaction with __alloc_pages_slowpath

The slowpath [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4525) sets [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) only when `defrag_mode` is set. The fast-path [`alloc_flags_nofragment()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3739) sets it more aggressively (also for the [`ZONE_DMA32`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) topology heuristic). When the fast path fails and the slowpath rebuilds [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) from scratch, the topology-driven [`ALLOC_NOFRAGMENT`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1288) is lost (because [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479) does not call [`alloc_flags_nofragment()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3739)). Only the `defrag_mode` contract survives.

The drop is intentional. A slowpath retry has already paid the cost of failing the fast-path scan, so the topology heuristic ("prefer local even if it fragments later") is no longer the right choice. The slowpath should accept fragmentation if that is what it takes to satisfy the request.

### sysctl interface for defrag_mode

[`mm/page_alloc.c`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L6760) exposes the variable as a sysctl.

```c
		.procname	= "defrag_mode",
		.data		= &defrag_mode,
		.maxlen		= sizeof(defrag_mode),
```

The setting is read at the producer site without any lock or barrier; updates take effect on subsequent allocations. There is no per-cpuset or per-process control; the setting is system-global.

### Summary of conditions that produce the bit

| Path | Condition |
| --- | --- |
| Fast path, `alloc_flags_nofragment()` | `defrag_mode` set, OR (zone idx is `ZONE_NORMAL` AND on UMA, OR on multi-node with `ZONE_DMA32` populated locally) |
| Slowpath rebuild, `gfp_to_alloc_flags()` | `defrag_mode` set |

Three conditions drop the bit.

| Path | Condition |
| --- | --- |
| `get_page_from_freelist()` line 3858 | `no_fallback` AND `!defrag_mode` AND `nr_online_nodes > 1` AND scan crossed to remote node |
| `get_page_from_freelist()` line 3981 | `no_fallback` AND `!defrag_mode` (after for-loop exit) |
| `__alloc_pages_slowpath()` line 4914 | `defrag_mode` set AND bit still set, immediately before OOM |

The asymmetry in the drop sites is deliberate. The first two only fire in the heuristic regime, leaving `defrag_mode` callers immune to mid-allocation drops. The third only fires in the contract regime, giving `defrag_mode` callers exactly one OOM-avoidance escape.
