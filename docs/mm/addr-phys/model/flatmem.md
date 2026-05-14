---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# Flatmem Memory Model

The flatmem memory model maps the entire physical address space to a single contiguous array of [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) descriptors called [`mem_map`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L45). Converting between a page frame number (PFN) and its [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) is a single pointer addition.

```
    Physical Memory (contiguous, starting at ARCH_PFN_OFFSET)

    PFN:   ARCH_PFN_OFFSET          ARCH_PFN_OFFSET + max_mapnr - 1
            |                                            |
            v                                            v
           +----+----+----+----+----+  ...  +----+----+----+
           |page|page|page|page|page|       |page|page|page|
           +----+----+----+----+----+  ...  +----+----+----+
            ^                                            ^
            |                                            |
    mem_map[0]                           mem_map[max_mapnr - 1]

    pfn_to_page(pfn) = mem_map + (pfn - ARCH_PFN_OFFSET)
    page_to_pfn(page) = (page - mem_map) + ARCH_PFN_OFFSET
```

## SUMMARY

Flatmem is the simplest memory model in Linux, suitable for non-NUMA systems with contiguous (or mostly contiguous) physical memory. It maintains a global array [`mem_map`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L45) that provides a one-to-one mapping between physical page frame numbers and [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) descriptors. The array is allocated during boot by [`alloc_node_mem_map()`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1647), called from the [`free_area_init()`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1824) initialization path.

Flatmem is selected when [`CONFIG_FLATMEM`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L386) is enabled. The conversion between PFN and [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) uses arithmetic on the global [`mem_map`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L45) pointer, offset by [`ARCH_PFN_OFFSET`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L15) to handle systems where physical memory does not start at address 0. The variable [`max_mapnr`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L42) records the total number of entries in the array, and [`pfn_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L26) uses it for bounds checking.

Holes in the physical address space still occupy entries in [`mem_map`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L45) (the corresponding [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) objects are never fully initialized). This makes flatmem unsuitable for systems with large sparse address spaces, where sparsemem is preferred.

## SPECIFICATIONS

(none; flatmem is a Linux kernel internal memory model abstraction)

## LINUX KERNEL

- [`'\<mem_map\>':'mm/mm_init.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L45): Global array of [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) descriptors covering the entire physical address space
- [`'\<max_mapnr\>':'mm/mm_init.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L42): Number of entries in the [`mem_map`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L45) array
- [`'\<ARCH_PFN_OFFSET\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L15): PFN of the first physical page frame (0 on most architectures)
- [`'\<__pfn_to_page\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L18): Flatmem macro converting PFN to [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) pointer via array index
- [`'\<__page_to_pfn\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L19): Flatmem macro converting [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) pointer back to PFN
- [`'\<pfn_valid\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L26): Flatmem PFN validity check using [`ARCH_PFN_OFFSET`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L15) and [`max_mapnr`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L42) bounds
- [`'\<for_each_valid_pfn\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L35): Iterator over valid PFNs within a range, clamped to [`ARCH_PFN_OFFSET`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L15) and [`max_mapnr`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L42)
- [`'\<alloc_node_mem_map\>':'mm/mm_init.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1647): Allocates the [`mem_map`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L45) array from memblock for node 0
- [`'\<free_area_init_node\>':'mm/mm_init.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1717): Per-node initialization that calls [`alloc_node_mem_map()`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1647)
- [`'\<free_area_init\>':'mm/mm_init.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1824): Top-level memory initialization called by architecture setup code
- [`'\<struct pglist_data\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1316): Per-node data structure containing [`node_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1398) under [`CONFIG_FLATMEM`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L386)
- [`'\<memmap_alloc\>':'mm/mm_init.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1622): Allocates memory map backing from memblock

## KERNEL DOCUMENTATION

- [`Documentation/mm/memory-model.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/memory-model.rst): Physical memory model overview covering flatmem and sparsemem

## OTHER SOURCES

## DETAILS

### PFN-to-page and page-to-PFN conversion

The flatmem model defines the simplest possible conversion between PFNs and [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) pointers. Both macros live in [`include/asm-generic/memory_model.h`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h) under `#if defined(CONFIG_FLATMEM)`:

```c
#ifndef ARCH_PFN_OFFSET
#define ARCH_PFN_OFFSET		(0UL)
#endif

#define __pfn_to_page(pfn)	(mem_map + ((pfn) - ARCH_PFN_OFFSET))
#define __page_to_pfn(page)	((unsigned long)((page) - mem_map) + \
				 ARCH_PFN_OFFSET)
```

[`__pfn_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L18) subtracts [`ARCH_PFN_OFFSET`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L15) from the PFN to produce a zero-based index into [`mem_map`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L45). The reverse macro [`__page_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L19) computes the pointer difference and adds [`ARCH_PFN_OFFSET`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L15) back. These are aliased to the public API at the bottom of the same file:

```c
#define page_to_pfn __page_to_pfn
#define pfn_to_page __pfn_to_page
```

[`ARCH_PFN_OFFSET`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L15) defaults to 0 but architectures where physical RAM does not start at address 0 override it. For example, ARM defines it as [`PHYS_PFN_OFFSET`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L15), RISC-V as `PFN_DOWN((unsigned long)phys_ram_base)`, and Xtensa as `PHYS_OFFSET >> PAGE_SHIFT`.

### PFN validity checking

The flatmem [`pfn_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L26) is a simple bounds check:

```c
static inline int pfn_valid(unsigned long pfn)
{
	unsigned long pfn_offset = ARCH_PFN_OFFSET;

	return pfn >= pfn_offset && (pfn - pfn_offset) < max_mapnr;
}
```

It verifies that the PFN falls within the range `[ARCH_PFN_OFFSET, ARCH_PFN_OFFSET + max_mapnr)`. Since flatmem covers holes with uninitialized [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) entries, [`pfn_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L26) returning true does not guarantee usable memory at that PFN. Some architectures (such as ARC and ARM) override [`pfn_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L26) with tighter implementations that consult memblock to exclude holes.

The [`for_each_valid_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L35) macro iterates over a PFN range while clamping to the valid bounds:

```c
#define for_each_valid_pfn(pfn, start_pfn, end_pfn)			 \
	for ((pfn) = max_t(unsigned long, (start_pfn), ARCH_PFN_OFFSET); \
	     (pfn) < min_t(unsigned long, (end_pfn),			 \
			   ARCH_PFN_OFFSET + max_mapnr);		 \
	     (pfn)++)
```

### The mem_map global and node_mem_map

The global [`mem_map`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L45) pointer and [`max_mapnr`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L42) are defined in [`mm/mm_init.c`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c) under `#ifndef CONFIG_NUMA`:

```c
#ifndef CONFIG_NUMA
unsigned long max_mapnr;
EXPORT_SYMBOL(max_mapnr);

struct page *mem_map;
EXPORT_SYMBOL(mem_map);
#endif
```

Under [`CONFIG_FLATMEM`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L386), each node's [`struct pglist_data`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1316) also carries a [`node_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1398) pointer:

```c
#ifdef CONFIG_FLATMEM	/* means !SPARSEMEM */
	struct page *node_mem_map;
#endif
```

The global [`mem_map`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L45) is set to point at node 0's [`node_mem_map`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1398) during initialization, making the per-node and global arrays the same object.

### Initialization: alloc_node_mem_map

The [`mem_map`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L45) array is allocated during boot through the call chain: [`free_area_init()`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1824) -> [`free_area_init_node()`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1717) -> [`alloc_node_mem_map()`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1647). Architecture [`setup_arch()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/kernel/setup.c#L605) code calls [`free_area_init()`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1824), which iterates over all online nodes and calls [`free_area_init_node()`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1717) for each.

[`alloc_node_mem_map()`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1647) performs the actual allocation:

```c
static void __init alloc_node_mem_map(struct pglist_data *pgdat)
{
	unsigned long start, offset, size, end;
	struct page *map;

	/* Skip empty nodes */
	if (!pgdat->node_spanned_pages)
		return;

	start = pgdat->node_start_pfn & ~(MAX_ORDER_NR_PAGES - 1);
	offset = pgdat->node_start_pfn - start;
	/*
	 * The zone's endpoints aren't required to be MAX_PAGE_ORDER
	 * aligned but the node_mem_map endpoints must be in order
	 * for the buddy allocator to function correctly.
	 */
	end = ALIGN(pgdat_end_pfn(pgdat), MAX_ORDER_NR_PAGES);
	size =  (end - start) * sizeof(struct page);
	map = memmap_alloc(size, SMP_CACHE_BYTES, MEMBLOCK_LOW_LIMIT,
			   pgdat->node_id, false);
	if (!map)
		panic("Failed to allocate %ld bytes for node %d memory map\n",
		      size, pgdat->node_id);
	pgdat->node_mem_map = map + offset;
	memmap_boot_pages_add(DIV_ROUND_UP(size, PAGE_SIZE));
	pr_debug("%s: node %d, pgdat %08lx, node_mem_map %08lx\n",
		 __func__, pgdat->node_id, (unsigned long)pgdat,
		 (unsigned long)pgdat->node_mem_map);

	/* the global mem_map is just set as node 0's */
	WARN_ON(pgdat != NODE_DATA(0));

	mem_map = pgdat->node_mem_map;
	if (page_to_pfn(mem_map) != pgdat->node_start_pfn)
		mem_map -= offset;

	max_mapnr = end - start;
}
```

The function aligns the start PFN down and the end PFN up to `MAX_ORDER_NR_PAGES` boundaries. This ensures the buddy allocator can pair pages across the full range. It allocates the array via [`memmap_alloc()`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1622), which obtains memory from the memblock allocator during early boot. The function stores the map in `pgdat->node_mem_map` (offset to account for alignment), then sets the global [`mem_map`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L45) to point at the same array. The `WARN_ON(pgdat != NODE_DATA(0))` confirms that flatmem expects only node 0 (it is a non-NUMA model). Finally, [`max_mapnr`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L42) is set to the total span (`end - start`), which [`pfn_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L26) uses for bounds checking.

### Kconfig selection

Flatmem is selected through `mm/Kconfig`:

```
config FLATMEM_MANUAL
	bool "Flat Memory"
	depends on !ARCH_SPARSEMEM_ENABLE || ARCH_FLATMEM_ENABLE

config FLATMEM
	def_bool y
	depends on !SPARSEMEM || FLATMEM_MANUAL
```

An architecture enables flatmem support by setting [`ARCH_FLATMEM_ENABLE`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/Kconfig#L1774). When both flatmem and sparsemem are available, the default depends on [`ARCH_SPARSEMEM_DEFAULT`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/Kconfig#L1777). Flatmem is the fallback when sparsemem is not selected.

### Limitations

Flatmem allocates [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) entries for every PFN in the range, including holes. On systems with large physical address spaces and sparse memory layout, this wastes significant memory. For example, a system with 4 GB of RAM at the bottom and 4 GB at the top of a 64 GB address space would allocate [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) entries for all 64 GB worth of PFNs. Flatmem also lacks support for memory hotplug, deferred struct page initialization, and alternative memory maps for persistent memory devices. These features require the sparsemem model.
