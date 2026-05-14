---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# Page Frame Number (PFN)

A Page Frame Number (PFN) is an integer index that uniquely identifies a physical page frame. The kernel uses PFNs as the fundamental unit for addressing physical memory, converting between PFNs, physical addresses, virtual addresses, and [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) descriptors through a family of macros and inline functions.

```
    Physical Address Space
    +---------------------------------------------------------------+
    | Page 0  | Page 1  | Page 2  |  ...  | Page N-1 | Page N      |
    +---------------------------------------------------------------+
    0x0000    0x1000    0x2000           (N-1)*4096   N*4096

    PFN  = Physical Address >> PAGE_SHIFT     (PHYS_PFN / PFN_DOWN)
    Addr = PFN << PAGE_SHIFT                  (PFN_PHYS)

    Conversion Map:

                    PHYS_PFN / PFN_DOWN
    phys_addr  --------------------------->  PFN
        |       <---------------------------  |
        |            PFN_PHYS                 |
        |                                     |
        |  __va / __pa                        |  pfn_to_page / page_to_pfn
        |                                     |
        v                                     v
    virt_addr  <------------------------  struct page
                    page_to_virt
```

## SUMMARY

The PFN is computed by right-shifting a physical address by [`PAGE_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/page.h#L11) (typically 12 for 4 KB pages). The core conversion macros [`PFN_DOWN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L11), [`PFN_UP()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L10), [`PFN_PHYS()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L12), and [`PHYS_PFN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L13) are defined in [`include/linux/pfn.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h).

The mapping between PFNs and [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) descriptors is provided by [`pfn_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L74) and [`page_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L73), whose implementation depends on the selected memory model (flatmem, classic sparse, or sparse vmemmap). Additional helpers convert between PFNs and virtual addresses ([`pfn_to_kaddr()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/page.h#L72), [`virt_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/xen/page.h#L298)) and between [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) and physical addresses ([`page_to_phys()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L77), [`phys_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L87)).

PFN validity is checked by [`pfn_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2167), which verifies that a memory map entry exists for a given PFN. For code that also requires the page to be online and fully initialized, [`pfn_to_online_page()`](https://elixir.bootlin.com/linux/v6.19/source/mm/memory_hotplug.c#L346) combines validation with section online checks.

The memblock allocator uses PFN-based iterators during early boot: [`for_each_mem_pfn_range()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/memblock.h#L323) walks memory regions in PFN terms, and [`memblock_region_memory_base_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/memblock.h#L527) / [`memblock_region_memory_end_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/memblock.h#L538) extract PFN boundaries from memblock regions using [`PFN_UP()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L10) and [`PFN_DOWN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L11) respectively, guaranteeing conservative rounding.

## SPECIFICATIONS

(none; PFN is a Linux kernel internal abstraction)

## LINUX KERNEL

- [`'\<PFN_DOWN\>':'include/linux/pfn.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L11): Converts a physical address to a PFN by right-shifting by [`PAGE_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/vdso/page.h#L13) (rounds down)
- [`'\<PFN_UP\>':'include/linux/pfn.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L10): Converts a physical address to a PFN, rounding up to the next page boundary
- [`'\<PFN_PHYS\>':'include/linux/pfn.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L12): Converts a PFN to its physical address by left-shifting by [`PAGE_SHIFT`](https://elixir.bootlin.com/linux/v6.19/source/include/vdso/page.h#L13)
- [`'\<PHYS_PFN\>':'include/linux/pfn.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L13): Converts a physical address to a PFN (same as [`PFN_DOWN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L11), cast to `unsigned long`)
- [`'\<PFN_ALIGN\>':'include/linux/pfn.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L9): Rounds a byte address up to the next page boundary
- [`'\<__pfn_to_phys\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L71): Alias for [`PFN_PHYS()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L12), converts PFN to physical address
- [`'\<__phys_to_pfn\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L70): Alias for [`PHYS_PFN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L13), converts physical address to PFN
- [`'\<pfn_to_page\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L74): Converts a PFN to its [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) pointer (memory-model dependent)
- [`'\<page_to_pfn\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L73): Converts a [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) pointer to its PFN (memory-model dependent)
- [`'\<page_to_phys\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L77): Converts a [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) to its physical address via [`page_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L73) and [`PFN_PHYS()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L12)
- [`'\<phys_to_page\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L87): Converts a physical address to its [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) via [`PHYS_PFN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L13) and [`pfn_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L74)
- [`'\<page_to_virt\>':'include/linux/mm.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L118): Converts a [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) to its kernel virtual address via [`page_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L73) and [`__va()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/page.h#L58)
- [`'\<virt_to_page\>':'arch/x86/include/asm/page.h'`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/page.h#L68): Converts a kernel virtual address to its [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) via [`__pa()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/page.h#L41) and [`pfn_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L74) (architecture-specific)
- [`'\<virt_to_pfn\>':'arch/x86/include/asm/xen/page.h'`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/xen/page.h#L298): Converts a kernel virtual address to a PFN via [`__pa()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/page.h#L41) and [`PFN_DOWN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L11) (architecture-specific)
- [`'\<pfn_to_kaddr\>':'arch/x86/include/asm/page.h'`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/page.h#L72): Converts a PFN to its kernel virtual address via [`__va()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/page.h#L58) (architecture-specific)
- [`'\<pfn_valid\>':'include/linux/mmzone.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2167): Checks if a memory map entry exists for a PFN (sparsemem implementation)
- [`'\<pfn_valid\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L26): Checks if a PFN falls within [`mem_map`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L45) bounds (flatmem implementation)
- [`'\<pfn_to_online_page\>':'mm/memory_hotplug.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/memory_hotplug.c#L346): Returns the [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) only if the PFN is valid and its section is online
- [`'\<for_each_mem_pfn_range\>':'include/linux/memblock.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/memblock.h#L323): Iterator over memblock memory regions expressed as PFN ranges
- [`'\<memblock_region_memory_base_pfn\>':'include/linux/memblock.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/memblock.h#L527): Returns the lowest PFN of a memblock region (rounds up with [`PFN_UP()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L10))
- [`'\<memblock_region_memory_end_pfn\>':'include/linux/memblock.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/memblock.h#L538): Returns the end PFN of a memblock region (rounds down with [`PFN_DOWN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L11))

## KERNEL DOCUMENTATION

- [`Documentation/mm/memory-model.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/memory-model.rst): Physical memory model overview describing the PFN-to-page mapping under flatmem and sparsemem

## OTHER SOURCES

## DETAILS

### Core PFN arithmetic macros

All PFN arithmetic macros are defined in [`include/linux/pfn.h`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h):

```c
#define PFN_ALIGN(x)	(((unsigned long)(x) + (PAGE_SIZE - 1)) & PAGE_MASK)
#define PFN_UP(x)	(((x) + PAGE_SIZE-1) >> PAGE_SHIFT)
#define PFN_DOWN(x)	((x) >> PAGE_SHIFT)
#define PFN_PHYS(x)	((phys_addr_t)(x) << PAGE_SHIFT)
#define PHYS_PFN(x)	((unsigned long)((x) >> PAGE_SHIFT))
```

[`PFN_DOWN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L11) truncates a physical address to the containing page frame (rounds down). [`PFN_UP()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L10) rounds up, returning the PFN of the first page frame that starts at or after the given address. [`PHYS_PFN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L13) is equivalent to [`PFN_DOWN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L11) but casts the result to `unsigned long`. [`PFN_PHYS()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L12) performs the reverse, converting a PFN back to the byte address of the page frame start. [`PFN_ALIGN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L9) rounds a byte address up to a page boundary, operating on addresses rather than PFNs.

[`include/asm-generic/memory_model.h`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h) provides shorthand aliases:

```c
#define	__phys_to_pfn(paddr)	PHYS_PFN(paddr)
#define	__pfn_to_phys(pfn)	PFN_PHYS(pfn)
```

### PFN_UP vs PFN_DOWN: conservative rounding in memblock

The distinction between [`PFN_UP()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L10) and [`PFN_DOWN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L11) is critical for the memblock allocator. When describing usable memory, the base address is rounded up (ensuring the region starts at a fully usable page) and the end address is rounded down (ensuring the region does not extend into a partial page). [`memblock_region_memory_base_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/memblock.h#L527) and [`memblock_region_memory_end_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/memblock.h#L538) demonstrate this:

```c
static inline unsigned long memblock_region_memory_base_pfn(const struct memblock_region *reg)
{
	return PFN_UP(reg->base);
}

static inline unsigned long memblock_region_memory_end_pfn(const struct memblock_region *reg)
{
	return PFN_DOWN(reg->base + reg->size);
}
```

The same rounding discipline appears in [`__next_mem_pfn_range()`](https://elixir.bootlin.com/linux/v6.19/source/mm/memblock.c#L1386), the workhorse behind [`for_each_mem_pfn_range()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/memblock.h#L323):

```c
void __init_memblock __next_mem_pfn_range(int *idx, int nid,
				unsigned long *out_start_pfn,
				unsigned long *out_end_pfn, int *out_nid)
{
	struct memblock_type *type = &memblock.memory;
	struct memblock_region *r;
	int r_nid;

	while (++*idx < type->cnt) {
		r = &type->regions[*idx];
		r_nid = memblock_get_region_node(r);

		if (PFN_UP(r->base) >= PFN_DOWN(r->base + r->size))
			continue;
		if (!numa_valid_node(nid) || nid == r_nid)
			break;
	}
	if (*idx >= type->cnt) {
		*idx = -1;
		return;
	}

	if (out_start_pfn)
		*out_start_pfn = PFN_UP(r->base);
	if (out_end_pfn)
		*out_end_pfn = PFN_DOWN(r->base + r->size);
	if (out_nid)
		*out_nid = r_nid;
}
```

The guard `PFN_UP(r->base) >= PFN_DOWN(r->base + r->size)` skips regions too small to contain a complete page frame. The start PFN is rounded up and the end PFN is rounded down, so callers always receive a range of fully usable pages.

### PFN to struct page conversion

[`pfn_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L74) and [`page_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L73) are the primary interface for converting between PFNs and [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) pointers. They are defined in [`include/asm-generic/memory_model.h`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h) as aliases:

```c
#define page_to_pfn __page_to_pfn
#define pfn_to_page __pfn_to_page
```

The actual implementation of [`__pfn_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L18) and [`__page_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L19) depends on the active memory model. Under flatmem, the conversion is a single array index operation on the global [`mem_map`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L45):

```c
#if defined(CONFIG_FLATMEM)

#define __pfn_to_page(pfn)	(mem_map + ((pfn) - ARCH_PFN_OFFSET))
#define __page_to_pfn(page)	((unsigned long)((page) - mem_map) + \
				 ARCH_PFN_OFFSET)
```

Under sparse vmemmap ([`CONFIG_SPARSEMEM_VMEMMAP`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L415)), the conversion is equally simple, using the global [`vmemmap`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/pgtable_64.h#L256) pointer:

```c
#elif defined(CONFIG_SPARSEMEM_VMEMMAP)

#define __pfn_to_page(pfn)	(vmemmap + (pfn))
#define __page_to_pfn(page)	(unsigned long)((page) - vmemmap)
```

Under classic sparse ([`CONFIG_SPARSEMEM`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L382) without vmemmap), the conversion requires a section lookup. [`__pfn_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L60) finds the section via [`__pfn_to_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2099), and [`__page_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L54) extracts the section number from the page flags via [`memdesc_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L2023):

```c
#elif defined(CONFIG_SPARSEMEM)

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

### Physical address and virtual address conversions

The remaining conversions build on [`pfn_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L74) / [`page_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L73) and [`PFN_PHYS()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L12) / [`PHYS_PFN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L13). [`page_to_phys()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L77) chains them together, with an optional debug check:

```c
#ifdef CONFIG_DEBUG_VIRTUAL
#define page_to_phys(page)						\
({									\
	unsigned long __pfn = page_to_pfn(page);			\
									\
	WARN_ON_ONCE(!pfn_valid(__pfn));				\
	PFN_PHYS(__pfn);						\
})
#else
#define page_to_phys(page)	PFN_PHYS(page_to_pfn(page))
#endif
```

[`phys_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L87) performs the reverse:

```c
#define phys_to_page(phys)	pfn_to_page(PHYS_PFN(phys))
```

The generic [`page_to_virt()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L118) converts a [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) to a kernel virtual address by composing [`page_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L73) with [`PFN_PHYS()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L12) and [`__va()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/page.h#L58):

```c
#define page_to_virt(x)	__va(PFN_PHYS(page_to_pfn(x)))
```

Architectures may override this. On arm64, [`page_to_virt()`](https://elixir.bootlin.com/linux/v6.19/source/arch/arm64/include/asm/memory.h#L404) computes the index from vmemmap directly:

```c
#define page_to_virt(x)	({						\
	__typeof__(x) __page = x;					\
	u64 __idx = ((u64)__page - VMEMMAP_START) / sizeof(struct page);\
	u64 __addr = PAGE_OFFSET + (__idx * PAGE_SIZE);			\
	(void *)__tag_set((const void *)__addr, page_kasan_tag(__page));\
})
```

The reverse path, [`virt_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/page.h#L68), is architecture-specific. On x86, it chains [`__pa()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/page.h#L41) and [`pfn_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L74):

```c
#define virt_to_page(kaddr)	pfn_to_page(__pa(kaddr) >> PAGE_SHIFT)
```

[`pfn_to_kaddr()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/page.h#L72) converts a PFN directly to a kernel virtual address, bypassing [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h):

```c
static __always_inline void *pfn_to_kaddr(unsigned long pfn)
{
	return __va(pfn << PAGE_SHIFT);
}
```

[`virt_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/xen/page.h#L298) performs the reverse:

```c
static inline unsigned long virt_to_pfn(const void *v)
{
	return PFN_DOWN(__pa(v));
}
```

### PFN validity checking

[`pfn_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2167) answers whether a memory map entry (a [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h)) exists for a given PFN. It does not guarantee that the memory at that PFN is usable. Under flatmem, it is a simple bounds check against [`ARCH_PFN_OFFSET`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L15) and [`max_mapnr`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L42):

```c
static inline int pfn_valid(unsigned long pfn)
{
	unsigned long pfn_offset = ARCH_PFN_OFFSET;

	return pfn >= pfn_offset && (pfn - pfn_offset) < max_mapnr;
}
```

Under sparsemem, [`pfn_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2167) performs a multi-step section lookup, checking that the PFN falls within [`NR_MEM_SECTIONS`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L1851), the section has a memory map ([`valid_section()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2031)), and the subsection is valid. It takes an RCU read lock to protect against concurrent hotplug.

### pfn_to_online_page: safe page access for walkers

Code that walks PFN ranges and needs fully initialized, online pages should use [`pfn_to_online_page()`](https://elixir.bootlin.com/linux/v6.19/source/mm/memory_hotplug.c#L346) instead of the [`pfn_valid()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mmzone.h#L2167) and [`pfn_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L74) pair:

```c
struct page *pfn_to_online_page(unsigned long pfn)
{
	unsigned long nr = pfn_to_section_nr(pfn);
	struct dev_pagemap *pgmap;
	struct mem_section *ms;

	if (nr >= NR_MEM_SECTIONS)
		return NULL;

	ms = __nr_to_section(nr);
	if (!online_section(ms))
		return NULL;

	if (IS_ENABLED(CONFIG_HAVE_ARCH_PFN_VALID) && !pfn_valid(pfn))
		return NULL;

	if (!pfn_section_valid(ms, pfn))
		return NULL;

	if (!online_device_section(ms))
		return pfn_to_page(pfn);

	pgmap = get_dev_pagemap(pfn);
	put_dev_pagemap(pgmap);

	if (pgmap)
		return NULL;

	return pfn_to_page(pfn);
}
```

It returns NULL for PFNs in offline sections, ZONE_DEVICE regions, and invalid subsections. The kernel's page walkers use it extensively. For example, [`saveable_page()`](https://elixir.bootlin.com/linux/v6.19/source/kernel/power/snapshot.c#L1375) in the hibernation code uses [`pfn_to_online_page()`](https://elixir.bootlin.com/linux/v6.19/source/mm/memory_hotplug.c#L346) to skip offline pages when building the hibernation image. The compaction code's [`__reset_isolation_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/mm/compaction.c#L276) uses it to safely access pageblock flags. The EDAC driver [`skx_mce_check_error()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/edac/skx_common.c#L745) uses it to validate PFNs from machine check error reports before accessing the [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h).

### Driver usage: PFN conversions in practice

PFN conversions are ubiquitous in kernel and driver code. A representative example is the i915 GPU driver setting up a hardware status page in [`ring_setup_phys_status_page()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/gpu/drm/i915/gt/intel_ring_submission.c#L74), which chains [`page_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L73) with [`PFN_PHYS()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L12) to obtain the physical address for hardware register programming:

```c
set_hws_pga(engine, PFN_PHYS(page_to_pfn(status_page(engine))));
```

The GVT-g virtual GPU code in [`gvt_pin_guest_page()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/gpu/drm/i915/gvt/kvmgt.c#L165) uses [`page_to_pfn()`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L73) to verify page contiguity:

```c
else if (page_to_pfn(base_page) + npage != page_to_pfn(cur_page)) {
```

The slab allocator in [`virt_to_slab()`](https://elixir.bootlin.com/linux/v6.19/source/mm/slab.h#L178) uses [`virt_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/page.h#L68) to find the slab page for a given kernel object:

```c
return page_slab(virt_to_page(addr));
```

The GVT-g GTT code in [`alloc_scratch_pages()`](https://elixir.bootlin.com/linux/v6.19/source/drivers/gpu/drm/i915/gvt/gtt.c#L2314) allocates memory with [`get_zeroed_page()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h#L365), then uses [`virt_to_page()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/page.h#L68) to get the [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) for DMA mapping:

```c
daddr = dma_map_page(dev, virt_to_page(scratch_pt), 0, 4096, DMA_BIDIRECTIONAL);
```

### PFN_ALIGN usage

[`PFN_ALIGN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L9) rounds a byte address up to the next page boundary. It is used during early boot and memory setup. For example, [`mark_rodata_ro()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1396) on x86-64 uses it to compute the page-aligned end of the kernel text and rodata sections before applying read-only protection. The per-CPU allocator [`pcpu_page_first_chunk()`](https://elixir.bootlin.com/linux/v6.19/source/mm/percpu.c#L3181) uses [`PFN_ALIGN()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/pfn.h#L9) to ensure chunk sizes are page-aligned for the page-based mapping.

### Memblock PFN iteration

During early boot (before the page allocator is available), the memblock allocator describes memory in terms of physical address ranges. The [`for_each_mem_pfn_range()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/memblock.h#L323) macro iterates over these ranges expressed as PFN pairs:

```c
#define for_each_mem_pfn_range(i, nid, p_start, p_end, p_nid)		\
	for (i = -1, __next_mem_pfn_range(&i, nid, p_start, p_end, p_nid); \
	     i >= 0; __next_mem_pfn_range(&i, nid, p_start, p_end, p_nid))
```

This iterator is used by the sparsemem initialization code in [`memblocks_present()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L244) to discover which sections contain memory, and by memory hotplug, NUMA setup, and various initialization paths that need to walk physical memory.
