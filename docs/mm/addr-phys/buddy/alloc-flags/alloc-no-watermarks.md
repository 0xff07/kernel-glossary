---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# ALLOC_NO_WATERMARKS

[`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) is the page allocator's emergency-reserve switch. When a caller is on a forward-progress path that the rest of the system depends on (a memory reclaimer, a network softirq draining a swap-over-NFS write, or a process with [`PF_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) set), the producer [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) returns this flag so that the freelist scan in [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3791) can bypass the per-zone watermark check entirely. Pages obtained that way are tagged via [`set_page_pfmemalloc()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2564) so downstream consumers (notably the socket layer) can recognise that the page came out of reserves and steer it only towards traffic that is part of the memory-freeing path.

```
        ALLOC_* bit field (12 bits live in v6.19)
        ─────────────────────────────────────────────────────────────
        bit:  11    10    9     8     7     6     5     4     3     2     1     0
              ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
              │  K  │  T  │ HA  │ NF  │ CMA │ CP  │ MR  │ NB  │ OOM │ NW  │  WMARK    │
              └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                                                                    ▲     ▲─────▲
                                                                    │        │
                                                  ALLOC_NO_WATERMARKS        ALLOC_WMARK_MASK
                                                          (0x04)              (= NO_WATERMARKS - 1
                                                                                = 0x03; the low 2
                                                                                bits index zone
                                                                                ->_watermark[])

        Legend: K  = ALLOC_KSWAPD          NF  = ALLOC_NOFRAGMENT     NB  = ALLOC_NON_BLOCK
                T  = ALLOC_TRYLOCK         CP  = ALLOC_CPUSET         OOM = ALLOC_OOM
                HA = ALLOC_HIGHATOMIC      MR  = ALLOC_MIN_RESERVE    NW  = ALLOC_NO_WATERMARKS
```

```
        Caller path -> ALLOC_NO_WATERMARKS -> freelist short-circuit
        ──────────────────────────────────────────────────────────────

          gfp_mask                          current->flags
          ┌───────────────────────┐         ┌───────────────────────┐
          │ __GFP_MEMALLOC        │         │ PF_MEMALLOC (set by   │
          │ (or context bits      │         │ memalloc_noreclaim_*, │
          │  PF_MEMALLOC, etc.)   │         │ kthread_use_mm, etc.) │
          └───────────┬───────────┘         └───────────┬───────────┘
                      │                                 │
                      ▼                                 ▼
          ┌─────────────────────────────────────────────────────────┐
          │ __gfp_pfmemalloc_flags(gfp_mask)                        │
          │   __GFP_NOMEMALLOC                ->  0                 │
          │   __GFP_MEMALLOC                  ->  ALLOC_NO_WATERMARKS │
          │   in_serving_softirq && PF_MEMALLOC -> ALLOC_NO_WATERMARKS │
          │   !in_interrupt && PF_MEMALLOC    ->  ALLOC_NO_WATERMARKS │
          │   !in_interrupt && oom_reserves   ->  ALLOC_OOM         │
          └─────────────────────────────┬───────────────────────────┘
                                        │ (returns reserve_flags)
                                        ▼
          __alloc_pages_slowpath, second pass:
              alloc_flags = gfp_to_alloc_flags_cma(gfp, reserve_flags)
                          | (alloc_flags & ALLOC_KSWAPD)
                                        │
                                        ▼
          get_page_from_freelist iterates the zonelist:
              mark = wmark_pages(zone, alloc_flags & ALLOC_WMARK_MASK)
              if (!zone_watermark_fast(...)) {
                  /* watermark fails ... but: */
                  BUILD_BUG_ON(ALLOC_NO_WATERMARKS < NR_WMARK);
                  if (alloc_flags & ALLOC_NO_WATERMARKS)
                          goto try_this_zone;        <── bypass
              }
                                        │
                                        ▼
          rmqueue() pulls a page from whatever pages remain
                                        │
                                        ▼
          prep_new_page:
              if (alloc_flags & ALLOC_NO_WATERMARKS)
                      set_page_pfmemalloc(page)      // page->lru.next |= BIT(1)
              else
                      clear_page_pfmemalloc(page)
                                        │
                                        ▼
          page_is_pfmemalloc(page) returns true downstream so that
          callers (e.g. the SK_MEMALLOC socket fast path) can refuse
          to use the page for traffic that is unrelated to reclaim.
```

## SUMMARY

[`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) lives at bit 2 of the per-allocation [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) word. The two bits below it form the watermark index (extracted by [`ALLOC_WMARK_MASK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1265) and selected by [`wmark_pages()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1077) into [`zone->_watermark[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L883)). The bit position is verified at compile time by [`BUILD_BUG_ON(ALLOC_NO_WATERMARKS < NR_WMARK)`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3915), which would fire if a future watermark added to [`enum zone_watermarks`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L708) overflowed into the no-watermarks bit.

The flag is produced exclusively by [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549), which is called from [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843) after the fast-path attempt has failed and from [`gfp_pfmemalloc_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4567). [`__alloc_pages_may_oom()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4053) sets it explicitly through [`__alloc_pages_cpuset_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4132) for [`__GFP_NOFAIL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) requests after [`out_of_memory()`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c) succeeds. The flag is consumed in two places. [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3916) skips the watermark-failure goto and jumps straight to [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c), and [`prep_new_page()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1903) marks the resulting [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) as pfmemalloc.

[`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) is stronger than every member of [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) (the bundle that scales the min watermark down by fractions). The reserves family bends the watermark, but the bypass flag ignores it entirely. [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585) does NOT itself test [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262), because the watermark-OK function is called only when the caller wants the watermark enforced. The bypass happens one level up, after the watermark function has returned false.

## SPECIFICATIONS

(none; [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) is a Linux kernel internal flag)

## LINUX KERNEL

### Flag definition

- [`'\<ALLOC_NO_WATERMARKS\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262): the bypass bit (`0x04`). Comment: "don't check watermarks at all".
- [`'\<ALLOC_WMARK_MASK\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1265): defined as `ALLOC_NO_WATERMARKS - 1` (= `0x03`); extracts the 2-bit watermark index from `alloc_flags`.
- [`'\<ALLOC_WMARK_MIN\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1259), [`'\<ALLOC_WMARK_LOW\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1260), [`'\<ALLOC_WMARK_HIGH\>':'mm/internal.h'`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1261): the encoded values that occupy the masked bits and index [`zone->_watermark[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L883).

### Producers

- [`'\<__gfp_pfmemalloc_flags\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549): the only function that returns the literal value [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262). Called from [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843) and exposed as [`gfp_pfmemalloc_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4567).
- [`'\<__alloc_pages_slowpath\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4693): the second-pass [`alloc_flags`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843) recomputation merges the result of [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) with the previously-set [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) bit and routes it through [`gfp_to_alloc_flags_cma()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c).
- [`'\<__alloc_pages_may_oom\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4053): on [`__GFP_NOFAIL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) requests after [`out_of_memory()`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c) returns true, hands [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) to [`__alloc_pages_cpuset_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4132).

### Consumers

- [`'\<get_page_from_freelist\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3916): the bypass site. After [`zone_watermark_fast()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) reports the watermark unmet and after deferred-grow fails, the test `if (alloc_flags & ALLOC_NO_WATERMARKS) goto try_this_zone;` jumps directly to [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c).
- [`'\<prep_new_page\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1889): tags the freshly-allocated [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) by calling [`set_page_pfmemalloc()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2564) so the network and slab layers can detect reserve-allocated pages later.
- [`'\<__alloc_pages_slowpath\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4968): comment-only consumer. The [`__GFP_NOFAIL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) loop deliberately uses [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) instead of [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) "because this could deplete whole memory reserves which would just make the situation worse".

### GFP mapping

- [`__GFP_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): the loud, explicit grant. [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) returns [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) immediately.
- [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h): the explicit denial. Forces a return of `0` even if the task carries [`PF_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h).
- [`PF_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h): per-task flag granted by [`memalloc_noreclaim_save()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/mm.h) and similar helpers. Carries the grant in either softirq context (when the task that hit the softirq itself owns [`PF_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h)) or process context.
- The OOM-victim case ([`oom_reserves_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/oom.h)) returns the weaker [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) instead, which is part of [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) (cuts the watermark to 1/4) rather than bypassing it.

### Watermarks and reserves

- [`'\<wmark_pages\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1077): looks up [`zone->_watermark[w]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L883) plus [`zone->watermark_boost`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L884). [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3897) calls it with `alloc_flags & ALLOC_WMARK_MASK`.
- [`'\<enum zone_watermarks\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L708): defines [`WMARK_MIN`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L709), [`WMARK_LOW`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L710), [`WMARK_HIGH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L711), [`WMARK_PROMO`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L712), and [`NR_WMARK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L713) (= 4 in v6.19).
- [`'\<__zone_watermark_ok\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585): does NOT itself check [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262); the bypass happens in the caller. Reads `alloc_flags & ALLOC_RESERVES` and the [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273)/[`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282)/[`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278) sub-bits to scale the threshold.

### Related types and helpers

- [`'\<set_page_pfmemalloc\>':'include/linux/mm.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2564): writes `BIT(1)` into [`page->lru.next`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h). Called only by [`prep_new_page()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1903).
- [`'\<clear_page_pfmemalloc\>':'include/linux/mm.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2569): clears the marker. Called by [`prep_new_page()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1905) on the non-bypass path.
- [`'\<page_is_pfmemalloc\>':'include/linux/mm.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2535): reads the marker. Used by SK_MEMALLOC paths and the page pool to decide whether a page may be recycled for non-emergency traffic.
- [`'\<folio_is_pfmemalloc\>':'include/linux/mm.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2550): folio variant of the same predicate.
- [`'\<gfp_pfmemalloc_allowed\>':'mm/page_alloc.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4567): boolean shim over [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) used by [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4787) to skip direct compaction when reserves are accessible.

## KERNEL DOCUMENTATION

- [`Documentation/admin-guide/sysctl/vm.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/admin-guide/sysctl/vm.rst): describes the user-tunable knobs (`min_free_kbytes`, `watermark_scale_factor`, `watermark_boost_factor`) that govern the very thresholds that [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) bypasses.
- [`Documentation/mm/physical_memory.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/physical_memory.rst): the per-zone watermark model (min/low/high) that the bypass overrides.

## OTHER SOURCES

## DETAILS

### Flag bit and surrounding mask

[`mm/internal.h`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) lays out the watermark-related encoding in a compact block.

```c
/* The ALLOC_WMARK bits are used as an index to zone->watermark */
#define ALLOC_WMARK_MIN		WMARK_MIN
#define ALLOC_WMARK_LOW		WMARK_LOW
#define ALLOC_WMARK_HIGH	WMARK_HIGH
#define ALLOC_NO_WATERMARKS	0x04 /* don't check watermarks at all */

/* Mask to get the watermark bits */
#define ALLOC_WMARK_MASK	(ALLOC_NO_WATERMARKS-1)
```

The arithmetic identity `ALLOC_WMARK_MASK = ALLOC_NO_WATERMARKS - 1` ties bit 2 to the upper boundary of the watermark-index field. As long as the field is the low 2 bits, the bypass bit sits exactly above it. [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3915) re-validates this invariant at compile time.

```c
/* Checked here to keep the fast path fast */
BUILD_BUG_ON(ALLOC_NO_WATERMARKS < NR_WMARK);
if (alloc_flags & ALLOC_NO_WATERMARKS)
	goto try_this_zone;
```

[`enum zone_watermarks`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L708) currently contains four members.

```c
enum zone_watermarks {
	WMARK_MIN,
	WMARK_LOW,
	WMARK_HIGH,
	WMARK_PROMO,
	NR_WMARK
};
```

[`NR_WMARK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L713) is therefore 4. The build assertion fires only if a fifth watermark is ever appended without first widening the [`ALLOC_WMARK_MASK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1265) field and shifting the rest of the [`ALLOC_*`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) bits up. The mask is read by [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3897) to look up the per-zone threshold.

```c
check_alloc_wmark:
	mark = wmark_pages(zone, alloc_flags & ALLOC_WMARK_MASK);
	if (!zone_watermark_fast(zone, order, mark,
			       ac->highest_zoneidx, alloc_flags,
			       gfp_mask)) {
```

[`wmark_pages()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1077) is a thin accessor over [`zone->_watermark[]`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L883).

```c
static inline unsigned long wmark_pages(const struct zone *z,
					enum zone_watermarks w)
{
	return z->_watermark[w] + z->watermark_boost;
}
```

### The producer: __gfp_pfmemalloc_flags

Every non-OOM path that produces [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) flows through this single function in [`mm/page_alloc.c`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549).

```c
/*
 * Distinguish requests which really need access to full memory
 * reserves from oom victims which can live with a portion of it
 */

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

The cases run from strongest to weakest. [`__GFP_NOMEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) overrides everything below it, letting a caller explicitly opt out of reserves even when running with [`PF_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) set (used by the OOM victim path itself, by some slab callbacks, and inside [`__alloc_pages_may_oom()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4053) before the high-watermark probe). [`__GFP_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) is the explicit grant, used by SK_MEMALLOC sockets, the swap subsystem, and similar reclaim-driving paths. Softirq with [`PF_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) on the interrupted task is granted because the check looks at [`current`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/current.h), and softirq inherits the per-task flag from the interrupted process. Process context with [`PF_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) is the weakest, granted by [`memalloc_noreclaim_save()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched/mm.h) and friends.

The OOM-victim branch returns [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) (= 0x08) instead. That value is part of [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) and is honoured by [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3627) (cut the watermark by an additional 50%, on top of the 50% from [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) when bundled), but it does NOT bypass the watermark check.

[`gfp_pfmemalloc_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4567) is the thin boolean shim used outside the slowpath proper.

```c
bool gfp_pfmemalloc_allowed(gfp_t gfp_mask)
{
	return !!__gfp_pfmemalloc_flags(gfp_mask);
}
```

### Slowpath wiring on the retry label

[`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4693) does NOT set [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) on the first attempt. The first pass uses [`gfp_to_alloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4479), which derives only the conservative watermark-bending bits ([`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282), [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278), [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292), [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294), etc.). Only after the first [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3791) attempt fails, after the optional direct compaction, and after re-checking cpuset and zonelist generations does the slowpath upgrade to the bypass.

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

The merge is non-additive. `alloc_flags` is overwritten by [`gfp_to_alloc_flags_cma(gfp_mask, reserve_flags)`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) (which only adds [`ALLOC_CMA`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1286) on top of `reserve_flags`), with [`ALLOC_KSWAPD`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1294) re-OR'd from the previous value so that kswapd waking is preserved across the upgrade. After this rewrite, `alloc_flags` is `ALLOC_NO_WATERMARKS [| ALLOC_CMA] [| ALLOC_KSWAPD]` (or `ALLOC_OOM | ...` for OOM victims). The watermark index in the low 2 bits is now zero, but [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3791) will short-circuit before that index is used.

The same block also resets the cpuset/nodemask iterator when reserve access has been granted.

```c
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

The `|| reserve_flags` clause lets a reserve-eligible allocation ignore the caller's [`mempolicy`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mempolicy.h) and consult every zone in the zonelist, on the grounds that emergency progress matters more than NUMA placement.

### The freelist-side bypass

[`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3791) does the watermark probe in two steps. First it tries [`high_wmark_pages(zone)`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) so it can flag the zone as below-high (for PCP draining heuristics). Then it falls back to the requested mark from `alloc_flags & ALLOC_WMARK_MASK`. When the fallback fails, the bypass triggers.

```c
check_alloc_wmark:
	mark = wmark_pages(zone, alloc_flags & ALLOC_WMARK_MASK);
	if (!zone_watermark_fast(zone, order, mark,
			       ac->highest_zoneidx, alloc_flags,
			       gfp_mask)) {
		int ret;

		if (cond_accept_memory(zone, order, alloc_flags))
			goto try_this_zone;

		/*
		 * Watermark failed for this zone, but see if we can
		 * grow this zone if it contains deferred pages.
		 */
		if (deferred_pages_enabled()) {
			if (_deferred_grow_zone(zone, order))
				goto try_this_zone;
		}
		/* Checked here to keep the fast path fast */
		BUILD_BUG_ON(ALLOC_NO_WATERMARKS < NR_WMARK);
		if (alloc_flags & ALLOC_NO_WATERMARKS)
			goto try_this_zone;
```

The order of the four bypass options matters. Deferred-page growth and unaccepted-memory acceptance both expand the pool of free pages without violating the watermark guarantee, so they are tried first. Only when no expansion is possible does the loop fall back to the unconditional bypass.

The destination label, [`try_this_zone`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c), calls [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) directly.

```c
try_this_zone:
		page = rmqueue(zonelist_zone(ac->preferred_zoneref), zone, order,
				gfp_mask, alloc_flags, ac->migratetype);
		if (page) {
			prep_new_page(page, order, gfp_mask, alloc_flags);
```

Whatever [`rmqueue()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c) returns is promoted directly to a freshly-allocated page. The bypass therefore does not stop the allocator from inspecting the per-CPU pageset or the buddy lists; it stops only the watermark gate.

[`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585) itself never inspects [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262).

```c
bool __zone_watermark_ok(struct zone *z, unsigned int order, unsigned long mark,
			 int highest_zoneidx, unsigned int alloc_flags,
			 long free_pages)
{
	long min = mark;
	int o;

	/* free_pages may go negative - that's OK */
	free_pages -= __zone_watermark_unusable_free(z, order, alloc_flags);

	if (unlikely(alloc_flags & ALLOC_RESERVES)) {
```

The watermark-OK function only adjusts `min` for the [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) bundle ([`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282), [`ALLOC_NON_BLOCK`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1278), [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273), [`ALLOC_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1292)). The bypass flag is never tested here because the caller jumps over the function entirely on the bypass path.

### Tagging the page: prep_new_page and pfmemalloc

[`prep_new_page()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1889) propagates the bypass into a per-page bit.

```c
static void prep_new_page(struct page *page, unsigned int order, gfp_t gfp_flags,
							unsigned int alloc_flags)
{
	post_alloc_hook(page, order, gfp_flags);

	if (order && (gfp_flags & __GFP_COMP))
		prep_compound_page(page, order);

	/*
	 * page is set pfmemalloc when ALLOC_NO_WATERMARKS was necessary to
	 * allocate the page. The expectation is that the caller is taking
	 * steps that will free more memory. The caller should avoid the page
	 * being used for !PFMEMALLOC purposes.
	 */
	if (alloc_flags & ALLOC_NO_WATERMARKS)
		set_page_pfmemalloc(page);
	else
		clear_page_pfmemalloc(page);
}
```

The marker piggy-backs on [`page->lru.next`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) (which is unused while the page is owned by the allocator's caller). [`set_page_pfmemalloc()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2564) and [`page_is_pfmemalloc()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2535) live in [`include/linux/mm.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h).

```c
static inline bool page_is_pfmemalloc(const struct page *page)
{
	/*
	 * lru.next has bit 1 set if the page is allocated from the
	 * pfmemalloc reserves.  Callers may simply overwrite it if
	 * they do not need to preserve that information.
	 */
	return (uintptr_t)page->lru.next & BIT(1);
}

static inline void set_page_pfmemalloc(struct page *page)
{
	page->lru.next = (void *)BIT(1);
}

static inline void clear_page_pfmemalloc(struct page *page)
{
	page->lru.next = NULL;
}
```

Downstream consumers (the network stack's [`SK_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/net/sock.h) gating, the page pool's pfmemalloc-skip in [`__page_pool_put_page()`](https://elixir.bootlin.com/linux/v6.19/source/net/core/page_pool.c)) read this bit to refuse using a reserve page for non-emergency traffic. That contract is what makes [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) safe. The allocator hands out the page, and the caller's stack agrees to spend it only on a path that itself frees memory.

### The OOM-victim NOFAIL helper

[`__alloc_pages_may_oom()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4053) is the only call site that passes [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) directly into a freelist scan. After [`out_of_memory()`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c) has selected and signalled a victim, a [`__GFP_NOFAIL`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) caller is given one shot at the deepest reserves through [`__alloc_pages_cpuset_fallback()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c).

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

The early high-watermark probe at the top of the same function uses [`ALLOC_WMARK_HIGH | ALLOC_CPUSET`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4087) instead, with [`__GFP_DIRECT_RECLAIM`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp_types.h) masked off. That probe deliberately keeps the watermark high; only after [`out_of_memory()`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c) returns does the function authorise the bypass.

Note the [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4968) NOFAIL fallback at the very end of that function does NOT use [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262).

```c
	if (unlikely(nofail)) {
		/* ... */
		/*
		 * Help non-failing allocations by giving some access to memory
		 * reserves normally used for high priority non-blocking
		 * allocations but do not use ALLOC_NO_WATERMARKS because this
		 * could deplete whole memory reserves which would just make
		 * the situation worse.
		 */
		page = __alloc_pages_cpuset_fallback(gfp_mask, order, ALLOC_MIN_RESERVE, ac);
```

The two NOFAIL escape hatches are deliberately asymmetric. The OOM-side escape is guarded by [`oom_lock`](https://elixir.bootlin.com/linux/v6.19/source/mm/oom_kill.c) and is reached only after a victim has been killed (which guarantees forward progress soon), so the bypass is tolerable. The slowpath-side escape is reached on every retry and does not have a forward-progress guarantee, so it stays at [`ALLOC_MIN_RESERVE`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1282) (the 50% min cut) instead.

### Direct compaction skip

[`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4787) checks [`gfp_pfmemalloc_allowed()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4567) before its first direct-compaction attempt.

```c
	/*
	 * For costly allocations, try direct compaction first, as it's likely
	 * that we have enough base pages and don't need to reclaim. For non-
	 * movable high-order allocations, do that as well, as compaction will
	 * try prevent permanent fragmentation by migrating from blocks of the
	 * same migratetype.
	 * Don't try this for allocations that are allowed to ignore
	 * watermarks, as the ALLOC_NO_WATERMARKS attempt didn't yet happen.
	 */
	if (can_direct_reclaim && can_compact &&
			(costly_order ||
			   (order > 0 && ac->migratetype != MIGRATE_MOVABLE))
			&& !gfp_pfmemalloc_allowed(gfp_mask)) {
```

Compaction is expensive and is intended to fix fragmentation, not to satisfy emergency allocations. A reserve-eligible caller is told to skip it because the next iteration will set [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) anyway and short-circuit the freelist scan.

### Interaction with ALLOC_OOM and ALLOC_RESERVES

[`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) and [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) are the two values [`__gfp_pfmemalloc_flags()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4549) can return. They are mutually exclusive; a single call returns one or the other (or zero), and the [`__alloc_pages_slowpath()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L4843) merge writes the result over `alloc_flags` rather than OR-ing it. [`ALLOC_OOM`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1273) belongs to [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297) (`NON_BLOCK | MIN_RESERVE | HIGHATOMIC | OOM`), so [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3627) cuts the min watermark by 50% when it is set and additionally lets the high-order check accept [`MIGRATE_HIGHATOMIC`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h) free areas. [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) sits outside [`ALLOC_RESERVES`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1297), [`__zone_watermark_ok()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3585) does not look at it, and the bypass happens in [`get_page_from_freelist()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L3916) one frame above.

An OOM victim is permitted to nibble at the reserve, while a [`PF_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/sched.h) reclaim driver is permitted to drain it. The driver path frees more memory as a side effect, while the victim path is on its way out and does not.

### Reading the bypass back: page_is_pfmemalloc

The pfmemalloc marker is the only durable trace of [`ALLOC_NO_WATERMARKS`](https://elixir.bootlin.com/linux/v6.19/source/mm/internal.h#L1262) once the page leaves the allocator. The socket layer's [`sk_memalloc_socks()`](https://elixir.bootlin.com/linux/v6.19/source/include/net/sock.h) gating drives the first reader path, where receive-side processing checks [`page_is_pfmemalloc()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2535) on incoming skb pages and drops or steers traffic so that pfmemalloc pages are spent only on connections marked [`SK_MEMALLOC`](https://elixir.bootlin.com/linux/v6.19/source/include/net/sock.h) (typically swap-over-NFS or similar memory-freeing flows). The page-pool driver framework drives the second reader path, where [`__page_pool_put_page()`](https://elixir.bootlin.com/linux/v6.19/source/net/core/page_pool.c) refuses to recycle a page whose pfmemalloc bit is set because the page came out of reserves and recycling it would extend the reserve consumption beyond the originally-justified emergency.

Because the marker is encoded in [`page->lru.next`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h), any caller that puts the page on an LRU or another list is expected to clear it (as [`prep_new_page()`](https://elixir.bootlin.com/linux/v6.19/source/mm/page_alloc.c#L1905) clears it for non-bypass pages). According to the encoding comment in [`include/linux/mm.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2538), "Callers may simply overwrite it if they do not need to preserve that information."
