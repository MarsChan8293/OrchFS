# OrchFS 架构分析：热写路径与冷数据整合路径

## 概述

OrchFS 通过异构 I/O 编排实现了高效的文件系统写入机制，主要包含两条核心路径：

1. **热写路径（Hot Write Path）**：面向随机、小块、频繁更新的写操作，将数据首先写入对随机写友好的 NVM
2. **冷数据整合路径（Cold Data Consolidation Path）**：在后台将随机写数据以对齐、大块的形式整理并迁移到高带宽 SSD

本文档详细分析了这两条路径在代码中的实现位置和工作机制。

---

## 一、热写路径（Hot Write Path）

热写路径负责处理未对齐到 SSD 页边界的小块、随机写入操作，将这些数据先写入 NVM 以提高写入性能。

### 1.1 核心文件

- **LibFS/lib_rwfunc.c** - 主要的读写函数实现
- **LibFS/index.c** - HRtree 索引管理
- **LibFS/index.h** - 索引数据结构定义

### 1.2 关键数据结构

#### 虚拟叶子节点（Virtual Leaf Node）
```c
// LibFS/index.h:73-81
struct virtual_node_t
{  
    uint64_t ndtype;                                    // 节点类型 = VIR_LEAF_NODE
    uint64_t ssd_dev_addr;                              // SSD 上的块地址
    int64_t max_pos;                                    
    int64_t nvm_page_id[VLN_SLOT_SUM];                  // NVM 页面 ID，用于缓存
    int64_t buf_meta_id[VLN_SLOT_SUM];                  // NVM 页面的元数据
};
```

#### 节点类型定义
```c
// LibFS/index.h:18-25
#define IDX_ROOT                      0
#define NOT_LEAF_NODE                 1
#define LEAF_NODE                     2
#define VIR_LEAF_NODE                 3    // 热写路径使用
#define STRATA_NODE                   4    // 混合节点（部分在 NVM，部分在 SSD）
#define SSD_BLOCK                     5    // 冷数据路径使用
```

### 1.3 热写路径执行流程

#### 流程图
```
用户写入请求
    ↓
orchfs_write() / orchfs_pwrite()
  (LibFS/lib_func.c:361, 266)
    ↓
write_into_file()
  (LibFS/lib_rwfunc.c:928)
    ↓
acquire_op_blk_info()
  (LibFS/lib_rwfunc.c:525)
    ↓
判断是否需要分配新块
    ↓
append_blk()
  (LibFS/lib_rwfunc.c:483)
    ↓
append_nvm_pages()      ← 热写路径：分配 NVM 页面
  (LibFS/index.c:615)
    ↓
unaligned_nvm_op()      ← 生成 NVM I/O 任务
  (LibFS/lib_rwfunc.c:336)
    ↓
do_nvm_task()          ← 执行 NVM 写入
  (LibFS/lib_rwfunc.c:622)
```

#### 详细代码路径

**1. 写入入口函数**
```c
// LibFS/lib_func.c:266
int64_t orchfs_pwrite(int fd, const void *buf, int64_t write_len, int64_t offset)
{
    // 调用核心写入函数
    write_into_file(fd, offset, write_len, buf);  // 行 323 或 350
}
```

**2. 核心写入函数**
```c
// LibFS/lib_rwfunc.c:928
void write_into_file(int fd, int64_t file_start_byte, int64_t write_len, void* write_buf)
{
    // 获取块信息
    acquire_op_blk_info(root_id, ino_id, blk_info_pt, start_blk_off, end_blk_off, end_blk_pos);
    
    // 处理未对齐的写入（热写路径）
    if(start_blk_pos != 0 || (start_blk_pos == 0 && write_len < ORCH_BLOCK_SIZE))
    {
        all_unaligned_op(blk_info_pt[0], start_blk_pos, first_blk_len, 
                        &ssd_io_task_end, &nvm_io_task_end, now_rbuf, WRITE_OP);
    }
}
```

**3. 判断和分配路径**
```c
// LibFS/lib_rwfunc.c:483-522
void append_blk(root_id_t root_id, ino_id_t ino_id, int64_t off_start, 
                int64_t off_end, int64_t file_end_byte)
{
    int64_t need_ssd_blk = 0, need_nvm_page = 0;
    int64_t end_blk_pos = (file_end_byte & ((1LL<<ORCH_BLOCK_BW)-1));
    
    // 判断是否需要 NVM 页面（未对齐的小块写入）
    if((end_blk_pos+1) % ORCH_BLOCK_SIZE != 0)
    {
        need_ssd_blk = off_end - off_start;
        need_nvm_page = end_blk_pos / ORCH_PAGE_SIZE + 1;  // 热写路径
    }
    
    // 分配 NVM 页面
    if(need_nvm_page != 0)
    {
        nvm_page_id_t* nvm_id_arr = malloc(sizeof(nvm_page_id_t) * need_nvm_page);
        for(int64_t j = 0; j < need_nvm_page; j++)
            nvm_id_arr[j] = require_nvm_page_id();
        append_nvm_pages(root_id, ino_id, need_nvm_page, nvm_id_arr);  // 行 519
    }
}
```

**4. NVM 页面分配**
```c
// LibFS/index.c:615-735
int append_nvm_pages(root_id_t root_id, int64_t inode_id, 
                    int32_t app_blk_num, nvm_page_id_t nvm_page_id_arr[])
{
    // 找到或创建虚拟叶子节点
    vir_nd_pt vir_leaf_pt = get_or_create_virnd(root_pt, inode_id, app_blk_offset);
    
    // 分配 NVM 页面 ID
    for(int32_t i = 0; i < app_blk_num; i++)
    {
        if(vir_leaf_pt->nvm_page_id[now_page_offset] == EMPTY_BLKID)
        {
            vir_leaf_pt->nvm_page_id[now_page_offset] = nvm_page_id_arr[now_page_cur];
            // 更新 HRtree 索引
        }
    }
}
```

**5. 执行 NVM I/O 操作**
```c
// LibFS/lib_rwfunc.c:336-380
void unaligned_nvm_op(off_info_t blk_info, int64_t blk_pos, int64_t io_len, 
                      io_task_pt* task_end_pt, void* buf, int io_type)
{
    // 将 NVM 写入任务添加到任务队列
    while(io_len > 0)
    {
        io_task_pt now_task_pt = *task_end_pt;
        now_task_pt->next = malloc(sizeof(io_task_t));
        
        now_task_pt->io_start_addr = blk_info.nvm_page_id[now_page_off] + now_page_pos;
        now_task_pt->io_type = io_type;
        now_task_pt->io_len = now_write_size;
        now_task_pt->io_data_buf = (void*)now_buf_addr;
    }
}

// LibFS/lib_rwfunc.c:622
void do_nvm_task(io_task_pt task_head_pt, io_task_pt task_end_pt)
{
    // 执行 NVM 写入任务
    while(now_task_pt != task_end_pt)
    {
        if((now_task_pt->io_type & WRITE_OP) != 0)
        {
            write_data_to_devs(nt_pt->io_data_buf, nt_pt->io_len, nt_pt->io_start_addr);
        }
    }
}
```

### 1.4 热写路径特征识别

代码中通过以下方式识别和处理热写：

1. **未对齐检测**：检查写入偏移是否对齐到 32KB 块边界
   ```c
   // LibFS/lib_rwfunc.c:987
   if(start_blk_pos != 0 || (start_blk_pos == 0 && write_len < ORCH_BLOCK_SIZE))
   ```

2. **节点类型判断**：检查块信息的节点类型
   ```c
   // LibFS/lib_rwfunc.c:412
   if(blk_info.ndtype == VIR_LEAF_NODE)  // NVM 节点
   ```

3. **小块写入检测**：文件末尾未填满一个完整块
   ```c
   // LibFS/lib_rwfunc.c:487
   if((end_blk_pos+1) % ORCH_BLOCK_SIZE != 0)  // 需要 NVM 页面
   ```

---

## 二、冷数据整合路径（Cold Data Consolidation Path）

冷数据整合路径负责在后台将 NVM 中的小块数据迁移到 SSD，以大块、对齐的方式写入，充分利用 SSD 的顺序 I/O 性能。

### 2.1 核心文件

- **LibFS/migrate.c** - 迁移操作的核心实现
- **LibFS/migrate.h** - 迁移相关数据结构
- **LibFS/io_thdpool.c** - 迁移线程池管理

### 2.2 关键数据结构

#### 迁移信息结构
```c
// LibFS/migrate.h:33-46
struct migrate_info_t
{
    struct LRU_t* LRU_info;                     // LRU 队列，管理待迁移数据
    int64_t migrate_num;                        // 每次迁移的块数量
    int64_t nvm_page_used;                      // 已使用的 NVM 页面数
    int64_t all_nvm_page;                       // 总 NVM 页面数
    int64_t can_use_page_num;                   // 可用页面数阈值
    int64_t mig_threshold;                      // 触发迁移的阈值
    int64_t mig_state;                          // 迁移状态
    struct timespec last_migrate_time;    
    int64_t all_mig_blk;                        // 已迁移的块总数
    int64_t max_page_use;                       // 最大页面使用量
};
```

#### LRU 节点信息
```c
// LibFS/migrate.h:24-30
struct LRU_node_info_t
{
    int64_t ino_id;                             // inode ID
    int64_t offset;                             // 文件偏移
    int64_t nvm_page_addr[10];                  // NVM 页面地址
};
```

### 2.3 冷数据整合路径执行流程

#### 流程图
```
VIR_LEAF_NODE 写满
    ↓
add_migrate_node()
  (LibFS/migrate.c:146)
    ↓
检查 NVM 使用量
    ↓
超过阈值？
    ↓ 是
唤醒迁移线程
    ↓
wait_and_exec_migrate()
  (LibFS/migrate.c:116)
    ↓
do_migrate_operation()
  (LibFS/migrate.c:37)
    ↓
1. 从 NVM 读取数据
2. 分配 SSD 块
3. 写入 SSD
4. 更新 HRtree 索引
    ↓
change_virnd_to_ssdblk()
  (LibFS/index.c)
    ↓
释放 NVM 页面
```

#### 详细代码路径

**1. 触发迁移条件**
```c
// LibFS/lib_rwfunc.c:568-571
#ifdef MIGRATTE_ON
    if(file_end_byte + 1 == ORCH_BLOCK_SIZE)  // VIR_LEAF_NODE 写满
        add_migrate_node(orch_rt.mig_rtinfo_pt, blk_info_pt + arr_idx, 
                        ino_id, VLN_SLOT_SUM);
#endif
```

**2. 添加到迁移队列**
```c
// LibFS/migrate.c:146-172
void add_migrate_node(migrate_info_pt mig_info, struct offset_info_t* off_info, 
                      int64_t ino_id, int64_t new_page_num)
{
    // 构建 LRU 节点
    LRU_node_info_t new_LRU_node;
    new_LRU_node.ino_id = ino_id;
    new_LRU_node.offset = off_info->offset_ans;
    for(int i = 0; i < VLN_SLOT_SUM; i++)
        new_LRU_node.nvm_page_addr[i] = off_info->nvm_page_id[i];
    
    // 添加到 LRU 队列
    add_LRU_node(mig_info->LRU_info, key, &new_LRU_node, sizeof(LRU_node_info_t));
    
    // 更新使用量
    __sync_fetch_and_add(&(mig_info->nvm_page_used), new_page_num);
    
    // 检查是否需要触发迁移
    if(mig_info->nvm_page_used > mig_info->mig_threshold)
    {
        if(__sync_fetch_and_or(&(mig_info->mig_state), DO_MIGRATE) == SLEEP)
        {
            sem_post(&(orch_io_scheduler.migrate_sem));  // 唤醒迁移线程
        }
    }
}
```

**3. 迁移线程等待和执行**
```c
// LibFS/migrate.c:116-144
void* wait_and_exec_migrate(void* para_arg)
{
    migrate_info_pt mig_info = arg->mig_info_pt;
    
    while(1)
    {
        sem_wait(&(pool->migrate_sem));  // 等待迁移信号
        
        // 批量迁移
        for(int i = 0; i < mig_info->migrate_num; i++)
        {
            int success_flag = do_migrate_operation(arg->mig_info_pt);
            if(success_flag == 0)
                break;
        }
        
        __sync_fetch_and_and(&(mig_info->mig_state), SLEEP);
    }
}
```

**4. 执行单次迁移操作**
```c
// LibFS/migrate.c:37-114
int do_migrate_operation(migrate_info_pt mig_info)
{
    // 1. 从 LRU 队列获取待迁移节点
    LRU_node_info_pt mig_pos_pt = get_and_eliminate_LRU_node(mig_info->LRU_info);
    
    // 2. 从 NVM 读取完整块数据（8 个 4KB 页面 = 32KB 块）
    void* blk_data_sp = malloc(ORCH_BLOCK_SIZE);
    void* page_data_sp = malloc(ORCH_PAGE_SIZE);
    for(int i = 0; i < VLN_SLOT_SUM; i++)  // VLN_SLOT_SUM = 8
    {
        read_data_from_devs(page_data_sp, ORCH_PAGE_SIZE, mig_pos_pt->nvm_page_addr[i]);
        memcpy(blk_data_sp + ORCH_PAGE_SIZE*i, page_data_sp, ORCH_PAGE_SIZE);
    }
    
    // 3. 分配 SSD 块
    int64_t new_ssd_blk_id = require_ssd_block_id();
    int64_t new_ssd_addr = ssdblk_to_devaddr(new_ssd_blk_id);
    
    // 4. 写入 SSD（通过共享内存发送给 KernelFS）
    int64_t* para_sp_pt = malloc(3 * sizeof(int64_t));
    para_sp_pt[0] = SHM_WRITE; 
    para_sp_pt[1] = new_ssd_addr;
    para_sp_pt[2] = ORCH_BLOCK_SIZE;
    sendreq_by_shm(para_sp_pt, 3*sizeof(int64_t), blk_data_sp, ORCH_BLOCK_SIZE);
    
    // 5. 更新 HRtree 索引：VIR_LEAF_NODE → SSD_BLOCK
    file_lock_rdlock(now_fd);
    lock_range_lock(root_id, mig_ino, mig_off, mig_off);
    
    change_virnd_to_ssdblk(root_id, mig_ino, mig_off, new_ssd_blk_id);
    
    unlock_range_lock(root_id, mig_ino, mig_off, mig_off);
    file_lock_unlock(now_fd);
    
    // 6. 释放 NVM 页面
    __sync_fetch_and_sub(&(mig_info->nvm_page_used), 8);
    
    return 1;
}
```

**5. 更新索引结构**
```c
// LibFS/index.c（需要查找 change_virnd_to_ssdblk 函数）
void change_virnd_to_ssdblk(root_id_t root_id, int64_t inode_id, 
                            int64_t blk_offset, int64_t changed_blkid)
{
    // 找到对应的索引节点
    idx_nd_pt leaf_pt = get_leafnode_pt(root_pt, inode_id, blk_offset);
    
    // 将虚拟节点标记清除，更新为 SSD 块
    FETCH_AND_unSET_BIT(leaf_pt->virnd_flag, leaf_pos);
    leaf_pt->son_blk_id[leaf_pos] = changed_blkid;
    
    // 写日志并释放虚拟节点
    write_change_log(...);
}
```

### 2.4 迁移触发条件

代码中通过以下机制触发迁移：

1. **NVM 使用量阈值**
   ```c
   // LibFS/migrate.c:14-19
   #define DEFAULT_MIGRATE_NUM         1024  // 每次迁移块数
   #define MIGRATE_PERCENTAGE          10    // 迁移阈值百分比
   #define CAN_USE_PERCENTAGE          10    // 可用空间百分比
   
   // LibFS/migrate.c:24-26
   ret_pt->can_use_page_num = ret_pt->all_nvm_page / 100 * CAN_USE_PERCENTAGE;
   ret_pt->mig_threshold = ret_pt->can_use_page_num / 100 * MIGRATE_PERCENTAGE;
   ```

2. **写满检测**
   ```c
   // 当一个虚拟叶子节点的所有 8 个 4KB 页面都写满时触发
   if(file_end_byte + 1 == ORCH_BLOCK_SIZE)  // 32KB 完整块
   ```

3. **LRU 策略**
   - 使用 LRU 队列管理所有写满的虚拟叶子节点
   - 优先迁移最少使用的冷数据

---

## 三、HRtree 元数据结构

HRtree（Heterogeneous-unit Range Tree）是 OrchFS 用于管理数据布局和映射的核心索引结构。

### 3.1 HRtree 层次结构

```
Index Root (根节点)
    ↓
Non-Leaf Nodes (非叶子索引节点，可多层)
    ↓
Leaf Nodes (叶子索引节点)
    ↓
    ├─ SSD_BLOCK (对齐的大块数据)
    ├─ VIR_LEAF_NODE (NVM 小块数据)
    └─ STRATA_NODE (混合节点)
```

### 3.2 核心数据结构定义

#### 索引根节点
```c
// LibFS/index.h:62-69
struct index_root_t
{  
    int64_t ndtype;                      // 节点类型 = IDX_ROOT
    int64_t idx_inode_id;                // 对应的 inode ID
    int64_t idx_entry_blkid;             // 索引入口块 ID
    int64_t max_blk_offset;              // 最大块偏移
};
```

#### 索引节点
```c
// LibFS/index.h:51-59
struct index_node_t
{  
    int64_t ndtype;                                      // 节点类型
    int64_t zipped_layer;                                // 压缩层数
    uint64_t virnd_flag[2];                              // 子节点是否为虚拟叶子节点的位图
    uint64_t bit_lock[2];                                // 细粒度位锁
    int64_t son_blk_id[NODE_SON_CAPACITY];               // 子节点块 ID 数组
};
```

#### 虚拟节点（热数据）
```c
// LibFS/index.h:73-81
struct virtual_node_t
{  
    uint64_t ndtype;                                     // = VIR_LEAF_NODE
    uint64_t ssd_dev_addr;                               // SSD 块地址（用于迁移后）
    int64_t max_pos;                                     // 最大位置
    int64_t nvm_page_id[VLN_SLOT_SUM];                   // 8 个 4KB NVM 页面
    int64_t buf_meta_id[VLN_SLOT_SUM];                   // 对应的元数据 ID
};
```

### 3.3 HRtree 在两条路径中的作用

#### 热写路径中的 HRtree 操作
```c
// LibFS/index.c:615
int append_nvm_pages(...)
{
    // 1. 找到或创建虚拟叶子节点
    // 2. 分配 NVM 页面 ID
    // 3. 更新虚拟节点的 nvm_page_id 数组
    // 4. 在索引节点中设置 virnd_flag 位图标记
}
```

#### 冷数据整合路径中的 HRtree 操作
```c
// LibFS/index.c
void change_virnd_to_ssdblk(...)
{
    // 1. 定位到对应的索引叶子节点
    // 2. 清除 virnd_flag 位图标记
    // 3. 更新 son_blk_id 为新的 SSD 块 ID
    // 4. 节点类型从 VIR_LEAF_NODE 变为 SSD_BLOCK
    // 5. 释放虚拟节点和 NVM 页面资源
}
```

### 3.4 Alignment-based Partitioning 机制

OrchFS 通过对齐检测实现数据分区：

```c
// LibFS/lib_rwfunc.c:483-522
void append_blk(...)
{
    int64_t end_blk_pos = (file_end_byte & ((1LL<<ORCH_BLOCK_BW)-1));
    
    // 对齐判断：32KB 块边界
    if((end_blk_pos+1) % ORCH_BLOCK_SIZE != 0)
    {
        // 未对齐 → 热写路径 → NVM
        need_nvm_page = end_blk_pos / ORCH_PAGE_SIZE + 1;
        append_nvm_pages(...);
    }
    else
    {
        // 对齐 → 冷数据路径 → SSD
        need_ssd_blk = off_end - off_start + 1;
        append_ssd_blocks(...);
    }
}
```

**对齐粒度配置**
- **块大小（ORCH_BLOCK_SIZE）**: 32KB（配置于 config/config.h:43）
- **页面大小（ORCH_PAGE_SIZE）**: 4KB（配置于 config/config.h:42）
- **分割粒度（ORCH_MAX_SPLIT_BLK）**: 可配置，影响并行 I/O 粒度

---

## 四、并行 I/O 引擎

OrchFS 通过专用线程池实现高并发 I/O。

### 4.1 线程池配置

```c
// LibFS/io_thdpool.h:10-12
#define MAX_NVM_THREADS            ORCH_CONFIG_NVMTHD  // NVM 线程数
#define MAX_SSD_THREADS            ORCH_CONFIG_SSDTHD  // SSD 线程数
#define IO_TOTAL_THREADS           ORCH_CONFIG_SSDTHD
```

通过 `config_parameter.py` 脚本配置：
```bash
python config_parameter.py /dev/dax0.0 /dev/nvme1n1 4 16 32k
                          [NVM设备] [SSD设备] [NVM线程] [SSD线程] [分割大小]
```

### 4.2 I/O 任务调度

```c
// LibFS/lib_rwfunc.c:1044-1050
int ssd_task_num = count_task_num(ssd_io_task_head, ssd_io_task_end);
int task_group_id = -1;

#ifndef SSD_NOT_PARALLEL
    if(ssd_task_num > 0)
        task_group_id = add_task(ssd_io_task_head, nvm_io_task_head, 
                                 ssd_task_num, 0, fd);
    do_nvm_task(nvm_io_task_head, nvm_io_task_end);  // 执行 NVM 任务
    if(ssd_task_num > 0 && task_group_id != -1)
        wait_task_done(task_group_id);                // 等待 SSD 任务完成
#endif
```

**执行策略**：
- NVM 任务：同步执行（因为 NVM 延迟低）
- SSD 任务：异步并行执行（充分利用 SSD 带宽）

---

## 五、STRATA 混合节点

STRATA_NODE 是一种特殊的混合节点，同时在 SSD 和 NVM 上存储数据。

### 5.1 STRATA 节点用途

用于处理 SSD 块的部分更新，避免 RMW（Read-Modify-Write）：
1. SSD 上保存完整的 32KB 块数据
2. NVM 上保存被修改的 4KB 页面
3. 读取时合并两者数据

### 5.2 STRATA 节点创建

```c
// LibFS/lib_rwfunc.c:262-295
void create_strata_structure(root_id_t root_id, int64_t ino_id, 
                              off_info_pt blk_info_pt, int64_t blk_offset, 
                              int64_t start_pos, int64_t len)
{
    // 将 SSD_BLOCK 转换为 STRATA_NODE
    if(blk_info_pt->ndtype == SSD_BLOCK)
        blk_info_pt->ndtype = STRATA_NODE;
    
    // 为修改的页面分配 NVM 页面和元数据
    for(int64_t i = start_page_id; i <= end_page_id; i++)
    {
        if(blk_info_pt->nvm_page_id[i] == EMPTY_BLKID)
        {
            int64_t new_nvm_page_id = require_nvm_page_id();
            int64_t bufmeta_id = require_buffer_metadata_id();
            insert_strata_page_and_metabuf(root_id, ino_id, blk_offset, 
                                           i, new_nvm_page_id, bufmeta_id);
        }
    }
}
```

### 5.3 STRATA 节点读写

```c
// LibFS/lib_rwfunc.c:64-216
void strata_write(off_info_t blk_info, int64_t write_pos, int64_t write_len, 
                  void* write_buf, ...)
{
    // 判断是否达到阈值
    int blk_4k_num = end_page_id - start_page_id + 1;
    if(blk_4k_num >= STRATA_THRESHOLD)  // STRATA_THRESHOLD = 6
    {
        // 大部分页面被修改 → 直接写 SSD
        ssd_write_flag = 1;
    }
    else
    {
        // 少数页面修改 → 写 NVM
        // 使用元数据记录修改的段
    }
}
```

---

## 六、代码路径总结

### 6.1 热写路径完整代码路径

```
用户调用 write()
    ↓
LibFS/lib_func.c:361 - orchfs_write()
    ↓
LibFS/lib_func.c:266 - orchfs_pwrite()
    ↓
LibFS/lib_func.c:323 - write_into_file()
    ↓
LibFS/lib_rwfunc.c:928 - write_into_file()
    ↓
LibFS/lib_rwfunc.c:966 - acquire_op_blk_info()
    ↓
LibFS/lib_rwfunc.c:525 - acquire_op_blk_info()
    ↓
LibFS/lib_rwfunc.c:538 - append_blk()
    ↓
LibFS/lib_rwfunc.c:519 - append_nvm_pages()
    ↓
LibFS/index.c:615 - append_nvm_pages()
    ↓
（分配 VIR_LEAF_NODE 和 NVM 页面）
    ↓
LibFS/lib_rwfunc.c:997 - all_unaligned_op()
    ↓
LibFS/lib_rwfunc.c:414 - unaligned_nvm_op()
    ↓
LibFS/lib_rwfunc.c:336 - unaligned_nvm_op()
    ↓
（创建 NVM I/O 任务）
    ↓
LibFS/lib_rwfunc.c:1048 - do_nvm_task()
    ↓
LibFS/lib_rwfunc.c:622 - do_nvm_task()
    ↓
（执行 NVM 写入到设备）
```

### 6.2 冷数据整合路径完整代码路径

```
VIR_LEAF_NODE 写满（8个4KB页面）
    ↓
LibFS/lib_rwfunc.c:570 - add_migrate_node()
    ↓
LibFS/migrate.c:146 - add_migrate_node()
    ↓
（添加到 LRU 队列，更新 NVM 使用量）
    ↓
LibFS/migrate.c:163 - 检查 nvm_page_used > mig_threshold
    ↓
LibFS/migrate.c:168 - sem_post(&migrate_sem)
    ↓
（唤醒迁移线程）
    ↓
LibFS/migrate.c:125 - sem_wait(&migrate_sem)
    ↓
LibFS/migrate.c:116 - wait_and_exec_migrate()
    ↓
LibFS/migrate.c:136 - do_migrate_operation()
    ↓
LibFS/migrate.c:37 - do_migrate_operation()
    ↓
（执行迁移）
    1. LibFS/migrate.c:39 - get_and_eliminate_LRU_node()
    2. LibFS/migrate.c:59 - read_data_from_devs() [从 NVM 读]
    3. LibFS/migrate.c:62 - require_ssd_block_id()
    4. LibFS/migrate.c:70 - sendreq_by_shm() [写入 SSD]
    5. LibFS/migrate.c:95 - change_virnd_to_ssdblk()
    6. LibFS/migrate.c:100 - __sync_fetch_and_sub(&nvm_page_used, 8)
    ↓
LibFS/index.c - change_virnd_to_ssdblk()
    ↓
（更新 HRtree：VIR_LEAF_NODE → SSD_BLOCK）
```

---

## 七、关键配置参数

### 7.1 系统配置

```c
// config/config.h
#define ORCH_PAGE_SIZE      4096    // 4KB NVM 页面
#define ORCH_BLOCK_SIZE     32768   // 32KB SSD 块
#define VLN_SLOT_SUM        8       // 每个虚拟节点 8 个页面槽位
```

### 7.2 迁移配置

```c
// LibFS/migrate.h
#define DEFAULT_MIGRATE_NUM         1024  // 每次迁移 1024 个块
#define MIGRATE_PERCENTAGE          10    // NVM 10% 使用触发迁移
#define CAN_USE_PERCENTAGE          10    // 可用空间 10%
```

### 7.3 STRATA 配置

```c
// LibFS/lib_rwfunc.c
#define STRATA_THRESHOLD    6  // 6 个或更多页面修改时直接写 SSD
```

---

## 八、性能优化机制

### 8.1 对齐优化
- 32KB 对齐的写入直接发往 SSD
- 未对齐的写入先发往 NVM，后台批量迁移

### 8.2 并行 I/O
- 多个 SSD 线程并行执行大块 I/O
- NVM 线程独立处理小块 I/O

### 8.3 批量迁移
- 每次迁移多个块（DEFAULT_MIGRATE_NUM = 1024）
- 使用 LRU 策略选择冷数据

### 8.4 细粒度锁
- 位锁（bit_lock）减少索引竞争
- 范围锁（lock_range_lock）保护文件区域

---

## 九、总结

OrchFS 通过以下机制实现了高效的异构 I/O 编排：

1. **热写路径（Path 1）**
   - **位置**: LibFS/lib_rwfunc.c, LibFS/index.c
   - **触发**: 未对齐的小块写入
   - **目标**: NVM (VIR_LEAF_NODE)
   - **特点**: 低延迟、高并发、随机写友好

2. **冷数据整合路径（Path 2）**
   - **位置**: LibFS/migrate.c
   - **触发**: NVM 使用量超过阈值
   - **目标**: SSD (SSD_BLOCK)
   - **特点**: 批量迁移、对齐写入、充分利用 SSD 带宽

3. **HRtree 元数据结构**
   - **位置**: LibFS/index.h, LibFS/index.c
   - **功能**: 统一管理异构数据布局
   - **机制**: Alignment-based partitioning

通过这种设计，OrchFS 能够同时利用 NVM 的低延迟和 SSD 的高带宽，实现了对现代高性能存储设备的充分利用。
