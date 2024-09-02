# Zephyr Arm64 页表管理

以下代码是Zephyr用于初始化页表的主要函数，函数调用为：*z_arm64_prep_c->z_arm64_mm_init->setup_page_tables*，并定义在*arch/arm64/core/mmu.c*中：

```C
static void setup_page_tables(struct arm_mmu_ptables *ptables)
{
 unsigned int index;
 const struct arm_mmu_flat_range *range;
 const struct arm_mmu_region *region;
 uintptr_t max_va = 0, max_pa = 0;

 MMU_DEBUG("xlat tables:\n");
 for (index = 0U; index < CONFIG_MAX_XLAT_TABLES; index++)
  MMU_DEBUG("%d: %p\n", index, xlat_tables + index * Ln_XLAT_NUM_ENTRIES);

 for (index = 0U; index < mmu_config.num_regions; index++) {
  region = &mmu_config.mmu_regions[index];
  max_va = MAX(max_va, region->base_va + region->size);
  max_pa = MAX(max_pa, region->base_pa + region->size);
 }

 __ASSERT(max_va <= (1ULL << CONFIG_ARM64_VA_BITS),
   "Maximum VA not supported\n");
 __ASSERT(max_pa <= (1ULL << CONFIG_ARM64_PA_BITS),
   "Maximum PA not supported\n");

 /* setup translation table for zephyr execution regions */
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

 invalidate_tlb_all();

 for (index = 0U; index < ARRAY_SIZE(mmu_zephyr_ranges); index++) {
  size_t size;

  range = &mmu_zephyr_ranges[index];
  size = POINTER_TO_UINT(range->end) - POINTER_TO_UINT(range->start);
  inv_dcache_after_map_helper(range->start, size, range->attrs);
 }

 for (index = 0U; index < mmu_config.num_regions; index++) {
  region = &mmu_config.mmu_regions[index];
  inv_dcache_after_map_helper(UINT_TO_POINTER(region->base_va), region->size,
         region->attrs);
 }
}
```

## Kernel Memory map

对于内核映像的映射，Zephyr使用静态数组的方式，将其内核映像的内存信息定义在内核源码中，为内核数据段、代码段，分别定义了起始地址（start）和终止地址（end），并设置默认内存属性（attrs）

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

其中所需的内核段的实际地址信息，则是从链接脚本 *include/zephyr/arch/arm64/scripts/linker.ld* 中获取。

之后通过`add_arm_mmu_flat_range`将其映射到虚拟内存空间中：

```C
 /* setup translation table for zephyr execution regions */
 for (index = 0U; index < ARRAY_SIZE(mmu_zephyr_ranges); index++) {
  range = &mmu_zephyr_ranges[index];
  add_arm_mmu_flat_range(ptables, range, 0);
 }
```

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

其中`add_arm_mmu_flat_range`和`add_arm_mmu_region`一样，都使用了__add_map实际去设置页表，而在内核映像映射中，其虚拟地址对应range->start，其物理地址同样对应range->start，表明它是1:1映射，而与add_arm_mmu_region不同的是，在映射内核映射时，并没有额外添加内存属性extra_flags，其余页表设置方式同Device Memory map。

## Device Memory map

Zephyr使用一种类似静态定义的方式，获取页表的 device memory 来源，对每一种硬件平台，它先通过解析设备树，得到了每个设备节点的信息，然后通过宏拼接的方式生成实例号，内核之后通过实例号使用设备节点的信息。

> [TODO]关于Zephyr的设备树如何解析，解析后的节点信息如何命名、存放，还未深入了解，目前知道的信息是：
> 在 Zephyr 操作系统中，设备树解析的过程主要发生在编译时，而不是在运行时。这与 Linux 内核有所不同。Zephyr 使用一系列 Python 脚本和宏在编译时解析设备树，并生成相应的头文件，这些头文件包含了与设备树相关的定义和配置数据。
> Zephyr利用一些python代码来解析设备树，该部分代码存放在./scripts/dts/中

而之后在一个mmu_regions.c文件中，定义并初始化该硬件平台的mmu_config，该mmu_config的初始化就利用到了先前设备树解析时生成的实例号，

以raspi 4b举例：

```C
static const struct arm_mmu_region mmu_regions[] = {
 MMU_REGION_FLAT_ENTRY("GIC",
         DT_REG_ADDR_BY_IDX(DT_INST(0, arm_gic), 0),
         DT_REG_SIZE_BY_IDX(DT_INST(0, arm_gic), 0),
         MT_DEVICE_nGnRnE | MT_P_RW_U_NA | MT_DEFAULT_SECURE_STATE),

 MMU_REGION_FLAT_ENTRY("GIC",
         DT_REG_ADDR_BY_IDX(DT_INST(0, arm_gic), 1),
         DT_REG_SIZE_BY_IDX(DT_INST(0, arm_gic), 1),
         MT_DEVICE_nGnRnE | MT_P_RW_U_NA | MT_DEFAULT_SECURE_STATE),
};

const struct arm_mmu_config mmu_config = {
 .num_regions = ARRAY_SIZE(mmu_regions),
 .mmu_regions = mmu_regions,
};
```

其中`MMU_REGION_FLAT_ENTRY`宏主要是做一个赋值操作，如下可以看到，*DT_REG_ADDR_BY_IDX(DT_INST(0, arm_gic), 0)*被同时赋值给了虚拟地址（base_va）和物理地址（base_pa），说明在建立页表时是使用1:1映射的，而且*DT_REG_ADDR_BY_IDX(DT_INST(0, arm_gic), 0)* 实际上是设备树解析后Zephyr生成的一个实例号，通过一些复杂的宏拼接的方式组合成该实例号，引用设备节点的地址：

```C
#define MMU_REGION_FLAT_ENTRY(name, adr, sz, attrs) \
 MMU_REGION_ENTRY(name, adr, adr, sz, attrs)

#define MMU_REGION_ENTRY(_name, _base_pa, _base_va, _size, _attrs) \
 {\
  .name = _name, \
  .base_pa = _base_pa, \
  .base_va = _base_va, \
  .size = _size, \
  .attrs = _attrs, \
 }
```

在完成mmu_config的初始化后，可以得到Device Memory的物理地址、虚拟地址（被映射到的，采用1:1）、内存属性， 之后通过`setup_page_table`，将其作为页表初始化的依据：

```C
static void setup_page_tables(struct arm_mmu_ptables *ptables)
{
 ...
 /*
  * Create translation tables for user provided platform regions.
  * Those must not conflict with our default mapping.
  */
 for (index = 0U; index < mmu_config.num_regions; index++) {
  region = &mmu_config.mmu_regions[index];
  add_arm_mmu_region(ptables, region, MT_NO_OVERWRITE);
 }
 ... 
}
```

可以通过注释看到，mmu_config中提供的内存信息，被解释为**user provided platform regions**，我的理解是，每个硬件平台有很多设备，但是实际上用到的不多，所以Zephyr提供了一种方式，也即是mmu_config的方式，让用户自定义哪些设备是被使用的，作为映射到虚拟内存空间中：

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

对于这部分内存**user provided platform regions**，除了mmu_config初始化时用户提供的内存属性，Zephyr还默认设置了一些内存属性，如上`add_arm_mmu_region`中的*extra_flags*，被设置为了**MT_NO_OVERWRITE**。

`__add_map`中的参数，phys是要映射的物理地址首地址，virt是虚拟地址首地址，根据mmu_config的初始化方法，两者等同，为1:1映射，而内存属性attrs为region->attrs | extra_flags，即Zephyr默认设置的内存属性加上用户提供的内存属性：

```C
static int __add_map(struct arm_mmu_ptables *ptables, const char *name,
       uintptr_t phys, uintptr_t virt, size_t size, uint32_t attrs)
{
 uint64_t desc = get_region_desc(attrs);
 bool may_overwrite = !(attrs & MT_NO_OVERWRITE);

 MMU_DEBUG("mmap [%s]: virt %lx phys %lx size %lx attr %llx %s overwrite\n",
    name, virt, phys, size, desc,
    may_overwrite ? "may" : "no");
 __ASSERT(((virt | phys | size) & (CONFIG_MMU_PAGE_SIZE - 1)) == 0,
   "address/size are not page aligned\n");
 desc |= phys;
 return set_mapping(ptables, virt, size, desc, may_overwrite);
}
```

而每个要映射的内存区域会根据其已经设置的标志（region->attrs | extra_flags），并为其设置内存属性：

```C
static uint64_t get_region_desc(uint32_t attrs)
{
 unsigned int mem_type;
 uint64_t desc = 0U;

 /* NS bit for security memory access from secure state */
 desc |= (attrs & MT_NS) ? PTE_BLOCK_DESC_NS : 0;

 /*
  * AP bits for EL0 / ELh Data access permission
  *
  *   AP[2:1]   ELh  EL0
  * +--------------------+
  *     00      RW   NA
  *     01      RW   RW
  *     10      RO   NA
  *     11      RO   RO
  */

 /* AP bits for Data access permission */
 desc |= (attrs & MT_RW) ? PTE_BLOCK_DESC_AP_RW : PTE_BLOCK_DESC_AP_RO;

 /* Mirror permissions to EL0 */
 desc |= (attrs & MT_RW_AP_ELx) ?
   PTE_BLOCK_DESC_AP_ELx : PTE_BLOCK_DESC_AP_EL_HIGHER;

 /* the access flag */
 desc |= PTE_BLOCK_DESC_AF;

 /* memory attribute index field */
 mem_type = MT_TYPE(attrs);
 desc |= PTE_BLOCK_DESC_MEMTYPE(mem_type);

 switch (mem_type) {
 case MT_DEVICE_nGnRnE:
 case MT_DEVICE_nGnRE:
 case MT_DEVICE_GRE:
  /* Access to Device memory and non-cacheable memory are coherent
   * for all observers in the system and are treated as
   * Outer shareable, so, for these 2 types of memory,
   * it is not strictly needed to set shareability field
   */
  desc |= PTE_BLOCK_DESC_OUTER_SHARE;
  /* Map device memory as execute-never */
  desc |= PTE_BLOCK_DESC_PXN;
  desc |= PTE_BLOCK_DESC_UXN;
  break;
 case MT_NORMAL_NC:
 case MT_NORMAL:
  /* Make Normal RW memory as execute never */
  if ((attrs & MT_RW) || (attrs & MT_P_EXECUTE_NEVER))
   desc |= PTE_BLOCK_DESC_PXN;

  if (((attrs & MT_RW) && (attrs & MT_RW_AP_ELx)) ||
       (attrs & MT_U_EXECUTE_NEVER))
   desc |= PTE_BLOCK_DESC_UXN;

  if (mem_type == MT_NORMAL)
   desc |= PTE_BLOCK_DESC_INNER_SHARE;
  else
   desc |= PTE_BLOCK_DESC_OUTER_SHARE;
 }

 /* non-Global bit */
 if (attrs & MT_NG) {
  desc |= PTE_BLOCK_DESC_NG;
 }

 return desc;
}
```

该函数所作工作：

1. 检测NS位，如果设置了MT_NS标志，则将其设置为PTE_BLOCK_DESC_NS
2. 检测AP位，如果设置了MT_RW标志，则将其设置为PTE_BLOCK_DESC_AP_RW，否则是PTE_BLOCK_DESC_AP_RO
3. AF位默认设置为PTE_BLOCK_DESC_AF
4. 检测内存类型，使用MT_TYPE从attr中提取内存类型，内存类型是在mmu_config初始化中定义的（用户提供）：
5. 根据第四步检测出的设备类型，设置内存共享属性，对于Device Memory，设置为Inner Shareable，对于Normal Memory，根据具体情况设置为内部或外部共享性。
6. 如果内存是可执行的，则清除 PXN（特权级执行禁止）和 UXN（用户级执行禁止）位；否则，将这些位设置为禁止执行。
7. 如果 attrs 中包含 MT_NG 标志，则设置非全局位。这通常用于控制 TLB 刷新的行为。

可选的设备类型有：

```C
#define MT_TYPE_MASK  0x7U
#define MT_TYPE(attr)  (attr & MT_TYPE_MASK)
#define MT_DEVICE_nGnRnE 0U
#define MT_DEVICE_nGnRE  1U
#define MT_DEVICE_GRE  2U
#define MT_NORMAL_NC  3U
#define MT_NORMAL  4U
#define MT_NORMAL_WT  5U
```

在经过`get_region_desc`之后，页表的内存属性来源已经确定了，最后通过`set_mapping`来设置页表

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
