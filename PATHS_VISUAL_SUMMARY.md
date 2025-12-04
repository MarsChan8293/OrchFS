# OrchFS Paths Visual Summary

## Overview Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         OrchFS Architecture                              │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    User Write Request                             │  │
│  └────────────────────────────┬─────────────────────────────────────┘  │
│                                │                                         │
│                                ▼                                         │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │           Alignment-Based Partitioning Decision                 │    │
│  │         (LibFS/lib_rwfunc.c:483, line 487-494)                 │    │
│  └─────────────────┬────────────────────┬─────────────────────────┘    │
│                    │                    │                               │
│         Unaligned Write      Aligned to 32KB boundary                   │
│         (< 32KB or offset != 0)    (≥ 32KB, offset % 32KB = 0)        │
│                    │                    │                               │
│                    ▼                    ▼                               │
│  ┌─────────────────────────┐  ┌─────────────────────────┐             │
│  │   PATH 1: HOT WRITE     │  │   PATH 2: COLD WRITE    │             │
│  │                         │  │                         │             │
│  │  Target: NVM            │  │  Target: SSD            │             │
│  │  Node: VIR_LEAF_NODE    │  │  Node: SSD_BLOCK        │             │
│  │  Size: 4KB pages        │  │  Size: 32KB blocks      │             │
│  │  Latency: Low           │  │  Bandwidth: High        │             │
│  │                         │  │                         │             │
│  │  Files:                 │  │  Files:                 │             │
│  │  - lib_rwfunc.c:519     │  │  - lib_rwfunc.c:507     │             │
│  │  - index.c:615          │  │  - index.c:553          │             │
│  │  - append_nvm_pages()   │  │  - append_ssd_blocks()  │             │
│  └────────────┬────────────┘  └─────────────────────────┘             │
│               │                                                         │
│               │  VIR_LEAF_NODE Full (8×4KB = 32KB)                    │
│               │  Trigger: lib_rwfunc.c:570                             │
│               │                                                         │
│               ▼                                                         │
│  ┌─────────────────────────────────────────────────────────────┐      │
│  │         BACKGROUND MIGRATION (Cold Data Consolidation)       │      │
│  │                    (LibFS/migrate.c)                         │      │
│  │                                                              │      │
│  │  Trigger: NVM usage > threshold (1% of total)               │      │
│  │  Strategy: LRU-based batch migration (1024 blocks)          │      │
│  │                                                              │      │
│  │  Process:                                                    │      │
│  │  1. Get LRU node (migrate.c:39)                             │      │
│  │  2. Read 32KB from NVM (migrate.c:59)                       │      │
│  │  3. Allocate SSD block (migrate.c:62)                       │      │
│  │  4. Write to SSD (migrate.c:70)                             │      │
│  │  5. Update HRtree: VIR_LEAF_NODE → SSD_BLOCK (migrate.c:95)│      │
│  │  6. Free NVM pages (migrate.c:100)                          │      │
│  └─────────────────────────────────────────────────────────────┘      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## HRtree Metadata Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                         HRtree Structure                             │
│                      (LibFS/index.h & index.c)                       │
│                                                                      │
│  ┌────────────────────┐                                             │
│  │   Index Root       │  (IDX_ROOT)                                 │
│  │  - idx_inode_id    │                                             │
│  │  - idx_entry_blkid │                                             │
│  │  - max_blk_offset  │                                             │
│  └─────────┬──────────┘                                             │
│            │                                                         │
│            ▼                                                         │
│  ┌─────────────────────┐                                            │
│  │  Non-Leaf Nodes     │  (NOT_LEAF_NODE)                           │
│  │  - zipped_layer     │                                            │
│  │  - son_blk_id[]     │  (multiple levels)                         │
│  │  - virnd_flag       │                                            │
│  └─────────┬───────────┘                                            │
│            │                                                         │
│            ▼                                                         │
│  ┌─────────────────────┐                                            │
│  │   Leaf Nodes        │  (LEAF_NODE)                               │
│  │  - son_blk_id[]     │                                            │
│  │  - virnd_flag       │  bitmap indicating child type              │
│  │  - bit_lock         │  fine-grained locking                      │
│  └─────────┬───────────┘                                            │
│            │                                                         │
│            ├─────────────┬─────────────┬──────────────┐            │
│            ▼             ▼             ▼              ▼            │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌─────────┐  │
│  │ VIR_LEAF_NODE│ │  SSD_BLOCK   │ │ STRATA_NODE  │ │  EMPTY  │  │
│  │              │ │              │ │              │ │         │  │
│  │ HOT DATA     │ │ COLD DATA    │ │ HYBRID       │ │         │  │
│  │ on NVM       │ │ on SSD       │ │ SSD + NVM    │ │         │  │
│  │              │ │              │ │              │ │         │  │
│  │ 8×4KB pages  │ │ 32KB block   │ │ 32KB + delta │ │         │  │
│  │              │ │              │ │              │ │         │  │
│  │ nvm_page_id[]│ │ ssd_dev_addr │ │ Both         │ │         │  │
│  │ buf_meta_id[]│ │              │ │ nvm_page_id[]│ │         │  │
│  │              │ │              │ │ ssd_dev_addr │ │         │  │
│  └──────────────┘ └──────────────┘ └──────────────┘ └─────────┘  │
│       ▲                                     │                       │
│       │                                     │                       │
│       │          Background Migration       │                       │
│       └─────────────────────────────────────┘                       │
│                (LibFS/migrate.c)                                    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

## Write Path State Transitions

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Data State Transitions                             │
│                                                                       │
│  New Write Request                                                    │
│         │                                                             │
│         ├─ Aligned (32KB) ───────────────────────┐                   │
│         │                                         ▼                   │
│         │                              ┌──────────────────┐           │
│         │                              │   SSD_BLOCK      │           │
│         │                              │   (Cold Data)    │           │
│         │                              │                  │           │
│         │                              │   - Direct write │           │
│         │                              │   - 32KB aligned │           │
│         │                              │   - High BW      │           │
│         │                              └──────────────────┘           │
│         │                                                             │
│         └─ Unaligned (<32KB) ────────────┐                           │
│                                           ▼                           │
│                              ┌──────────────────────┐                 │
│                              │  VIR_LEAF_NODE       │                 │
│                              │  (Hot Data)          │                 │
│                              │                      │                 │
│                              │  - Write to NVM      │                 │
│                              │  - 4KB pages         │                 │
│                              │  - Low latency       │                 │
│                              └──────────┬───────────┘                 │
│                                         │                             │
│                                         │ 8 pages full (32KB)        │
│                                         │ Add to LRU queue           │
│                                         │ (lib_rwfunc.c:570)         │
│                                         │                             │
│                                         ▼                             │
│                              ┌──────────────────────┐                 │
│                              │  LRU Migration Queue │                 │
│                              │  (migrate.c:156)     │                 │
│                              └──────────┬───────────┘                 │
│                                         │                             │
│                                         │ NVM usage > threshold      │
│                                         │ (migrate.c:163)            │
│                                         │                             │
│                                         ▼                             │
│                              ┌──────────────────────┐                 │
│                              │  Background Thread   │                 │
│                              │  Migration           │                 │
│                              │  (migrate.c:116)     │                 │
│                              │                      │                 │
│                              │  - Read from NVM     │                 │
│                              │  - Write to SSD      │                 │
│                              │  - Update HRtree     │                 │
│                              └──────────┬───────────┘                 │
│                                         │                             │
│                                         ▼                             │
│                              ┌──────────────────────┐                 │
│                              │   SSD_BLOCK          │                 │
│                              │   (Migrated Cold)    │                 │
│                              │                      │                 │
│                              │  - Free NVM pages    │                 │
│                              │  - 32KB on SSD       │                 │
│                              └──────────────────────┘                 │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

## STRATA Hybrid Node

```
┌────────────────────────────────────────────────────────────────────┐
│                    STRATA Node (Hybrid)                             │
│                    (LibFS/lib_rwfunc.c:262)                         │
│                                                                     │
│  Use Case: Partial update to existing SSD block                    │
│                                                                     │
│  ┌─────────────────────────────────────────────────┐               │
│  │         Existing SSD_BLOCK                       │               │
│  │         32KB on SSD                              │               │
│  └───────────────────┬─────────────────────────────┘               │
│                      │                                              │
│                      │ Partial update (< 6 pages)                  │
│                      │ (STRATA_THRESHOLD = 6)                      │
│                      │                                              │
│                      ▼                                              │
│  ┌─────────────────────────────────────────────────┐               │
│  │         STRATA_NODE                              │               │
│  │                                                  │               │
│  │  SSD Part:                NVM Part:              │               │
│  │  ┌──────────────┐        ┌──────────────┐       │               │
│  │  │ Original     │        │ Modified     │       │               │
│  │  │ 32KB block   │        │ pages only   │       │               │
│  │  │              │        │ (< 6 × 4KB)  │       │               │
│  │  │ ssd_dev_addr │        │ nvm_page_id[]│       │               │
│  │  └──────────────┘        │ buf_meta_id[]│       │               │
│  │                          └──────────────┘       │               │
│  │                                                  │               │
│  │  Read: Merge SSD data + NVM deltas              │               │
│  │  Write: Update NVM pages, keep SSD unchanged    │               │
│  │                                                  │               │
│  │  When >= 6 pages modified:                      │               │
│  │    → Write entire 32KB to SSD (lib_rwfunc.c:91)│               │
│  └─────────────────────────────────────────────────┘               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Key Code Locations Quick Reference

### Path 1: Hot Write (NVM)
```
Entry:     LibFS/lib_func.c:361 (orchfs_write)
Decision:  LibFS/lib_rwfunc.c:487 (alignment check)
Allocate:  LibFS/lib_rwfunc.c:519 (append_nvm_pages)
           LibFS/index.c:615 (append_nvm_pages implementation)
Execute:   LibFS/lib_rwfunc.c:336 (unaligned_nvm_op)
           LibFS/lib_rwfunc.c:622 (do_nvm_task)
```

### Path 2: Cold Data Consolidation (Migration)
```
Trigger:   LibFS/lib_rwfunc.c:570 (add_migrate_node call)
Queue:     LibFS/migrate.c:146 (add_migrate_node)
Thread:    LibFS/migrate.c:116 (wait_and_exec_migrate)
Migrate:   LibFS/migrate.c:37 (do_migrate_operation)
Update:    LibFS/index.c (change_virnd_to_ssdblk)
```

### HRtree Operations
```
Structures:  LibFS/index.h:51-92 (node definitions)
Hot data:    LibFS/index.c:615 (append_nvm_pages)
Cold data:   LibFS/index.c:553 (append_ssd_blocks)
Query:       LibFS/index.c (query_offset_info)
```

### STRATA Hybrid
```
Create:    LibFS/lib_rwfunc.c:262 (create_strata_structure)
Write:     LibFS/lib_rwfunc.c:64 (strata_write)
Read:      LibFS/lib_rwfunc.c:219 (read_strata_info)
Threshold: LibFS/lib_rwfunc.c:21 (STRATA_THRESHOLD = 6)
```

## Configuration Parameters

### Sizes and Alignment
```c
ORCH_PAGE_SIZE   = 4KB     // NVM page granularity
ORCH_BLOCK_SIZE  = 32KB    // SSD block granularity
VLN_SLOT_SUM     = 8       // Pages per virtual node
```

### Migration Thresholds
```c
DEFAULT_MIGRATE_NUM    = 1024  // Blocks per batch
MIGRATE_PERCENTAGE     = 10    // Trigger threshold (% of usable)
CAN_USE_PERCENTAGE     = 10    // Usable NVM space (% of total)
```

### Thread Configuration
```bash
# Set via config_parameter.py
MAX_NVM_THREADS = 4   // Recommended for Intel Optane
MAX_SSD_THREADS = 16  // Match CPU cores, >= 8
```

## Performance Characteristics

| Aspect              | Hot Path (NVM)        | Cold Path (SSD)       |
|---------------------|----------------------|----------------------|
| **Granularity**     | 4KB pages            | 32KB blocks          |
| **Alignment**       | Unaligned OK         | Must be aligned      |
| **Latency**         | Low (~100ns)         | Higher (~10-100μs)   |
| **Bandwidth**       | Moderate             | Very High (GB/s)     |
| **Concurrency**     | High (4 threads)     | Very High (16 threads)|
| **Use Case**        | Random small writes  | Sequential large I/O |
| **Node Type**       | VIR_LEAF_NODE        | SSD_BLOCK            |
| **Optimization**    | Low latency          | High throughput      |

## Summary

OrchFS achieves high performance by:

1. **Alignment-Based Partitioning**: Automatically routes data based on write alignment
2. **Heterogeneous Storage**: Leverages both NVM (low latency) and SSD (high bandwidth)
3. **HRtree Metadata**: Unified index structure managing diverse data layouts
4. **Background Migration**: Converts hot data to cold transparently
5. **Parallel I/O**: Maximizes device utilization with dedicated thread pools

This architecture enables OrchFS to handle both random small writes efficiently (via NVM) and maximize SSD bandwidth for large sequential I/O (via aligned blocks and batch migration).
