---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# struct mem_section

[`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) is the per-section descriptor that the sparsemem memory model uses to track physical memory. Each section covers a fixed-size region of the physical address space (128 MB on x86-64). The [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1917) field serves a dual purpose: during early boot it encodes the NUMA node ID for allocation guidance, and after initialization it stores a biased pointer to the section's [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array combined with status flags in its low bits.

```
    section_mem_map bit layout (after initialization)
    ──────────────────────────────────────────────────

    63                                    N   ...  3   2   1   0
    ┌────────────────────────────────────┬─────┬───┬───┬───┬───┐
    │  encoded struct page * pointer     │ NID │...│ E │ O │ M │ P │
    └────────────────────────────────────┴─────┴───┴───┴───┴───┘
                                                    │   │   │   │
                       SECTION_IS_EARLY ────────────┘   │   │   │
                       SECTION_IS_ONLINE ───────────────┘   │   │
                       SECTION_HAS_MEM_MAP ─────────────────┘   │
                       SECTION_MARKED_PRESENT ──────────────────┘

    Pointer = section_mem_map & SECTION_MAP_MASK
            = mem_map - section_nr_to_pfn(pnum)
    (biased so that pointer + pfn yields the correct struct page)

    section_mem_map bit layout (during early boot, before mem_map)
    ──────────────────────────────────────────────────────────────

    63                                    N   ...        1   0
    ┌────────────────────────────────────┬─────┬────┬───┬───┐
    │            0 (unused)              │ NID │ 0  │ O │ P │
    └────────────────────────────────────┴─────┴────┴───┴───┘
                                                        │   │
                       SECTION_IS_ONLINE ───────────────┘   │
                       SECTION_MARKED_PRESENT ──────────────┘
```

## SUMMARY

```
    PFN bit layout on x86-64 (40-bit PFN, 4 KB pages, 5-level paging)
    ────────────────────────────────────────────────────────────────

    bit:    39 ............ 23  22 ........ 15  14 ..... 9  8 ........ 0
            ┌──────────────────┬────────────────┬───────────┬────────────┐
    PFN:    │   root index     │ section in root│ subsection│  page in   │
            │    (17 bits)     │   (8 bits)     │  (6 bits) │ subsection │
            │                  │                │           │  (9 bits)  │
            └────────┬─────────┴───────┬────────┴─────┬─────┴────────────┘
                     │                 │              │
                     │                 │              │  subsection_map_index(pfn)
                     │                 │              │  = (pfn & ~PAGE_SECTION_MASK)
                     │                 │              │    / PAGES_PER_SUBSECTION
                     │                 │              │
                     │                 │              ▼
                     │                 │         struct mem_section_usage
                     │                 │         ┌─────────────────────────┐
                     │                 │         │ subsection_map[]        │
                     │                 │         │ ┌──┬──┬──┬─────┬──┐     │
                     │                 │         │ │ 0│ 1│ 2│ ... │63│ bit │
                     │                 │         │ └──┴──┴──┴─────┴──┘     │
                     │                 │         └─────────────────────────┘
                     │                 │
                     │                 │   pfn_to_section_nr(pfn)
                     │                 │   = pfn >> PFN_SECTION_SHIFT
                     │                 │     (right-shift by 15)
                     │                 │
                     │                 │   sec_nr & SECTION_ROOT_MASK
                     │                 │   (low log2(SECTIONS_PER_ROOT) bits)
                     │                 ▼
                     │             struct mem_section *root[SECTIONS_PER_ROOT]
                     │             ┌────┬────┬────┬────┬─────┬────┐
                     │             │ s0 │ s1 │ s2 │ s3 │ ... │sN-1│
                     │             └────┴────┴────┴────┴─────┴────┘
                     │                ▲
                     │                │
                     │   SECTION_NR_TO_ROOT(sec_nr) = sec_nr / SECTIONS_PER_ROOT
                     ▼
              struct mem_section **mem_section
              ┌────────┐
              │ root 0 │──┐
              │ root 1 │  │  each entry points to a page-sized block
              │ root 2 │  │  of struct mem_section (SECTIONS_PER_ROOT
              │  ...   │  │   entries, allocated on demand)
              │ root R │  │
              └────────┘  ▼

    Constants on x86-64 (5-level paging):
      PAGE_SHIFT                = 12   (4 KB pages)
      PFN_SUBSECTION_SHIFT      = 9    (512 pages per 2 MB subsection)
      PFN_SECTION_SHIFT         = 15   (32768 pages per 128 MB section)
      SUBSECTIONS_PER_SECTION   = 64   ((1 << 27) / (1 << 21))
      SECTIONS_PER_ROOT         = 256  (PAGE_SIZE / sizeof(struct mem_section)
                                        = 4096 / 16, w/o CONFIG_PAGE_EXTENSION)
      MAX_PHYSMEM_BITS          = 52   (=> 40-bit PFN, 25-bit section number)
```

[`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) is the fundamental building block of the sparsemem physical memory model ([`CONFIG_SPARSEMEM`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L382)). It connects a section number (derived from a PFN) to an array of [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) descriptors and per-section metadata. The struct has three primary fields:

- [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1917): an `unsigned long` that packs an encoded [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array pointer with status flags in its low bits. During early boot, it temporarily stores the NUMA node ID to guide allocation. After initialization, the pointer is biased by subtracting the section's base PFN so that `section_mem_map + pfn` yields the correct [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) without additional arithmetic.
- [`usage`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1919): a pointer to [`struct mem_section_usage`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1891) that holds the subsection validity bitmap and pageblock flags for the buddy allocator.
- [`page_ext`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1925) (conditional on `CONFIG_PAGE_EXTENSION`): a pointer to extended per-page metadata.

[`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) objects are organized in a two-level array called [`mem_section`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L27). With [`CONFIG_SPARSEMEM_EXTREME`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L408) (the default on 64-bit), the top-level is a dynamically allocated pointer array; each root entry points to a page-sized block of [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) objects. Without it, the array is a statically allocated flat table.

## SPECIFICATIONS

(none; [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) is a Linux kernel internal data structure)

## LINUX KERNEL

- [`'\<struct mem_section\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904): per-section descriptor containing [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1917), [`usage`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1919) pointer, and optional [`page_ext`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1925)
- [`'\<struct mem_section_usage\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1891): per-section usage metadata with [`subsection_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1894) bitmap and [`pageblock_flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1897)
- [`'\<__nr_to_section\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1955): converts a section number to a [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) pointer via the two-level array
- [`'\<__pfn_to_section\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2099): converts a PFN to its [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) via [`pfn_to_section_nr()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1863) and [`__nr_to_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1955)
- [`'\<pfn_to_section_nr\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1863): right-shifts PFN by [`PFN_SECTION_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1849) to get the section number
- [`'\<section_nr_to_pfn\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1867): left-shifts section number by [`PFN_SECTION_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1849) to get the base PFN
- [`'\<__section_mem_map_addr\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2014): extracts the [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array pointer from [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1917) by masking off flag bits
- [`'\<sparse_encode_mem_map\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L268): encodes a mem_map pointer by subtracting the section base PFN so that `encoded + pfn` yields the correct [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h)
- [`'\<sparse_decode_mem_map\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L281): decodes an encoded [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1917) back to the actual [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array base
- [`'\<sparse_encode_early_nid\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L118): encodes a NUMA node ID into the high bits of [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1917) during early boot
- [`'\<sparse_early_nid\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L123): extracts the NUMA node ID from [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1917) during early boot
- [`'\<sparse_init_one_section\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L289): writes the final encoded mem_map pointer and flags into a section
- [`'\<valid_section\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2031): tests [`SECTION_HAS_MEM_MAP`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2002) to check whether a section has a memory map
- [`'\<present_section\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2021): tests [`SECTION_MARKED_PRESENT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2001) to check whether a section has been registered
- [`'\<early_section\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2036): tests [`SECTION_IS_EARLY`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2004) to distinguish boot-time sections from hotplugged ones
- [`'\<online_section\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2046): tests [`SECTION_IS_ONLINE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2003) to check whether a section is available for allocation
- [`'\<pfn_valid\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2167): sparsemem PFN validity check using section lookup, [`valid_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2031), and [`pfn_section_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2112)
- [`'\<pfn_section_valid\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2112): subsection-granularity validity check using the [`subsection_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1894) bitmap
- [`'\<subsection_map_index\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2106): computes the subsection bitmap index for a PFN within its section
- [`'\<section_to_usemap\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1950): returns the [`pageblock_flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1897) array from a section's [`usage`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1919)
- [`'\<sparse_index_init\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L82): allocates a root array entry for [`CONFIG_SPARSEMEM_EXTREME`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L408)
- [`'\<sparse_index_alloc\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L63): allocates a page-sized block of [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) objects for one root entry
- [`'\<mem_section_usage_size\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L304): computes the allocation size for a [`struct mem_section_usage`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1891) including the trailing pageblock flags
- [`'\<set_page_section\>':'include/linux/mm.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2017): encodes the section number into [`page->flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L1751) (classic sparse only)
- [`'\<memdesc_section\>':'include/linux/mm.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2023): extracts the section number from page flags (classic sparse only)

## KERNEL DOCUMENTATION

- [`Documentation/mm/memory-model.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/memory-model.rst): physical memory model overview covering FLATMEM and SPARSEMEM, describes [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) and the two PFN conversion strategies
- [`Documentation/mm/physical_memory.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/physical_memory.rst): physical memory organization including zones, nodes, and how sparsemem sections relate to them

## OTHER SOURCES

## DETAILS

### Struct definition

[`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) is defined in [`include/linux/mmzone.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h):

```c
struct mem_section {
	/*
	 * This is, logically, a pointer to an array of struct
	 * pages.  However, it is stored with some other magic.
	 * (see sparse.c::sparse_init_one_section())
	 *
	 * Additionally during early boot we encode node id of
	 * the location of the section here to guide allocation.
	 * (see sparse.c::memory_present())
	 *
	 * Making it a UL at least makes someone do a cast
	 * before using it wrong.
	 */
	unsigned long section_mem_map;

	struct mem_section_usage *usage;
#ifdef CONFIG_PAGE_EXTENSION
	/*
	 * If SPARSEMEM, pgdat doesn't have page_ext pointer. We use
	 * section. (see page_ext.h about this.)
	 */
	struct page_ext *page_ext;
	unsigned long pad;
#endif
	/*
	 * WARNING: mem_section must be a power-of-2 in size for the
	 * calculation and use of SECTION_ROOT_MASK to make sense.
	 */
};
```

The struct must be a power-of-2 in size because [`SECTION_ROOT_MASK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1942) (used for indexing within a root entry) is derived from [`SECTIONS_PER_ROOT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1935) via bitmask arithmetic. [`sparse_init()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L594) enforces this with a compile-time check:

```c
BUILD_BUG_ON(!is_power_of_2(sizeof(struct mem_section)));
```

The `pad` field under `CONFIG_PAGE_EXTENSION` exists to maintain the power-of-2 size constraint when [`page_ext`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1925) is present.

### Section sizing constants

Section size is architecture-defined by [`SECTION_SIZE_BITS`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/sparsemem.h#L28). On x86-64, [`arch/x86/include/asm/sparsemem.h`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/sparsemem.h) sets it to 27 (128 MB sections):

```c
# define SECTION_SIZE_BITS	27 /* matt - 128 is convenient right now */
# define MAX_PHYSMEM_BITS	(pgtable_l5_enabled() ? 52 : 46)
```

Derived constants in [`include/linux/mmzone.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h):

```c
#define PA_SECTION_SHIFT	(SECTION_SIZE_BITS)
#define PFN_SECTION_SHIFT	(SECTION_SIZE_BITS - PAGE_SHIFT)
#define NR_MEM_SECTIONS		(1UL << SECTIONS_SHIFT)
#define PAGES_PER_SECTION       (1UL << PFN_SECTION_SHIFT)
#define PAGE_SECTION_MASK	(~(PAGES_PER_SECTION-1))
```

On x86-64 with 4K pages, [`PFN_SECTION_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1849) is 15 (27 - 12), so each section covers 32768 page frames. [`SECTIONS_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/page-flags-layout.h#L31) is computed in [`include/linux/page-flags-layout.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/page-flags-layout.h) as `MAX_PHYSMEM_BITS - SECTION_SIZE_BITS`. With 5-level paging (52-bit physical), this yields [`SECTIONS_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/page-flags-layout.h#L31) = 25 and [`NR_MEM_SECTIONS`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1851) = 33,554,432.

Subsection granularity (used for ZONE_DEVICE hotplug) is defined as:

```c
#define SUBSECTION_SHIFT 21
#define SUBSECTION_SIZE (1UL << SUBSECTION_SHIFT)
#define PFN_SUBSECTION_SHIFT (SUBSECTION_SHIFT - PAGE_SHIFT)
#define PAGES_PER_SUBSECTION (1UL << PFN_SUBSECTION_SHIFT)
#define SUBSECTIONS_PER_SECTION (1UL << (SECTION_SIZE_BITS - SUBSECTION_SHIFT))
```

On x86-64, each 128 MB section contains `2^(27-21)` = 64 subsections of 2 MB each.

### The two-level mem_section array

[`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) objects are arranged in a two-dimensional array. The layout depends on [`CONFIG_SPARSEMEM_EXTREME`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L408):

```c
#ifdef CONFIG_SPARSEMEM_EXTREME
#define SECTIONS_PER_ROOT       (PAGE_SIZE / sizeof (struct mem_section))
#else
#define SECTIONS_PER_ROOT	1
#endif

#define SECTION_NR_TO_ROOT(sec)	((sec) / SECTIONS_PER_ROOT)
#define NR_SECTION_ROOTS	DIV_ROUND_UP(NR_MEM_SECTIONS, SECTIONS_PER_ROOT)
#define SECTION_ROOT_MASK	(SECTIONS_PER_ROOT - 1)

#ifdef CONFIG_SPARSEMEM_EXTREME
extern struct mem_section **mem_section;
#else
extern struct mem_section mem_section[NR_SECTION_ROOTS][SECTIONS_PER_ROOT];
#endif
```

With [`CONFIG_SPARSEMEM_EXTREME`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L408), the global [`mem_section`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L27) is a pointer-to-pointer. The top-level pointer array is allocated by [`memblocks_present()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L244) during boot:

```c
static void __init memblocks_present(void)
{
	unsigned long start, end;
	int i, nid;

#ifdef CONFIG_SPARSEMEM_EXTREME
	if (unlikely(!mem_section)) {
		unsigned long size, align;

		size = sizeof(struct mem_section *) * NR_SECTION_ROOTS;
		align = 1 << (INTERNODE_CACHE_SHIFT);
		mem_section = memblock_alloc_or_panic(size, align);
	}
#endif

	for_each_mem_pfn_range(i, MAX_NUMNODES, &start, &end, &nid)
		memory_present(nid, start, end);
}
```

Individual root entries are allocated on demand by [`sparse_index_init()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L82), which calls [`sparse_index_alloc()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L63) to allocate a page-sized block of [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) objects:

```c
static int __meminit sparse_index_init(unsigned long section_nr, int nid)
{
	unsigned long root = SECTION_NR_TO_ROOT(section_nr);
	struct mem_section *section;

	if (mem_section[root])
		return 0;

	section = sparse_index_alloc(nid);
	if (!section)
		return -ENOMEM;

	mem_section[root] = section;

	return 0;
}
```

[`__nr_to_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1955) navigates this two-level structure. It divides the section number by [`SECTIONS_PER_ROOT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1935) to get the root index, then uses [`SECTION_ROOT_MASK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1942) to index within the root:

```c
static inline struct mem_section *__nr_to_section(unsigned long nr)
{
	unsigned long root = SECTION_NR_TO_ROOT(nr);

	if (unlikely(root >= NR_SECTION_ROOTS))
		return NULL;

#ifdef CONFIG_SPARSEMEM_EXTREME
	if (!mem_section || !mem_section[root])
		return NULL;
#endif
	return &mem_section[root][nr & SECTION_ROOT_MASK];
}
```

The convenience function [`__pfn_to_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2099) composes [`pfn_to_section_nr()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1863) with [`__nr_to_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1955):

```c
static inline struct mem_section *__pfn_to_section(unsigned long pfn)
{
	return __nr_to_section(pfn_to_section_nr(pfn));
}
```

### section_mem_map flag bits

The low bits of [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1917) store status flags. The available bits come from the alignment guarantee: the encoded pointer is computed as `mem_map - section_nr_to_pfn(pnum)`, and both values are well-aligned, leaving at least 6 low bits free on all architectures (15 on x86-64). The flags are defined as an enum in [`include/linux/mmzone.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1987):

```c
enum {
	SECTION_MARKED_PRESENT_BIT,
	SECTION_HAS_MEM_MAP_BIT,
	SECTION_IS_ONLINE_BIT,
	SECTION_IS_EARLY_BIT,
#ifdef CONFIG_ZONE_DEVICE
	SECTION_TAINT_ZONE_DEVICE_BIT,
#endif
#ifdef CONFIG_SPARSEMEM_VMEMMAP_PREINIT
	SECTION_IS_VMEMMAP_PREINIT_BIT,
#endif
	SECTION_MAP_LAST_BIT,
};

#define SECTION_MARKED_PRESENT		BIT(SECTION_MARKED_PRESENT_BIT)
#define SECTION_HAS_MEM_MAP		BIT(SECTION_HAS_MEM_MAP_BIT)
#define SECTION_IS_ONLINE		BIT(SECTION_IS_ONLINE_BIT)
#define SECTION_IS_EARLY		BIT(SECTION_IS_EARLY_BIT)
#define SECTION_MAP_MASK		(~(BIT(SECTION_MAP_LAST_BIT) - 1))
#define SECTION_NID_SHIFT		SECTION_MAP_LAST_BIT
```

[`SECTION_MARKED_PRESENT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2001) (bit 0): set by [`__section_mark_present()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L164) when a section is registered via [`memory_present()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L217) or [`sparse_add_section()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L933). Tested by [`present_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2021).

[`SECTION_HAS_MEM_MAP`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2002) (bit 1): set by [`sparse_init_one_section()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L289) after the [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array is populated. Tested by [`valid_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2031). Cleared by [`section_deactivate()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L817) during hot-remove.

[`SECTION_IS_ONLINE`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2003) (bit 2): set during [`memory_present()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L217) for boot-time sections. Tested by [`online_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2046).

[`SECTION_IS_EARLY`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2004) (bit 3): set by [`sparse_init_early_section()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L497) to distinguish boot-time sections from hotplugged ones. Tested by [`early_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2036). This distinction matters for [`pfn_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2167): early sections are considered fully valid across their entire span, while hotplugged sections require per-subsection validation via [`pfn_section_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2112).

[`SECTION_MAP_MASK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2011) clears all flag bits to extract the encoded pointer. [`SECTION_NID_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2012) equals [`SECTION_MAP_LAST_BIT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1998), placing the early-boot NID encoding immediately above the flag bits.

### The section_mem_map encoding

The [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1917) field stores an encoded pointer that enables direct PFN-to-page conversion with a single addition. [`sparse_encode_mem_map()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L268) computes this encoding by subtracting the section's base PFN from the mem_map pointer:

```c
static unsigned long sparse_encode_mem_map(struct page *mem_map, unsigned long pnum)
{
	unsigned long coded_mem_map =
		(unsigned long)(mem_map - (section_nr_to_pfn(pnum)));
	BUILD_BUG_ON(SECTION_MAP_LAST_BIT > PFN_SECTION_SHIFT);
	BUG_ON(coded_mem_map & ~SECTION_MAP_MASK);
	return coded_mem_map;
}
```

The `BUILD_BUG_ON` verifies that the flag bits fit within the alignment guaranteed by [`PFN_SECTION_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1849). The `BUG_ON` checks that the computed pointer does not collide with the flag bit region.

The inverse operation is [`sparse_decode_mem_map()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L281):

```c
struct page *sparse_decode_mem_map(unsigned long coded_mem_map, unsigned long pnum)
{
	coded_mem_map &= SECTION_MAP_MASK;
	return ((struct page *)coded_mem_map) + section_nr_to_pfn(pnum);
}
```

To simply extract the encoded (biased) pointer without reversing the bias, [`__section_mem_map_addr()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2014) is used:

```c
static inline struct page *__section_mem_map_addr(struct mem_section *section)
{
	unsigned long map = section->section_mem_map;
	map &= SECTION_MAP_MASK;
	return (struct page *)map;
}
```

This biased pointer is used directly by the classic sparse [`__pfn_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L60) macro in [`include/asm-generic/memory_model.h`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h), where `__section_mem_map_addr(__sec) + __pfn` yields the correct [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) because the bias cancels out the section offset.

### Early boot NID encoding

During early boot, before a section's [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array is allocated, [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1917) temporarily stores the NUMA node ID. [`memory_present()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L217) writes this encoding:

```c
ms->section_mem_map = sparse_encode_early_nid(nid) |
				SECTION_IS_ONLINE;
__section_mark_present(ms, section_nr);
```

[`sparse_encode_early_nid()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L118) shifts the node ID above the flag bits:

```c
static inline unsigned long sparse_encode_early_nid(int nid)
{
	return ((unsigned long)nid << SECTION_NID_SHIFT);
}
```

[`sparse_early_nid()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L123) extracts it back:

```c
static inline int sparse_early_nid(struct mem_section *section)
{
	return (section->section_mem_map >> SECTION_NID_SHIFT);
}
```

This temporary NID encoding is consumed by [`sparse_init()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L594) to group sections by NUMA node before allocating their mem_maps from node-local memory. The encoding is overwritten when [`sparse_init_one_section()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L289) stores the final encoded mem_map pointer.

### struct mem_section_usage

Each section's [`usage`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1919) pointer references a [`struct mem_section_usage`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1891):

```c
struct mem_section_usage {
	struct rcu_head rcu;
#ifdef CONFIG_SPARSEMEM_VMEMMAP
	DECLARE_BITMAP(subsection_map, SUBSECTIONS_PER_SECTION);
#endif
	/* See declaration of similar field in struct zone */
	unsigned long pageblock_flags[0];
};
```

The [`subsection_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1894) bitmap tracks which 2 MB subsections within a section have valid memory. This enables sub-section granularity for ZONE_DEVICE hotplug. During boot, [`subsection_map_init()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L188) populates this bitmap by calling [`subsection_mask_set()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L179) for each memory range within a section.

The trailing [`pageblock_flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1897) zero-length array provides per-pageblock migrate type information for the buddy allocator. Its actual size is computed by [`usemap_size()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L299), and the total allocation size is returned by [`mem_section_usage_size()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L304):

```c
static unsigned long usemap_size(void)
{
	return BITS_TO_LONGS(SECTION_BLOCKFLAGS_BITS) * sizeof(unsigned long);
}

size_t mem_section_usage_size(void)
{
	return sizeof(struct mem_section_usage) + usemap_size();
}
```

The `rcu` field enables safe concurrent access during memory hot-remove: [`section_deactivate()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L817) uses [`kfree_rcu()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/rcupdate.h) to free the usage structure, and [`pfn_section_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2112) uses [`READ_ONCE()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/rwonce.h) to safely read the usage pointer under RCU protection.

### Section state query helpers

A family of inline functions in [`include/linux/mmzone.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h) test the flag bits in [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1917):

```c
static inline int present_section(const struct mem_section *section)
{
	return (section && (section->section_mem_map & SECTION_MARKED_PRESENT));
}

static inline int valid_section(const struct mem_section *section)
{
	return (section && (section->section_mem_map & SECTION_HAS_MEM_MAP));
}

static inline int early_section(const struct mem_section *section)
{
	return (section && (section->section_mem_map & SECTION_IS_EARLY));
}

static inline int online_section(const struct mem_section *section)
{
	return (section && (section->section_mem_map & SECTION_IS_ONLINE));
}
```

[`present_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2021) indicates a section has been registered (physical memory exists at that address range). [`valid_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2031) indicates the [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array has been allocated and wired in. A section can be present but invalid during early boot (between [`memory_present()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L217) and [`sparse_init_one_section()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L289)).

The iteration macro [`for_each_present_section_nr()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2269) walks all present sections using [`next_present_section_nr()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2259), which scans linearly from a starting section number up to [`__highest_present_section_nr`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L163):

```c
#define for_each_present_section_nr(start, section_nr)		\
	for (section_nr = next_present_section_nr(start - 1);	\
	     section_nr != -1;					\
	     section_nr = next_present_section_nr(section_nr))
```

### PFN validity and subsection validation

The sparsemem [`pfn_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2167) uses [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) to determine whether a PFN has a valid [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h):

```c
static inline int pfn_valid(unsigned long pfn)
{
	struct mem_section *ms;
	int ret;

	if (PHYS_PFN(PFN_PHYS(pfn)) != pfn)
		return 0;

	if (pfn_to_section_nr(pfn) >= NR_MEM_SECTIONS)
		return 0;
	ms = __pfn_to_section(pfn);
	rcu_read_lock_sched();
	if (!valid_section(ms)) {
		rcu_read_unlock_sched();
		return 0;
	}
	ret = early_section(ms) || pfn_section_valid(ms, pfn);
	rcu_read_unlock_sched();

	return ret;
}
```

For early (boot-time) sections, the entire section span is considered valid (the [`early_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2036) short-circuit). For hotplugged sections, [`pfn_section_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2112) checks the specific subsection bit:

```c
static inline int pfn_section_valid(struct mem_section *ms, unsigned long pfn)
{
	int idx = subsection_map_index(pfn);
	struct mem_section_usage *usage = READ_ONCE(ms->usage);

	return usage ? test_bit(idx, usage->subsection_map) : 0;
}
```

[`subsection_map_index()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2106) computes the subsection index from a PFN:

```c
static inline int subsection_map_index(unsigned long pfn)
{
	return (pfn & ~(PAGE_SECTION_MASK)) / PAGES_PER_SUBSECTION;
}
```

The RCU read lock in [`pfn_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2167) protects against concurrent hot-remove, where [`section_deactivate()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L817) clears [`SECTION_HAS_MEM_MAP`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2002) and frees the usage structure via RCU.

### Section number in page->flags (classic sparse)

In classic sparse mode ([`CONFIG_SPARSEMEM`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L382) without [`CONFIG_SPARSEMEM_VMEMMAP`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L415)), the section number is encoded in the high bits of [`page->flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L1751). [`SECTIONS_WIDTH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/page-flags-layout.h#L53) equals [`SECTIONS_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/page-flags-layout.h#L31) in this configuration, allocating bits for the section number in the flags layout:

```
Page flags: | SECTION | NODE | ZONE | [LAST_CPUPID] | ... | FLAGS |
```

[`set_page_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2017) writes the section number and [`memdesc_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2023) reads it back. This enables [`__page_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L54) to determine which section a page belongs to without any external lookup. With [`CONFIG_SPARSEMEM_VMEMMAP`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L415), [`SECTIONS_WIDTH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/page-flags-layout.h#L55) is 0 and the section number is not stored in page flags, because vmemmap conversion uses pointer arithmetic against the global [`vmemmap`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/pgtable_64.h#L256) base instead.

### Connection to PFN-to-page conversion

The classic sparse PFN conversion macros in [`include/asm-generic/memory_model.h`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h) rely directly on [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904):

```c
#define __pfn_to_page(pfn)				\
({	unsigned long __pfn = (pfn);			\
	struct mem_section *__sec = __pfn_to_section(__pfn);	\
	__section_mem_map_addr(__sec) + __pfn;		\
})

#define __page_to_pfn(pg)					\
({	const struct page *__pg = (pg);				\
	int __sec = memdesc_section(__pg->flags);		\
	(unsigned long)(__pg - __section_mem_map_addr(__nr_to_section(__sec)));	\
})
```

[`__pfn_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L60) finds the section via [`__pfn_to_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2099), extracts the biased pointer via [`__section_mem_map_addr()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2014), and adds the PFN. The bias from [`sparse_encode_mem_map()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L268) ensures the result is the correct [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h).

With vmemmap ([`CONFIG_SPARSEMEM_VMEMMAP`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L415)), the conversion bypasses [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) entirely:

```c
#define __pfn_to_page(pfn)	(vmemmap + (pfn))
#define __page_to_pfn(page)	(unsigned long)((page) - vmemmap)
```

In this case, [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) still exists and is used for PFN validation, section state tracking, subsection management, and memory hotplug, but is not in the critical PFN conversion path.
