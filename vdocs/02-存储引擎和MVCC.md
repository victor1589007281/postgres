# PostgreSQL 存储引擎和MVCC

## 目录
- [1. 存储引擎概述](#1-存储引擎概述)
- [2. Heap存储引擎](#2-heap存储引擎)
- [3. MVCC多版本并发控制](#3-mvcc多版本并发控制)
- [4. 元组可见性判断](#4-元组可见性判断)
- [5. VACUUM机制](#5-vacuum机制)
- [6. 索引访问方法](#6-索引访问方法)

---

## 1. 存储引擎概述

PostgreSQL采用**插件式存储引擎**架构，但默认且最主要的是**Heap堆表**存储引擎。

### 1.1 存储引擎架构

```mermaid
graph TB
    subgraph "**查询执行器**"
        style Executor fill:#bbdefb,stroke:#1565c0,stroke-width:2px
        Executor[**Executor**<br/>执行计划]
    end
    
    subgraph "**表访问方法层 Table AM**"
        style TAM fill:#c5cae9,stroke:#283593,stroke-width:2px
        TAM[**Table Access Method API**<br/>tableam.h<br/>scan/insert/update/delete]
    end
    
    subgraph "**具体存储引擎**"
        style Heap fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
        
        Heap[**Heap存储引擎**<br/>heapam.c<br/>默认引擎<br/>支持MVCC]
    end
    
    subgraph "**索引访问方法 Index AM**"
        style BTree fill:#fff9c4,stroke:#f57f17,stroke-width:2px
        style GIN fill:#fff9c4,stroke:#f57f17,stroke-width:2px
        style GiST fill:#fff9c4,stroke:#f57f17,stroke-width:2px
        style BRIN fill:#fff9c4,stroke:#f57f17,stroke-width:2px
        style Hash fill:#fff9c4,stroke:#f57f17,stroke-width:2px
        style SPGiST fill:#fff9c4,stroke:#f57f17,stroke-width:2px
        
        BTree[**B-Tree**<br/>nbtree/<br/>通用索引<br/>支持排序]
        GIN[**GIN**<br/>gin/<br/>倒排索引<br/>全文搜索]
        GiST[**GiST**<br/>gist/<br/>通用搜索树<br/>地理数据]
        BRIN[**BRIN**<br/>brin/<br/>块范围索引<br/>大表优化]
        Hash[**Hash**<br/>hash/<br/>等值查询]
        SPGiST[**SP-GiST**<br/>spgist/<br/>空间分区]
    end
    
    subgraph "**缓冲区管理层**"
        style BufferMgr fill:#ffccbc,stroke:#bf360c,stroke-width:2px
        BufferMgr[**Buffer Manager**<br/>bufmgr.c<br/>共享缓冲池<br/>页面换入换出]
    end
    
    subgraph "**存储管理器**"
        style SMGR fill:#e0e0e0,stroke:#424242,stroke-width:2px
        SMGR[**Storage Manager**<br/>smgr.c<br/>文件I/O<br/>页面读写]
    end
    
    Executor --> TAM
    TAM --> Heap
    
    Executor --> BTree
    Executor --> GIN
    Executor --> GiST
    Executor --> BRIN
    Executor --> Hash
    Executor --> SPGiST
    
    Heap --> BufferMgr
    BTree --> BufferMgr
    GIN --> BufferMgr
    GiST --> BufferMgr
    BRIN --> BufferMgr
    Hash --> BufferMgr
    SPGiST --> BufferMgr
    
    BufferMgr --> SMGR
```

---

## 2. Heap存储引擎

### 2.1 Heap表的存储结构

PostgreSQL的Heap表是一种**堆组织表**（Heap-Organized Table），数据以无序方式存储，不按主键聚簇。

#### **页面结构**

```mermaid
graph TB
    subgraph "**8KB页面结构**"
        style PageHeader fill:#bbdefb,stroke:#1565c0,stroke-width:3px
        style ItemArray fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
        style FreeSpace fill:#fff9c4,stroke:#f57f17,stroke-width:2px
        style Tuples fill:#ffccbc,stroke:#bf360c,stroke-width:2px
        style Special fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
        
        PageHeader[**PageHeaderData 24字节**<br/>━━━━━━━━━━━━━━━━<br/>pd_lsn: 8字节 WAL位置<br/>pd_checksum: 2字节<br/>pd_flags: 2字节<br/>pd_lower: 2字节 指向ItemId末尾<br/>pd_upper: 2字节 指向空闲空间末尾<br/>pd_special: 2字节<br/>pd_pagesize_version: 2字节<br/>pd_prune_xid: 4字节]
        
        ItemArray[**ItemIdData数组**<br/>━━━━━━━━━━━━━━━━<br/>每个4字节:<br/>lp_off: 15位 偏移量<br/>lp_flags: 2位 状态标志<br/>lp_len: 15位 长度<br/>━━━━━━━━━━━━━━━━<br/>从页首向后增长]
        
        FreeSpace[**空闲空间**<br/>━━━━━━━━━━━━━━━━<br/>可用于插入新元组<br/>大小=pd_upper-pd_lower]
        
        Tuples[**元组数据**<br/>━━━━━━━━━━━━━━━━<br/>HeapTupleHeader+数据<br/>从页尾向前增长<br/>━━━━━━━━━━━━━━━━<br/>t_xmin t_xmax t_ctid<br/>t_infomask t_infomask2<br/>NULL位图<br/>实际数据列]
        
        Special[**Special Space**<br/>━━━━━━━━━━━━━━━━<br/>Heap表为空<br/>索引页使用]
    end
    
    PageHeader --> ItemArray
    ItemArray --> FreeSpace
    FreeSpace --> Tuples
    Tuples --> Special
```

### 2.2 元组结构（HeapTuple）

**源码位置**: `src/include/access/htup_details.h`

```mermaid
graph TB
    subgraph "**HeapTupleHeaderData结构**"
        style Union fill:#bbdefb,stroke:#1565c0,stroke-width:3px
        style CTID fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
        style Infomask fill:#fff9c4,stroke:#f57f17,stroke-width:2px
        style Bitmap fill:#ffccbc,stroke:#bf360c,stroke-width:2px
        style Data fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
        
        Union[**t_choice联合体 12字节**<br/>━━━━━━━━━━━━━━━━<br/>HeapTupleFields:<br/>• t_xmin: 4字节 插入事务ID<br/>• t_xmax: 4字节 删除/锁定事务ID<br/>• t_cid/t_xvac: 4字节<br/>━━━━━━━━━━━━━━━━<br/>或 DatumTupleFields<br/>用于内存中的复合类型]
        
        CTID[**t_ctid ItemPointerData 6字节**<br/>━━━━━━━━━━━━━━━━<br/>block: 4字节 页面号<br/>offset: 2字节 页内偏移<br/>━━━━━━━━━━━━━━━━<br/>指向当前或新版本元组]
        
        Infomask[**标志位 5字节**<br/>━━━━━━━━━━━━━━━━<br/>t_infomask2: 2字节<br/>• 列数 11位<br/>• HOT标志位<br/>━━━━━━━━━━━━━━━━<br/>t_infomask: 2字节<br/>• HASNULL/HASVARWIDTH<br/>• XMIN/XMAX状态<br/>• 锁信息<br/>━━━━━━━━━━━━━━━━<br/>t_hoff: 1字节 头部长度]
        
        Bitmap[**NULL位图 可变长度**<br/>━━━━━━━━━━━━━━━━<br/>t_bits数组<br/>每列1bit<br/>1=NOT NULL, 0=NULL]
        
        Data[**实际数据**<br/>━━━━━━━━━━━━━━━━<br/>对齐到MAXALIGN<br/>定长列在前<br/>变长列在后]
    end
    
    Union --> CTID
    CTID --> Infomask
    Infomask --> Bitmap
    Bitmap --> Data
```

#### **关键字段说明**

| **字段** | **大小** | **说明** |
|---------|---------|---------|
| **t_xmin** | 4字节 | 插入该元组的事务ID，用于判断元组对当前事务是否可见 |
| **t_xmax** | 4字节 | 删除或锁定该元组的事务ID。0表示未删除 |
| **t_cid** | 4字节 | 命令ID（Command ID），在单个事务内区分不同SQL语句 |
| **t_ctid** | 6字节 | 指向当前版本或更新后的新版本元组位置（用于MVCC链） |
| **t_infomask** | 2字节 | 状态标志位：是否有NULL、XMIN/XMAX状态、锁信息等 |
| **t_infomask2** | 2字节 | 列数（11位）+ HOT更新标志 |
| **t_hoff** | 1字节 | 头部长度（包括NULL位图），实际数据从此偏移开始 |

### 2.3 元组的生命周期

```mermaid
sequenceDiagram
    participant T1 as **事务T1**
    participant Heap as **Heap表**
    participant Buffer as **共享缓冲区**
    participant WAL as **WAL日志**
    
    Note over T1: **INSERT操作**
    T1->>T1: **1. 分配新的XID**
    T1->>Heap: **2. 找到有空闲空间的页面**
    Heap->>Buffer: **3. 从缓冲区获取页面**
    T1->>T1: **4. 构建HeapTuple**<br/>t_xmin=当前XID<br/>t_xmax=0<br/>t_ctid=自身TID
    T1->>WAL: **5. 写WAL日志**<br/>XLOG_HEAP_INSERT
    T1->>Buffer: **6. 插入元组到页面**<br/>更新ItemId<br/>标记页面为脏
    T1->>T1: **7. COMMIT提交**
    T1->>WAL: **8. 写COMMIT日志**
    
    Note over T1: **UPDATE操作**
    T1->>Buffer: **9. 读取旧元组**
    T1->>T1: **10. 创建新版本元组**<br/>t_xmin=当前XID<br/>t_xmax=0
    T1->>Buffer: **11. 更新旧元组**<br/>设置t_xmax=当前XID<br/>设置t_ctid=新元组TID
    T1->>WAL: **12. 写WAL日志**<br/>XLOG_HEAP_UPDATE
    T1->>Buffer: **13. 插入新元组**
    
    Note over T1: **DELETE操作**
    T1->>Buffer: **14. 读取元组**
    T1->>Buffer: **15. 设置t_xmax=当前XID**
    T1->>WAL: **16. 写WAL日志**<br/>XLOG_HEAP_DELETE
    T1->>T1: **17. 元组标记为删除**<br/>物理空间仍然存在
    
    Note over Heap: **VACUUM回收**
    Heap->>Buffer: **18. 扫描死元组**
    Heap->>Buffer: **19. 移除死元组**<br/>更新FSM
    Heap->>Heap: **20. 回收空间**
```

---

## 3. MVCC多版本并发控制

PostgreSQL的MVCC实现是其核心特性之一，允许读不阻塞写、写不阻塞读。

### 3.1 MVCC原理

```mermaid
graph TB
    subgraph "**MVCC版本链**"
        style V1 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
        style V2 fill:#fff9c4,stroke:#f57f17,stroke-width:2px
        style V3 fill:#ffccbc,stroke:#bf360c,stroke-width:2px
        style V4 fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
        
        V1[**版本1 原始元组**<br/>━━━━━━━━━━━━━━━━<br/>TID: 0,1<br/>t_xmin: 100<br/>t_xmax: 200<br/>t_ctid: 0,2<br/>━━━━━━━━━━━━━━━━<br/>data: name=Alice, age=25]
        
        V2[**版本2 UPDATE by 200**<br/>━━━━━━━━━━━━━━━━<br/>TID: 0,2<br/>t_xmin: 200<br/>t_xmax: 300<br/>t_ctid: 0,3<br/>━━━━━━━━━━━━━━━━<br/>data: name=Alice, age=26]
        
        V3[**版本3 UPDATE by 300**<br/>━━━━━━━━━━━━━━━━<br/>TID: 0,3<br/>t_xmin: 300<br/>t_xmax: 0<br/>t_ctid: 0,3<br/>━━━━━━━━━━━━━━━━<br/>data: name=Alice, age=27]
        
        V4[**版本4 如果DELETE**<br/>━━━━━━━━━━━━━━━━<br/>t_xmax: 400<br/>━━━━━━━━━━━━━━━━<br/>标记为删除<br/>等待VACUUM清理]
    end
    
    V1 -->|**t_ctid指向**| V2
    V2 -->|**t_ctid指向**| V3
    V3 -.->|**如果删除**| V4
    
    subgraph "**不同事务看到的版本**"
        style Tx100 fill:#bbdefb,stroke:#1565c0,stroke-width:2px
        style Tx200 fill:#bbdefb,stroke:#1565c0,stroke-width:2px
        style Tx300 fill:#bbdefb,stroke:#1565c0,stroke-width:2px
        
        Tx100[**事务XID=150**<br/>快照: xmin=100, xmax=200<br/>━━━━━━━━━━━━━━━━<br/>看到: 版本1<br/>age=25]
        
        Tx200[**事务XID=250**<br/>快照: xmin=200, xmax=300<br/>━━━━━━━━━━━━━━━━<br/>看到: 版本2<br/>age=26]
        
        Tx300[**事务XID=350**<br/>快照: xmin=300, xmax=400<br/>━━━━━━━━━━━━━━━━<br/>看到: 版本3<br/>age=27]
    end
    
    V1 -.->|**可见**| Tx100
    V2 -.->|**可见**| Tx200
    V3 -.->|**可见**| Tx300
```

### 3.2 事务快照（Snapshot）

每个事务开始时获取一个快照，记录当前活跃事务的状态。

**源码位置**: `src/include/utils/snapshot.h`

```c
typedef struct SnapshotData {
    TransactionId xmin;    // 最小活跃事务ID，小于它的都已提交或回滚
    TransactionId xmax;    // 下一个要分配的事务ID
    TransactionId *xip;    // 活跃事务ID数组
    uint32 xcnt;           // 活跃事务数量
    // ...
} SnapshotData;
```

### 3.3 可见性判断流程

```mermaid
graph TB
    Start[**开始判断元组可见性**] --> CheckXmin{**检查t_xmin**}
    
    CheckXmin -->|**t_xmin > xmax**| Invisible1[**不可见**<br/>插入事务在快照后]
    CheckXmin -->|**t_xmin in xip**| Invisible2[**不可见**<br/>插入事务未提交]
    CheckXmin -->|**t_xmin已中止**| Invisible3[**不可见**<br/>插入事务已回滚]
    CheckXmin -->|**t_xmin已提交**| CheckXmax{**检查t_xmax**}
    
    CheckXmax -->|**t_xmax = 0**| Visible1[**可见**<br/>未被删除]
    CheckXmax -->|**t_xmax > xmax**| Visible2[**可见**<br/>删除事务在快照后]
    CheckXmax -->|**t_xmax in xip**| Visible3[**可见**<br/>删除事务未提交]
    CheckXmax -->|**t_xmax已中止**| Visible4[**可见**<br/>删除事务已回滚]
    CheckXmax -->|**t_xmax已提交**| Invisible4[**不可见**<br/>已被删除]
    
    style Start fill:#bbdefb,stroke:#1565c0,stroke-width:2px
    style CheckXmin fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style CheckXmax fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style Visible1 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style Visible2 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style Visible3 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style Visible4 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style Invisible1 fill:#ffccbc,stroke:#bf360c,stroke-width:2px
    style Invisible2 fill:#ffccbc,stroke:#bf360c,stroke-width:2px
    style Invisible3 fill:#ffccbc,stroke:#bf360c,stroke-width:2px
    style Invisible4 fill:#ffccbc,stroke:#bf360c,stroke-width:2px
```

**关键函数**: `HeapTupleSatisfiesMVCC()` 在 `src/backend/access/heap/heapam_visibility.c`

---

## 4. 元组可见性判断

### 4.1 CLOG（提交日志）

PostgreSQL使用CLOG来记录事务的提交状态。

```mermaid
graph LR
    subgraph "**CLOG结构**"
        style CLOG fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
        
        CLOG[**CLOG pg_xact/**<br/>━━━━━━━━━━━━━━━━<br/>每个事务占2位:<br/>00 = IN_PROGRESS<br/>01 = COMMITTED<br/>10 = ABORTED<br/>11 = SUB_COMMITTED<br/>━━━━━━━━━━━━━━━━<br/>8KB页面=32K事务]
    end
    
    subgraph "**CLOG缓冲区**"
        style Buffer fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
        Buffer[**SLRU Buffers**<br/>Simple LRU缓冲<br/>默认128个8KB页<br/>缓存最近事务状态]
    end
    
    subgraph "**事务状态查询**"
        style Query fill:#fff9c4,stroke:#f57f17,stroke-width:2px
        Query[**TransactionIdGetStatus**<br/>输入: XID<br/>输出: 事务状态<br/>━━━━━━━━━━━━━━━━<br/>1. 计算页号=XID/32K<br/>2. 计算页内偏移<br/>3. 从缓冲/磁盘读取<br/>4. 提取2位状态]
    end
    
    Query -->|**查询**| Buffer
    Buffer <-->|**缺页**| CLOG
```

### 4.2 可见性规则矩阵

| **元组状态** | **t_xmin状态** | **t_xmax状态** | **是否可见** | **说明** |
|-------------|---------------|---------------|-------------|---------|
| **新插入** | IN_PROGRESS（当前事务） | 0 | 是（当前事务）<br/>否（其他事务） | 自己插入的可见 |
| **已提交** | COMMITTED（<xmin） | 0 | 是 | 正常可见 |
| **已删除** | COMMITTED（<xmin） | COMMITTED（<xmin） | 否 | 已删除不可见 |
| **正在删除** | COMMITTED（<xmin） | IN_PROGRESS | 是（非删除事务）<br/>否（删除事务） | 删除未提交 |
| **删除已回滚** | COMMITTED（<xmin） | ABORTED | 是 | 删除失败，仍可见 |
| **未来版本** | >xmax | - | 否 | 快照后创建的 |

---

## 5. VACUUM机制

由于MVCC会产生大量过时元组（死元组），PostgreSQL需要VACUUM来回收空间。

### 5.1 VACUUM工作原理

```mermaid
sequenceDiagram
    participant AV as **AutoVacuum进程**
    participant Table as **表页面**
    participant Index as **索引**
    participant FSM as **FSM空闲空间映射**
    participant VM as **VM可见性映射**
    
    AV->>Table: **1. 扫描表页面**
    Table-->>AV: **2. 识别死元组**<br/>t_xmax已提交且<br/>所有活跃事务都不可见
    
    AV->>AV: **3. 收集死元组TID列表**<br/>使用maintenance_work_mem
    
    AV->>Index: **4. 扫描索引**<br/>删除指向死元组的索引项
    Index-->>AV: **5. 清理完成**
    
    AV->>Table: **6. 回收死元组空间**<br/>压缩页面<br/>更新ItemId
    
    AV->>FSM: **7. 更新空闲空间映射**<br/>记录每页空闲字节数
    
    AV->>VM: **8. 更新可见性映射**<br/>标记all-visible页面
    
    AV->>AV: **9. 更新统计信息**<br/>pg_stat_all_tables
    
    Note over AV,VM: **如果需要FREEZE**
    AV->>Table: **10. 冻结老旧元组**<br/>t_xmin替换为FrozenXID<br/>防止事务ID回卷
```

### 5.2 VACUUM类型

```mermaid
graph TB
    subgraph "**VACUUM操作类型**"
        style Normal fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
        style Full fill:#ffccbc,stroke:#bf360c,stroke-width:3px
        style Auto fill:#fff9c4,stroke:#f57f17,stroke-width:3px
        style Freeze fill:#e1bee7,stroke:#6a1b9a,stroke-width:3px
        
        Normal[**普通VACUUM**<br/>━━━━━━━━━━━━━━━━<br/>• 标记死元组为可重用<br/>• 不返还空间给OS<br/>• 不锁表，可并发<br/>• 更新FSM和VM<br/>━━━━━━━━━━━━━━━━<br/>命令: VACUUM table;]
        
        Full[**VACUUM FULL**<br/>━━━━━━━━━━━━━━━━<br/>• 重写整个表<br/>• 返还空间给OS<br/>• 需要AccessExclusive锁<br/>• 阻塞所有操作<br/>━━━━━━━━━━━━━━━━<br/>命令: VACUUM FULL table;]
        
        Auto[**AutoVacuum**<br/>━━━━━━━━━━━━━━━━<br/>• 自动触发<br/>• 后台工作进程<br/>• 基于统计信息<br/>━━━━━━━━━━━━━━━━<br/>触发条件:<br/>死元组数 > threshold +<br/>scale_factor × 表行数]
        
        Freeze[**FREEZE冻结**<br/>━━━━━━━━━━━━━━━━<br/>• 防止XID回卷<br/>• t_xmin替换为2<br/>• 超过vacuum_freeze_min_age<br/>━━━━━━━━━━━━━━━━<br/>强制: vacuum_freeze_table_age]
    end
```

### 5.3 HOT（Heap-Only Tuple）优化

当UPDATE不修改索引列时，PostgreSQL使用HOT优化避免更新索引。

```mermaid
graph LR
    subgraph "**HOT更新链**"
        style Old fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
        style New fill:#fff9c4,stroke:#f57f17,stroke-width:2px
        style Index fill:#ffccbc,stroke:#bf360c,stroke-width:2px
        
        Old[**旧版本元组**<br/>━━━━━━━━━━━━━━━━<br/>TID: 0,1<br/>t_xmax: 200<br/>t_ctid: 0,5<br/>data: id=1, name=Alice<br/>age=25]
        
        New[**新版本元组**<br/>━━━━━━━━━━━━━━━━<br/>TID: 0,5<br/>t_xmin: 200<br/>t_xmax: 0<br/>━━━━━━━━━━━━━━━━<br/>data: id=1, name=Alice<br/>age=26<br/>━━━━━━━━━━━━━━━━<br/>t_infomask2设置HEAP_HOT_UPDATED]
        
        Index[**索引不变**<br/>━━━━━━━━━━━━━━━━<br/>id=1 → TID 0,1<br/>━━━━━━━━━━━━━━━━<br/>索引仍指向旧版本<br/>通过t_ctid链找到新版本]
    end
    
    Old -->|**同页HOT链**| New
    Index -.->|**索引指向**| Old
```

**HOT的优点**:
- 避免索引膨胀
- 减少索引更新开销
- 加快UPDATE性能

**HOT的条件**:
- 新旧版本在同一页面内
- 页面有足够空闲空间
- 没有更新索引列

---

## 6. 索引访问方法

PostgreSQL支持多种索引类型，每种适用于不同场景。

### 6.1 索引类型对比

```mermaid
graph TB
    subgraph "**PostgreSQL索引类型**"
        subgraph "**B-Tree索引**"
            style BT fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
            BT[**B-Tree**<br/>━━━━━━━━━━━━━━━━<br/>• 最常用的索引<br/>• 支持<、<=、=、>=、>、BETWEEN<br/>• 支持ORDER BY<br/>• 支持UNIQUE约束<br/>━━━━━━━━━━━━━━━━<br/>使用场景:<br/>通用查询、排序、范围查询<br/>━━━━━━━━━━━━━━━━<br/>实现: src/backend/access/nbtree/]
        end
        
        subgraph "**GIN索引**"
            style GIN fill:#fff9c4,stroke:#f57f17,stroke-width:3px
            GIN[**GIN倒排索引**<br/>━━━━━━━━━━━━━━━━<br/>• Generalized Inverted Index<br/>• 一个值对应多个行<br/>• 支持数组、全文搜索、jsonb<br/>━━━━━━━━━━━━━━━━<br/>使用场景:<br/>全文搜索、数组包含查询<br/>jsonb键值查询<br/>━━━━━━━━━━━━━━━━<br/>操作符: @>、<@、&&、@?]
        end
        
        subgraph "**GiST索引**"
            style GIST fill:#ffccbc,stroke:#bf360c,stroke-width:3px
            GIST[**GiST通用搜索树**<br/>━━━━━━━━━━━━━━━━<br/>• Generalized Search Tree<br/>• 平衡树结构<br/>• 支持空间数据、范围类型<br/>━━━━━━━━━━━━━━━━<br/>使用场景:<br/>地理位置查询PostGIS<br/>范围类型重叠查询<br/>━━━━━━━━━━━━━━━━<br/>操作符: <<、>>、&&、@>]
        end
        
        subgraph "**BRIN索引**"
            style BRIN fill:#e1bee7,stroke:#6a1b9a,stroke-width:3px
            BRIN[**BRIN块范围索引**<br/>━━━━━━━━━━━━━━━━<br/>• Block Range Index<br/>• 存储块范围的最小/最大值<br/>• 索引体积极小<br/>━━━━━━━━━━━━━━━━<br/>使用场景:<br/>大表时间序列数据<br/>物理顺序与逻辑顺序一致<br/>━━━━━━━━━━━━━━━━<br/>适用: 日志表、时间戳列]
        end
        
        subgraph "**Hash索引**"
            style HASH fill:#b3e5fc,stroke:#0277bd,stroke-width:3px
            HASH[**Hash索引**<br/>━━━━━━━━━━━━━━━━<br/>• 哈希表实现<br/>• 仅支持等值查询<br/>• 不支持排序<br/>━━━━━━━━━━━━━━━━<br/>使用场景:<br/>精确匹配查询<br/>不需要范围查询<br/>━━━━━━━━━━━━━━━━<br/>操作符: =]
        end
        
        subgraph "**SP-GiST索引**"
            style SPGIST fill:#dcedc8,stroke:#558b2f,stroke-width:3px
            SPGIST[**SP-GiST空间分区**<br/>━━━━━━━━━━━━━━━━<br/>• Space-Partitioned GiST<br/>• 非平衡树<br/>• 支持四叉树、k-d树<br/>━━━━━━━━━━━━━━━━<br/>使用场景:<br/>电话号码、IP地址<br/>空间分区数据]
        end
    end
```

### 6.2 索引选择指南

| **场景** | **推荐索引** | **示例** |
|---------|-------------|---------|
| 通用查询、排序 | **B-Tree** | `CREATE INDEX ON users(name);` |
| 全文搜索 | **GIN** | `CREATE INDEX ON docs USING GIN(to_tsvector('english', content));` |
| 数组查询 | **GIN** | `CREATE INDEX ON posts USING GIN(tags);` |
| JSONB查询 | **GIN** | `CREATE INDEX ON data USING GIN(json_col);` |
| 地理位置 | **GiST** | `CREATE INDEX ON places USING GIST(location);` |
| 大表时间序列 | **BRIN** | `CREATE INDEX ON logs USING BRIN(created_at);` |
| 仅等值查询 | **Hash** | `CREATE INDEX ON cache USING HASH(key);` |

---

## 总结

PostgreSQL的存储引擎和MVCC机制是其高性能和高并发的基础：

1. **Heap存储**: 堆组织表，页面+元组结构，支持高效的顺序扫描
2. **MVCC**: 通过元组多版本实现读写不阻塞，每个事务看到一致的快照
3. **可见性判断**: 基于t_xmin、t_xmax和事务快照判断元组是否可见
4. **VACUUM**: 回收死元组空间，防止表膨胀和事务ID回卷
5. **多种索引**: B-Tree、GIN、GiST、BRIN等满足不同查询需求

**关键优势**:
- **并发性高**: 读不阻塞写，写不阻塞读
- **一致性强**: 每个事务看到一致的数据快照
- **灵活性强**: 支持多种索引类型和访问方法

**代价**:
- **空间开销**: 需要存储多版本元组
- **维护成本**: 需要定期VACUUM清理死元组

---

**相关文档**:
- [01-架构总览](./01-架构总览.md)
- [03-事务系统](./03-事务系统.md)
- [04-WAL和崩溃恢复](./04-WAL和崩溃恢复.md)

