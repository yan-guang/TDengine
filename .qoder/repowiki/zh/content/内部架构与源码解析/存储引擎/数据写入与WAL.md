# 数据写入与WAL

<cite>
**本文档引用的文件**   
- [vnodeSvr.c](file://source/dnode/vnode/src/vnd/vnodeSvr.c)
- [walWrite.c](file://source/libs/wal/src/walWrite.c)
- [walMgmt.c](file://source/libs/wal/src/walMgmt.c)
- [walMeta.c](file://source/libs/wal/src/walMeta.c)
- [walRead.c](file://source/libs/wal/src/walRead.c)
- [tsdbWrite.c](file://source/dnode/vnode/src/tsdb/tsdbWrite.c)
- [tsdbMemTable.c](file://source/dnode/vnode/src/tsdb/tsdbMemTable.c)
- [wal.h](file://include/libs/wal/wal.h)
</cite>

## 目录
1. [引言](#引言)
2. [数据写入流程概述](#数据写入流程概述)
3. [WAL机制详解](#wal机制详解)
4. [MemTable内存管理](#memtable内存管理)
5. [关键函数分析](#关键函数分析)
6. [故障恢复机制](#故障恢复机制)
7. [总结](#总结)

## 引言
TDengine是一款高性能、分布式、支持SQL的时序数据库，其存储引擎设计精巧，尤其在数据写入路径上采用了Write-Ahead Log（WAL）和内存表（MemTable）相结合的策略，以确保数据的持久性和高吞吐量。本文档将深入剖析TDengine存储引擎的数据写入流程，从客户端请求到达vnode开始，详细描述数据如何首先写入WAL以确保持久性，然后被加载到内存中的MemTable。同时，本文还将解释WAL的文件结构、同步策略和故障恢复机制，以及MemTable的内存管理、大小控制和触发落盘的条件。

## 数据写入流程概述
当客户端向TDengine发送数据写入请求时，该请求首先被路由到相应的vnode（虚拟节点）。vnode接收到请求后，会进行一系列预处理操作，包括验证请求的有效性、检查表是否存在等。一旦预处理完成，vnode将调用WAL模块将数据写入WAL文件，以确保即使在系统崩溃的情况下，数据也不会丢失。随后，数据被加载到内存中的MemTable，供后续查询使用。当MemTable达到一定大小或满足其他条件时，它会被冻结并触发落盘过程，将数据持久化到磁盘上的SSTable文件中。

## WAL机制详解
### WAL文件结构
WAL（Write-Ahead Log）是TDengine中用于确保数据持久性的关键组件。WAL文件由一系列日志记录组成，每个记录包含一个头部和一个主体。头部包含了版本号、消息类型、校验和等信息，而主体则包含了实际的数据内容。WAL文件的结构设计使得它可以高效地进行追加写入，并且支持快速的读取和恢复操作。

### 同步策略
WAL的同步策略决定了何时将数据从内存刷新到磁盘。TDengine提供了多种同步级别，包括`TAOS_WAL_SKIP`、`TAOS_WAL_WRITE`和`TAOS_WAL_FSYNC`。`TAOS_WAL_SKIP`表示不进行任何同步操作，适用于对数据持久性要求不高的场景；`TAOS_WAL_WRITE`表示每次写入后都会调用`write`系统调用，但不保证数据立即写入磁盘；`TAOS_WAL_FSYNC`则表示每次写入后都会调用`fsync`系统调用，确保数据被强制写入磁盘，提供最高的数据安全性。

### 故障恢复机制
在系统重启或发生故障后，TDengine通过WAL文件来恢复未持久化的数据。恢复过程首先读取WAL元数据文件，确定最新的快照版本和最后一个已提交的版本。然后，系统会从WAL文件中读取从快照版本之后的所有日志记录，并将其重放至MemTable中，从而恢复到故障前的状态。此外，WAL还支持日志截断和文件滚动功能，以防止日志文件无限增长。

**Section sources**
- [walWrite.c](file://source/libs/wal/src/walWrite.c#L0-L739)
- [walMgmt.c](file://source/libs/wal/src/walMgmt.c#L0-L417)
- [walMeta.c](file://source/libs/wal/src/walMeta.c#L0-L799)
- [walRead.c](file://source/libs/wal/src/walRead.c#L0-L537)
- [wal.h](file://include/libs/wal/wal.h#L0-L215)

## MemTable内存管理
### 内存管理
MemTable是TDengine中用于缓存最近写入数据的内存数据结构。它采用跳表（Skip List）作为底层数据结构，以支持高效的插入和查询操作。每个MemTable实例都与一个特定的vnode关联，并且可以包含多个子表（STbData），每个子表对应一个具体的表。MemTable通过引用计数机制来管理其生命周期，确保在没有活跃引用时能够被安全地销毁。

### 大小控制
为了防止MemTable占用过多内存，TDengine对MemTable的大小进行了严格控制。当MemTable的大小超过预设阈值时，它会被冻结并触发落盘过程。此外，系统还会定期检查MemTable的大小，并根据需要进行调整。例如，当系统内存紧张时，可能会提前触发落盘过程，以释放内存资源。

### 触发落盘的条件
MemTable的落盘过程通常由以下几种条件触发：
1. **大小阈值**：当MemTable的大小超过预设的最大值时。
2. **时间间隔**：当距离上次落盘的时间超过预设的时间间隔时。
3. **系统负载**：当系统检测到内存压力较大时，可能会提前触发落盘过程。
4. **手动触发**：管理员可以通过命令行工具手动触发落盘过程。

**Section sources**
- [tsdbMemTable.c](file://source/dnode/vnode/src/tsdb/tsdbMemTable.c#L0-L799)
- [tsdbWrite.c](file://source/dnode/vnode/src/tsdb/tsdbWrite.c#L0-L104)

## 关键函数分析
### vnodeProcessSubmitReq
`vnodeProcessSubmitReq`函数是处理数据写入请求的核心函数。它首先对请求进行预处理，包括验证请求的有效性和检查表是否存在。然后，函数调用WAL模块将数据写入WAL文件，并将数据加载到MemTable中。最后，函数返回响应结果给客户端。

### walAppendLog
`walAppendLog`函数负责将日志记录追加到WAL文件中。它首先检查当前WAL文件是否需要滚动，如果需要，则创建新的WAL文件。然后，函数将日志记录的头部和主体分别写入WAL文件和索引文件中，并更新相关的元数据信息。

### tsdbInsertTableData
`tsdbInsertTableData`函数负责将数据插入到MemTable中。它首先获取或创建对应的子表（STbData），然后根据数据格式（行格式或列格式）将数据插入到跳表中。插入完成后，函数会更新MemTable的统计信息，如最小/最大版本号、最小/最大键值等。

**Section sources**
- [vnodeSvr.c](file://source/dnode/vnode/src/vnd/vnodeSvr.c#L0-L3204)
- [walWrite.c](file://source/libs/wal/src/walWrite.c#L0-L739)
- [tsdbWrite.c](file://source/dnode/vnode/src/tsdb/tsdbWrite.c#L0-L104)
- [tsdbMemTable.c](file://source/dnode/vnode/src/tsdb/tsdbMemTable.c#L0-L799)

## 故障恢复机制
### 日志重放
在系统重启或发生故障后，TDengine通过日志重放机制来恢复未持久化的数据。恢复过程首先读取WAL元数据文件，确定最新的快照版本和最后一个已提交的版本。然后，系统会从WAL文件中读取从快照版本之后的所有日志记录，并将其重放至MemTable中，从而恢复到故障前的状态。

### 快照管理
TDengine支持定期生成快照，以减少日志重放的时间。快照包含了某个时间点上所有数据的完整副本，可以用于快速恢复。当系统启动时，它会首先加载最新的快照，然后从快照版本之后的日志记录开始重放，直到恢复到最新的状态。

### 日志截断
为了防止WAL文件无限增长，TDengine支持日志截断功能。当日志文件中的数据已经被持久化到SSTable文件中时，这些日志记录可以被安全地删除。系统会定期检查哪些日志记录可以被截断，并执行相应的清理操作。

**Section sources**
- [walWrite.c](file://source/libs/wal/src/walWrite.c#L0-L739)
- [walMgmt.c](file://source/libs/wal/src/walMgmt.c#L0-L417)
- [walMeta.c](file://source/libs/wal/src/walMeta.c#L0-L799)
- [walRead.c](file://source/libs/wal/src/walRead.c#L0-L537)

## 总结
TDengine的存储引擎通过WAL和MemTable的结合，实现了高效的数据写入和持久化。WAL确保了数据的持久性，即使在系统崩溃的情况下也能恢复数据；MemTable则提供了高性能的内存缓存，支持快速的插入和查询操作。通过合理的内存管理和落盘策略，TDengine能够在保证数据安全的同时，提供出色的性能表现。本文档详细介绍了TDengine存储引擎的数据写入流程，包括WAL的文件结构、同步策略、故障恢复机制，以及MemTable的内存管理、大小控制和触发落盘的条件，为开发者和运维人员提供了宝贵的参考。

**Section sources**
- [vnodeSvr.c](file://source/dnode/vnode/src/vnd/vnodeSvr.c#L0-L3204)
- [walWrite.c](file://source/libs/wal/src/walWrite.c#L0-L739)
- [walMgmt.c](file://source/libs/wal/src/walMgmt.c#L0-L417)
- [walMeta.c](file://source/libs/wal/src/walMeta.c#L0-L799)
- [walRead.c](file://source/libs/wal/src/walRead.c#L0-L537)
- [tsdbWrite.c](file://source/dnode/vnode/src/tsdb/tsdbWrite.c#L0-L104)
- [tsdbMemTable.c](file://source/dnode/vnode/src/tsdb/tsdbMemTable.c#L0-L799)
- [wal.h](file://include/libs/wal/wal.h#L0-L215)