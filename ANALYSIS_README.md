# OrchFS Hot and Cold Path Analysis - Documentation Guide

## Overview

This repository contains detailed analysis of OrchFS's hot write path and cold data consolidation path as described in the FAST'25 paper. The analysis covers how OrchFS distinguishes between these two paths using alignment-based partitioning and the HRtree metadata structure.

## Available Documentation

### 1. [PATHS_VISUAL_SUMMARY.md](PATHS_VISUAL_SUMMARY.md) - **START HERE** 🚀
**Quick visual reference guide with diagrams**

Best for: Getting a quick understanding of the architecture
- ASCII diagrams showing data flow
- HRtree structure visualization
- State transition diagrams
- Quick code location reference
- Performance characteristics table

**Estimated reading time: 10-15 minutes**

### 2. [HOT_COLD_PATH_ANALYSIS.md](HOT_COLD_PATH_ANALYSIS.md) - **English Version** 📖
**Comprehensive English analysis**

Best for: Detailed understanding with code references
- Complete path analysis with code locations
- Line-by-line breakdown
- Function call chains
- Configuration parameters
- Implementation details

**Estimated reading time: 30-40 minutes**

### 3. [ARCHITECTURE_ANALYSIS.md](ARCHITECTURE_ANALYSIS.md) - **中文版本** 📚
**详细的中文架构分析**

适合: 需要中文详细说明的读者
- 完整的代码路径分析
- 详细的实现机制说明
- HRtree 元数据结构分析
- STRATA 混合节点说明
- 性能优化机制

**预计阅读时间: 30-40 分钟**

## Quick Answer to the Problem Statement

**问题**: 代码仓里1和2是如何区分的？

**Answer**: OrchFS uses **alignment-based partitioning** to distinguish the two paths:

### Path 1: Hot Write Path (热写路径)
- **Decision Point**: `LibFS/lib_rwfunc.c:487`
- **Condition**: `if((end_blk_pos+1) % ORCH_BLOCK_SIZE != 0)`
- **Result**: Unaligned writes → allocate NVM pages (VIR_LEAF_NODE)
- **Implementation**: `LibFS/lib_rwfunc.c:519` → `LibFS/index.c:615` (append_nvm_pages)

### Path 2: Cold Data Consolidation Path (冷数据整合路径)
- **Trigger Point**: `LibFS/lib_rwfunc.c:570` (when VIR_LEAF_NODE is full)
- **Condition**: `if(file_end_byte + 1 == ORCH_BLOCK_SIZE)` + NVM usage threshold
- **Result**: Background migration via `LibFS/migrate.c:37-172`
- **Process**: Read from NVM → Write to SSD → Update HRtree (VIR_LEAF_NODE → SSD_BLOCK)

### HRtree's Role
- **Structure**: `LibFS/index.h:51-92`
- **Purpose**: Unified metadata managing heterogeneous data layout
- **Node Types**:
  - `VIR_LEAF_NODE` (value 3) = Hot data on NVM
  - `SSD_BLOCK` (value 5) = Cold data on SSD
  - `STRATA_NODE` (value 4) = Hybrid (SSD + NVM deltas)

## Key Insights

1. **Alignment is the Key**: 
   - 32KB-aligned writes → direct to SSD
   - Unaligned writes → NVM first, migrate later

2. **Two-Stage Process**:
   - Stage 1 (Hot): Fast writes to NVM (4KB granularity)
   - Stage 2 (Cold): Batch migration to SSD (32KB blocks)

3. **Automatic Management**:
   - No user intervention needed
   - Background migration thread
   - LRU-based cold data selection

## Code Location Quick Reference

```
Hot Path Entry:
  LibFS/lib_func.c:361 → orchfs_write()
  LibFS/lib_rwfunc.c:928 → write_into_file()
  LibFS/lib_rwfunc.c:487 → alignment check (PATH DECISION)
  LibFS/lib_rwfunc.c:519 → append_nvm_pages()
  LibFS/index.c:615 → append_nvm_pages() implementation

Cold Path (Migration):
  LibFS/lib_rwfunc.c:570 → add_migrate_node()
  LibFS/migrate.c:146 → add_migrate_node() implementation
  LibFS/migrate.c:37 → do_migrate_operation()
  [Background thread migrates NVM → SSD]

HRtree Metadata:
  LibFS/index.h:51-92 → data structure definitions
  LibFS/index.c → index operations (query, update, migrate)
```

## Reading Recommendations

### For Developers
1. Start with [PATHS_VISUAL_SUMMARY.md](PATHS_VISUAL_SUMMARY.md) for overview
2. Read [HOT_COLD_PATH_ANALYSIS.md](HOT_COLD_PATH_ANALYSIS.md) for details
3. Reference specific code files mentioned

### For Researchers
1. Review [HOT_COLD_PATH_ANALYSIS.md](HOT_COLD_PATH_ANALYSIS.md) or [ARCHITECTURE_ANALYSIS.md](ARCHITECTURE_ANALYSIS.md)
2. Cross-reference with FAST'25 paper
3. Examine code implementation in detail

### For Quick Reference
- Use [PATHS_VISUAL_SUMMARY.md](PATHS_VISUAL_SUMMARY.md)
- Jump to "Key Code Locations Quick Reference" section
- Check configuration parameters table

## Related Files in Repository

- **Implementation**: `LibFS/lib_rwfunc.c`, `LibFS/migrate.c`, `LibFS/index.c`
- **Headers**: `LibFS/index.h`, `LibFS/migrate.h`, `LibFS/io_thdpool.h`
- **Configuration**: `config/config.h`, `config_parameter.py`
- **Main README**: [README.MD](README.MD)

## Questions?

If you have questions about the analysis, please refer to:
1. The comprehensive documentation files listed above
2. The OrchFS FAST'25 paper
3. Contact: zhanyekang@foxmail.com (from main README)

---

**Note**: This analysis was created to answer the question about how OrchFS distinguishes between the hot write path and cold data consolidation path. No code modifications were made - this is purely documentation based on code analysis.
