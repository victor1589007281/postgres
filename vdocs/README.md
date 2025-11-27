# PostgreSQL 深度解析文档

## 📚 文档总览

本系列文档基于 PostgreSQL 源码深入分析，全面解析 PostgreSQL 的架构、原理、实现和应用场景。文档包含大量架构图、流程图和时序图，帮助读者从源码层面理解 PostgreSQL。

---

## 📖 文档目录

### 1. [架构总览](./01-架构总览.md)
**关键内容**：
- PostgreSQL 整体系统架构
- 核心模块划分与职责
- 多进程架构详解
- 内存架构与共享内存
- 存储架构与数据目录结构

**核心图表**：
- 系统整体架构图
- 模块分层架构图
- 进程架构图
- 内存布局图
- 数据目录结构图

**适合人群**：初学者、架构师、需要全局了解的开发者

---

### 2. [存储引擎和MVCC](./02-存储引擎和MVCC.md)
**关键内容**：
- Heap 堆表存储引擎
- 页面和元组结构
- MVCC 多版本并发控制原理
- 元组可见性判断算法
- VACUUM 机制与 HOT 优化
- 多种索引访问方法（B-Tree、GIN、GiST、BRIN等）

**核心图表**：
- 存储引擎架构图
- 页面详细结构图
- 元组结构图
- MVCC 版本链图
- 可见性判断流程图
- VACUUM 工作流程图

**适合人群**：数据库内核开发者、性能优化工程师

---

### 3. [事务系统](./03-事务系统.md)
**关键内容**：
- ACID 特性实现机制
- 事务 ID (XID) 管理与回卷
- 四种事务隔离级别详解
- 锁机制（表锁、行锁、锁兼容性）
- 死锁检测算法
- 两阶段提交 (2PC)

**核心图表**：
- ACID 实现架构图
- 事务处理流程图
- XID 循环比较图
- 隔离级别对比图
- 锁兼容性矩阵
- 死锁检测流程图

**适合人群**：应用开发者、DBA、并发控制研究者

---

### 4. [WAL和崩溃恢复](./04-WAL和崩溃恢复.md)
**关键内容**：
- WAL 预写日志原理
- WAL 文件结构与 LSN
- WAL 写入流程与 Full Page Write
- 检查点机制
- 崩溃恢复流程
- WAL 归档与 PITR

**核心图表**：
- WAL 原理示意图
- WAL 文件组织图
- WAL 记录结构图
- 检查点流程图
- 崩溃恢复流程图
- PITR 恢复时序图

**适合人群**：DBA、灾难恢复工程师、可靠性工程师

---

### 5. [备份和复制原理](./07-备份和复制原理.md)
**关键内容**：
- 逻辑备份 (pg_dump/pg_dumpall)
- 物理备份 (pg_basebackup)
- 流复制原理与配置
- 同步复制 vs 异步复制
- 逻辑复制 (发布/订阅)
- 复制槽机制

**核心图表**：
- 备份类型对比图
- pg_dump 工作流程图
- pg_basebackup 流程图
- 流复制架构图
- 逻辑复制架构图
- 复制槽原理图

**适合人群**：DBA、备份恢复工程师、高可用架构师

---

### 6. [高可用方案](./08-高可用方案.md)
**关键内容**：
- 高可用架构设计
- 主备架构配置
- Patroni 自动故障切换
- repmgr 和 pg_auto_failover
- 读写分离方案（Pgpool-II、HAProxy）
- 连接池方案（Pgbouncer）

**核心图表**：
- 典型主备架构图
- Patroni 架构图
- 故障切换流程图
- Pgpool-II 读写分离图
- HAProxy 负载均衡图
- Pgbouncer 连接池图

**适合人群**：DBA、架构师、运维工程师

---

### 7. [网络模型和进程架构](./09-网络模型和进程架构.md)
**关键内容**：
- 多进程模型 vs 多线程模型
- 连接建立完整流程
- 前后端协议详解
- 身份验证方法
- 进程间通信（共享内存、信号量、Latch）
- 并发处理模型

**核心图表**：
- 整体网络架构图
- 连接建立时序图
- 协议消息类型图
- 查询执行流程图
- 共享内存布局图
- 连接池对比图

**适合人群**：网络编程开发者、性能调优工程师

---

### 8. [使用场景和案例](./10-使用场景和案例.md)
**关键内容**：
- 典型应用场景（OLTP、地理空间、JSONB、全文搜索）
- 行业应用案例（金融、电商、LBS、CMS）
- 高级数据类型应用（数组、范围、枚举、JSONB）
- 性能优化案例
- 数据库迁移案例（MySQL、Oracle）

**核心图表**：
- 场景分类图
- 金融系统架构图
- 电商平台架构图
- LBS 系统架构图
- 迁移流程图

**适合人群**：应用开发者、架构师、DBA

---

## 🎯 学习路径建议

### 入门路径
```mermaid
graph LR
    A[**01-架构总览**] --> B[**09-网络模型和进程架构**]
    B --> C[**10-使用场景和案例**]
    
    style A fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style B fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style C fill:#ffccbc,stroke:#bf360c,stroke-width:2px
```

**说明**：先理解整体架构，再学习连接和通信，最后了解实际应用。

### 内核开发路径
```mermaid
graph LR
    A[**01-架构总览**] --> B[**02-存储引擎和MVCC**]
    B --> C[**03-事务系统**]
    C --> D[**04-WAL和崩溃恢复**]
    
    style A fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style B fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style C fill:#ffccbc,stroke:#bf360c,stroke-width:2px
    style D fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
```

**说明**：深入理解存储、事务和恢复机制，适合内核开发者。

### DBA运维路径
```mermaid
graph LR
    A[**01-架构总览**] --> B[**07-备份和复制原理**]
    B --> C[**08-高可用方案**]
    C --> D[**04-WAL和崩溃恢复**]
    
    style A fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style B fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style C fill:#ffccbc,stroke:#bf360c,stroke-width:2px
    style D fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
```

**说明**：专注于备份、高可用和恢复，适合运维人员。

---

## 📊 文档特色

### ✅ 基于源码分析
所有内容基于 PostgreSQL 最新源码，引用实际代码位置和关键函数，确保准确性。

### ✅ 图表丰富
每个文档包含大量高质量图表：
- **架构图**：展示系统结构
- **流程图**：展示处理流程
- **时序图**：展示交互过程
- **对比图**：对比不同方案

### ✅ 实践导向
不仅讲解原理，还包含：
- 配置示例
- SQL 示例
- 最佳实践
- 性能调优

### ✅ 深度与广度兼备
既有底层实现细节，又有高层架构设计，满足不同层次读者需求。

---

## 🔗 关键概念索引

### 架构相关
- **多进程模型**：[01-架构总览](./01-架构总览.md#3-进程架构)、[09-网络模型](./09-网络模型和进程架构.md#1-网络模型概述)
- **共享内存**：[01-架构总览](./01-架构总览.md#4-内存架构)、[09-网络模型](./09-网络模型和进程架构.md#41-共享内存)
- **Postmaster**：[01-架构总览](./01-架构总览.md#321-postmaster主进程)
- **Backend进程**：[01-架构总览](./01-架构总览.md#322-backend后端进程)

### 存储相关
- **Heap表**：[02-存储引擎和MVCC](./02-存储引擎和MVCC.md#2-heap存储引擎)
- **页面结构**：[01-架构总览](./01-架构总览.md#53-页面结构)、[02-存储引擎](./02-存储引擎和MVCC.md#21-heap表的存储结构)
- **元组结构**：[02-存储引擎和MVCC](./02-存储引擎和MVCC.md#22-元组结构heaptuple)
- **索引类型**：[02-存储引擎和MVCC](./02-存储引擎和MVCC.md#6-索引访问方法)

### MVCC相关
- **多版本并发控制**：[02-存储引擎和MVCC](./02-存储引擎和MVCC.md#3-mvcc多版本并发控制)
- **事务快照**：[02-存储引擎和MVCC](./02-存储引擎和MVCC.md#32-事务快照snapshot)
- **可见性判断**：[02-存储引擎和MVCC](./02-存储引擎和MVCC.md#4-元组可见性判断)
- **VACUUM**：[02-存储引擎和MVCC](./02-存储引擎和MVCC.md#5-vacuum机制)

### 事务相关
- **ACID**：[03-事务系统](./03-事务系统.md#11-acid实现机制)
- **事务ID**：[03-事务系统](./03-事务系统.md#2-事务id管理)
- **隔离级别**：[03-事务系统](./03-事务系统.md#3-事务隔离级别)
- **锁机制**：[03-事务系统](./03-事务系统.md#4-锁机制)
- **死锁检测**：[03-事务系统](./03-事务系统.md#5-死锁检测)

### WAL相关
- **预写日志**：[04-WAL和崩溃恢复](./04-WAL和崩溃恢复.md#1-wal概述)
- **WAL结构**：[04-WAL和崩溃恢复](./04-WAL和崩溃恢复.md#2-wal结构)
- **检查点**：[04-WAL和崩溃恢复](./04-WAL和崩溃恢复.md#4-检查点机制)
- **崩溃恢复**：[04-WAL和崩溃恢复](./04-WAL和崩溃恢复.md#5-崩溃恢复)
- **WAL归档**：[04-WAL和崩溃恢复](./04-WAL和崩溃恢复.md#6-wal归档)

### 复制相关
- **流复制**：[07-备份和复制原理](./07-备份和复制原理.md#4-流复制原理)
- **逻辑复制**：[07-备份和复制原理](./07-备份和复制原理.md#5-逻辑复制)
- **复制槽**：[07-备份和复制原理](./07-备份和复制原理.md#6-复制槽)
- **同步复制**：[07-备份和复制原理](./07-备份和复制原理.md#43-同步复制)

### 高可用相关
- **主备架构**：[08-高可用方案](./08-高可用方案.md#2-主备架构)
- **Patroni**：[08-高可用方案](./08-高可用方案.md#31-patroni架构)
- **读写分离**：[08-高可用方案](./08-高可用方案.md#4-读写分离)
- **连接池**：[08-高可用方案](./08-高可用方案.md#5-连接池方案)

---

## 🛠️ 配置参数速查

### 内存相关
| **参数** | **推荐值** | **说明** | **文档** |
|---------|-----------|---------|---------|
| `shared_buffers` | 系统内存的25% | 共享缓冲区 | [01-架构总览](./01-架构总览.md#42-共享内存组件) |
| `work_mem` | 4MB-64MB | 排序/Hash工作内存 | [01-架构总览](./01-架构总览.md#43-本地内存backend进程) |
| `maintenance_work_mem` | 64MB-1GB | VACUUM等维护操作内存 | [02-存储引擎](./02-存储引擎和MVCC.md#51-vacuum工作原理) |
| `effective_cache_size` | 系统内存的50-75% | 查询优化器估算 | - |

### WAL相关
| **参数** | **推荐值** | **说明** | **文档** |
|---------|-----------|---------|---------|
| `wal_level` | `replica` | WAL级别 | [04-WAL](./04-WAL和崩溃恢复.md#32-wal-buffers管理) |
| `max_wal_size` | 1GB-10GB | WAL最大大小 | [04-WAL](./04-WAL和崩溃恢复.md#42-检查点触发条件) |
| `wal_buffers` | 16MB | WAL缓冲区 | [04-WAL](./04-WAL和崩溃恢复.md#32-wal-buffers管理) |
| `checkpoint_timeout` | 5min-15min | 检查点超时 | [04-WAL](./04-WAL和崩溃恢复.md#42-检查点触发条件) |

### 连接相关
| **参数** | **推荐值** | **说明** | **文档** |
|---------|-----------|---------|---------|
| `max_connections` | 100-200 | 最大连接数 | [09-网络模型](./09-网络模型和进程架构.md#51-连接数限制) |
| `superuser_reserved_connections` | 3 | 超级用户预留 | [09-网络模型](./09-网络模型和进程架构.md#51-连接数限制) |

### 复制相关
| **参数** | **推荐值** | **说明** | **文档** |
|---------|-----------|---------|---------|
| `max_wal_senders` | 10 | WAL发送进程数 | [07-备份和复制](./07-备份和复制原理.md#42-流复制配置) |
| `synchronous_commit` | `on` / `remote_apply` | 同步提交级别 | [07-备份和复制](./07-备份和复制原理.md#43-同步复制) |
| `hot_standby` | `on` | 备库只读 | [07-备份和复制](./07-备份和复制原理.md#42-流复制配置) |

---

## 📝 常用SQL命令速查

### 监控相关
```sql
-- 当前连接和查询
SELECT * FROM pg_stat_activity;

-- 数据库统计
SELECT * FROM pg_stat_database;

-- 表统计
SELECT * FROM pg_stat_user_tables;

-- 索引统计
SELECT * FROM pg_stat_user_indexes;

-- 复制状态
SELECT * FROM pg_stat_replication;

-- WAL位置
SELECT pg_current_wal_lsn();

-- 数据库大小
SELECT pg_size_pretty(pg_database_size('mydb'));

-- 表大小
SELECT pg_size_pretty(pg_total_relation_size('mytable'));
```

### 维护相关
```sql
-- VACUUM
VACUUM VERBOSE ANALYZE mytable;

-- 重建索引
REINDEX TABLE mytable;

-- 更新统计信息
ANALYZE mytable;

-- 检查点
CHECKPOINT;

-- 查看锁
SELECT * FROM pg_locks;

-- 查看等待
SELECT * FROM pg_stat_activity WHERE wait_event IS NOT NULL;
```

---

## 🎓 推荐资源

### 官方文档
- [PostgreSQL 官方文档](https://www.postgresql.org/docs/)
- [PostgreSQL Wiki](https://wiki.postgresql.org/)

### 源码相关
- [PostgreSQL GitHub](https://github.com/postgres/postgres)
- [源码浏览](https://doxygen.postgresql.org/)

### 社区资源
- [Planet PostgreSQL](https://planet.postgresql.org/)
- [PostgreSQL 中文社区](http://www.postgres.cn/)

### 书籍推荐
- 《PostgreSQL 数据库内核分析》
- 《PostgreSQL 实战》
- 《PostgreSQL 即学即用》

---

## 🤝 贡献指南

欢迎贡献文档改进、错误修正或新内容！

### 文档规范
- 使用 Markdown 格式
- Mermaid 图表字体加粗
- 图表背景颜色浅色
- 代码示例包含注释

### 提交流程
1. Fork 仓库
2. 创建分支
3. 提交修改
4. 发起 Pull Request

---

## 📮 反馈与联系

如有问题、建议或发现错误，欢迎提交 Issue 或 Pull Request。

---

**文档版本**: v1.0  
**最后更新**: 2024-11  
**基于版本**: PostgreSQL 15/16  

---

## 📜 许可证

本文档遵循 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 许可证。

PostgreSQL 本身遵循 [PostgreSQL License](https://www.postgresql.org/about/licence/)（类似 BSD/MIT）。

