---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# Sparsemem Memory Model

The sparsemem memory model divides physical memory into fixed-size sections, each tracked by a [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904). Only sections that contain present memory receive a [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array, so holes in the physical address space consume no memory map storage.

```
    Physical Address Space (possibly sparse, with holes)

    Section 0        Section 1        Section 2        Section 3
    [  present  ]    [   hole   ]     [  present  ]    [  present  ]
         |                                 |                |
         v                                 v                v
    +-----------+                     +-----------+    +-----------+
    | struct    |                     | struct    |    | struct    |
    | page[]    |                     | page[]    |    | page[]    |
    +-----------+                     +-----------+    +-----------+
         ^                                 ^                ^
         |                                 |                |
    mem_section[0]                    mem_section[2]    mem_section[3]
    .section_mem_map                  .section_mem_map  .section_mem_map

    (Section 1 has no mem_map allocation -- the hole is skipped)

    Classic Sparse:
      pfn_to_page(pfn) = section[pfn >> PFN_SECTION_SHIFT].section_mem_map + pfn

    VMEMMAP:
      pfn_to_page(pfn) = vmemmap + pfn
      (vmemmap is a virtually contiguous array, page tables map only present sections)
```

## SUMMARY

```
    Sparsemem data structure hierarchy
    ──────────────────────────────────

    struct mem_section
    ┌──────────────────────────────────────────────────────────┐
    │  section_mem_map  (unsigned long, encoded)               │
    │  ┌──────────────────────────────────┬───┬───┬───┬───┐    │
    │  │   encoded struct page * pointer  │ E │ O │ M │ P │    │
    │  └──────────────────┬───────────────┴───┴───┴───┴───┘    │
    │                     │                                    │
    │  usage ──┐          │  (biased: pointer + pfn = page)    │
    │          │          │                                    │
    │  page_ext (only with CONFIG_PAGE_EXTENSION)              │
    └──────────┼──────────┼────────────────────────────────────┘
               │          │
               │          ▼
               │     struct page[PAGES_PER_SECTION]
               │     ┌────┬────┬────┬────┬─────┬─────┐
               │     │ p0 │ p1 │ p2 │ p3 │ ... │pN-1 │
               │     └────┴────┴────┴────┴─────┴─────┘
               │     |<── one entry per PFN in the section ──>|
               │
               ▼
    struct mem_section_usage
    ┌──────────────────────────────────────────────────────────┐
    │  rcu                                                     │
    │                                                          │
    │  subsection_map[BITS_TO_LONGS(SUBSECTIONS_PER_SECTION)]  │
    │       bit:   0    1    2    3    4   ...   N-1           │
    │       ┌────┬────┬────┬────┬────┬─────┬────┐              │
    │       │ 1  │ 0  │ 1  │ 1  │ 0  │ ... │ 1  │              │
    │       └─┬──┴─┬──┴─┬──┴─┬──┴─┬──┴─────┴─┬──┘              │
    │         │    │    │    │    │          │                 │
    │  pageblock_flags[0]  (trailing variable-length array)    │
    └─────────┼────┼────┼────┼────┼──────────┼─────────────────┘
              │    │    │    │    │          │
              ▼    ▼    ▼    ▼    ▼          ▼
              ┌────┬────┬────┬────┬────┬────┬─────┐
              │ SS0│ SS1│ SS2│ SS3│ SS4│ ...│SSN-1│  2 MB subsections
              └────┴────┴────┴────┴────┴────┴─────┘  within the section
              |<──── Section (128 MB on x86-64) ────────────>|

    Flag bits in section_mem_map (low bits):
      P = SECTION_MARKED_PRESENT  (bit 0)
      M = SECTION_HAS_MEM_MAP     (bit 1)
      O = SECTION_IS_ONLINE       (bit 2)
      E = SECTION_IS_EARLY        (bit 3)

    On x86-64: each section spans 128 MB and holds 64 subsections of
    2 MB each. The subsection_map bitmap tracks which subsections hold
    valid memory (used by ZONE_DEVICE memory hotplug).
```

Sparsemem is the most versatile memory model in Linux, selected by [`CONFIG_SPARSEMEM`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L382). It partitions the physical address space into sections of size `2^SECTION_SIZE_BITS` bytes (128 MB on x86-64, 512 MB or 128 MB on arm64). Each section is represented by a [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) whose [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1916) field encodes a pointer to an array of [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) descriptors plus status flags.

Two PFN-to-page conversion strategies exist:

- Classic sparse ([`CONFIG_SPARSEMEM`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L382) without [`CONFIG_SPARSEMEM_VMEMMAP`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L415)): the section number is encoded in [`page->flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L1751) and the PFN high bits select a [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904). Conversion requires a two-level lookup.
- Sparse vmemmap ([`CONFIG_SPARSEMEM_VMEMMAP`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L415)): a global [`vmemmap`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/pgtable_64.h#L256) pointer provides a virtually contiguous [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array. Conversion is a single pointer addition (same cost as flatmem), but the virtual-to-physical page table mappings only cover present sections.

Sparsemem supports memory hotplug via [`sparse_add_section()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L933), deferred struct page initialization, and alternative memory maps for persistent memory devices (ZONE_DEVICE).

## SPECIFICATIONS

(none; sparsemem is a Linux kernel internal memory model abstraction)

## LINUX KERNEL

- [`'\<struct mem_section\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904): Per-section descriptor containing [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1916), usage pointer, and optional page extension
- [`'\<struct mem_section_usage\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1891): Per-section usage metadata with subsection bitmap and pageblock flags
- [`'\<sparse_init\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L594): Top-level sparsemem initialization called from architecture setup code
- [`'\<sparse_init_nid\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L532): Initializes all sections belonging to a single NUMA node
- [`'\<memblocks_present\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L244): Marks all memblock regions as present sections
- [`'\<memory_present\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L217): Records a memory range against a node, initializing section entries
- [`'\<sparse_init_one_section\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L289): Writes the encoded mem_map pointer and flags into a section
- [`'\<sparse_init_early_section\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L497): Helper that assigns usage and calls [`sparse_init_one_section()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L289) during boot
- [`'\<sparse_encode_mem_map\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L268): Encodes a mem_map pointer with a section offset so that `section_mem_map + pfn` yields the correct [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h)
- [`'\<sparse_decode_mem_map\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L281): Decodes an encoded [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1916) back to the actual [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array base
- [`'\<sparse_index_init\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L82): Allocates a root array entry for [`CONFIG_SPARSEMEM_EXTREME`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L408)
- [`'\<sparse_buffer_init\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L446): Pre-allocates a contiguous buffer for section mem_maps within a node
- [`'\<__populate_section_memmap\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L417): Allocates the [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array for a section (non-vmemmap path)
- [`'\<__populate_section_memmap\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L561): Allocates the [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array for a section (vmemmap path, calls [`vmemmap_populate()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1562))
- [`'\<__pfn_to_page\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L60): Classic sparse PFN-to-page conversion via section lookup
- [`'\<__page_to_pfn\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L54): Classic sparse page-to-PFN conversion via section number in page flags
- [`'\<__pfn_to_section\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2099): Converts a PFN to its [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) via [`pfn_to_section_nr()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1863) and [`__nr_to_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1955)
- [`'\<__nr_to_section\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1955): Converts a section number to its [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) pointer via the two-level [`mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1605) array
- [`'\<__section_mem_map_addr\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2014): Extracts the [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array pointer from [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1916) by masking off flag bits
- [`'\<pfn_to_section_nr\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1863): Right-shifts PFN by [`PFN_SECTION_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1849) to get the section number
- [`'\<section_nr_to_pfn\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1867): Left-shifts section number by [`PFN_SECTION_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1849) to get the base PFN
- [`'\<valid_section\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2031): Checks whether a section has a memory map ([`SECTION_HAS_MEM_MAP`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2002) flag)
- [`'\<pfn_valid\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2167): Sparsemem PFN validity check using section lookup and subsection validation
- [`'\<set_page_section\>':'include/linux/mm.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2017): Encodes the section number into [`page->flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L1751) (classic sparse only)
- [`'\<memdesc_section\>':'include/linux/mm.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2023): Extracts the section number from page flags (classic sparse only)
- [`'\<vmemmap_populate\>':'arch/x86/mm/init_64.c'`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1562): x86-64 implementation that creates page table entries mapping vmemmap virtual addresses to physical [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) storage
- [`'\<vmemmap_populate_basepages\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L299): Generic vmemmap population using base pages (no huge page optimization)
- [`'\<sparse_add_section\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L933): Adds or populates a section for memory hotplug

## KERNEL DOCUMENTATION

- [`Documentation/mm/memory-model.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/memory-model.rst): Physical memory model overview covering flatmem and sparsemem

## OTHER SOURCES

## DETAILS

### Struct hierarchy

The diagram below shows how a [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) connects to its [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array via [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1917) and to its [`struct mem_section_usage`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1891) via [`usage`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1919). The low bits of [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1917) hold status flags (the encoded pointer is biased so that `pointer + pfn` directly yields the correct [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h)); the [`subsection_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1894) bitmap inside [`struct mem_section_usage`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1891) records which 2 MB subsections of the section contain valid memory.

```
    Sparsemem data structure hierarchy
    ──────────────────────────────────

    struct mem_section
    ┌──────────────────────────────────────────────────────────┐
    │  section_mem_map  (unsigned long, encoded)               │
    │  ┌──────────────────────────────────┬───┬───┬───┬───┐    │
    │  │   encoded struct page * pointer  │ E │ O │ M │ P │    │
    │  └──────────────────┬───────────────┴───┴───┴───┴───┘    │
    │                     │                                    │
    │  usage ──┐          │  (biased: pointer + pfn = page)    │
    │          │          │                                    │
    │  page_ext (only with CONFIG_PAGE_EXTENSION)              │
    └──────────┼──────────┼────────────────────────────────────┘
               │          │
               │          ▼
               │     struct page[PAGES_PER_SECTION]
               │     ┌────┬────┬────┬────┬─────┬─────┐
               │     │ p0 │ p1 │ p2 │ p3 │ ... │pN-1 │
               │     └────┴────┴────┴────┴─────┴─────┘
               │     |<── one entry per PFN in the section ──>|
               │
               ▼
    struct mem_section_usage
    ┌──────────────────────────────────────────────────────────┐
    │  rcu                                                     │
    │                                                          │
    │  subsection_map[BITS_TO_LONGS(SUBSECTIONS_PER_SECTION)]  │
    │       bit:   0    1    2    3    4   ...   N-1           │
    │       ┌────┬────┬────┬────┬────┬─────┬────┐              │
    │       │ 1  │ 0  │ 1  │ 1  │ 0  │ ... │ 1  │              │
    │       └─┬──┴─┬──┴─┬──┴─┬──┴─┬──┴─────┴─┬──┘              │
    │         │    │    │    │    │          │                 │
    │  pageblock_flags[0]  (trailing variable-length array)    │
    └─────────┼────┼────┼────┼────┼──────────┼─────────────────┘
              │    │    │    │    │          │
              ▼    ▼    ▼    ▼    ▼          ▼
              ┌────┬────┬────┬────┬────┬────┬─────┐
              │ SS0│ SS1│ SS2│ SS3│ SS4│ ...│SSN-1│  2 MB subsections
              └────┴────┴────┴────┴────┴────┴─────┘  within the section
              |<──── Section (128 MB on x86-64) ────────────>|

    Flag bits in section_mem_map (low bits):
      P = SECTION_MARKED_PRESENT  (bit 0)
      M = SECTION_HAS_MEM_MAP     (bit 1)
      O = SECTION_IS_ONLINE       (bit 2)
      E = SECTION_IS_EARLY        (bit 3)

    On x86-64: each section spans 128 MB and holds 64 subsections of
    2 MB each. The subsection_map bitmap tracks which subsections hold
    valid memory (used by ZONE_DEVICE memory hotplug).
```

The pointer extraction is performed by [`__section_mem_map_addr()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2014), which masks off the flag bits using [`SECTION_MAP_MASK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2011). The encoding (subtracting [`section_nr_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1867) from the real mem_map base) is set up by [`sparse_encode_mem_map()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L268) and reversed by [`sparse_decode_mem_map()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L281). The [`subsection_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1894) bit corresponding to a PFN is computed by [`subsection_map_index()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2106) and consumed by [`pfn_section_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2112) to validate hotplugged sections at sub-section granularity.

### Section constants and sizing

The section size is architecture-defined by [`SECTION_SIZE_BITS`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/sparsemem.h#L27). On x86-64, [`arch/x86/include/asm/sparsemem.h`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/sparsemem.h) sets it to 27 (128 MB sections). On arm64, [`arch/arm64/include/asm/sparsemem.h`](https://elixir.bootlin.com/linux/v6.19/source/arch/arm64/include/asm/sparsemem.h) uses 27 for 16K/64K pages or 29 for 4K pages. Derived constants in [`include/linux/mmzone.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h):

```c
#define PA_SECTION_SHIFT	(SECTION_SIZE_BITS)
#define PFN_SECTION_SHIFT	(SECTION_SIZE_BITS - PAGE_SHIFT)
#define NR_MEM_SECTIONS		(1UL << SECTIONS_SHIFT)
#define PAGES_PER_SECTION       (1UL << PFN_SECTION_SHIFT)
```

[`SECTIONS_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/page-flags-layout.h#L31) is defined in [`include/linux/page-flags-layout.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/page-flags-layout.h) as `MAX_PHYSMEM_BITS - SECTION_SIZE_BITS`. On x86-64 with 5-level paging, [`MAX_PHYSMEM_BITS`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/sparsemem.h#L28) is 52, so [`SECTIONS_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1600) is 25 and [`NR_MEM_SECTIONS`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1851) is 33 million.

### The mem_section array

[`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) objects are arranged in a two-dimensional array called [`mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1605). The organization depends on [`CONFIG_SPARSEMEM_EXTREME`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L408):

```c
#ifdef CONFIG_SPARSEMEM_EXTREME
extern struct mem_section **mem_section;
#else
extern struct mem_section mem_section[NR_SECTION_ROOTS][SECTIONS_PER_ROOT];
#endif
```

With [`CONFIG_SPARSEMEM_EXTREME`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L408) (the default on most 64-bit architectures), [`mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1605) is a pointer-to-pointer that is dynamically allocated. Root entries are allocated on demand by [`sparse_index_init()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L82). Each root holds `PAGE_SIZE / sizeof(struct mem_section)` entries (defined as [`SECTIONS_PER_ROOT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1935)). [`__nr_to_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1955) navigates this two-level structure:

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

### struct mem_section internals

The [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1916) field stores both a pointer and status flags in its low bits:

```c
enum {
	SECTION_HAS_MEM_MAP_BIT,
	SECTION_IS_ONLINE_BIT,
	...
	SECTION_MAP_LAST_BIT,
};

#define SECTION_HAS_MEM_MAP		BIT(SECTION_HAS_MEM_MAP_BIT)
#define SECTION_IS_ONLINE		BIT(SECTION_IS_ONLINE_BIT)
#define SECTION_MAP_MASK		(~(BIT(SECTION_MAP_LAST_BIT) - 1))
```

[`SECTION_MAP_MASK`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2011) clears the low flag bits to extract the pointer. [`__section_mem_map_addr()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2014) applies this mask:

```c
static inline struct page *__section_mem_map_addr(struct mem_section *section)
{
	unsigned long map = section->section_mem_map;
	map &= SECTION_MAP_MASK;
	return (struct page *)map;
}
```

The pointer stored in [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1916) is encoded with a bias so that direct indexing by PFN works. [`sparse_encode_mem_map()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L268) computes this:

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

By subtracting `section_nr_to_pfn(pnum)` (the base PFN of the section), the encoded pointer satisfies `encoded_ptr + pfn == &page[pfn - section_base_pfn]`. This allows [`__pfn_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L60) to simply add the PFN to the encoded pointer without first subtracting the section base.

### Classic sparse PFN conversion

When [`CONFIG_SPARSEMEM`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L382) is set without [`CONFIG_SPARSEMEM_VMEMMAP`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L415), the classic sparse conversion macros in [`include/asm-generic/memory_model.h`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h) are used:

```c
#define __page_to_pfn(pg)					\
({	const struct page *__pg = (pg);				\
	int __sec = memdesc_section(__pg->flags);		\
	(unsigned long)(__pg - __section_mem_map_addr(__nr_to_section(__sec)));	\
})

#define __pfn_to_page(pfn)				\
({	unsigned long __pfn = (pfn);			\
	struct mem_section *__sec = __pfn_to_section(__pfn);	\
	__section_mem_map_addr(__sec) + __pfn;		\
})
```

[`__pfn_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L60) converts a PFN to a [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) in two steps: first it finds the section via [`__pfn_to_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2099) (which right-shifts the PFN by [`PFN_SECTION_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1849) to get the section number, then indexes into the [`mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1605) array), then adds the PFN to the encoded [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1916) pointer.

[`__page_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L54) extracts the section number from [`page->flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L1751) via [`memdesc_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2023), looks up the section, and computes the pointer difference from the section's mem_map base.

### Section number encoding in page->flags

In classic sparse, the section number is stored in the high bits of [`page->flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L1751). The flag word layout from [`include/linux/mmzone.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1143):

```
Page flags: | [SECTION] | [NODE] | ZONE | [LAST_CPUPID] | ... | FLAGS |
```

[`SECTIONS_WIDTH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/page-flags-layout.h#L53) equals [`SECTIONS_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/page-flags-layout.h#L31) for classic sparse, and 0 for vmemmap (since vmemmap does not need the section number in the flags). [`set_page_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2017) writes the section number:

```c
static inline void set_page_section(struct page *page, unsigned long section)
{
	page->flags.f &= ~(SECTIONS_MASK << SECTIONS_PGSHIFT);
	page->flags.f |= (section & SECTIONS_MASK) << SECTIONS_PGSHIFT;
}
```

[`memdesc_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2023) reads it back:

```c
static inline unsigned long memdesc_section(memdesc_flags_t mdf)
{
	return (mdf.f >> SECTIONS_PGSHIFT) & SECTIONS_MASK;
}
```

[`SECTIONS_PGSHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1157) is computed as `(sizeof(unsigned long)*8) - SECTIONS_WIDTH`, placing the section number in the topmost bits.

### Sparse vmemmap PFN conversion

When [`CONFIG_SPARSEMEM_VMEMMAP`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L415) is enabled, the conversion becomes trivial:

```c
/* memmap is virtually contiguous.  */
#define __pfn_to_page(pfn)	(vmemmap + (pfn))
#define __page_to_pfn(page)	(unsigned long)((page) - vmemmap)
```

The [`vmemmap`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/pgtable_64.h#L256) pointer is architecture-defined. On x86-64:

```c
#define VMEMMAP_START		vmemmap_base
#define vmemmap ((struct page *)VMEMMAP_START)
```

On arm64, [`vmemmap`](https://elixir.bootlin.com/linux/v6.19/source/arch/arm64/include/asm/pgtable.h#L31) accounts for the physical memory start offset:

```c
#define vmemmap		((struct page *)VMEMMAP_START - (memstart_addr >> PAGE_SHIFT))
```

The key insight is that [`vmemmap`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/pgtable_64.h#L256) points to a virtual address range large enough to cover all possible PFNs, but only sections containing present memory have their page table entries populated. This is done by [`vmemmap_populate()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1562), which each architecture implements. The x86-64 version uses 2 MB huge pages where possible:

```c
int __meminit vmemmap_populate(unsigned long start, unsigned long end, int node,
		struct vmem_altmap *altmap)
{
	int err;

	VM_BUG_ON(!PAGE_ALIGNED(start));
	VM_BUG_ON(!PAGE_ALIGNED(end));

	if (end - start < PAGES_PER_SECTION * sizeof(struct page))
		err = vmemmap_populate_basepages(start, end, node, NULL);
	else if (boot_cpu_has(X86_FEATURE_PSE))
		err = vmemmap_populate_hugepages(start, end, node, altmap);
	else if (altmap) {
		pr_err_once("%s: no cpu support for altmap allocations\n",
				__func__);
		err = -ENOMEM;
	} else
		err = vmemmap_populate_basepages(start, end, node, NULL);
	if (!err)
		sync_global_pgds(start, end - 1);
	return err;
}
```

### Sparsemem PFN validity

The sparsemem [`pfn_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2167) performs a multi-step validation:

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

It first checks that the PFN round-trips through physical address conversion (catching stale high bits). Then it verifies the section number is within bounds, looks up the section, checks that the section has a memory map ([`valid_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2031) tests [`SECTION_HAS_MEM_MAP`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2002)), and finally verifies subsection-level validity. The RCU read lock protects against concurrent hotplug removal.

### Boot-time initialization: sparse_init

Architecture [`setup_arch()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/kernel/setup.c#L605) code calls [`sparse_init()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L594) to initialize the sparsemem data structures. On x86-64, this happens in [`setup_arch()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/kernel/setup.c#L880); on arm64, in `bootmem_init()`.

```c
void __init sparse_init(void)
{
	unsigned long pnum_end, pnum_begin, map_count = 1;
	int nid_begin;

	BUILD_BUG_ON(!is_power_of_2(sizeof(struct mem_section)));
	memblocks_present();

	pnum_begin = first_present_section_nr();
	nid_begin = sparse_early_nid(__nr_to_section(pnum_begin));

	set_pageblock_order();

	for_each_present_section_nr(pnum_begin + 1, pnum_end) {
		int nid = sparse_early_nid(__nr_to_section(pnum_end));

		if (nid == nid_begin) {
			map_count++;
			continue;
		}
		sparse_init_nid(nid_begin, pnum_begin, pnum_end, map_count);
		nid_begin = nid;
		pnum_begin = pnum_end;
		map_count = 1;
	}
	sparse_init_nid(nid_begin, pnum_begin, pnum_end, map_count);
	vmemmap_populate_print_last();
}
```

[`sparse_init()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L594) first calls [`memblocks_present()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L244) to scan all memblock regions and mark the corresponding sections as present via [`memory_present()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L217). Then it groups consecutive present sections by NUMA node and processes each group with [`sparse_init_nid()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L532). This per-node batching allows contiguous allocation of section mem_maps from node-local memory.

### Per-node initialization: sparse_init_nid

[`sparse_init_nid()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L532) handles all sections belonging to a single NUMA node:

```c
static void __init sparse_init_nid(int nid, unsigned long pnum_begin,
				   unsigned long pnum_end,
				   unsigned long map_count)
{
	unsigned long pnum;
	struct page *map;
	struct mem_section *ms;

	if (sparse_usage_init(nid, map_count)) {
		pr_err("%s: node[%d] usemap allocation failed", __func__, nid);
		goto failed;
	}

	sparse_buffer_init(map_count * section_map_size(), nid);

	sparse_vmemmap_init_nid_early(nid);

	for_each_present_section_nr(pnum_begin, pnum) {
		unsigned long pfn = section_nr_to_pfn(pnum);

		if (pnum >= pnum_end)
			break;

		ms = __nr_to_section(pnum);
		if (!preinited_vmemmap_section(ms)) {
			map = __populate_section_memmap(pfn, PAGES_PER_SECTION,
					nid, NULL, NULL);
			if (!map) {
				pr_err("%s: node[%d] memory map backing failed. Some memory will not be available.",
				       __func__, nid);
				pnum_begin = pnum;
				sparse_usage_fini();
				sparse_buffer_fini();
				goto failed;
			}
			memmap_boot_pages_add(DIV_ROUND_UP(PAGES_PER_SECTION * sizeof(struct page),
							   PAGE_SIZE));
			sparse_init_early_section(nid, map, pnum, 0);
		}
	}
	sparse_usage_fini();
	sparse_buffer_fini();
	return;
failed:
	for_each_present_section_nr(pnum_begin, pnum) {
		if (pnum >= pnum_end)
			break;
		ms = __nr_to_section(pnum);
		if (!preinited_vmemmap_section(ms))
			ms->section_mem_map = 0;
		ms->section_mem_map = 0;
	}
}
```

It first allocates [`struct mem_section_usage`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1891) structures for all sections in the node, then calls [`sparse_buffer_init()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L446) to pre-allocate a contiguous buffer sized for all the section mem_maps. For each present section, it calls [`__populate_section_memmap()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L417) to allocate the [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array (either from the pre-allocated buffer or via [`vmemmap_populate()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1562) on the vmemmap path), then calls [`sparse_init_early_section()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L497) to wire the mem_map into the section via [`sparse_init_one_section()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L289).

### Marking sections as present: memory_present

[`memblocks_present()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L244) iterates over all memblock memory regions and calls [`memory_present()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L217) for each:

```c
static void __init memory_present(int nid, unsigned long start, unsigned long end)
{
	unsigned long pfn;

	start &= PAGE_SECTION_MASK;
	mminit_validate_memmodel_limits(&start, &end);
	for (pfn = start; pfn < end; pfn += PAGES_PER_SECTION) {
		unsigned long section_nr = pfn_to_section_nr(pfn);
		struct mem_section *ms;

		sparse_index_init(section_nr, nid);
		set_section_nid(section_nr, nid);

		ms = __nr_to_section(section_nr);
		if (!ms->section_mem_map) {
			ms->section_mem_map = sparse_encode_early_nid(nid) |
							SECTION_IS_ONLINE;
			__section_mark_present(ms, section_nr);
		}
	}
}
```

For each section-aligned PFN in the range, it calls [`sparse_index_init()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L82) to ensure the root array entry exists (for [`CONFIG_SPARSEMEM_EXTREME`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L408)), records the NUMA node ID, and marks the section as present and online. During this early phase, [`section_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1916) temporarily holds the encoded node ID (so that [`sparse_init_nid()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L532) can later determine which node each section belongs to).

### Wiring a section's mem_map: sparse_init_one_section

[`sparse_init_one_section()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L289) writes the final encoded pointer into a section:

```c
static void __meminit sparse_init_one_section(struct mem_section *ms,
		unsigned long pnum, struct page *mem_map,
		struct mem_section_usage *usage, unsigned long flags)
{
	ms->section_mem_map &= ~SECTION_MAP_MASK;
	ms->section_mem_map |= sparse_encode_mem_map(mem_map, pnum)
		| SECTION_HAS_MEM_MAP | flags;
	ms->usage = usage;
}
```

It clears any existing pointer bits, encodes the new mem_map with [`sparse_encode_mem_map()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L268), ORs in the [`SECTION_HAS_MEM_MAP`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2002) flag and any additional flags (such as `SECTION_IS_EARLY`), and stores the usage pointer.

### Memory hotplug: sparse_add_section

[`sparse_add_section()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L933) extends the memory model at runtime for hotplugged memory:

```c
int __meminit sparse_add_section(int nid, unsigned long start_pfn,
		unsigned long nr_pages, struct vmem_altmap *altmap,
		struct dev_pagemap *pgmap)
{
	unsigned long section_nr = pfn_to_section_nr(start_pfn);
	struct mem_section *ms;
	struct page *memmap;
	int ret;

	ret = sparse_index_init(section_nr, nid);
	if (ret < 0)
		return ret;

	memmap = section_activate(nid, start_pfn, nr_pages, altmap, pgmap);
	if (IS_ERR(memmap))
		return PTR_ERR(memmap);

	page_init_poison(memmap, sizeof(struct page) * nr_pages);

	ms = __nr_to_section(section_nr);
	set_section_nid(section_nr, nid);
	__section_mark_present(ms, section_nr);

	if (section_nr_to_pfn(section_nr) != start_pfn)
		memmap = pfn_to_page(section_nr_to_pfn(section_nr));
	sparse_init_one_section(ms, section_nr, memmap, ms->usage, 0);

	return 0;
}
```

It allocates the [`struct mem_section`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1904) root entry if needed, activates the section (allocating or reusing a mem_map), poisons the new [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) entries for debugging, and wires the section via [`sparse_init_one_section()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L289). With vmemmap, sub-section granularity (2 MB) is supported for ZONE_DEVICE mappings.

### Kconfig selection

Sparsemem is selected through `mm/Kconfig`:

```
config SPARSEMEM_MANUAL
	bool "Sparse Memory"
	depends on ARCH_SPARSEMEM_ENABLE

config SPARSEMEM
	def_bool y
	depends on (!SELECT_MEMORY_MODEL && ARCH_SPARSEMEM_ENABLE) || SPARSEMEM_MANUAL

config SPARSEMEM_VMEMMAP
	bool "Sparse Memory virtual memmap"
	depends on SPARSEMEM && SPARSEMEM_VMEMMAP_ENABLE
```

An architecture enables sparsemem by setting `ARCH_SPARSEMEM_ENABLE` and optionally [`ARCH_SPARSEMEM_DEFAULT`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/Kconfig#L1777). The vmemmap optimization requires `SPARSEMEM_VMEMMAP_ENABLE`. Most 64-bit architectures (x86-64, arm64, riscv64, powerpc64) default to [`CONFIG_SPARSEMEM_VMEMMAP`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L415).
