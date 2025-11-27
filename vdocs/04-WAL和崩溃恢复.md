# PostgreSQL WAL和崩溃恢复

## 目录
- [1. WAL概述](#1-wal概述)
- [2. WAL结构](#2-wal结构)
- [3. WAL写入流程](#3-wal写入流程)
- [4. 检查点机制](#4-检查点机制)
- [5. 崩溃恢复](#5-崩溃恢复)
- [6. WAL归档](#6-wal归档)

---

## 1. WAL概述

**WAL（Write-Ahead Logging，预写日志）**是PostgreSQL实现ACID中持久性（Durability）的核心机制。

### 1.1 WAL基本原理

```mermaid
graph TB
    subgraph "**WAL核心原则**"
        style Rule1 fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
        style Rule2 fill:#fff9c4,stroke:#f57f17,stroke-width:3px
        style Rule3 fill:#ffccbc,stroke:#bf360c,stroke-width:3px
        
        Rule1[**原则1: 先写日志**<br/>━━━━━━━━━━━━━━━━<br/>数据页修改前<br/>必须先写WAL日志<br/>━━━━━━━━━━━━━━━━<br/>保证崩溃后可以重做<br/>Redo恢复]
        
        Rule2[**原则2: LSN递增**<br/>━━━━━━━━━━━━━━━━<br/>每个WAL记录有唯一LSN<br/>Log Sequence Number<br/>━━━━━━━━━━━━━━━━<br/>LSN单调递增<br/>标识WAL位置]
        
        Rule3[**原则3: 页面LSN**<br/>━━━━━━━━━━━━━━━━<br/>每个数据页记录pd_lsn<br/>最后修改该页的WAL位置<br/>━━━━━━━━━━━━━━━━<br/>恢复时判断是否需要重做<br/>如果页面LSN ≥ WAL LSN则跳过]
    end
    
    Rule1 --> Rule2
    Rule2 --> Rule3
```

### 1.2 WAL的作用

```mermaid
graph LR
    subgraph "**WAL的核心作用**"
        style Crash fill:#bbdefb,stroke:#1565c0,stroke-width:3px
        style Performance fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
        style Replication fill:#fff9c4,stroke:#f57f17,stroke-width:3px
        style PITR fill:#ffccbc,stroke:#bf360c,stroke-width:3px
        
        Crash[**1. 崩溃恢复**<br/>━━━━━━━━━━━━━━━━<br/>系统崩溃后<br/>通过WAL重做已提交事务<br/>回滚未提交事务<br/>━━━━━━━━━━━━━━━━<br/>保证数据一致性<br/>不丢失已提交数据]
        
        Performance[**2. 提升性能**<br/>━━━━━━━━━━━━━━━━<br/>顺序写WAL日志<br/>比随机写数据页快<br/>━━━━━━━━━━━━━━━━<br/>延迟刷写数据页<br/>批量写入降低IO<br/>━━━━━━━━━━━━━━━━<br/>提交只需fsync WAL]
        
        Replication[**3. 流复制基础**<br/>━━━━━━━━━━━━━━━━<br/>主库发送WAL给备库<br/>备库应用WAL重放<br/>━━━━━━━━━━━━━━━━<br/>实现主备同步<br/>高可用架构]
        
        PITR[**4. PITR时间点恢复**<br/>━━━━━━━━━━━━━━━━<br/>归档WAL日志<br/>基础备份+WAL归档<br/>━━━━━━━━━━━━━━━━<br/>恢复到任意时间点<br/>误操作恢复]
    end
    
    Crash --> Performance
    Performance --> Replication
    Replication --> PITR
```

---

## 2. WAL结构

### 2.1 WAL文件组织

```mermaid
graph TB
    subgraph "**WAL目录结构**"
        style PGWAL fill:#e3f2fd,stroke:#1565c0,stroke-width:3px
        
        PGWAL[**pg_wal/ 目录**<br/>━━━━━━━━━━━━━━━━<br/>存储WAL段文件]
        
        subgraph "**WAL段文件**"
            style Seg1 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
            style Seg2 fill:#fff9c4,stroke:#f57f17,stroke-width:2px
            style Seg3 fill:#ffccbc,stroke:#bf360c,stroke-width:2px
            
            Seg1[**000000010000000000000001**<br/>16MB WAL段文件<br/>━━━━━━━━━━━━━━━━<br/>时间线 Timeline: 1<br/>逻辑文件号: 0<br/>段号: 1]
            
            Seg2[**000000010000000000000002**<br/>16MB WAL段文件<br/>━━━━━━━━━━━━━━━━<br/>时间线 Timeline: 1<br/>逻辑文件号: 0<br/>段号: 2]
            
            Seg3[**000000010000000000000003**<br/>16MB WAL段文件<br/>━━━━━━━━━━━━━━━━<br/>时间线 Timeline: 1<br/>逻辑文件号: 0<br/>段号: 3]
        end
        
        subgraph "**WAL归档**"
            style Archive fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
            Archive[**archive_status/**<br/>━━━━━━━━━━━━━━━━<br/>.ready 文件: 待归档<br/>.done 文件: 已归档]
        end
        
        PGWAL --> Seg1
        PGWAL --> Seg2
        PGWAL --> Seg3
        PGWAL --> Archive
    end
```

**文件命名规则**:
- 文件名长度: 24个十六进制字符
- 格式: `TTTTTTTLLLLLLLLSSSSSSSS`
  - `TTTTTTTT`: 8位时间线ID（Timeline）
  - `LLLLLLLL`: 8位逻辑文件号
  - `SSSSSSSS`: 8位段号
- 示例: `000000010000000000000001`
  - Timeline = 1
  - 逻辑文件号 = 0
  - 段号 = 1

### 2.2 WAL记录结构

```mermaid
graph TB
    subgraph "**单条WAL记录 XLogRecord**"
        style Header fill:#bbdefb,stroke:#1565c0,stroke-width:3px
        style Data fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
        style Block fill:#fff9c4,stroke:#f57f17,stroke-width:3px
        
        Header[**XLogRecord头部**<br/>━━━━━━━━━━━━━━━━<br/>xl_tot_len: 记录总长度<br/>xl_xid: 事务ID<br/>xl_prev: 前一条记录LSN<br/>xl_rmid: 资源管理器ID<br/>xl_info: 操作类型<br/>━━━━━━━━━━━━━━━━<br/>xl_crc: CRC校验和]
        
        Data[**主数据 Main Data**<br/>━━━━━━━━━━━━━━━━<br/>操作相关的核心数据<br/>如: xl_heap_insert结构<br/>━━━━━━━━━━━━━━━━<br/>新元组数据]
        
        Block[**块引用 Block References**<br/>━━━━━━━━━━━━━━━━<br/>涉及的数据页信息<br/>━━━━━━━━━━━━━━━━<br/>• 表空间OID<br/>• 数据库OID<br/>• 关系OID<br/>• 块号<br/>━━━━━━━━━━━━━━━━<br/>Full Page Image<br/>首次修改后完整页面]
    end
    
    Header --> Data
    Data --> Block
```

### 2.3 LSN结构

```mermaid
graph LR
    subgraph "**LSN Log Sequence Number**"
        style LSN fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
        
        LSN[**LSN: 64位整数**<br/>━━━━━━━━━━━━━━━━<br/>高32位: WAL段文件号<br/>低32位: 段内偏移量<br/>━━━━━━━━━━━━━━━━<br/>示例: 0/1A000028<br/>━━━━━━━━━━━━━━━━<br/>文件号: 0<br/>偏移: 0x1A000028<br/>━━━━━━━━━━━━━━━━<br/>对应文件:<br/>000000010000000000000001<br/>偏移: 436207656字节]
    end
```

---

## 3. WAL写入流程

### 3.1 完整的WAL写入过程

```mermaid
sequenceDiagram
    participant Backend as **Backend进程**
    participant WALInsert as **WAL Insert**
    participant WALBuffer as **WAL Buffers**
    participant WALWriter as **WAL Writer**
    participant Disk as **磁盘pg_wal/**
    
    Backend->>Backend: **修改数据页**
    Backend->>WALInsert: **XLogInsert构建WAL记录**
    WALInsert->>WALInsert: **分配LSN**
    WALInsert->>WALInsert: **保留WAL空间**<br/>WAL_INSERT_LOCK
    
    WALInsert->>WALBuffer: **复制WAL记录到缓冲区**
    WALInsert-->>Backend: **返回LSN**
    
    Backend->>Backend: **在数据页设置pd_lsn**
    Backend->>Backend: **标记缓冲区页为脏**
    
    Note over Backend,Disk: **异步写入WAL**
    WALWriter->>WALBuffer: **定期刷写**<br/>wal_writer_delay 200ms
    WALBuffer->>Disk: **write系统调用**
    
    Note over Backend,Disk: **事务提交时同步**
    Backend->>Backend: **COMMIT**
    Backend->>WALBuffer: **XLogFlush刷写到LSN**
    WALBuffer->>Disk: **fsync确保持久化**
    Disk-->>Backend: **fsync完成**
    Backend->>Backend: **COMMIT成功**
```

### 3.2 WAL Buffers管理

```mermaid
graph TB
    subgraph "**WAL缓冲区**"
        style Buffer fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
        
        Buffer[**WAL Buffers**<br/>━━━━━━━━━━━━━━━━<br/>共享内存中的环形缓冲区<br/>默认大小: 16MB<br/>配置参数: wal_buffers<br/>━━━━━━━━━━━━━━━━<br/>多个8KB页面<br/>━━━━━━━━━━━━━━━━<br/>LSN范围: [写入位置, 刷盘位置]]
        
        subgraph "**缓冲区状态**"
            style Insert fill:#bbdefb,stroke:#1565c0,stroke-width:2px
            style Write fill:#fff9c4,stroke:#f57f17,stroke-width:2px
            style Flush fill:#ffccbc,stroke:#bf360c,stroke-width:2px
            
            Insert[**Insert Pointer**<br/>当前插入位置]
            Write[**Write Pointer**<br/>已写入磁盘位置]
            Flush[**Flush Pointer**<br/>已fsync位置]
        end
        
        Buffer --> Insert
        Buffer --> Write
        Buffer --> Flush
    end
    
    Insert -.->|**write**| Write
    Write -.->|**fsync**| Flush
```

### 3.3 Full Page Write（FPW）

```mermaid
graph TB
    Start[**修改数据页**] --> Check{**检查点后首次修改？**}
    
    Check -->|**是**| FPW[**写入完整页面镜像**<br/>━━━━━━━━━━━━━━━━<br/>WAL中包含整个8KB页<br/>保证页面完整性]
    Check -->|**否**| Delta[**只写增量数据**<br/>━━━━━━━━━━━━━━━━<br/>WAL中只记录修改部分<br/>节省空间]
    
    FPW --> Record[**写入WAL记录**]
    Delta --> Record
    
    Record --> Done[**完成**]
    
    style Start fill:#bbdefb,stroke:#1565c0,stroke-width:2px
    style Check fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style FPW fill:#ffccbc,stroke:#bf360c,stroke-width:2px
    style Delta fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style Record fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
    style Done fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
```

**FPW的作用**:
- 防止部分写（torn page）问题
- 操作系统页面大小（通常4KB）小于PostgreSQL页面（8KB）
- 崩溃时可能只写入页面的一部分
- FPW保证页面数据完整

**配置参数**:
- `full_page_writes`: 是否启用FPW（默认on，强烈建议开启）
- `wal_compression`: 压缩FPW数据（默认off）

---

## 4. 检查点机制

检查点（Checkpoint）是WAL管理的关键机制，定期将脏页刷写到磁盘，缩短恢复时间。

### 4.1 检查点流程

```mermaid
sequenceDiagram
    participant Checkpointer as **Checkpointer进程**
    participant SharedBuffer as **共享缓冲区**
    participant WAL as **WAL日志**
    participant DataFiles as **数据文件**
    participant ControlFile as **pg_control**
    
    Note over Checkpointer: **触发检查点**<br/>超时或WAL大小
    
    Checkpointer->>WAL: **写CHECKPOINT_START日志**
    Checkpointer->>Checkpointer: **记录Redo Point**<br/>检查点开始的LSN
    
    Checkpointer->>SharedBuffer: **扫描所有脏页**
    
    loop **批量刷写脏页**
        SharedBuffer->>DataFiles: **写入脏页到磁盘**<br/>限速避免IO峰值<br/>checkpoint_completion_target
    end
    
    Checkpointer->>DataFiles: **fsync所有数据文件**
    
    Checkpointer->>WAL: **写CHECKPOINT_COMPLETE日志**
    
    Checkpointer->>ControlFile: **更新pg_control**<br/>记录Redo Point LSN<br/>fsync pg_control
    
    Checkpointer->>WAL: **回收旧WAL文件**<br/>Redo Point之前的可删除
    
    Note over Checkpointer: **检查点完成**
```

### 4.2 检查点触发条件

```mermaid
graph TB
    subgraph "**检查点触发机制**"
        style Time fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
        style WALSize fill:#fff9c4,stroke:#f57f17,stroke-width:3px
        style Manual fill:#ffccbc,stroke:#bf360c,stroke-width:3px
        style Shutdown fill:#e1bee7,stroke:#6a1b9a,stroke-width:3px
        
        Time[**1. 超时触发**<br/>━━━━━━━━━━━━━━━━<br/>checkpoint_timeout<br/>默认: 5分钟<br/>━━━━━━━━━━━━━━━━<br/>定期执行检查点<br/>避免WAL积累过多]
        
        WALSize[**2. WAL大小触发**<br/>━━━━━━━━━━━━━━━━<br/>max_wal_size<br/>默认: 1GB<br/>━━━━━━━━━━━━━━━━<br/>WAL文件超过阈值<br/>立即执行检查点]
        
        Manual[**3. 手动触发**<br/>━━━━━━━━━━━━━━━━<br/>CHECKPOINT命令<br/>━━━━━━━━━━━━━━━━<br/>管理员主动执行<br/>阻塞直到完成]
        
        Shutdown[**4. 关闭触发**<br/>━━━━━━━━━━━━━━━━<br/>数据库正常关闭<br/>pg_ctl stop<br/>━━━━━━━━━━━━━━━━<br/>智能关闭执行检查点<br/>快速启动无需恢复]
    end
```

### 4.3 检查点调优

```mermaid
graph LR
    subgraph "**检查点性能优化**"
        style Spread fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
        style Buffers fill:#fff9c4,stroke:#f57f17,stroke-width:2px
        style WAL fill:#ffccbc,stroke:#bf360c,stroke-width:2px
        
        Spread[**分散IO checkpoint_completion_target**<br/>━━━━━━━━━━━━━━━━<br/>默认: 0.9<br/>━━━━━━━━━━━━━━━━<br/>检查点在timeout的90%时间内完成<br/>避免IO峰值<br/>━━━━━━━━━━━━━━━━<br/>值越大，IO越平滑<br/>但恢复时间越长]
        
        Buffers[**增大缓冲区 shared_buffers**<br/>━━━━━━━━━━━━━━━━<br/>建议: 系统内存的25%<br/>━━━━━━━━━━━━━━━━<br/>减少磁盘IO<br/>提高缓存命中率<br/>━━━━━━━━━━━━━━━━<br/>但检查点时间更长]
        
        WAL[**增大WAL限制 max_wal_size**<br/>━━━━━━━━━━━━━━━━<br/>默认: 1GB<br/>建议: 5-10GB<br/>━━━━━━━━━━━━━━━━<br/>减少检查点频率<br/>但恢复时间更长<br/>━━━━━━━━━━━━━━━━<br/>需要更多磁盘空间]
    end
```

---

## 5. 崩溃恢复

### 5.1 崩溃恢复流程

```mermaid
sequenceDiagram
    participant Startup as **Startup进程**
    participant ControlFile as **pg_control**
    participant WAL as **WAL文件**
    participant Buffer as **共享缓冲区**
    participant DataFiles as **数据文件**
    
    Note over Startup: **数据库启动**
    
    Startup->>ControlFile: **读取pg_control**
    ControlFile-->>Startup: **获取Redo Point LSN**<br/>最后检查点的位置
    
    Startup->>ControlFile: **检查状态**
    
    alt **正常关闭**
        ControlFile-->>Startup: **DB_SHUTDOWNED**
        Startup->>Startup: **无需恢复，直接启动**
    else **异常关闭**
        ControlFile-->>Startup: **DB_IN_PRODUCTION**<br/>需要恢复
        
        Startup->>WAL: **打开WAL文件**<br/>从Redo Point开始
        
        loop **重放WAL记录**
            WAL-->>Startup: **读取WAL记录**
            Startup->>Startup: **检查LSN**
            
            Startup->>Buffer: **读取数据页**
            Buffer->>DataFiles: **缺页时从磁盘读取**
            
            Startup->>Startup: **比较页面pd_lsn vs WAL LSN**
            
            alt **pd_lsn < WAL LSN**
                Startup->>Buffer: **重做WAL记录**<br/>应用修改到页面
                Startup->>Buffer: **更新pd_lsn**
            else **pd_lsn >= WAL LSN**
                Startup->>Startup: **跳过此记录**<br/>页面已更新
            end
        end
        
        Startup->>Startup: **恢复完成**
        Startup->>ControlFile: **更新状态**<br/>DB_SHUTDOWNED
    end
    
    Note over Startup: **数据库进入正常运行**
```

### 5.2 恢复模式

```mermaid
graph TB
    subgraph "**恢复类型**"
        style Crash fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
        style Archive fill:#fff9c4,stroke:#f57f17,stroke-width:3px
        style Standby fill:#ffccbc,stroke:#bf360c,stroke-width:3px
        
        Crash[**崩溃恢复 Crash Recovery**<br/>━━━━━━━━━━━━━━━━<br/>数据库异常关闭后启动<br/>━━━━━━━━━━━━━━━━<br/>• 从最后检查点开始<br/>• 重放pg_wal/中的WAL<br/>• 恢复到一致状态<br/>━━━━━━━━━━━━━━━━<br/>自动执行，无需配置]
        
        Archive[**归档恢复 Archive Recovery**<br/>━━━━━━━━━━━━━━━━<br/>从基础备份+归档WAL恢复<br/>━━━━━━━━━━━━━━━━<br/>• 恢复基础备份<br/>• 应用归档WAL<br/>• PITR时间点恢复<br/>━━━━━━━━━━━━━━━━<br/>recovery_target_*参数<br/>━━━━━━━━━━━━━━━━<br/>恢复后变为正常运行]
        
        Standby[**备库恢复 Standby Recovery**<br/>━━━━━━━━━━━━━━━━<br/>持续从主库接收WAL<br/>━━━━━━━━━━━━━━━━<br/>• 流复制接收WAL<br/>• 持续应用WAL<br/>• 可提供只读查询<br/>━━━━━━━━━━━━━━━━<br/>Hot Standby模式<br/>━━━━━━━━━━━━━━━━<br/>一直处于恢复状态]
    end
    
    Crash --> Archive
    Archive --> Standby
```

### 5.3 Redo操作类型

PostgreSQL使用**资源管理器（Resource Manager）**来处理不同类型的WAL记录。

```mermaid
graph TB
    subgraph "**资源管理器 RM**"
        style Heap fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
        style Btree fill:#fff9c4,stroke:#f57f17,stroke-width:2px
        style Xact fill:#ffccbc,stroke:#bf360c,stroke-width:2px
        style CLOG fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
        
        Heap[**RM_HEAP_ID**<br/>━━━━━━━━━━━━━━━━<br/>堆表操作<br/>━━━━━━━━━━━━━━━━<br/>• INSERT<br/>• DELETE<br/>• UPDATE<br/>• HOT_UPDATE]
        
        Btree[**RM_BTREE_ID**<br/>━━━━━━━━━━━━━━━━<br/>B-Tree索引操作<br/>━━━━━━━━━━━━━━━━<br/>• INSERT<br/>• DELETE<br/>• SPLIT页面分裂<br/>• VACUUM]
        
        Xact[**RM_XACT_ID**<br/>━━━━━━━━━━━━━━━━<br/>事务操作<br/>━━━━━━━━━━━━━━━━<br/>• COMMIT<br/>• ABORT<br/>• PREPARE 2PC]
        
        CLOG[**RM_CLOG_ID**<br/>━━━━━━━━━━━━━━━━<br/>CLOG操作<br/>━━━━━━━━━━━━━━━━<br/>• 零页初始化<br/>• 截断]
    end
```

**常见资源管理器**:
| **RM ID** | **名称** | **用途** |
|----------|---------|---------|
| 0 | XLOG | WAL内部操作（检查点等） |
| 10 | Heap | 堆表操作 |
| 11 | Heap2 | 堆表其他操作（VACUUM等） |
| 2 | Btree | B-Tree索引 |
| 12 | Xact | 事务提交/回滚 |
| 0 | CLOG | 事务状态日志 |

---

## 6. WAL归档

WAL归档是实现PITR（Point-In-Time Recovery）和备份的基础。

### 6.1 WAL归档流程

```mermaid
sequenceDiagram
    participant Backend as **Backend进程**
    participant WAL as **pg_wal/**
    participant Archiver as **Archiver进程**
    participant Archive as **归档存储**
    
    Backend->>WAL: **写入WAL记录**
    
    Note over WAL: **WAL段文件写满16MB**
    WAL->>WAL: **切换到新段文件**
    WAL->>WAL: **创建.ready文件**<br/>archive_status/段名.ready
    
    Archiver->>WAL: **定期扫描.ready文件**<br/>archive_timeout默认0秒
    
    Archiver->>WAL: **读取待归档段文件**
    
    Archiver->>Archive: **执行archive_command**<br/>复制到归档位置
    
    Archive-->>Archiver: **归档成功**
    
    Archiver->>WAL: **删除.ready文件**
    Archiver->>WAL: **创建.done文件**<br/>archive_status/段名.done
    
    Note over WAL: **可以回收.done的WAL文件**
```

### 6.2 配置WAL归档

```ini
# postgresql.conf

# 启用归档
archive_mode = on

# 归档命令（复制到归档目录）
archive_command = 'cp %p /mnt/archive/%f'

# 归档超时（强制切换WAL段）
archive_timeout = 300  # 5分钟

# WAL级别必须为replica或logical
wal_level = replica
```

**archive_command变量**:
- `%p`: 要归档的WAL文件路径
- `%f`: WAL文件名

**常见归档命令**:
```bash
# 本地复制
archive_command = 'cp %p /mnt/archive/%f'

# 远程复制（rsync）
archive_command = 'rsync -a %p user@backup:/archive/%f'

# AWS S3
archive_command = 'aws s3 cp %p s3://my-bucket/wal/%f'

# 使用pg_receivewal（推荐）
# 独立进程持续接收WAL流
pg_receivewal -D /mnt/archive -h primary_host
```

### 6.3 PITR恢复流程

```mermaid
sequenceDiagram
    participant Admin as **管理员**
    participant BaseBackup as **基础备份**
    participant Archive as **WAL归档**
    participant PG as **PostgreSQL**
    
    Admin->>BaseBackup: **1. 恢复基础备份**<br/>pg_basebackup的tar包
    BaseBackup->>PG: **解压到数据目录**
    
    Admin->>PG: **2. 创建recovery信号文件**<br/>touch recovery.signal
    
    Admin->>PG: **3. 配置postgresql.conf**<br/>restore_command<br/>recovery_target_time
    
    Note over Admin,PG: **recovery.conf内容示例**<br/>restore_command='cp /archive/%f %p'<br/>recovery_target_time='2024-01-01 12:00:00'
    
    Admin->>PG: **4. 启动PostgreSQL**
    
    PG->>PG: **检测recovery.signal**<br/>进入恢复模式
    
    loop **应用WAL**
        PG->>Archive: **通过restore_command获取WAL**
        Archive-->>PG: **返回WAL文件**
        PG->>PG: **重放WAL记录**
        
        alt **达到目标时间**
            PG->>PG: **停止恢复**
        end
    end
    
    PG->>PG: **删除recovery.signal**
    PG->>PG: **创建时间线历史文件**<br/>新时间线+1
    
    PG-->>Admin: **恢复完成，数据库可用**
```

### 6.4 时间线（Timeline）

```mermaid
graph LR
    subgraph "**时间线概念**"
        style TL1 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
        style TL2 fill:#fff9c4,stroke:#f57f17,stroke-width:2px
        style TL3 fill:#ffccbc,stroke:#bf360c,stroke-width:2px
        
        TL1[**时间线1**<br/>━━━━━━━━━━━━━━━━<br/>初始运行<br/>LSN: 0/1000000-0/5000000]
        
        TL2[**时间线2**<br/>━━━━━━━━━━━━━━━━<br/>PITR恢复到0/3000000<br/>新历史分支<br/>LSN: 0/3000000-0/8000000]
        
        TL3[**时间线3**<br/>━━━━━━━━━━━━━━━━<br/>再次PITR<br/>新分支<br/>LSN: 0/4000000-...]
    end
    
    TL1 -->|**PITR恢复**| TL2
    TL2 -->|**再次PITR**| TL3
```

**时间线作用**:
- 避免混淆不同历史分支的WAL
- PITR后生成新时间线
- 时间线历史文件（`.history`）记录分支关系

---

## 总结

PostgreSQL的WAL和崩溃恢复机制是其持久性和可靠性的保证：

1. **WAL预写日志**: 所有修改先写日志，保证崩溃后可恢复
2. **检查点机制**: 定期刷写脏页，缩短恢复时间，回收WAL空间
3. **崩溃恢复**: 从检查点开始重放WAL，恢复到一致状态
4. **WAL归档**: 支持PITR时间点恢复和长期备份

**关键特性**:
- **Full Page Write**: 防止部分写问题
- **LSN机制**: 确保WAL顺序和页面一致性
- **资源管理器**: 模块化处理不同类型的WAL记录
- **时间线**: 支持多次PITR恢复

**性能优化**:
- `checkpoint_completion_target`: 平滑IO，避免峰值
- `max_wal_size`: 控制WAL大小和检查点频率
- `wal_buffers`: WAL缓冲区大小
- `wal_compression`: 压缩FPW节省空间

**监控指标**:
- `pg_current_wal_lsn()`: 当前WAL位置
- `pg_wal_lsn_diff()`: 计算LSN差值
- `pg_stat_bgwriter`: 检查点统计信息
- `pg_stat_archiver`: 归档进度

---

**相关文档**:
- [01-架构总览](./01-架构总览.md)
- [02-存储引擎和MVCC](./02-存储引擎和MVCC.md)
- [03-事务系统](./03-事务系统.md)
- [07-备份和复制原理](./07-备份和复制原理.md)

