# Zephyr页表初始化

## 早期初始化阶段

汇编启动代码: 在最早的启动阶段，Zephyr 的汇编启动代码会初始化基本的硬件设置，包括堆栈指针、CPU 寄存器等。这一阶段通常不涉及复杂的内存管理。
内核初始化入口: 随后，控制权转交给 C 语言的内核初始化代码，通常是 z_cstart 函数。这是内核初始化的入口，负责设置基本的内核数据结构。

cpu0在经过【reset.S】—【z_prep_c】后完成bss清理/数据拷贝/mmu/中断控制器初始化等流程，正式跳转至内核【z_cstart】，调用架构初始化接口【arch_kernel_init】（arm64/cortex-a/r 此接口为空）。

```C
void z_arm64_prep_c(void)
{
 /* Initialize tpidrro_el0 with our struct _cpu instance address */
 write_tpidrro_el0((uintptr_t)&_kernel.cpus[0]);

 z_bss_zero();
 z_data_copy();
#ifdef CONFIG_ARM64_SAFE_EXCEPTION_STACK
 /* After bss clean, _kernel.cpus is in bss section */
 z_arm64_safe_exception_stack_init();
#endif
 z_arm64_mm_init(true);
 z_arm64_interrupt_init();
 z_cstart();

 CODE_UNREACHABLE;
}
```

`z_arm64_mm_init`会完成系统启动时设置页表并启用 MMU：

```C
 /*
  * Only booting core setup up the page tables.
  */
 if (is_primary_core) {
  kernel_ptables.base_xlat_table = new_table();
  setup_page_tables(&kernel_ptables);
 }
```

它主要是调用`setup_page_tables`根据预定义的内存区域和配置，设置虚拟地址空间到物理地址空间的映射，在这之前，它会通过`new_table`来获取第一个未曾使用的空闲页表，返回其地址：

```C
static uint64_t *new_table(void)
{
 uint64_t *table;
 unsigned int i;

 /* Look for a free table. */
 for (i = 0U; i < CONFIG_MAX_XLAT_TABLES; i++) {
  if (xlat_use_count[i] == 0U) {
   table = &xlat_tables[i * Ln_XLAT_NUM_ENTRIES];
   xlat_use_count[i] = 1U;
   MMU_DEBUG("allocating table [%d]%p\n", i, table);
   return table;
  }
 }

 LOG_ERR("CONFIG_MAX_XLAT_TABLES, too small");
 return NULL;
}
```

zephyr似乎是通过**xlat_use_count**来跟踪页表使用状态，而只有在`new_table`、`free_table`、table_usage被显式使用。

`kernel_ptables`是Zephyr 中用于管理 ARM64 架构下页表的结构体。这一结构体包含了与页表配置和管理相关的关键信息：

```C
struct arm_mmu_ptables {
    uint64_t *base_xlat_table;
    uint64_t ttbr0;
};
```

**base_xlat_table**具体指向的是基础地址转换表（PGD），`setup_page_tables`就是在其基础上修改页表的：

```C
static void setup_page_tables(struct arm_mmu_ptables *ptables)
{
 ...
 // 从mmu_config.num_regions中获取最大虚拟地址和最大物理地址，判断是否超出地址范围
 for (index = 0U; index < mmu_config.num_regions; index++) {
  region = &mmu_config.mmu_regions[index];
  max_va = MAX(max_va, region->base_va + region->size);
  max_pa = MAX(max_pa, region->base_pa + region->size);
 }

 __ASSERT(max_va <= (1ULL << CONFIG_ARM64_VA_BITS),
   "Maximum VA not supported\n");
 __ASSERT(max_pa <= (1ULL << CONFIG_ARM64_PA_BITS),
   "Maximum PA not supported\n");

 // 设置 Zephyr 内核的内存映射，包括内核代码段、数据段、以及其他关键内存区域
 for (index = 0U; index < ARRAY_SIZE(mmu_zephyr_ranges); index++) {
  range = &mmu_zephyr_ranges[index];
  add_arm_mmu_flat_range(ptables, range, 0);
 }

 /*
  * Create translation tables for user provided platform regions.
  * Those must not conflict with our default mapping.
  */
 for (index = 0U; index < mmu_config.num_regions; index++) {
  region = &mmu_config.mmu_regions[index];
  add_arm_mmu_region(ptables, region, MT_NO_OVERWRITE);
 }
}
```

`setup_page_tables`使用了`mmu_config.mmu_regions`遍历了所有需要映射的内存区域，通过遍历这个数组，代码获取每个内存区域的基地址（base_va 和 base_pa）以及大小（size），并计算这些区域的最大虚拟地址（max_va）和最大物理地址（max_pa），判断是否超出了内核配置。（**但是，我并没有找到zephyr中哪里初始化了这个变量，而针对于不同的平台，该变量有不同的定义，我怀疑可能是在设备注册时的驱动程序中添加的**）：

```C
struct arm_mmu_config {
 /* Number of regions */
 unsigned int num_regions;
 /* Regions */
 const struct arm_mmu_region *mmu_regions;
};
```

而主要的映射过程是由以下代码完成的：

```C
static void setup_page_tables(struct arm_mmu_ptables *ptables)
{
 ...
 for (index = 0U; index < ARRAY_SIZE(mmu_zephyr_ranges); index++) {
  range = &mmu_zephyr_ranges[index];
  add_arm_mmu_flat_range(ptables, range, 0);
 }

 // 设置用户定义的内存区域映射
 for (index = 0U; index < mmu_config.num_regions; index++) {
  region = &mmu_config.mmu_regions[index];
  add_arm_mmu_region(ptables, region, MT_NO_OVERWRITE);
 }
 ...
}
```

其中首先从`mmu_zephyr_ranges`中读取内存区域，然后调用`add_arm_mmu_flat_range`添加到页表中。而`mmu_zephyr_ranges`是Zephyr定义的一个静态常量数组，它定义了 Zephyr 中的数据段（只读数据段、数据段、BSS 段和未初始化数据段）、代码段的虚拟地址到物理地址的映射规则，以及这些段的访问权限。这个数组中的每个元素都是一个 struct arm_mmu_flat_range 结构体实例，它描述了一个内存区域的名称、起始地址、结束地址和内存属性：

```C
static const struct arm_mmu_flat_range mmu_zephyr_ranges[] = {

 /* Mark the zephyr execution regions (data, bss, noinit, etc.)
  * cacheable, read-write
  * Note: read-write region is marked execute-never internally
  */
 { .name  = "zephyr_data",
   .start = _image_ram_start,
   .end   = _image_ram_end,
   .attrs = MT_NORMAL | MT_P_RW_U_NA | MT_DEFAULT_SECURE_STATE },

 /* Mark text segment cacheable,read only and executable */
 { .name  = "zephyr_code",
   .start = __text_region_start,
   .end   = __text_region_end,
   .attrs = MT_NORMAL | MT_P_RX_U_RX | MT_DEFAULT_SECURE_STATE },

 /* Mark rodata segment cacheable, read only and execute-never */
 { .name  = "zephyr_rodata",
   .start = __rodata_region_start,
   .end   = __rodata_region_end,
   .attrs = MT_NORMAL | MT_P_RO_U_RO | MT_DEFAULT_SECURE_STATE },

#ifdef CONFIG_NOCACHE_MEMORY
 /* Mark nocache segment noncachable, read-write and execute-never */
 { .name  = "nocache_data",
   .start = _nocache_ram_start,
   .end   = _nocache_ram_end,
   .attrs = MT_NORMAL_NC | MT_P_RW_U_RW | MT_DEFAULT_SECURE_STATE },
#endif
};
```

`add_arm_mmu_flat_range`的定义如下：

```C
static inline void add_arm_mmu_flat_range(struct arm_mmu_ptables *ptables,
       const struct arm_mmu_flat_range *range,
       uint32_t extra_flags)
{
 uintptr_t address = (uintptr_t)range->start;
 size_t size = (uintptr_t)range->end - address;

 if (size) {
  /* MMU not yet active: must use unlocked version */
  __add_map(ptables, range->name, address, address,
     size, range->attrs | extra_flags);
 }
}
```

而之后则会读取`mmu_config.num_regions`中的每个内存区域，并依次调用`add_arm_mmu_region`将这些内存区域注册到页表中：

```C
static inline void add_arm_mmu_region(struct arm_mmu_ptables *ptables,
          const struct arm_mmu_region *region,
          uint32_t extra_flags)
{
 if (region->size || region->attrs) {
  /* MMU not yet active: must use unlocked version */
  __add_map(ptables, region->name, region->base_pa, region->base_va,
     region->size, region->attrs | extra_flags);
 }
}
```

都是调用的`__add_map`来将内存区域添加到页表中：

```C
static int __add_map(struct arm_mmu_ptables *ptables, const char *name,
       uintptr_t phys, uintptr_t virt, size_t size, uint32_t attrs)
{
 // 内存区域属性（如访问权限、内存类型）转换为 ARM64 页表项的描述符
 uint64_t desc = get_region_desc(attrs);
 // 判断是否可以覆盖已有映射
 bool may_overwrite = !(attrs & MT_NO_OVERWRITE);

 MMU_DEBUG("mmap [%s]: virt %lx phys %lx size %lx attr %llx %s overwrite\n",
    name, virt, phys, size, desc,
    may_overwrite ? "may" : "no");

 // 对齐检查: 确保虚拟地址、物理地址和区域大小都是页面对齐的
 __ASSERT(((virt | phys | size) & (CONFIG_MMU_PAGE_SIZE - 1)) == 0,
   "address/size are not page aligned\n");
 desc |= phys;
 return set_mapping(ptables, virt, size, desc, may_overwrite);
}
```

`set_mapping`是实际的映射函数：

```C
static int set_mapping(struct arm_mmu_ptables *ptables,
         uintptr_t virt, size_t size,
         uint64_t desc, bool may_overwrite)
{
 uint64_t *pte, *ptes[XLAT_LAST_LEVEL + 1];
 uint64_t level_size;
 uint64_t *table = ptables->base_xlat_table;
 unsigned int level = BASE_XLAT_LEVEL;
 int ret = 0;

 while (size) {
  __ASSERT(level <= XLAT_LAST_LEVEL,
    "max translation table level exceeded\n");

  /* Locate PTE for given virtual address and page table level */
  pte = &table[XLAT_TABLE_VA_IDX(virt, level)];
  ptes[level] = pte;

  if (is_table_desc(*pte, level)) {
   /* Move to the next translation table level */
   level++;
   table = pte_desc_table(*pte);
   continue;
  }

  if (!may_overwrite && !is_free_desc(*pte)) {
   /* the entry is already allocated */
   LOG_ERR("entry already in use: "
    "level %d pte %p *pte 0x%016llx",
    level, pte, *pte);
   ret = -EBUSY;
   break;
  }

  level_size = 1ULL << LEVEL_TO_VA_SIZE_SHIFT(level);

  if (is_desc_superset(*pte, desc, level)) {
   /* This block already covers our range */
   level_size -= (virt & (level_size - 1));
   if (level_size > size) {
    level_size = size;
   }
   goto move_on;
  }

  if ((size < level_size) || (virt & (level_size - 1)) ||
      !is_desc_block_aligned(desc, level_size)) {
   /* Range doesn't fit, create subtable */
   table = expand_to_table(pte, level);
   if (!table) {
    ret = -ENOMEM;
    break;
   }
   level++;
   continue;
  }

  /* Adjust usage count for corresponding table */
  if (is_free_desc(*pte)) {
   table_usage(pte, 1);
  }
  if (!desc) {
   table_usage(pte, -1);
  }
  /* Create (or erase) block/page descriptor */
  set_pte_block_desc(pte, desc, level);

  /* recursively free unused tables if any */
  while (level != BASE_XLAT_LEVEL &&
         is_table_unused(pte)) {
   free_table(pte);
   pte = ptes[--level];
   set_pte_block_desc(pte, 0, level);
   table_usage(pte, -1);
  }

move_on:
  virt += level_size;
  desc += desc ? level_size : 0;
  size -= level_size;

  /* Range is mapped, start again for next range */
  table = ptables->base_xlat_table;
  level = BASE_XLAT_LEVEL;
 }

 return ret;
}
```

