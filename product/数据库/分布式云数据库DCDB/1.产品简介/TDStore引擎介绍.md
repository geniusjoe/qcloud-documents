
TDStore 是腾讯云全自研的金融级新敏态引擎，该引擎可以有效解决对于客户业务发展过程中业务形态、业务量的不可预知性，适配金融敏态业务。

## TDStore 引擎特性
### 高度兼容 MySQL 语法
TDSQL TDStore 引擎版计算节点基于 MySQL 8.0 实现，除个别受限的系统操作，TDStore 可以100%兼容原生 MySQL 语法。单机 MySQL 的业务可以无损迁移到 TDStore 上，真正实现对业务应用无入侵。

###  存储计算分离/独立弹性伸缩
TDStore 采用计算和存储分离的原生分布式的架构设计，计算层和存储层的节点均可根据业务需求独立弹性扩缩容，而且无须额外的人工运维干预，实现扩缩容过程对业务零感知。

- **计算层**采用多主架构，而且每个计算节点均可读写，用户可以随着业务量的增长而弹性扩展计算节点，单实例可支撑千万级 QPS 流量，帮助用户轻松应对突如其来的业务峰值压力。
- 对于**存储层**资源，用户也可以随着业务数据量的增长而弹性扩展存储节点。数据在不同节点之间的迁移、均衡、路由变更等操作均由 TDStore 实例内部自洽完成。

### 云原生的管控系统
TDStore 的管控部分采用了云原生的方式，借助云原生的能力，能够快速且方便地管理 TDStore 实例，免除了繁琐的物理机上架，配置等资源管理运维操作，同时也无需关心资源的使用率情况，即买即用，支持高效弹性扩缩容。

### 原生 Online DDL 支持
TDStore 支持原生 Online DDL 操作，用户在业务运行过程中，有动态更改表结构的需求时，无须依赖如 pt 或 ghost 等外部工具组件，直接使用原生 MySQL DDL 语句便可完成。

TDStore 覆盖 MySQL 原生可支持的 instant 类型 DDL 操作，并且对于大部分类型（除涉及主键外的）DDL，均能以不阻塞业务的正常 DML 请求下完成。同时，TDStore 的 Online DDL 可以在多个计算节点之间保持一致性，不同表对象的 Online DDL 可以并行执行。

### 完整分布式事务支持
TDStore 以原生分布式的架构完整支持事务 [ACID](https://cloud.tencent.com/document/product/1121/36888#A) 特性，默认的事务隔离级别为快照隔离级别（Snapshot Isolation），支持全局一致性读特性，整体事务并发控制框架基于 MVCC + Time-Ordering 的方式实现。

分布式事务协调者由分布式存储层节点担任，而当存储节点在线扩容遇到数据分裂或切主等状态变更的场景时，TDStore 均可实现不中断事务，将底层数据状态的变更对事务请求的影响降到最低，从而做到无感知的集群扩缩容。

### 低成本海量存储
TDStore 存储层基于 LSM-Tree + SSTable 结构存放和管理数据，具有较高的压缩率，能有效降低海量数据规模下的存储成本。对于一些数据行重复度较大的业务场景，对比 InnoDB 存储引擎，TDStore 版最高可实现高达20倍的压缩率，单实例可支撑 EB 级别的存储量。

## TDStore 引擎架构

### 计算节点 SQLEngine
SQLEngine 是计算节点，负责接收和响应客户端的 SQL 请求。SQLEngine 基于 MySQL 8.0 实现，完全兼容原生 MySQL 语法，从原生 MySQL 迁移过来的业务在使用时无须对业务语句进行任何改造。

SQLEngine 采用无状态化的设计方式，节点本身不保存任何用户数据，并将多线程框架替换为协程框架，与集群内的 TDStore 节点进行交互。

一个实例内可以包含多个 SQLEngine 节点，节点之间彼此独立，均可读写。

### 存储节点 TDStore
TDStore 是存储节点，负责用户数据的存储。它是一个基于 Multi-Raft 协议实现的分布式存储集群。

Region 是 TDStore 存储和管理数据的最小单位，以及 TDStore 节点之间进行数据复制同步的单位，一个 Region 代表一段左闭右开的数据区间，每个 Region 包含一主 N 备的多个副本，不同副本分散在不同的 TDStore 节点上。客户端对某一行数据的访问，在经过 SQLEngine 编码后，会将请求发送到对应的 TDStore 上对应的 Region 上。

在分布式事务中，TDStore 承担协调者的角色，由 Region 的 Leader 副本进行响应。

### 管控节点 TDMC
TDMC 为管控节点集群，负责实例内部的资源管控调度，以及全局唯一性的数据资源的管理。
- 在资源调度方面，TDMC 主要负责根据 TDStore 心跳上报的信息，对 Region 下发进行分裂、切主、合并、迁移等任务，同时向 SQLEngine 提供最新的全局的 Region 路由信息。
- 在数据管理方面，TDMC 主要负责向计算和存储节点提供全局唯一且严格递增的事务时间戳，用以实现数据多版本管理以及可见性的判断。另外，MC 还负责管理 MDL 锁、系统变量参数等等。
