---
topics: mm
tags:
    - "mm"
    - "verification-needed"
---

# Sparse Virtual Memory Map (vmemmap)

The sparse virtual memory map ([`vmemmap`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/pgtable_64.h#L256)) is a virtually contiguous array of [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) descriptors that covers the entire physical address space. It is the default PFN-to-page conversion mechanism on 64-bit Linux systems, enabled by [`CONFIG_SPARSEMEM_VMEMMAP`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L415). Only sections containing present memory have their page table entries populated, so holes in the physical address space consume page table pages (a few KB) rather than full [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) arrays (megabytes).

```
    vmemmap: a virtually contiguous struct page array

    Virtual Address Space                    Physical Memory
    ─────────────────                        ───────────────

    vmemmap + 0       ─────────┐
    ┌──────────────┐           │   Page tables map only
    │ struct page  │◄──────────┘   present sections
    │ for PFN 0    │
    ├──────────────┤               ┌──────────────┐
    │ struct page  │──────────────►│ Physical page │
    │ for PFN 1    │               │ backing the   │
    ├──────────────┤               │ struct page   │
    │     ...      │               │ array         │
    ├──────────────┤               └──────────────┘
    │ struct page  │
    │ for PFN N    │               Section with no memory:
    ├──────────────┤               page table entries absent,
    │  (hole)      │ ◄─── no PT   access faults
    ├──────────────┤
    │ struct page  │
    │ for PFN M    │
    └──────────────┘

    Conversion:
      pfn_to_page(pfn) = vmemmap + pfn
      page_to_pfn(page) = page - vmemmap
```

## SUMMARY

[`vmemmap`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/pgtable_64.h#L256) provides O(1) PFN-to-page conversion at the same cost as FLATMEM (a single pointer addition), while retaining the sparsemem memory model's ability to handle holes, hotplug, and NUMA. The key insight is that a 64-bit virtual address space is large enough to reserve a contiguous virtual range covering every possible PFN, and then selectively populate only the page table entries that map to real physical memory.

On x86-64, [`vmemmap`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/pgtable_64.h#L256) is defined as `(struct page *)VMEMMAP_START`, where [`VMEMMAP_START`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/pgtable_64_types.h#L118) is the runtime value stored in [`vmemmap_base`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/kernel/head64.c#L67) (`0xffffea0000000000` with 4-level paging, `0xffd4000000000000` with 5-level paging). On arm64, [`vmemmap`](https://elixir.bootlin.com/linux/v6.19/source/arch/arm64/include/asm/pgtable.h#L31) includes a bias for the physical memory start address so that PFN 0 maps to the correct entry.

The conversion macros in [`include/asm-generic/memory_model.h`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h) are:

- [`__pfn_to_page(pfn)`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L46) = `vmemmap + pfn`
- [`__page_to_pfn(page)`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L47) = `page - vmemmap`

Each architecture implements [`vmemmap_populate()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1562) to create the page table entries for a given virtual address range. The generic code in [`mm/sparse-vmemmap.c`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c) provides a full set of page table population helpers that most architectures use: [`vmemmap_pgd_populate()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L237), [`vmemmap_p4d_populate()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L224), [`vmemmap_pud_populate()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L211), [`vmemmap_pmd_populate()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L198), and [`vmemmap_pte_populate()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L154). A hugepage-optimized path ([`vmemmap_populate_hugepages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L416)) maps entire sections with 2 MB PMD entries for reduced TLB pressure. Memory backing for the page table entries and [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) arrays comes from [`vmemmap_alloc_block_buf()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L87), which draws from the sparse pre-allocated buffer during boot or the page allocator at runtime.

## SPECIFICATIONS

(none; vmemmap is a Linux kernel internal virtual memory map abstraction)

## LINUX KERNEL

- [`'\<vmemmap\>':'arch/x86/include/asm/pgtable_64.h'`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/pgtable_64.h#L256): x86-64 vmemmap pointer, defined as `(struct page *)VMEMMAP_START`
- [`'\<vmemmap\>':'arch/arm64/include/asm/pgtable.h'`](https://elixir.bootlin.com/linux/v6.19/source/arch/arm64/include/asm/pgtable.h#L31): arm64 vmemmap pointer, biased by `memstart_addr` for physical memory offset
- [`'\<vmemmap_base\>':'arch/x86/kernel/head64.c'`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/kernel/head64.c#L67): x86-64 runtime base address for vmemmap (adjustable for KASLR)
- [`'\<__pfn_to_page\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L46): vmemmap PFN-to-page conversion (`vmemmap + pfn`)
- [`'\<__page_to_pfn\>':'include/asm-generic/memory_model.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L47): vmemmap page-to-PFN conversion (`page - vmemmap`)
- [`'\<vmemmap_populate\>':'arch/x86/mm/init_64.c'`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1562): x86-64 arch callback that creates page table entries for a vmemmap range
- [`'\<vmemmap_populate_basepages\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L299): generic vmemmap population using base (4K) page table entries
- [`'\<vmemmap_populate_hugepages\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L416): generic vmemmap population using PMD-level (2 MB) huge page entries
- [`'\<vmemmap_populate_range\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L280): core loop that populates PTE entries for a vmemmap address range
- [`'\<vmemmap_populate_address\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L249): populates a single vmemmap address through the full page table hierarchy
- [`'\<vmemmap_pgd_populate\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L237): ensures PGD entry exists for a vmemmap address
- [`'\<vmemmap_p4d_populate\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L224): ensures P4D entry exists for a vmemmap address
- [`'\<vmemmap_pud_populate\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L211): ensures PUD entry exists for a vmemmap address
- [`'\<vmemmap_pmd_populate\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L198): ensures PMD entry exists for a vmemmap address
- [`'\<vmemmap_pte_populate\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L154): fills a PTE entry for a vmemmap address, allocating or reusing a backing page
- [`'\<vmemmap_set_pmd\>':'arch/x86/mm/init_64.c'`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1522): x86-64 arch callback to set a PMD entry for vmemmap huge page mapping
- [`'\<vmemmap_check_pmd\>':'arch/x86/mm/init_64.c'`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1549): x86-64 arch callback to check whether a PMD already maps a huge page
- [`'\<vmemmap_alloc_block\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L59): allocates a block for vmemmap backing (memblock during boot, page allocator at runtime)
- [`'\<vmemmap_alloc_block_buf\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L87): allocates vmemmap backing from sparse buffer, vmemmap_alloc_block, or altmap
- [`'\<vmemmap_alloc_block_zero\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L187): allocates and zeroes a page table page for vmemmap
- [`'\<__earlyonly_bootmem_alloc\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L51): boot-time allocation wrapper that calls [`memmap_alloc()`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1622)
- [`'\<vmemmap_verify\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L143): warns if vmemmap backing pages are allocated from a remote NUMA node
- [`'\<struct vmem_altmap\>':'include/linux/memremap.h'`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/memremap.h#L21): pre-allocated storage descriptor for device-local vmemmap backing
- [`'\<altmap_alloc_block_buf\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L116): allocates vmemmap backing pages from a [`struct vmem_altmap`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/memremap.h#L21)
- [`'\<vmemmap_free\>':'arch/x86/mm/init_64.c'`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1277): x86-64 teardown of vmemmap page table entries for memory hot-remove
- [`'\<depopulate_section_memmap\>':'mm/sparse.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L676): converts PFN range to vmemmap addresses and calls [`vmemmap_free()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1277)
- [`'\<sync_global_pgds\>':'arch/x86/mm/init_64.c'`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L218): x86-64 synchronization of PGD entries across all page tables after vmemmap population
- [`'\<__populate_section_memmap\>':'mm/sparse-vmemmap.c'`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L561): entry point from sparsemem that calls [`vmemmap_populate()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1562) for a section
- [`'\<vmemmap_populate_print_last\>':'arch/x86/mm/init_64.c'`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1651): prints the last accumulated PMD mapping range for debug output

## KERNEL DOCUMENTATION

- [`Documentation/mm/memory-model.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/memory-model.rst): physical memory model overview covering vmemmap as the sparse vmemmap variant of SPARSEMEM
- [`Documentation/mm/vmemmap_dedup.rst`](https://elixir.bootlin.com/linux/v6.19/source/Documentation/mm/vmemmap_dedup.rst): HugeTLB Vmemmap Optimization (HVO) and Device DAX vmemmap deduplication

## OTHER SOURCES

## DETAILS

### The vmemmap pointer

Each architecture defines a [`vmemmap`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/pgtable_64.h#L256) macro that expands to a `struct page *` pointing to the base of the virtual memory map. On x86-64, [`VMEMMAP_START`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/pgtable_64_types.h#L118) is the runtime variable [`vmemmap_base`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/kernel/head64.c#L67):

```c
/* arch/x86/include/asm/pgtable_64_types.h */
#define __VMEMMAP_BASE_L4	0xffffea0000000000UL
#define __VMEMMAP_BASE_L5	0xffd4000000000000UL
# define VMEMMAP_START		vmemmap_base

/* arch/x86/include/asm/pgtable_64.h */
#define vmemmap ((struct page *)VMEMMAP_START)
```

[`vmemmap_base`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/kernel/head64.c#L67) defaults to [`__VMEMMAP_BASE_L4`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/pgtable_64_types.h#L113) and is switched to [`__VMEMMAP_BASE_L5`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/include/asm/pgtable_64_types.h#L114) during early boot if 5-level paging is enabled. KASLR randomizes the value to prevent predictable virtual address layouts.

On arm64, [`vmemmap`](https://elixir.bootlin.com/linux/v6.19/source/arch/arm64/include/asm/pgtable.h#L31) includes a physical memory offset bias:

```c
/* arch/arm64/include/asm/pgtable.h */
#define vmemmap		((struct page *)VMEMMAP_START - (memstart_addr >> PAGE_SHIFT))
```

By subtracting `memstart_addr >> PAGE_SHIFT`, arm64 ensures that `vmemmap[pfn]` yields the correct [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) even though physical memory may start at an address above 0. The [`VMEMMAP_START`](https://elixir.bootlin.com/linux/v6.19/source/arch/arm64/include/asm/memory.h#L50) and [`VMEMMAP_END`](https://elixir.bootlin.com/linux/v6.19/source/arch/arm64/include/asm/memory.h#L51) are computed from the virtual address space layout.

### PFN-to-page conversion

With [`CONFIG_SPARSEMEM_VMEMMAP`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig#L415), the conversion macros in [`include/asm-generic/memory_model.h`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h) are trivial pointer arithmetic:

```c
/* memmap is virtually contiguous.  */
#define __pfn_to_page(pfn)	(vmemmap + (pfn))
#define __page_to_pfn(page)	(unsigned long)((page) - vmemmap)
```

This is the same cost as FLATMEM (a single addition or subtraction), but the underlying page tables only map sections that contain present memory. Accessing a [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) for a PFN in a hole causes a page fault because the page table entries for that vmemmap range are absent.

The section number is not stored in [`page->flags`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm.h#L1751) when vmemmap is enabled. [`SECTIONS_WIDTH`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/page-flags-layout.h#L55) is 0 for vmemmap configurations, freeing those bits in the flags field for other uses.

### Kconfig selection

vmemmap is selected through [`mm/Kconfig`](https://elixir.bootlin.com/linux/v6.19/source/mm/Kconfig):

```
config SPARSEMEM_VMEMMAP
	def_bool y
	depends on SPARSEMEM && SPARSEMEM_VMEMMAP_ENABLE
	help
	  SPARSEMEM_VMEMMAP uses a virtually mapped memmap to optimise
	  pfn_to_page and page_to_pfn operations.  This is the most
	  efficient option when sufficient kernel resources are available.
```

An architecture must set `SPARSEMEM_VMEMMAP_ENABLE` to allow vmemmap. Most 64-bit architectures (x86-64, arm64, riscv64, powerpc64, s390, loongarch, sparc64) enable it by default.

### Architecture callback: vmemmap_populate

Each architecture implements [`vmemmap_populate()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1562) to create page table mappings for a given vmemmap virtual address range. The x86-64 implementation chooses between huge pages and base pages:

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

For full section mappings on CPUs that support PSE (Page Size Extensions, present on all modern x86-64), it uses [`vmemmap_populate_hugepages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L416) which maps at 2 MB (PMD) granularity. For sub-section mappings or CPUs without PSE, it falls back to [`vmemmap_populate_basepages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L299) using 4K PTEs. After populating, [`sync_global_pgds()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L218) ensures the new PGD entries are visible across all CPU page tables.

Other architectures have similar implementations: arm64 uses [`pmd_set_huge()`](https://elixir.bootlin.com/linux/v6.19/source/arch/arm64/mm/mmu.c#L1755) for huge page mapping, RISC-V always uses huge pages, and s390 uses its own page table management.

### Generic page table populate helpers

The generic code in [`mm/sparse-vmemmap.c`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c) provides a full set of page table population helpers. Each level ensures the entry exists by allocating a page table page if needed:

```c
pgd_t * __meminit vmemmap_pgd_populate(unsigned long addr, int node)
{
	pgd_t *pgd = pgd_offset_k(addr);
	if (pgd_none(*pgd)) {
		void *p = vmemmap_alloc_block_zero(PAGE_SIZE, node);
		if (!p)
			return NULL;
		pgd_populate_kernel(addr, pgd, p);
	}
	return pgd;
}
```

[`vmemmap_p4d_populate()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L224), [`vmemmap_pud_populate()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L211), and [`vmemmap_pmd_populate()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L198) follow the same pattern, each using [`vmemmap_alloc_block_zero()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L187) to allocate zeroed page table pages and the corresponding `*_populate` function to wire them in.

The lowest level, [`vmemmap_pte_populate()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L154), fills a PTE entry with either a freshly allocated backing page or a reuse reference:

```c
pte_t * __meminit vmemmap_pte_populate(pmd_t *pmd, unsigned long addr, int node,
				       struct vmem_altmap *altmap,
				       unsigned long ptpfn, unsigned long flags)
{
	pte_t *pte = pte_offset_kernel(pmd, addr);
	if (pte_none(ptep_get(pte))) {
		pte_t entry;
		void *p;

		if (ptpfn == (unsigned long)-1) {
			p = vmemmap_alloc_block_buf(PAGE_SIZE, node, altmap);
			if (!p)
				return NULL;
			ptpfn = PHYS_PFN(__pa(p));
		} else {
			/*
			 * When a PTE/PMD entry is freed from the init_mm
			 * there's a free_pages() call to this page allocated
			 * above. Thus this get_page() is paired with the
			 * put_page_testzero() on the freeing path.
			 * This can only called by certain ZONE_DEVICE path,
			 * and through vmemmap_populate_compound_pages() when
			 * slab is available.
			 */
			if (flags & VMEMMAP_POPULATE_PAGEREF)
				get_page(pfn_to_page(ptpfn));
		}
		entry = pfn_pte(ptpfn, PAGE_KERNEL);
		set_pte_at(&init_mm, addr, pte, entry);
	}
	return pte;
}
```

When `ptpfn` is -1 (the normal case), a new backing page is allocated. When `ptpfn` is specified, the PTE reuses an existing physical page, enabling the vmemmap deduplication optimization used by HugeTLB Vmemmap Optimization (HVO) and ZONE_DEVICE compound pages.

[`vmemmap_populate_address()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L249) composes the full hierarchy walk for a single address:

```c
static pte_t * __meminit vmemmap_populate_address(unsigned long addr, int node,
					      struct vmem_altmap *altmap,
					      unsigned long ptpfn,
					      unsigned long flags)
{
	pgd_t *pgd;
	p4d_t *p4d;
	pud_t *pud;
	pmd_t *pmd;
	pte_t *pte;

	pgd = vmemmap_pgd_populate(addr, node);
	if (!pgd)
		return NULL;
	p4d = vmemmap_p4d_populate(pgd, addr, node);
	if (!p4d)
		return NULL;
	pud = vmemmap_pud_populate(p4d, addr, node);
	if (!pud)
		return NULL;
	pmd = vmemmap_pmd_populate(pud, addr, node);
	if (!pmd)
		return NULL;
	pte = vmemmap_pte_populate(pmd, addr, node, altmap, ptpfn, flags);
	if (!pte)
		return NULL;
	vmemmap_verify(pte, node, addr, addr + PAGE_SIZE);

	return pte;
}
```

### Huge page vmemmap mapping

[`vmemmap_populate_hugepages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L416) maps vmemmap at PMD granularity (2 MB on x86-64). This reduces TLB pressure because a single 2 MB PMD entry covers the [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array for an entire section (128 MB / 4K pages * 64 bytes = 2 MB):

```c
int __meminit vmemmap_populate_hugepages(unsigned long start, unsigned long end,
					 int node, struct vmem_altmap *altmap)
{
	unsigned long addr;
	unsigned long next;
	pgd_t *pgd;
	p4d_t *p4d;
	pud_t *pud;
	pmd_t *pmd;

	for (addr = start; addr < end; addr = next) {
		next = pmd_addr_end(addr, end);

		pgd = vmemmap_pgd_populate(addr, node);
		if (!pgd)
			return -ENOMEM;

		p4d = vmemmap_p4d_populate(pgd, addr, node);
		if (!p4d)
			return -ENOMEM;

		pud = vmemmap_pud_populate(p4d, addr, node);
		if (!pud)
			return -ENOMEM;

		pmd = pmd_offset(pud, addr);
		if (pmd_none(pmdp_get(pmd))) {
			void *p;

			p = vmemmap_alloc_block_buf(PMD_SIZE, node, altmap);
			if (p) {
				vmemmap_set_pmd(pmd, p, node, addr, next);
				continue;
			} else if (altmap) {
				return -ENOMEM;
			}
		} else if (vmemmap_check_pmd(pmd, node, addr, next))
			continue;
		if (vmemmap_populate_basepages(addr, next, node, altmap))
			return -ENOMEM;
	}
	return 0;
}
```

The loop iterates at PMD-aligned boundaries. For each PMD entry, it walks the higher page table levels using the generic populate helpers, then attempts to allocate a 2 MB block via [`vmemmap_alloc_block_buf()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L87). If the 2 MB allocation succeeds, [`vmemmap_set_pmd()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1522) (an arch callback) installs the PMD entry. If the PMD already has a huge page mapping, [`vmemmap_check_pmd()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1549) verifies it and skips. If the 2 MB allocation fails (and no altmap is in use), it falls back to [`vmemmap_populate_basepages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L299) for that PMD range.

The x86-64 [`vmemmap_set_pmd()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1522) creates a large page PTE and also tracks contiguous mapping blocks for debug output:

```c
void __meminit vmemmap_set_pmd(pmd_t *pmd, void *p, int node,
			       unsigned long addr, unsigned long next)
{
	pte_t entry;

	entry = pfn_pte(__pa(p) >> PAGE_SHIFT,
			PAGE_KERNEL_LARGE);
	set_pmd(pmd, __pmd(pte_val(entry)));

	/* check to see if we have contiguous blocks */
	if (p_end != p || node_start != node) {
		if (p_start)
			pr_debug(" [%lx-%lx] PMD -> [%p-%p] on node %d\n",
				addr_start, addr_end-1, p_start, p_end-1, node_start);
		addr_start = addr;
		node_start = node;
		p_start = p;
	}

	addr_end = addr + PMD_SIZE;
	p_end = p + PMD_SIZE;

	if (!IS_ALIGNED(addr, PMD_SIZE) ||
		!IS_ALIGNED(next, PMD_SIZE))
		vmemmap_use_new_sub_pmd(addr, next);
}
```

### Memory allocation: vmemmap_alloc_block_buf

[`vmemmap_alloc_block_buf()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L87) is the central allocation function for vmemmap backing pages. It uses a three-tier strategy:

```c
void * __meminit vmemmap_alloc_block_buf(unsigned long size, int node,
					 struct vmem_altmap *altmap)
{
	void *ptr;

	if (altmap)
		return altmap_alloc_block_buf(size, altmap);

	ptr = sparse_buffer_alloc(size);
	if (!ptr)
		ptr = vmemmap_alloc_block(size, node);
	return ptr;
}
```

1. If a [`struct vmem_altmap`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/memremap.h#L21) is provided (persistent memory devices), allocation comes from device-local reserved pages via [`altmap_alloc_block_buf()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L116).
2. During boot, [`sparse_buffer_alloc()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L468) draws from the pre-allocated contiguous buffer that [`sparse_buffer_init()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L446) set up for the current NUMA node.
3. If the pre-allocated buffer is exhausted (or after boot), [`vmemmap_alloc_block()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L59) allocates from the page allocator or memblock:

```c
void * __meminit vmemmap_alloc_block(unsigned long size, int node)
{
	/* If the main allocator is up use that, fallback to bootmem. */
	if (slab_is_available()) {
		gfp_t gfp_mask = GFP_KERNEL|__GFP_RETRY_MAYFAIL|__GFP_NOWARN;
		int order = get_order(size);
		static bool warned;
		struct page *page;

		page = alloc_pages_node(node, gfp_mask, order);
		if (page)
			return page_address(page);

		if (!warned) {
			warn_alloc(gfp_mask & ~__GFP_NOWARN, NULL,
				   "vmemmap alloc failure: order:%u", order);
			warned = true;
		}
		return NULL;
	} else
		return __earlyonly_bootmem_alloc(node, size, size,
				__pa(MAX_DMA_ADDRESS));
}
```

During boot, [`__earlyonly_bootmem_alloc()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L51) calls [`memmap_alloc()`](https://elixir.bootlin.com/linux/v6.19/source/mm/mm_init.c#L1622) to allocate from memblock. At runtime (for hotplug), [`alloc_pages_node()`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/gfp.h) allocates from the page allocator with `GFP_KERNEL|__GFP_RETRY_MAYFAIL|__GFP_NOWARN` to allow retries without being fatal.

### NUMA locality verification

[`vmemmap_verify()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L143) is called after every PTE or PMD entry is populated to check that the backing page was allocated from a NUMA-local node:

```c
void __meminit vmemmap_verify(pte_t *pte, int node,
				unsigned long start, unsigned long end)
{
	unsigned long pfn = pte_pfn(ptep_get(pte));
	int actual_node = early_pfn_to_nid(pfn);

	if (node_distance(actual_node, node) > LOCAL_DISTANCE)
		pr_warn_once("[%lx-%lx] potential offnode page_structs\n",
			start, end - 1);
}
```

If the backing page for a vmemmap range ended up on a remote NUMA node, performance of [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) accesses for that memory region degrades due to remote memory latency. The warning fires once to flag the configuration issue.

### struct vmem_altmap for persistent memory

[`struct vmem_altmap`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/memremap.h#L21) allows persistent memory (PMEM) device drivers to store the [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array on the device itself rather than consuming system DRAM:

```c
struct vmem_altmap {
	unsigned long base_pfn;
	const unsigned long end_pfn;
	const unsigned long reserve;
	unsigned long free;
	unsigned long align;
	unsigned long alloc;
};
```

The `base_pfn` and `end_pfn` define the PFN range of the device. The `reserve` field specifies pages reserved for driver use (mapped but excluded from memmap storage). The `free` field tracks available pages for memmap allocation. [`altmap_alloc_block_buf()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L116) consumes pages from the altmap to back the vmemmap page table entries:

```c
static void * __meminit altmap_alloc_block_buf(unsigned long size,
					       struct vmem_altmap *altmap)
{
	unsigned long pfn, nr_pfns, nr_align;

	if (size & ~PAGE_MASK) {
		pr_warn_once("%s: allocations must be multiple of PAGE_SIZE (%ld)\n",
				__func__, size);
		return NULL;
	}

	pfn = vmem_altmap_next_pfn(altmap);
	nr_pfns = size >> PAGE_SHIFT;
	nr_align = 1UL << find_first_bit(&nr_pfns, BITS_PER_LONG);
	nr_align = ALIGN(pfn, nr_align) - pfn;
	if (nr_pfns + nr_align > vmem_altmap_nr_free(altmap))
		return NULL;

	altmap->alloc += nr_pfns;
	altmap->align += nr_align;
	pfn += nr_align;

	pr_debug("%s: pfn: %#lx alloc: %ld align: %ld nr: %#lx\n",
			__func__, pfn, altmap->alloc, altmap->align, nr_pfns);
	return __va(__pfn_to_phys(pfn));
}
```

This is passed through the call chain from [`devm_memremap_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/memremap.c) -> [`arch_add_memory()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c) -> [`sparse_add_section()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L933) -> [`__populate_section_memmap()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L561) -> [`vmemmap_populate()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1562).

### vmemmap teardown: vmemmap_free

Each architecture implements [`vmemmap_free()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1277) to tear down vmemmap page table entries during memory hot-remove. The x86-64 implementation removes the page tables:

```c
void __ref vmemmap_free(unsigned long start, unsigned long end,
		struct vmem_altmap *altmap)
{
	VM_BUG_ON(!PAGE_ALIGNED(start));
	VM_BUG_ON(!PAGE_ALIGNED(end));

	remove_pagetable(start, end, false, altmap);
}
```

[`vmemmap_free()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1277) is called from [`depopulate_section_memmap()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse.c#L676), which converts a PFN range to vmemmap virtual addresses:

```c
static void depopulate_section_memmap(unsigned long pfn, unsigned long nr_pages,
		struct vmem_altmap *altmap)
{
	unsigned long start = (unsigned long) pfn_to_page(pfn);
	unsigned long end = start + nr_pages * sizeof(struct page);

	vmemmap_free(start, end, altmap);
}
```

When an altmap is provided, the freed pages are returned to the device's reserved pool rather than the page allocator.

### Integration with sparsemem

The vmemmap path is invoked through [`__populate_section_memmap()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L561) in [`mm/sparse-vmemmap.c`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c):

```c
struct page * __meminit __populate_section_memmap(unsigned long pfn,
		unsigned long nr_pages, int nid, struct vmem_altmap *altmap,
		struct dev_pagemap *pgmap)
{
	unsigned long start = (unsigned long) pfn_to_page(pfn);
	unsigned long end = start + nr_pages * sizeof(struct page);
	int r;

	if (WARN_ON_ONCE(!IS_ALIGNED(pfn, PAGES_PER_SUBSECTION) ||
		!IS_ALIGNED(nr_pages, PAGES_PER_SUBSECTION)))
		return NULL;

	if (vmemmap_can_optimize(altmap, pgmap))
		r = vmemmap_populate_compound_pages(pfn, start, end, nid, pgmap);
	else
		r = vmemmap_populate(start, end, nid, altmap);

	if (r < 0)
		return NULL;

	return pfn_to_page(pfn);
}
```

It computes the vmemmap virtual address range corresponding to the PFN range (`pfn_to_page(pfn)` yields the vmemmap address), then calls the architecture's [`vmemmap_populate()`](https://elixir.bootlin.com/linux/v6.19/source/arch/x86/mm/init_64.c#L1562) (or [`vmemmap_populate_compound_pages()`](https://elixir.bootlin.com/linux/v6.19/source/mm/sparse-vmemmap.c#L506) for optimizable ZONE_DEVICE pages) to create the page table entries. After population, [`pfn_to_page(pfn)`](https://elixir.bootlin.com/linux/v6.19/source/include/asm-generic/memory_model.h#L73) returns the now-accessible vmemmap address as the section's [`struct page`](https://elixir.bootlin.com/linux/v6.19/source/include/linux/mm_types.h) array.
