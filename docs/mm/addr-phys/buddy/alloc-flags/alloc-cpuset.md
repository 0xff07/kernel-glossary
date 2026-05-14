---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# ALLOC_CPUSET

[`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) is the page allocator's "honour the current task's cpuset" bit. When set, the per-zone scan inside [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3816) calls [`__cpuset_zone_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cpuset.h#L88) on every candidate zone and skips zones outside the task's [`mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) (or its nearest hardwalled ancestor). The flag is the default for the first allocation attempt. [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4482) seeds [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) with `ALLOC_WMARK_MIN | ALLOC_CPUSET`, and [`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L5005) re-confirms it when [`cpusets_enabled()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cpuset.h#L39) is true and the caller did not provide a [`nodemask`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/nodemask.h). Three sites drop the flag. [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4519) clears it for non-blocking [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) callers (typical [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) so the request does not fail on cpuset isolation. [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4854) implicitly drops it via the [`reserve_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843) rebuild when the caller earns [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) or [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273). The dedicated retry helper [`__alloc_pages_cpuset_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4034) deliberately drops it on the second pass after the cpuset-restricted first pass returned NULL.

```
        ALLOC_* bit field (12 bits live in v6.19)
        ─────────────────────────────────────────────────────────────
        bit:  11    10    9     8     7     6     5     4     3     2     1     0
              ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
              │  K  │  T  │ HA  │ NF  │ CMA │ CP  │ MR  │ NB  │ OOM │ NW  │  WMARK    │
              └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                                      ▲
                                      │
                           ALLOC_CPUSET = 0x40

        Legend: K  = ALLOC_KSWAPD          NF  = ALLOC_NOFRAGMENT     NB  = ALLOC_NON_BLOCK
                T  = ALLOC_TRYLOCK         CP  = ALLOC_CPUSET         OOM = ALLOC_OOM
                HA = ALLOC_HIGHATOMIC      MR  = ALLOC_MIN_RESERVE    NW  = ALLOC_NO_WATERMARKS
```

```
        ALLOC_CPUSET state machine across an allocation
        ────────────────────────────────────────────────────────────────

         Caller (alloc_pages or __folio_alloc)
                                │
                                ▼
         __alloc_frozen_pages_noprof:
             alloc_flags = ALLOC_WMARK_LOW
                                │
                                ▼
         prepare_alloc_pages:
             if (cpusets_enabled()) {
                     if (in_task() && !ac->nodemask)
                             ac->nodemask = &cpuset_current_mems_allowed;
                     else
                             *alloc_flags |= ALLOC_CPUSET;        <── set
             }
                                │
                                ▼
         get_page_from_freelist (first attempt):
             for_next_zone_zonelist_nodemask(zone, ...) {
                     if (cpusets_enabled() &&
                         (alloc_flags & ALLOC_CPUSET) &&
                         !__cpuset_zone_allowed(zone, gfp_mask))
                             continue;                               <── skip
                     ...
             }
                                │
                                ▼ (NULL)
         __alloc_pages_slowpath:
             ac->nodemask = nodemask;             /* restore caller's */
             alloc_flags = gfp_to_alloc_flags(gfp_mask, order);
                                │
                  alloc_flags = ALLOC_WMARK_MIN | ALLOC_CPUSET (initial)
                  alloc_flags |= __GFP_HIGH | __GFP_KSWAPD_RECLAIM
                                │
                                ▼
              if (alloc_flags & ALLOC_MIN_RESERVE)
                      alloc_flags &= ~ALLOC_CPUSET;     <── clear (GFP_ATOMIC)
                                │
                                ▼
              reserve_flags = __gfp_pfmemalloc_flags(gfp_mask);
              if (reserve_flags)
                      alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, reserve_flags) |
                                    (alloc_flags & ALLOC_KSWAPD);
                                                       <── implicit clear
                                                           (no CPUSET in
                                                            reserve_flags)

              if (!(alloc_flags & ALLOC_CPUSET) || reserve_flags) {
                      ac->nodemask = NULL;          /* clear restriction */
                      ac->preferred_zoneref = first_zones_zonelist(...);
              }

         __alloc_pages_cpuset_fallback (used by may_oom NOFAIL and the
                                        slowpath final NOFAIL escape):
             page = get_page_from_freelist(gfp_mask, order,
                                           alloc_flags|ALLOC_CPUSET, ac);
             if (!page)
                     page = get_page_from_freelist(gfp_mask, order,
                                                   alloc_flags, ac);
             /* deliberate two-shot: cpuset first, then ignore cpuset */
```

## SUMMARY

[`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) (= `0x40`) is bit 6 of [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c). It controls whether the freelist scan calls [`__cpuset_zone_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cpuset.h#L88) on each zone. The actual cpuset enforcement lives in [`cpuset_current_node_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/kernel/cgroup/cpuset.c#L4395), which has a multi-tier rule set. Interrupt context always passes, nodes in [`current->mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) always pass, OOM victims always pass, [`__GFP_HARDWALL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) callers fail outside [`mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h), and non-hardwall (i.e. [`GFP_KERNEL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) callers may walk up to the nearest hardwalled ancestor cpuset.

The producer side is layered. [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4482) seeds the bit unconditionally. [`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4986) keeps the bit only when cpusets are enabled, sets [`ac->nodemask`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) to [`&cpuset_current_mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cpuset.h) for in-task callers without an explicit nodemask, or sets the [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) bit for callers that do supply one. [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4519) clears the bit for [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) non-blocking shapes (typical [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) so isolation does not starve interrupt-time allocations.

The slowpath uses cpuset retries to handle two race classes. [`check_retry_cpuset()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4660) detects cases where [`ac->nodemask`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) and [`current->mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) disagreed at decision time but agree now (cpuset update race), or where [`mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) was updated mid-allocation; in either case the slowpath restarts. The cookie machinery is provided by [`read_mems_allowed_begin()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cpuset.h) and [`read_mems_allowed_retry()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cpuset.h), called at the top of the slowpath.

[`__alloc_pages_cpuset_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4034) is a deliberate two-attempt helper. It always tries with [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) first (honouring the cpuset) and, if that fails, retries with [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) cleared (ignoring the cpuset). The helper is used by [`__alloc_pages_may_oom()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4132) for the [`__GFP_NOFAIL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) escape after [`out_of_memory()`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c#L1119) and by [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4972) for its final NOFAIL fallback.

## SPECIFICATIONS

(none; [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) is a Linux kernel internal flag)

## LINUX KERNEL

### Flag definition

- [`'\<ALLOC_CPUSET\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285): bit `0x40`. Comment: "check for correct cpuset".

### Producers

- [`'\<gfp_to_alloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4482): the seed. Sets [`ALLOC_WMARK_MIN | ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) as the initial value of [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) inside the slowpath rebuild.
- [`'\<prepare_alloc_pages\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L5005): the fast-path producer. Sets the bit when cpusets are enabled and the caller did not pre-supply [`ac->nodemask`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) (or sets [`ac->nodemask`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) to [`&cpuset_current_mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cpuset.h) when the caller did supply one).
- [`'\<__alloc_pages_cpuset_fallback\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4042): re-applies [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) to its first attempt regardless of the caller's bit.
- [`'\<__alloc_pages_may_oom\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4087): the high-watermark probe before invoking [`out_of_memory()`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c#L1119) uses [`ALLOC_WMARK_HIGH | ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h).
- [`'\<gfp_to_alloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4519): the clear. Drops [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) when [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) is set inside the non-blocking branch.
- [`'\<__alloc_pages_slowpath\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843): rebuild that overwrites [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) when [`reserve_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) is non-zero. The new value lacks [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) (because [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) and [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776) never set it). The subsequent `if (!(alloc_flags & ALLOC_CPUSET) || reserve_flags)` clears [`ac->nodemask`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h).

### Consumers

- [`'\<get_page_from_freelist\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3816): the per-zone gate. `if (cpusets_enabled() && (alloc_flags & ALLOC_CPUSET) && !__cpuset_zone_allowed(zone, gfp_mask)) continue;`
- [`'\<alloc_pages_bulk_noprof\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L5139): the bulk allocator's own per-zone gate. Same predicate.
- [`'\<should_reclaim_retry\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4620): the reclaim-retry zone scan also honours [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) before checking whether reclaim could make the threshold reachable.
- [`'\<try_to_compact_pages\>':'mm/compaction.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/compaction.c#L2845): direct compaction restricts its zone iteration to cpuset-allowed zones when [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) is set.
- [`'\<__alloc_pages_slowpath\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4854): the rebuild test that decides whether to clear [`ac->nodemask`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h).

### GFP mapping

- [`__GFP_HARDWALL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): orthogonal to the [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) bit; [`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4998) ORs [`__GFP_HARDWALL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) into [`alloc_gfp`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) when cpusets are enabled. [`cpuset_current_node_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/kernel/cgroup/cpuset.c#L4395) treats [`__GFP_HARDWALL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) callers as strict (no escape to ancestor cpusets).
- [`GFP_USER`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) (carries [`__GFP_HARDWALL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)): strict isolation. Outside [`mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) the request fails unless the task is OOM-killed or exiting.
- [`GFP_KERNEL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) (no [`__GFP_HARDWALL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)): can walk up to the nearest hardwalled ancestor.
- [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) (yields [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282)): [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4519) clears [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) for these.

### Cpuset interaction

- [`'\<cpusets_enabled\>':'include/linux/cpuset.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cpuset.h#L39): the static-key gate that lets [`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4995) and [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3816) skip cpuset logic entirely when no cpuset has ever been configured.
- [`'\<__cpuset_zone_allowed\>':'include/linux/cpuset.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cpuset.h#L88): the per-zone wrapper that calls [`cpuset_current_node_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/kernel/cgroup/cpuset.c#L4395) with [`zone_to_nid(z)`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h).
- [`'\<cpuset_current_node_allowed\>':'kernel/cgroup/cpuset.c'`](https://elixir.bootlin.com/linux/v6.19/source/kernel/cgroup/cpuset.c#L4395): the actual policy, with tiers ordered as interrupt -> always, node in [`mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) -> always, OOM victim -> always, [`__GFP_HARDWALL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) -> deny, [`PF_EXITING`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) -> always, otherwise scan up to nearest hardwalled ancestor.
- [`'\<cpuset_current_mems_allowed\>':'kernel/cgroup/cpuset.c'`](https://elixir.bootlin.com/linux/v6.19/source/kernel/cgroup/cpuset.c): the per-task [`nodemask_t`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/nodemask.h) cached in [`current`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/current.h). [`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L5003) points [`ac->nodemask`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) at it for in-task callers without an explicit mask.
- [`'\<check_retry_cpuset\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4660): cpuset-update race detection. Drops [`ac->nodemask`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) and returns true when the saved nodemask no longer fits the current cpuset; the slowpath then jumps to `restart`.
- [`'\<read_mems_allowed_begin\>':'include/linux/cpuset.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cpuset.h): seqlock cookie used by [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) to detect parallel cpuset updates.
- [`'\<__alloc_pages_cpuset_fallback\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4034): the two-attempt helper that retries with [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) cleared when the cpuset-restricted attempt failed.

### Related types

- [`'\<struct alloc_context\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h): owns [`nodemask`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) (the effective cpuset), [`zonelist`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h), [`preferred_zoneref`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h), [`highest_zoneidx`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h), and [`migratetype`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h).
- [`'\<struct cpuset\>':'kernel/cgroup/cpuset.c'`](https://elixir.bootlin.com/linux/v6.19/source/kernel/cgroup/cpuset.c): walked by [`nearest_hardwall_ancestor()`](https://elixir.bootlin.com/linux/v6.19/source/kernel/cgroup/cpuset.c) when a [`GFP_KERNEL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) request falls outside [`mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h).

## KERNEL DOCUMENTATION

- [`Documentation/admin-guide/cgroup-v2.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/admin-guide/cgroup-v2.rst): cgroup v2 cpuset controller, including [`cpuset.mems`](https://elixir.bootlin.com/linux/v6.19/source/kernel/cgroup/cpuset.c) (the source of [`mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h)) and `cpuset.mems.partition`.
- [`Documentation/admin-guide/cgroup-v1/cpusets.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/admin-guide/cgroup-v1/cpusets.rst): legacy cgroup v1 cpuset semantics, including the hardwall vs softwall distinction that [`__GFP_HARDWALL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) propagates.
- [`Documentation/admin-guide/sysctl/vm.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/admin-guide/sysctl/vm.rst): allocation policy and zone-reclaim tunables.

## OTHER SOURCES

## DETAILS

### Producer chain

[`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4986) is the fast-path producer. Its cpuset block.

```c
	if (cpusets_enabled()) {
		*alloc_gfp |= __GFP_HARDWALL;
		/*
		 * When we are in the interrupt context, it is irrelevant
		 * to the current task context. It means that any node ok.
		 */
		if (in_task() && !ac->nodemask)
			ac->nodemask = &cpuset_current_mems_allowed;
		else
			*alloc_flags |= ALLOC_CPUSET;
	}
```

An in-task caller without an explicit nodemask gets [`ac->nodemask`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) pointed at [`cpuset_current_mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/kernel/cgroup/cpuset.c), so the zonelist iterator inside [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) visits only allowed nodes and the [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) bit is unnecessary (the nodemask already constrains iteration). A caller with an explicit nodemask, or one running in interrupt context, leaves [`ac->nodemask`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) alone and sets [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) so the per-zone gate runs the full cpuset check on each candidate.

The `*alloc_gfp |= __GFP_HARDWALL` adds the hardwall bit unconditionally when cpusets are enabled, on the assumption that callers without an explicit override want strict isolation. This is what triggers the `if (gfp_mask & __GFP_HARDWALL) return false;` branch in [`cpuset_current_node_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/kernel/cgroup/cpuset.c#L4413) for nodes outside [`mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h).

[`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479) seeds the slowpath rebuild.

```c
static inline unsigned int
gfp_to_alloc_flags(gfp_t gfp_mask, unsigned int order)
{
	unsigned int alloc_flags = ALLOC_WMARK_MIN | ALLOC_CPUSET;
```

This unconditional seeding is correct because [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479) is called only from [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4757), which itself is reached only after [`prepare_alloc_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4986) has already validated whether cpusets matter for the request. The [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) bit will be honoured by [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) only when [`cpusets_enabled()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cpuset.h#L39) is also true, so the seeding is harmless on systems without cpusets.

### Producer-side clear: GFP_ATOMIC carve-out

The same producer also drops the bit for non-blocking [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) callers.

```c
	if (!(gfp_mask & __GFP_DIRECT_RECLAIM)) {
		/* ... ALLOC_NON_BLOCK / ALLOC_HIGHATOMIC handling ... */

		/*
		 * Ignore cpuset mems for non-blocking __GFP_HIGH (probably
		 * GFP_ATOMIC) rather than fail, see the comment for
		 * cpuset_current_node_allowed().
		 */
		if (alloc_flags & ALLOC_MIN_RESERVE)
			alloc_flags &= ~ALLOC_CPUSET;
	}
```

A [`GFP_ATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) caller running in interrupt context cannot block, often holds critical kernel locks, and would be impossible to reschedule onto an allowed node. Letting cpuset isolation cause an outright failure here would translate "cpuset configuration mismatch" into "kernel cannot service interrupt", which is unacceptable. The clearing is gated on [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) (which equates to [`__GFP_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) so that only "important" non-blocking callers get the carve-out, and a bare [`GFP_NOWAIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) request keeps cpuset isolation.

### Consumer: get_page_from_freelist per-zone gate

The actual enforcement lives in the per-zone scan.

```c
	for_next_zone_zonelist_nodemask(zone, z, ac->highest_zoneidx,
					ac->nodemask) {
		struct page *page;
		unsigned long mark;

		if (cpusets_enabled() &&
			(alloc_flags & ALLOC_CPUSET) &&
			!__cpuset_zone_allowed(zone, gfp_mask))
				continue;
```

The triple condition is short-circuited. [`cpusets_enabled()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cpuset.h#L39) is a static-branch check (zero overhead when no cpuset has been configured), [`alloc_flags & ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) is a single bit test, and [`__cpuset_zone_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cpuset.h#L88) is the function call, taken only when both predecessors hold.

[`__cpuset_zone_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/cpuset.h#L88) is a one-line wrapper.

```c
static inline bool __cpuset_zone_allowed(struct zone *z, gfp_t gfp_mask)
{
	return cpuset_current_node_allowed(zone_to_nid(z), gfp_mask);
}
```

The actual policy lives in [`cpuset_current_node_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/kernel/cgroup/cpuset.c#L4395).

```c
bool cpuset_current_node_allowed(int node, gfp_t gfp_mask)
{
	struct cpuset *cs;
	bool allowed;
	unsigned long flags;

	if (in_interrupt())
		return true;
	if (node_isset(node, current->mems_allowed))
		return true;
	/*
	 * Allow tasks that have access to memory reserves because they have
	 * been OOM killed to get memory anywhere.
	 */
	if (unlikely(tsk_is_oom_victim(current)))
		return true;
	if (gfp_mask & __GFP_HARDWALL)	/* If hardwall request, stop here */
		return false;

	if (current->flags & PF_EXITING) /* Let dying task have memory */
		return true;

	/* Not hardwall and node outside mems_allowed: scan up cpusets */
	spin_lock_irqsave(&callback_lock, flags);

	cs = nearest_hardwall_ancestor(task_cs(current));
	allowed = node_isset(node, cs->mems_allowed);

	spin_unlock_irqrestore(&callback_lock, flags);
	return allowed;
}
```

The function checks six tiers in order. Interrupt context returns true because cpuset enforcement makes no sense for an asynchronous interrupt that does not belong to the original task. A node already in [`current->mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) returns true. An OOM victim returns true because the dying task may need memory anywhere to finish exiting. A [`__GFP_HARDWALL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) caller (typically [`GFP_USER`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) returns false under strict isolation. A [`PF_EXITING`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) task returns true with the same rationale as the OOM victim. Anything else (typically [`GFP_KERNEL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) walks up to the nearest hardwalled ancestor and checks its [`mems_allowed`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h).

The takeaway for [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) is that the bit only gates whether this rule set runs at all. The rule set itself is universal and is short-circuited at every level.

### Slowpath rebuild and implicit clear

[`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843) recomputes [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) on retry.

```c
	reserve_flags = __gfp_pfmemalloc_flags(gfp_mask);
	if (reserve_flags)
		alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, reserve_flags) |
					  (alloc_flags & ALLOC_KSWAPD);

	/*
	 * Reset the nodemask and zonelist iterators if memory policies can be
	 * ignored. These allocations are high priority and system rather than
	 * user oriented.
	 */
	if (!(alloc_flags & ALLOC_CPUSET) || reserve_flags) {
		ac->nodemask = NULL;
		ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
					ac->highest_zoneidx, ac->nodemask);
	}
```

When [`reserve_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) is non-zero (the caller has earned [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) or [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273)), the rewrite drops [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) by overwrite (because [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) and [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3776) never produce that bit). Only [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) is carried over from the previous value. The follow-up `if (... || reserve_flags)` clears [`ac->nodemask`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) so the freelist scan covers every zone.

The `|| reserve_flags` clause is what makes a [`PF_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) reclaim driver or OOM victim ignore the cpuset, on the assumption that a caller deep enough to need reserves is also high-priority enough to be allowed across cpuset boundaries. The same intent is encoded in the third tier of [`cpuset_current_node_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/kernel/cgroup/cpuset.c#L4408), where [`tsk_is_oom_victim()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/oom.h#L74) returns true for the same task that gets [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273), so the two layers cooperate.

### Cpuset-update race detection

[`check_retry_cpuset()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4660) handles two race classes between the allocator and concurrent cpuset updates.

```c
static inline bool
check_retry_cpuset(int cpuset_mems_cookie, struct alloc_context *ac)
{
	/*
	 * It's possible that cpuset's mems_allowed and the nodemask from
	 * mempolicy don't intersect. This should be normally dealt with by
	 * policy_nodemask(), but it's possible to race with cpuset update in
	 * such a way the check therein was true, and then it became false
	 * before we got our cpuset_mems_cookie here.
	 * This assumes that for all allocations, ac->nodemask can come only
	 * from MPOL_BIND mempolicy (whose documented semantics is to be ignored
	 * when it does not intersect with the cpuset restrictions) or the
	 * caller can deal with a violated nodemask.
	 */
	if (cpusets_enabled() && ac->nodemask &&
			!cpuset_nodemask_valid_mems_allowed(ac->nodemask)) {
		ac->nodemask = NULL;
		return true;
	}

	/*
	 * When updating a task's mems_allowed or mempolicy nodemask, it is
	 * possible to race with parallel threads in such a way that our
	 * allocation can fail while the mask is being updated. If we are about
	 * to fail, check if the cpuset changed during allocation and if so,
	 * retry.
	 */
	if (read_mems_allowed_retry(cpuset_mems_cookie))
		return true;

	return false;
}
```

The function is called from [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) at the `retry:` label and again at `nopage:` before deciding to fail. A returned `true` triggers `goto restart` (which re-seeds the cpuset cookie and reruns [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479)); the slowpath retries the entire allocation with the updated cpuset state.

### __alloc_pages_cpuset_fallback: the deliberate two-attempt helper

The helper that the NOFAIL paths use to guarantee an attempt with cpuset isolation followed by an attempt without.

```c
static inline struct page *
__alloc_pages_cpuset_fallback(gfp_t gfp_mask, unsigned int order,
			      unsigned int alloc_flags,
			      const struct alloc_context *ac)
{
	struct page *page;

	page = get_page_from_freelist(gfp_mask, order,
			alloc_flags|ALLOC_CPUSET, ac);
	/*
	 * fallback to ignore cpuset restriction if our nodes
	 * are depleted
	 */
	if (!page)
		page = get_page_from_freelist(gfp_mask, order,
				alloc_flags, ac);
	return page;
}
```

The first attempt forces [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) on top of the caller's [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c). The second attempt uses the original [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) without the OR. This means the fallback only ignores the cpuset if the original `alloc_flags` did not contain [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) to begin with. A caller that wants the strict-then-fallback semantics passes `alloc_flags = 0` (or `alloc_flags = ALLOC_NO_WATERMARKS`); a caller that wants always-strict passes `alloc_flags |= ALLOC_CPUSET` so both attempts honour the cpuset.

[`__alloc_pages_may_oom()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4132) calls this helper with [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) for the [`__GFP_NOFAIL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) escape, attempting cpuset + no-watermarks first and then ignoring cpuset. [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4972) calls it with [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) for its NOFAIL final escape, attempting cpuset + min-reserve first and then ignoring cpuset.

### Consumer: try_to_compact_pages

Direct compaction also honours [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285).

```c
/* mm/compaction.c, in try_to_compact_pages, line ~2845 */
		if (cpusets_enabled() &&
		    (alloc_flags & ALLOC_CPUSET) &&
		    !__cpuset_zone_allowed(zone, gfp_mask))
			continue;
```

The compaction zone iteration runs before the freelist scan (when `__alloc_pages_direct_compact()` is called), so respecting [`ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1285) at this level avoids spending compaction effort on zones whose pages would be unusable to the caller anyway.

### Consumer: should_reclaim_retry

The reclaim-retry zone scan also runs through the cpuset gate before doing the watermark probe.

```c
/* mm/page_alloc.c, in should_reclaim_retry, line ~4620 */
		if (cpusets_enabled() &&
			(alloc_flags & ALLOC_CPUSET) &&
			!__cpuset_zone_allowed(zone, gfp_mask))
				continue;
```

The function uses `[__zone_watermark_ok()](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585)` with `available = NR_FREE_PAGES + zone_reclaimable_pages(zone)` to ask "would reclaim be enough?" If no cpuset-allowed zone could pass the watermark even after full reclaim, the slowpath stops retrying and proceeds to OOM. Cpuset filtering here is essential. A system with enough total reclaimable memory but no allowed zone would otherwise loop forever on direct reclaim.
