---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# ALLOC_OOM

[`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) is the page allocator's grant for tasks the OOM killer has marked as victims. It is weaker than [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) (the bypass) and is meant to give a doomed task one cheap pass through the freelist so it can finish its exit path and return memory. The flag is produced exclusively by [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4566) when [`oom_reserves_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4530) returns true, which in turn requires [`tsk_is_oom_victim()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/oom.h#L74) to return true (a victim has had [`mark_oom_victim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c#L767) attach an [`oom_mm`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/signal.h) to its [`signal_struct`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/signal.h)). Once set, the flag participates in the [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) bundle. [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3627) cuts the min watermark in half, and the high-order check in the same function lets the victim consume from [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) free areas. [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3250) lets victims dip into the high-atomic pool as a last-chance fallback before the zone returns NULL.

```
        ALLOC_* bit field (12 bits live in v6.19)
        ─────────────────────────────────────────────────────────────
        bit:  11    10    9     8     7     6     5     4     3     2     1     0
              ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
              │  K  │  T  │ HA  │ NF  │ CMA │ CP  │ MR  │ NB  │ OOM │ NW  │  WMARK    │
              └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                                                              ▲
                                                              │
                                                  ALLOC_OOM = 0x08 (MMU only)

        Legend: K  = ALLOC_KSWAPD          NF  = ALLOC_NOFRAGMENT     NB  = ALLOC_NON_BLOCK
                T  = ALLOC_TRYLOCK         CP  = ALLOC_CPUSET         OOM = ALLOC_OOM
                HA = ALLOC_HIGHATOMIC      MR  = ALLOC_MIN_RESERVE    NW  = ALLOC_NO_WATERMARKS
```

```
        OOM victim path -> ALLOC_OOM -> reserves access
        ──────────────────────────────────────────────────────────────

         Process context allocator runs out of memory.
                              │
                              ▼
              __alloc_pages_slowpath:
                  page = __alloc_pages_may_oom(...)
                              │
                              ▼
              __alloc_pages_may_oom:
                  mutex_trylock(&oom_lock)
                  page = get_page_from_freelist(
                      gfp_mask & ~__GFP_DIRECT_RECLAIM,
                      order,
                      ALLOC_WMARK_HIGH | ALLOC_CPUSET, ac);
                  /* high-watermark probe to catch parallel OOM */
                              │
                              ▼  (still NULL)
                  out_of_memory(&oc):
                      select_bad_process / current is exiting / NOFS ...
                      mark_oom_victim(victim) -> tsk->signal->oom_mm = mm
                                                 set TIF_MEMDIE
                              │
                              ▼  (returns true)
                  if (gfp_mask & __GFP_NOFAIL)
                      page = __alloc_pages_cpuset_fallback(
                          gfp_mask, order, ALLOC_NO_WATERMARKS, ac);
                  /* NOFAIL gets the bypass; everyone else returns NULL
                     and re-enters __alloc_pages_slowpath retry loop */
                              │
                              ▼
              Next slowpath iteration:
                  reserve_flags = __gfp_pfmemalloc_flags(gfp_mask):
                      !__GFP_NOMEMALLOC && !__GFP_MEMALLOC
                      && !in_interrupt && !PF_MEMALLOC
                      && oom_reserves_allowed(current)
                                  └─ tsk_is_oom_victim(current)
                                       └─ current->signal->oom_mm
                      -> ALLOC_OOM   (= 0x08)
                              │
                              ▼
                  alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, ALLOC_OOM)
                                | (alloc_flags & ALLOC_KSWAPD)
                  ac->nodemask = NULL  (cpuset/policy ignored)
                              │
                              ▼
              get_page_from_freelist + __zone_watermark_ok:
                  alloc_flags & ALLOC_RESERVES is true
                  if (alloc_flags & ALLOC_OOM)
                          min -= min / 2;          (50% off)
                  high-order: MIGRATE_HIGHATOMIC free area is acceptable
                              │
                              ▼
              rmqueue_buddy:
                  __rmqueue() returns NULL?
                  if (alloc_flags & (ALLOC_OOM|ALLOC_NON_BLOCK))
                          page = __rmqueue_smallest(zone, order,
                                                    MIGRATE_HIGHATOMIC);
                              │
                              ▼
              On success: prep_new_page (no pfmemalloc tag because
                          ALLOC_NO_WATERMARKS is NOT set).
                              │
              On retry exhaust:
                  if (tsk_is_oom_victim(current) && (alloc_flags & ALLOC_OOM))
                          goto nopage;   /* avoid infinite loop */
```

## SUMMARY

[`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) is bit 3 of [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) (= `0x08`) when [`CONFIG_MMU`](https://elixir.bootlin.com/linux/v6.19/source/init/Kconfig) is set, which this page assumes throughout. Together with [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278), [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282), and [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292), it forms [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297), the bundle that [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3602) tests as a single condition before applying per-bit watermark cuts.

The flag enters the allocator only through [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549). Inside that function, the precondition cascade requires the absence of [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) and [`__GFP_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h), the task to be outside interrupt context, the absence of [`PF_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) (any earlier branch wins and returns [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) instead), and finally [`oom_reserves_allowed(current)`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4530) returning true. The outer entry that drives this path is the retry loop in [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843), which calls [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) on every iteration after the first attempt has failed.

[`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) is read in three places. [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3627) cuts the min watermark by half and allows [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) high-order free areas, [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3250) does a last-chance pull from [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) when the regular [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) returns NULL, and the slowpath retry termination test in [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4934) boots a victim that is still spinning on the OOM grant to the no-page exit. Indirect interaction also goes through [`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567), where membership in [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) suppresses the `nr_free_highatomic` subtraction so the victim can see the high-atomic reserves as available.

The flag is mutually exclusive with [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) at production. [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) returns one or the other, never both. A [`PF_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) reclaim driver gets the stronger bypass; an OOM victim gets the watermark cut. The asymmetry exists because the reclaim driver actively frees memory while the victim is on its way out.

## SPECIFICATIONS

(none; [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) is a Linux kernel internal flag)

## LINUX KERNEL

### Flag definition

- [`'\<ALLOC_OOM\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273): bit `0x08` under [`CONFIG_MMU`](https://elixir.bootlin.com/linux/v6.19/source/init/Kconfig). The non-MMU branch aliases [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1275) to [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262); this page assumes the MMU branch.
- [`'\<ALLOC_RESERVES\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297): the bundle macro `(ALLOC_NON_BLOCK | ALLOC_MIN_RESERVE | ALLOC_HIGHATOMIC | ALLOC_OOM)`. Used by [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3602) and [`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567) as a single test before per-bit handling.

### Producers

- [`'\<__gfp_pfmemalloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549): the only function that returns the literal value [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273).
- [`'\<oom_reserves_allowed\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4530): the predicate. Returns true when the current task is an OOM victim. Wraps [`tsk_is_oom_victim()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/oom.h#L74) and adds the !MMU [`TIF_MEMDIE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) special case (skipped here).
- [`'\<tsk_is_oom_victim\>':'include/linux/oom.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/oom.h#L74): one-line accessor that reads [`tsk->signal->oom_mm`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/signal.h). The pointer is set when [`mark_oom_victim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c#L767) attaches the victim's [`mm_struct`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) to its [`signal_struct`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/signal.h).
- [`'\<__alloc_pages_slowpath\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843): the call site that actually wires [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) into [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843) (via [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549)) on every retry iteration.

### Consumers

- [`'\<__zone_watermark_ok\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3627): cuts `min` by 50% when [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) is set, on top of the [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) entry test.
- [`'\<__zone_watermark_ok\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3662): the high-order check accepts a [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) free area when [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) (or [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292)) is set.
- [`'\<__zone_watermark_unusable_free\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567): suppresses the `unusable_free += READ_ONCE(z->nr_free_highatomic)` subtraction when [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) is set, so the high-atomic reserve counts as free.
- [`'\<rmqueue_buddy\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3250): pulls from the [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) freelist via [`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) when [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) returned NULL and either [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) or [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) is set.
- [`'\<__alloc_pages_slowpath\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4934): the loop-termination check `tsk_is_oom_victim(current) && (alloc_flags & ALLOC_OOM || (gfp_mask & __GFP_NOMEMALLOC))` jumps to the no-page exit so a victim does not spin forever.

### GFP mapping

- [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): blocks [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) (returns 0 from [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) before the OOM branch is reached).
- [`__GFP_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): wins over the OOM branch and yields [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) instead. Even an OOM victim that asked for the explicit grant gets the bypass.
- [`PF_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h): same outcome. The OOM branch is reached only when the task does NOT have [`PF_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) set.
- [`__GFP_NOFAIL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): orthogonal but interacts. After [`out_of_memory()`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c#L1119) returns, [`__alloc_pages_may_oom()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4053) hands a NOFAIL caller the stronger [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) bypass directly, without going through the [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) path.

### Watermarks and reserves

- [`'\<__zone_watermark_ok\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585): the function that consumes the bit. The min cut from [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) (50%) compounds with the cut from [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) (also 50%) when both are set, leaving the threshold at 25% of the per-zone min.
- [`'\<wmark_pages\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1077): provides `mark` to [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585) but does not see the OOM bit; the cut is applied inside [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585).
- [`'\<MIGRATE_HIGHATOMIC\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h): the migrate-type pool that [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) victims are allowed to drain through both [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3662) and [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3250).

### OOM control plane

- [`'\<__alloc_pages_may_oom\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4053): the oom-lock-protected entry from [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4922). Calls [`out_of_memory()`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c#L1119) and on success may hand [`__GFP_NOFAIL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) callers the [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) escape via [`__alloc_pages_cpuset_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4034).
- [`'\<out_of_memory\>':'mm/oom_kill.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c#L1119): selects a victim and calls [`mark_oom_victim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c#L767). The marking is what flips [`tsk_is_oom_victim()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/oom.h#L74) for the chosen task, which is what eventually flips [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) into the [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) branch.
- [`'\<mark_oom_victim\>':'mm/oom_kill.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c#L767): sets [`TIF_MEMDIE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) and `tsk->signal->oom_mm`. Held under [`oom_lock`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c).
- [`'\<__alloc_pages_cpuset_fallback\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4034): the two-shot wrapper that retries [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3791) once with the caller's cpuset and once without.

## KERNEL DOCUMENTATION

- [`Documentation/admin-guide/mm/concepts.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/admin-guide/mm/concepts.rst): user-facing description of the OOM killer and reclaim behaviour that produce the conditions under which [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) is granted.
- [`Documentation/admin-guide/sysctl/vm.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/admin-guide/sysctl/vm.rst): describes `panic_on_oom`, `oom_kill_allocating_task`, and the watermark tunables that frame the OOM path.
- [`Documentation/mm/physical_memory.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/physical_memory.rst): the per-zone watermark model that the [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) bundle scales down.

## OTHER SOURCES

## DETAILS

### Flag bit

[`mm/internal.h`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1267) keeps the OOM grant in a small `#ifdef`.

```c
/*
 * Only MMU archs have async oom victim reclaim - aka oom_reaper so we
 * cannot assume a reduced access to memory reserves is sufficient for
 * !MMU
 */
#ifdef CONFIG_MMU
#define ALLOC_OOM		0x08
#else
#define ALLOC_OOM		ALLOC_NO_WATERMARKS
#endif
```

The MMU branch is what this page documents. On MMU systems the [`oom_reaper`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c) thread asynchronously frees the victim's anonymous memory, so a partial-reserve grant is enough to let the victim finish exiting; on !MMU systems there is no reaper, and the kernel cannot assume forward progress, so the alias to [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) gives the victim full bypass instead. The MMU branch is the only one analysed below.

The bit is part of the [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) bundle.

```c
/* Flags that allow allocations below the min watermark. */
#define ALLOC_RESERVES (ALLOC_NON_BLOCK|ALLOC_MIN_RESERVE|ALLOC_HIGHATOMIC|ALLOC_OOM)
```

[`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3558) and [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585) test the bundle as a single condition before drilling into per-bit behaviour.

### Producer chain

The OOM grant is driven by the same [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) that produces [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262), but on a different branch.

```c
static inline int __gfp_pfmemalloc_flags(gfp_t gfp_mask)
{
	if (unlikely(gfp_mask & __GFP_NOMEMALLOC))
		return 0;
	if (gfp_mask & __GFP_MEMALLOC)
		return ALLOC_NO_WATERMARKS;
	if (in_serving_softirq() && (current->flags & PF_MEMALLOC))
		return ALLOC_NO_WATERMARKS;
	if (!in_interrupt()) {
		if (current->flags & PF_MEMALLOC)
			return ALLOC_NO_WATERMARKS;
		else if (oom_reserves_allowed(current))
			return ALLOC_OOM;
	}

	return 0;
}
```

The OOM branch is reached only after every stronger grant has been ruled out, with no [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) opt-out, no [`__GFP_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) explicit grant, no interrupt context, and no [`PF_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) on the task. [`oom_reserves_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4530) is the gate.

```c
static bool oom_reserves_allowed(struct task_struct *tsk)
{
	if (!tsk_is_oom_victim(tsk))
		return false;

	/*
	 * !MMU doesn't have oom reaper so give access to memory reserves
	 * only to the thread with TIF_MEMDIE set
	 */
	if (!IS_ENABLED(CONFIG_MMU) && !test_thread_flag(TIF_MEMDIE))
		return false;

	return true;
}
```

The MMU branch reduces to `tsk_is_oom_victim(tsk)`, which in turn is one assignment-comparison.

```c
static inline bool tsk_is_oom_victim(struct task_struct * tsk)
{
	return tsk->signal->oom_mm;
}
```

[`tsk->signal->oom_mm`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/signal.h) is a [`struct mm_struct`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) pointer that [`mark_oom_victim()`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c#L767) installs once per victim.

```c
static void mark_oom_victim(struct task_struct *tsk)
{
	const struct cred *cred;
	struct mm_struct *mm = tsk->mm;

	WARN_ON(oom_killer_disabled);
	/* OOM killer might race with memcg OOM */
	if (test_and_set_tsk_thread_flag(tsk, TIF_MEMDIE))
		return;

	/* oom_mm is bound to the signal struct life time. */
	if (!cmpxchg(&tsk->signal->oom_mm, NULL, mm))
		mmgrab(tsk->signal->oom_mm);
	/* ... thaw + bookkeeping ... */
}
```

The [`cmpxchg`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/atomic.h) and [`TIF_MEMDIE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) make the marking idempotent across racing OOM events. Once marked, the victim's [`tsk_is_oom_victim()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/oom.h#L74) is permanently true for the lifetime of the [`signal_struct`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/signal.h), and every subsequent [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) call from that task that does not have a stronger grant returns [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273).

### Slowpath wiring through retry

[`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843) re-evaluates [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) on every retry. The relevant block lives inside the `retry:` label.

```c
retry:
	/* ... cpuset/zonelist re-check ... */

	/* Ensure kswapd doesn't accidentally go to sleep as long as we loop */
	if (alloc_flags & ALLOC_KSWAPD)
		wake_all_kswapds(order, gfp_mask, ac);

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

The first time a task becomes an OOM victim, the next iteration of this block flips `reserve_flags` from `0` to [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273). [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) is rewritten to [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) (plus optional [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) and the carried-over [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) bit), and the `|| reserve_flags` clause clears the cpuset nodemask so the freelist scan can spread across every zone. After that, [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3791) sees [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) inside [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) and forwards it down to [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585) and [`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3250).

### Watermark cut: the ALLOC_RESERVES bundle arithmetic

[`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585) carves out the reserves in a layered way. The unconditional path starts with `min = mark` and subtracts unusable free pages. Then the bundle test triggers per-bit cuts.

```c
	if (unlikely(alloc_flags & ALLOC_RESERVES)) {
		/*
		 * __GFP_HIGH allows access to 50% of the min reserve as well
		 * as OOM.
		 */
		if (alloc_flags & ALLOC_MIN_RESERVE) {
			min -= min / 2;

			/*
			 * Non-blocking allocations (e.g. GFP_ATOMIC) can
			 * access more reserves than just __GFP_HIGH. Other
			 * non-blocking allocations requests such as GFP_NOWAIT
			 * or (GFP_KERNEL & ~__GFP_DIRECT_RECLAIM) do not get
			 * access to the min reserve.
			 */
			if (alloc_flags & ALLOC_NON_BLOCK)
				min -= min / 4;
		}

		/*
		 * OOM victims can try even harder than the normal reserve
		 * users on the grounds that it's definitely going to be in
		 * the exit path shortly and free memory. Any allocation it
		 * makes during the free path will be small and short-lived.
		 */
		if (alloc_flags & ALLOC_OOM)
			min -= min / 2;
	}
```

Each cut shrinks the remaining `min`, so the cuts compound rather than add. Starting from the per-zone min watermark `M`, bare [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) (no other bundle bits) takes `min` to `M - M/2 = M/2`, allowing the victim down to half the configured min. [`ALLOC_OOM | ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) runs the [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) cut first (`M/2`), then the [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) cut halves what remains (`M/4`). [`ALLOC_OOM | ALLOC_MIN_RESERVE | ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) runs [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) (50%), then [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) inside the same `if` (an additional 25% of the prior `M/2`, leaving `M/2 - M/8 = 3M/8`), then [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) (50% of `3M/8`, leaving `3M/16`).

In practice, an OOM victim almost always reaches [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585) with only [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) set (and possibly [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) which does not affect the cut). The slowpath rebuild at line 4843 overwrites `alloc_flags` with `ALLOC_OOM | (alloc_flags & ALLOC_KSWAPD)`, so [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) and [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) from the first attempt get dropped in the rebuild.

### High-order check: MIGRATE_HIGHATOMIC acceptance

After the order-0 watermark passes, [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3640) walks the orders looking for a free area.

```c
	/* For a high-order request, check at least one suitable page is free */
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

The standard PCP migrate-types ([`MIGRATE_UNMOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), [`MIGRATE_MOVABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), [`MIGRATE_RECLAIMABLE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h)) are always considered. [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) opens up [`MIGRATE_CMA`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h). [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) (or [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292)) opens up [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h). The point of the bundle test in [`__zone_watermark_unusable_free()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3567) becomes clear here. Since the high-atomic reserve is going to count as available later, it would be wrong to subtract `nr_free_highatomic` from the order-0 free pool, so the bundle membership tells the unusable-free routine to leave it in.

### Last-chance pull from MIGRATE_HIGHATOMIC

[`rmqueue_buddy()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3222) is the slow buddy-list path inside [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c). After the regular [`__rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) walks the migrate-type fallback chain and returns NULL, the OOM (or non-block) caller gets one more shot.

```c
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

[`__rmqueue_smallest()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) is the buddy-list walker that returns the smallest free block of the requested order from the named migrate-type. For the OOM victim path the migrate-type is forced to [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h), which is the pool that [`reserve_highatomic_pageblock()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) (called from [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) on successful [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292) allocations) populates. According to the comment, an order-0 atomic or OOM caller failing now is worse than a future high-order atomic caller failing, so the reserve is opened up.

### Loop-termination check

[`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4934) protects against an OOM victim that keeps spinning without making progress.

```c
	/* Reclaim has failed us, start killing things */
	page = __alloc_pages_may_oom(gfp_mask, order, ac, &did_some_progress);
	if (page)
		goto got_pg;

	/* Avoid allocations with no watermarks from looping endlessly */
	if (tsk_is_oom_victim(current) &&
	    (alloc_flags & ALLOC_OOM ||
	     (gfp_mask & __GFP_NOMEMALLOC)))
		goto nopage;
```

A task that is itself the OOM victim and is still asking the allocator to retry with [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) (or that explicitly opted out of reserves with [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h)) is bumped to the no-page exit. The first time through the loop, the victim is granted [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) and gets to try the discounted watermark; if that still fails after another retry, the loop terminates because the victim's job is to exit, not to satisfy unbounded allocations.

### The __GFP_NOFAIL OOM escape uses the stronger flag

[`__alloc_pages_may_oom()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4053) handles the [`__GFP_NOFAIL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) special case differently from a normal victim allocation. After [`out_of_memory()`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c#L1119) reports success, NOFAIL goes straight to [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262).

```c
	/* Exhausted what can be done so it's blame time */
	if (out_of_memory(&oc) ||
	    WARN_ON_ONCE_GFP(gfp_mask & __GFP_NOFAIL, gfp_mask)) {
		*did_some_progress = 1;

		/*
		 * Help non-failing allocations by giving them access to memory
		 * reserves
		 */
		if (gfp_mask & __GFP_NOFAIL)
			page = __alloc_pages_cpuset_fallback(gfp_mask, order,
					ALLOC_NO_WATERMARKS, ac);
	}
```

The reason this NOFAIL helper does NOT use [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) is that the caller asking for NOFAIL is probably not the victim and probably does not have `oom_mm` attached, so a later [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) call would return `0` and lose the grant on the next retry. Using [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) directly here gives the NOFAIL allocation a single clean shot under [`oom_lock`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c) protection.

### ALLOC_OOM is weaker than ALLOC_NO_WATERMARKS

The two flags act on different layers of the watermark machinery. [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) is the bypass switch in [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3916), where the watermark check is skipped entirely, and the caller is expected to free more memory than it consumes, justifying the deeper reach. [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) is a bend, not a bypass, since the watermark is still consulted but cut by 50% (and [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) becomes spendable), reflecting that the victim is on its way out and frees memory only as it dies.

The asymmetry shows up in [`prep_new_page()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1903) too. Only [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) sets [`page->lru.next |= BIT(1)`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2566) (the pfmemalloc marker that downstream consumers like [`SK_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/net/sock.h) read). An OOM-allocated page is indistinguishable from a normal allocation once it leaves the allocator, with no per-page tag, only the per-task victim status carried in [`signal->oom_mm`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/signal.h).
