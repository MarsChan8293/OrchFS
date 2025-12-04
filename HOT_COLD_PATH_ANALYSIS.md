# OrchFS Hot and Cold Path Analysis

## Overview

This document analyzes how OrchFS distinguishes and implements the two critical data paths mentioned in the problem statement:

1. **Hot Write Path**: For random, small, frequent write operations → writes to NVM first
2. **Cold Data Consolidation Path**: Background migration of data to SSD in aligned, large blocks

## Quick Summary

### Path Distinction Mechanism

OrchFS uses **alignment-based partitioning** to distinguish between hot and cold paths:

```
Write Request
    ↓
Is the write aligned to 32KB boundary?
    ↓
  No (unaligned)              Yes (aligned)
    ↓                              ↓
HOT PATH                      COLD PATH
Write to NVM                  Write to SSD
(VIR_LEAF_NODE)              (SSD_BLOCK)
    ↓
After NVM fills up
    ↓
Background Migration
    ↓
Migrate to SSD
(becomes SSD_BLOCK)
```

---

## 1. Hot Write Path Implementation

### Location: `LibFS/lib_rwfunc.c` and `LibFS/index.c`

### Entry Point
```
User write() call
    ↓
LibFS/lib_func.c:361 - orchfs_write()
    ↓
LibFS/lib_func.c:266 - orchfs_pwrite()
    ↓
LibFS/lib_func.c:323 - write_into_file()
```

### Hot Path Decision Logic

**File**: `LibFS/lib_rwfunc.c`

**Line 987-998**: Checks if write is unaligned
```c
// Check if write starts at non-block-aligned position
if(start_blk_pos != 0 || (start_blk_pos == 0 && write_len < ORCH_BLOCK_SIZE))
{
    // This is a HOT PATH - use NVM
    iter_start++;
    // ...
    all_unaligned_op(blk_info_pt[0], start_blk_pos, first_blk_len, 
                    &ssd_io_task_end, &nvm_io_task_end, now_rbuf, WRITE_OP);
}
```

**Line 483-522**: Allocates NVM pages for unaligned writes
```c
void append_blk(root_id_t root_id, ino_id_t ino_id, ...)
{
    int64_t end_blk_pos = (file_end_byte & ((1LL<<ORCH_BLOCK_BW)-1));
    
    // Check if unaligned (HOT PATH)
    if((end_blk_pos+1) % ORCH_BLOCK_SIZE != 0)
    {
        need_nvm_page = end_blk_pos / ORCH_PAGE_SIZE + 1;  // HOT: Use NVM
        append_nvm_pages(root_id, ino_id, need_nvm_page, nvm_id_arr);
    }
    else
    {
        need_ssd_blk = off_end - off_start + 1;  // COLD: Use SSD
        append_ssd_blocks(root_id, ino_id, need_ssd_blk, ssd_id_arr);
    }
}
```

### Hot Path Execution

**File**: `LibFS/index.c`

**Line 615-735**: `append_nvm_pages()` - Allocates NVM pages
- Creates or updates VIR_LEAF_NODE (Virtual Leaf Node)
- Each VIR_LEAF_NODE contains 8 × 4KB NVM pages = 32KB total
- Updates HRtree index structure

**File**: `LibFS/lib_rwfunc.c`

**Line 336-380**: `unaligned_nvm_op()` - Creates NVM I/O tasks
**Line 622-660**: `do_nvm_task()` - Executes NVM writes

### Hot Path Characteristics

- **Granularity**: 4KB pages
- **Target Device**: NVM (ORCH_DEV_NVM_PATH)
- **Node Type**: VIR_LEAF_NODE
- **Performance**: Low latency, high concurrency
- **Use Case**: Random, small, frequent writes

---

## 2. Cold Data Consolidation Path Implementation

### Location: `LibFS/migrate.c`

### Migration Trigger

**File**: `LibFS/lib_rwfunc.c`

**Line 568-571**: Adds data to migration queue when VIR_LEAF_NODE is full
```c
#ifdef MIGRATTE_ON
    if(file_end_byte + 1 == ORCH_BLOCK_SIZE)  // Full 32KB block
        add_migrate_node(orch_rt.mig_rtinfo_pt, blk_info_pt + arr_idx, 
                        ino_id, VLN_SLOT_SUM);
#endif
```

### Migration Process

**File**: `LibFS/migrate.c`

**Line 146-172**: `add_migrate_node()` - Adds node to LRU queue
```c
void add_migrate_node(migrate_info_pt mig_info, struct offset_info_t* off_info, ...)
{
    // Add to LRU queue for migration
    add_LRU_node(mig_info->LRU_info, key, &new_LRU_node, sizeof(LRU_node_info_t));
    
    // Update NVM usage counter
    __sync_fetch_and_add(&(mig_info->nvm_page_used), new_page_num);
    
    // Check if migration threshold exceeded
    if(mig_info->nvm_page_used > mig_info->mig_threshold)
    {
        // Wake up migration thread
        sem_post(&(orch_io_scheduler.migrate_sem));
    }
}
```

**Line 116-144**: `wait_and_exec_migrate()` - Migration thread
```c
void* wait_and_exec_migrate(void* para_arg)
{
    while(1)
    {
        sem_wait(&(pool->migrate_sem));  // Wait for migration signal
        
        // Batch migrate blocks
        for(int i = 0; i < mig_info->migrate_num; i++)
        {
            do_migrate_operation(arg->mig_info_pt);
        }
    }
}
```

**Line 37-114**: `do_migrate_operation()` - Single migration operation
```c
int do_migrate_operation(migrate_info_pt mig_info)
{
    // 1. Get node from LRU queue
    LRU_node_info_pt mig_pos_pt = get_and_eliminate_LRU_node(mig_info->LRU_info);
    
    // 2. Read full 32KB block from NVM (8 × 4KB pages)
    for(int i = 0; i < VLN_SLOT_SUM; i++)  // VLN_SLOT_SUM = 8
    {
        read_data_from_devs(page_data_sp, ORCH_PAGE_SIZE, 
                           mig_pos_pt->nvm_page_addr[i]);
        memcpy(blk_data_sp + ORCH_PAGE_SIZE*i, page_data_sp, ORCH_PAGE_SIZE);
    }
    
    // 3. Allocate SSD block
    int64_t new_ssd_blk_id = require_ssd_block_id();
    int64_t new_ssd_addr = ssdblk_to_devaddr(new_ssd_blk_id);
    
    // 4. Write to SSD (aligned 32KB write)
    sendreq_by_shm(para_sp_pt, 3*sizeof(int64_t), blk_data_sp, ORCH_BLOCK_SIZE);
    
    // 5. Update HRtree: VIR_LEAF_NODE → SSD_BLOCK
    change_virnd_to_ssdblk(root_id, mig_ino, mig_off, new_ssd_blk_id);
    
    // 6. Free NVM pages
    __sync_fetch_and_sub(&(mig_info->nvm_page_used), 8);
    
    return 1;
}
```

### Cold Path Characteristics

- **Granularity**: 32KB blocks (aligned)
- **Target Device**: SSD (ORCH_DEV_SSD_PATH)
- **Node Type**: SSD_BLOCK
- **Performance**: High bandwidth, sequential I/O optimized
- **Use Case**: Large, aligned writes for maximum SSD performance

---

## 3. HRtree Metadata Structure

HRtree (Heterogeneous-unit Range Tree) is the core index structure managing data layout.

### Location: `LibFS/index.h` and `LibFS/index.c`

### Key Node Types

**File**: `LibFS/index.h`

**Line 18-25**: Node type definitions
```c
#define IDX_ROOT                      0  // Index root
#define NOT_LEAF_NODE                 1  // Non-leaf index node
#define LEAF_NODE                     2  // Leaf index node
#define VIR_LEAF_NODE                 3  // HOT PATH: NVM pages
#define STRATA_NODE                   4  // Mixed node (partial updates)
#define SSD_BLOCK                     5  // COLD PATH: SSD blocks
```

### Data Structures

**Line 51-59**: Index Node
```c
struct index_node_t
{  
    int64_t ndtype;                                     // Node type
    int64_t zipped_layer;                               // Compressed layers
    uint64_t virnd_flag[2];                             // Bitmap: is child a virtual node?
    uint64_t bit_lock[2];                               // Fine-grained bit locks
    int64_t son_blk_id[NODE_SON_CAPACITY];              // Child block IDs
};
```

**Line 73-81**: Virtual Node (Hot Data)
```c
struct virtual_node_t
{  
    uint64_t ndtype;                                    // = VIR_LEAF_NODE
    uint64_t ssd_dev_addr;                              // SSD address (for migration)
    int64_t max_pos;                                    
    int64_t nvm_page_id[VLN_SLOT_SUM];                  // 8 × 4KB NVM pages
    int64_t buf_meta_id[VLN_SLOT_SUM];                  // Metadata IDs
};
```

### HRtree Role in Both Paths

**Hot Path**:
- Creates/updates VIR_LEAF_NODE
- Sets virnd_flag bitmap in index node
- Stores NVM page addresses

**Cold Path**:
- Clears virnd_flag bitmap
- Changes node type: VIR_LEAF_NODE → SSD_BLOCK
- Updates block ID to point to SSD

---

## 4. Alignment-Based Partitioning Mechanism

### Configuration

**File**: `config/config.h`

```c
#define ORCH_PAGE_SIZE      4096    // 4KB - NVM page size
#define ORCH_BLOCK_SIZE     32768   // 32KB - SSD block size
#define VLN_SLOT_SUM        8       // 8 pages per virtual node
```

### Decision Logic

```
Write Size and Alignment Check
    ↓
    ├─ Write size < 32KB OR not aligned to 32KB boundary
    │      ↓
    │  HOT PATH: Use NVM (VIR_LEAF_NODE)
    │      ↓
    │  Write to NVM in 4KB pages
    │      ↓
    │  Add to LRU queue when full
    │
    └─ Write size >= 32KB AND aligned to 32KB boundary
           ↓
       COLD PATH: Use SSD (SSD_BLOCK)
           ↓
       Write directly to SSD in 32KB blocks
```

---

## 5. Migration Configuration

**File**: `LibFS/migrate.h`

```c
#define DEFAULT_MIGRATE_NUM         1024  // Migrate 1024 blocks per batch
#define MIGRATE_PERCENTAGE          10    // Trigger at 10% usage
#define CAN_USE_PERCENTAGE          10    // 10% of total NVM
```

### Migration Threshold Calculation

**File**: `LibFS/migrate.c`

**Line 14-33**: Initialization
```c
migrate_info_pt init_migrate_info()
{
    ret_pt->all_nvm_page = MAX_PAGE_NUM;
    ret_pt->can_use_page_num = ret_pt->all_nvm_page / 100 * CAN_USE_PERCENTAGE;
    ret_pt->mig_threshold = ret_pt->can_use_page_num / 100 * MIGRATE_PERCENTAGE;
    
    // Example: If MAX_PAGE_NUM = 100000
    // can_use_page_num = 10000 (10%)
    // mig_threshold = 1000 (1% of total, 10% of usable)
}
```

---

## 6. STRATA Hybrid Nodes

STRATA_NODE is a special hybrid node for partial SSD block updates.

### Location: `LibFS/lib_rwfunc.c`

**Line 262-295**: `create_strata_structure()`
- Converts SSD_BLOCK to STRATA_NODE
- Allocates NVM pages for modified pages only
- Maintains full block on SSD + deltas on NVM

**Line 21**: STRATA threshold
```c
#define STRATA_THRESHOLD    6  // If >= 6 pages modified, write entire block to SSD
```

### STRATA Use Case

```
Existing SSD block needs partial update
    ↓
    ├─ Modified pages < 6
    │      ↓
    │  Create STRATA_NODE
    │  Write only modified pages to NVM
    │  Keep full block on SSD
    │
    └─ Modified pages >= 6
           ↓
       Write entire 32KB block to SSD
```

---

## 7. Parallel I/O Engine

### Thread Configuration

**File**: `LibFS/io_thdpool.h`

```c
#define MAX_NVM_THREADS            ORCH_CONFIG_NVMTHD  // NVM threads
#define MAX_SSD_THREADS            ORCH_CONFIG_SSDTHD  // SSD threads
```

Configured via `config_parameter.py`:
```bash
python config_parameter.py /dev/dax0.0 /dev/nvme1n1 4 16 32k
                          [NVM device] [SSD device] [NVM threads] [SSD threads] [split size]
```

### I/O Execution Strategy

**File**: `LibFS/lib_rwfunc.c`

**Line 1044-1050**: Task scheduling
```c
// Execute tasks
if(ssd_task_num > 0)
    task_group_id = add_task(...);  // SSD: parallel execution
do_nvm_task(...);                    // NVM: synchronous execution
if(ssd_task_num > 0)
    wait_task_done(task_group_id);   // Wait for SSD completion
```

---

## 8. Complete Code Path Summary

### Hot Write Path

```
User write() → LibFS/lib_func.c:361 (orchfs_write)
             → LibFS/lib_func.c:266 (orchfs_pwrite)
             → LibFS/lib_func.c:323 (write_into_file)
             → LibFS/lib_rwfunc.c:928 (write_into_file)
             → LibFS/lib_rwfunc.c:966 (acquire_op_blk_info)
             → LibFS/lib_rwfunc.c:538 (append_blk)
             → LibFS/lib_rwfunc.c:519 (append_nvm_pages)
             → LibFS/index.c:615 (append_nvm_pages)
             → [Allocate VIR_LEAF_NODE and NVM pages]
             → LibFS/lib_rwfunc.c:997 (all_unaligned_op)
             → LibFS/lib_rwfunc.c:336 (unaligned_nvm_op)
             → [Create NVM I/O tasks]
             → LibFS/lib_rwfunc.c:622 (do_nvm_task)
             → [Execute NVM writes]
```

### Cold Data Consolidation Path

```
VIR_LEAF_NODE full → LibFS/lib_rwfunc.c:570 (add_migrate_node)
                   → LibFS/migrate.c:146 (add_migrate_node)
                   → [Add to LRU queue]
                   → LibFS/migrate.c:163 (check threshold)
                   → LibFS/migrate.c:168 (wake migration thread)
                   → LibFS/migrate.c:125 (wait_and_exec_migrate)
                   → LibFS/migrate.c:136 (do_migrate_operation)
                   → LibFS/migrate.c:37 (do_migrate_operation)
                   → [Execute migration]
                      1. Read from NVM (line 59)
                      2. Allocate SSD block (line 62)
                      3. Write to SSD (line 70)
                      4. Update HRtree (line 95)
                      5. Free NVM pages (line 100)
```

---

## 9. Key Insights

### How the Two Paths Are Distinguished

1. **By Write Alignment**:
   - Unaligned writes (not on 32KB boundary) → Hot Path → NVM
   - Aligned writes (32KB boundary) → Cold Path → SSD

2. **By Node Type in HRtree**:
   - VIR_LEAF_NODE = Hot data on NVM
   - SSD_BLOCK = Cold data on SSD
   - STRATA_NODE = Hybrid (SSD + NVM deltas)

3. **By Write Size**:
   - Small writes (< 32KB or partial) → Hot Path
   - Large aligned writes (≥ 32KB) → Cold Path

### Performance Optimization

1. **Hot Path**: Optimizes for random, small writes
   - Low latency NVM
   - 4KB granularity
   - High concurrency

2. **Cold Path**: Optimizes for sequential, large writes
   - High bandwidth SSD
   - 32KB aligned blocks
   - Batch migration (1024 blocks)

3. **Migration**: Converts hot data to cold
   - Background operation
   - LRU-based selection
   - Threshold-triggered (1% total NVM usage)

---

## 10. References

### Key Files

- **Hot Path**: `LibFS/lib_rwfunc.c`, `LibFS/index.c`, `LibFS/lib_func.c`
- **Cold Path**: `LibFS/migrate.c`, `LibFS/io_thdpool.c`
- **Metadata**: `LibFS/index.h`, `LibFS/index.c`
- **Config**: `config/config.h`, `config_parameter.py`

### Key Functions

- **Hot Path**: `append_nvm_pages()`, `unaligned_nvm_op()`, `do_nvm_task()`
- **Cold Path**: `add_migrate_node()`, `do_migrate_operation()`, `change_virnd_to_ssdblk()`
- **Decision**: `append_blk()`, `acquire_op_blk_info()`

This analysis demonstrates how OrchFS leverages alignment-based partitioning and the HRtree metadata structure to efficiently manage heterogeneous I/O across NVM and SSD devices.
